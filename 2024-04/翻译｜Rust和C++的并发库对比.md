> 本文原作者：[Mara Bos](https://blog.m-ou.se/format-args/)，原文链接：[https://blog.m-ou.se/super-let/](https://blog.m-ou.se/rust-cpp-concurrency)
>
> 原文发布于2022年8月16日。

Rust标准库里包含的并发特性和C++11真的很相似：线程、原子量、互斥锁、条件变量等。不过，在过去的几年里，作为C++17和C++20的一部分，C++得到了大量和并发相关的新特性，并且在未来的版本中还有更多相关的提案。

让我们花点时间来评阅一下C++的并发特性，并讨论一下它们的Rust等效替代可以像什么样，以及实现这些特性需要做什么。

## atomic_ref

[P0019R8](https://wg21.link/p0019r8)为C++引入了[`std::atomic_ref`](https://zh.cppreference.com/w/cpp/atomic/atomic_ref)。这是一个允许你将一个不是原子量的对象以原子量的方式来使用的类型。例如，你可以创建一个引用了常规`int`的`atomic_ref<int>`，它向你提供了和`atomic<int>`一致的功能。

尽管这在C++里需要一个复制了绝大部分`atomic`接口的全新类型，与之对等的Rust特性却是一个单行函数：`Atomic*::from_mut`。以`u32`为例，这个函数允许你将一个`&mut u32`转换为`&AtomicU32`，这是Rust中一种非常合理的别名形式。

C++的`atomic_ref`类型附带了需要手动确保的安全要求。当你使用`atomic_ref`来访问一个对象时，所有对该对象的访问都必须通过`atomic_ref`完成。在`atomic_ref`还存在时直接访问该对象会导致未定义行为。

不过，在Rust里，这个问题已经被借用检查器完全解决了。编译器知道在一个`u32`被可变借用的期间，它不能被直接以任何形式访问。被传入`from_mut`的`&mut u32`的生命周期被作为从这个函数中得到的`&AtomicU32`的一部分被保留。你可以拷贝任意数量的`&AtomicU32`，但仅仅在所有引用的副本都消失后，原始借用才会结束。

[`from_mut`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicBool.html#method.from_mut)函数目前还是不稳定的，但也许是时候把它稳定下来了。

## 泛型的原子类型

在C++里，`std::atomic`是泛型的：你既可以使用`atomic<int>`，又可以使用`atomic<MyOwnStruct>`。在Rust里，另一方面，我们只有特定的原子类型：`AtomicU32`、`AtomicBool`、`AtomicUsize`，等等。

C++的原子类型支持任意size的对象，也不考虑平台支持什么。对于那些size不受平台本地的原子操作支持的对象，C++会自动回退到一个基于锁的实现。在另一方面，Rust只提供被平台本地支持的类型。如果你正在为一个没有64位原子量的平台编译代码的话，那`AtomicU64`就不会存在。

这既有好处又有坏处。它既意味着使用`AtomicU64`的Rust代码可能无法在特定平台上编译，又意味着不会有因为部分类型悄悄回落到一个非常不同的实现而导致的突如其来的性能相关的变化。它还意味着我们可以假定`AtomicU64`在内存中的排布和`u64`是确切一致的，这允许了类似`AtomicU64::from_mut`的函数存在。

在Rust中维护一个为任意size的类型工作的`Atomic<T>`将会很难办。如果不进行特殊化，我们就不能让`Atomic<LargeThing>`包含一个`Mutex`，而`Atomic<SmallThing>`不包含。不过，我们可以做的是将互斥锁都存放在一个全局的、由内存地址作为索引的`HashMap`里，这样`Atomic<T>`的size就可以和`T`一致，并在需要时使用来自这个全局哈希表的`Mutex`。

这就是流行的[`atomic`包](https://docs.rs/atomic/)实际上的做法。

对于将这个通用的`Atomic<T>`类型添加到Rust标准库的提案来说，它需要讨论这个类型是否应当支持`no_std`程序。`HashMap`通常需要进行堆内存分配，但这对`no_std`程序而言并不可能。（译注：当然，只要提供`#[global_allocator]`就可以了。）虽然设置一个固定长度的表格就能为`no_std`的程序解决问题，但这种做法可能由于种种原因而无法被接受。

## 带填充的compare-exchange

[P0528R3](https://wg21.link/p0528r3)改变了`compare_exchange`处理填充的方式。对`atomic<TypeWithPadding>`的比较-交换操作也会比较填充位的内容，但这被证明是个坏主意。现在，填充位不再被包括在比较中。

由于Rust当前只为没有任何填充的整数提供了原子类型，这个变化和Rust是没有关系的。

不过，具有`compare_exchange`方法的`Atomic<T>`类型的提案将会需要讨论如何处理填充，并且很可能会采纳上述C++提案的意见。

## compare-exchange的内存顺序

在C++11，[`compare_exchange`](https://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange)函数要求成功内存顺序强于或等于失败内存顺序。诸如`compare_exchange(…, …, memory_order_release, memory_order_acquire)`的用法是不被接受的。这个要求被原封不动地照搬到了Rust的[`compare_exchange`](https://doc.rust-lang.org/stable/std/sync/atomic/struct.AtomicU32.html#method.compare_exchange)函数中。

[P0418R2](https://wg21.link/p0418r2)认为应该取消这种限制，这现在已经成为了C++17的一部分。

同样的限制也作为[rust-lang/rust#98383](https://github.com/rust-lang/rust/pull/98383)的一部分，在Rust 1.64中被取消了。

## constexpr的Mutex构造函数

C++的`std::mutex`拥有一个`constexpr`的构造函数，这意味着它在编译期可以作为常量计算的一部分被构造。不过，事实上不是所有实现都提供了这个特性。举例来说，微软的`std::mutex`实现并没有包含一个`constexpr`构造函数。因此，对可移植代码来说，依赖这个特性是个坏主意。

另外，有趣的是，C++的`std::condition_variable`和`std::shared_mutex`根本没有提供`constexpr`的构造函数。

这在[Rust 1.63.0](https://blog.rust-lang.org/2022/08/11/Rust-1.63.0.html)中作为[rust-lang/rust#93740](https://github.com/rust-lang/rust/issues/93740)的一部分被解决了：`Mutex::new`、`RwLock::new`和`Condvar::new`*全都*是`const`函数。

## 锁存器和屏障

`std::latch`和`std::barrier`在[P1135R6](https://wg21.link/p1135r6)中，和一众别的工具一起，被引入了C++20。这两个类型都允许等待多个线程达到某个特定点。锁存器基本上就是一个计数器，每个线程递减一次，并允许你等待它减到零。它只能被使用一次。屏障是这个想法的更高级版本，它可以被重复使用，并接受一个“完成函数”，在计数器到达零时自动执行。

Rust自1.0起就有一个类似的[`Barrier`](https://doc.rust-lang.org/stable/std/sync/struct.Barrier.html)类型，它的灵感来源于pthread（`pthread_barrier_t`）而不是C++。

Rust（和pthread）的屏障不如C++现在的灵活。它只有一个“递减并等待”函数（叫作`wait`），并缺乏C++的[`std::barrier`](https://en.cppreference.com/w/cpp/thread/barrier)中所拥有的“仅等待”、“仅递减”、“递减并丢弃”函数。

在另一方面，不像C++，Rust（和ptrhead）的“递减并等待”操作会指派一个线程为小组领导。这是（可能更灵活的）完成函数的替代方案。

Rust版本中缺少的函数可以在任何时候被轻松地添加。我们只需要一个好的提案来给这些方法命名:)。

## 信号量

同样的[P1135R6](https://wg21.link/p1135r6)还向C++20中加入了信号量：[`std::counting_semaphore`和`std::binary_semaphore`](https://en.cppreference.com/w/cpp/thread/counting_semaphore)。

Rust并没有一个通用的信号量类型，尽管它通过[`thread::park`和`unpark`](https://doc.rust-lang.org/stable/std/thread/fn.park.html)为每个线程配备了实际上等同于二元信号量的功能。

可以使用`Mutex<u32>`和`Condvar`来轻松地手动构造信号量，但绝大多数操作系统允许使用单个`Atomic<u32>`来实现更高效也更小的信号量。例如，通过Linux的[`futex()`](https://man7.org/linux/man-pages/man2/futex.2.html)和Windows的[`WaitOnAddress()`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitonaddress)。这些操作可以使用的原子量的size取决于操作系统及其版本。

C++的`counting_semaphore`是一个模板，其模版参数是一个整数，这个整数表示我们需要它能数到多大。举个例子，`counting_semaphore<1000>`可以数到至少1000，并因此至少是16位的。`binary_semaphore`类型是`counting_semaphore<1>`的别名，在某些平台上可以是单字节。

在Rust里，我们可能不会很快准备好使用这种泛型类型。Rust的泛型强制执行一定的一致性，这对我们使用常量作为泛型参数进行了一定的限制。

我们可以设置独立的`Semaphore32`、`Semaphore64`，等等，但那看起来有点适得其反。设置`Semaphore<u32>`、`Semaphore<u64>`、甚至`Semaphore<bool>`是可能的，但这在标准库中还没有过先例。我们的原子量只是简单的`AtomicU32`、`AtomicU64`，等等。

正如之前提过的，对我们的原子量来说，我们只提供被目标平台原生支持的类型。如果我们把相同的哲学应用在`Semaphore`上，那它在没有`futex`或`WaitOnAddress`函数的平台上就无法存在了，例如macOS。并且，如果我们将信号量的类型按照不同size拆分，有的size在（部分版本的）Linux和众多的BSD上也不会存在。

如果想把信号量引入Rust标准库，我们首先需要确定是否确实需要不同size的信号量，并且为了让它们有用，需要什么形式的灵活性和可移植性。或许我们应该提供一个总是可用的32-bit`Semaphore`类型（使用一个基于互斥锁的回退），但任何类似的提案都需要包括一个详细的、对用例和局限性的解释。

## 原子等待和通知

[P1135R6](https://wg21.link/p1135r6)剩余的、为C++20添加的新特性是原子[`wait`和`notify`函数](https://en.cppreference.com/w/cpp/atomic/atomic/wait)。

这些函数有效地通过一个标准接口直接暴露了Linux的[`futex()`](https://man7.org/linux/man-pages/man2/futex.2.html)和Windows的[`WaitOnAddress()`](https://docs.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-waitonaddress)。

不过，无论操作系统支持与否，它们都对所有平台、所有size的原子量可用。尽管Linux的futex总是32位的，但C++允许`atomic<uint64_t>::wait`正常运行。

一种实现这一功能的方法是采用一个类似于[“停车场”](https://webkit.org/blog/6161/locking-in-webkit/)的工具：一个高效地将内存地址映射到锁和队列的全局`HashMap`。这意味着在Linux上，32位的等待操作可以使用非常快的基于futex的实现，而其他的size将使用一个非常不同的实现。

如果遵循只提供本地支持的类型和函数的哲学（像我们为原子类型做的），我们就不会提供这样的回退实现。那意味着我们在Linux上只有`AtomicU32::wait`（和`AtomicI32::wait`），而在Windows上所有原子类型都有这个`wait`方法。

关于Rust里`Atomic*::wait`和`Atomic*::notify`的提案，将需要讨论回退到全局表在Rust里是否可取。

