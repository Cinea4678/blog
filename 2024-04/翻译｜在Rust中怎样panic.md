---
title: 翻译｜在Rust中怎样panic
zhihu-tags: rust 
zhihu-url: https://zhuanlan.zhihu.com/p/692679205
---
> 本文原作者：Ralf Jung，原文地址：[https://www.ralfj.de/blog/2019/11/25/how-to-panic-in-rust.html](https://www.ralfj.de/blog/2019/11/25/how-to-panic-in-rust.html)
>
> 我之前也写过一篇介绍Rust的panic机制的文章，不过，最近看到这篇文章，感觉其中值得学习的地方很多，因此也一并搬运+翻译过来了。

当你`panic!()`的时候到底发生了什么？我最近花了很多时间来研究标准库中与此相关的部分，结果发现答案相当复杂！我一直没有找到能解释清楚Rust panic这幅宏大画卷的文档，因此我觉得这值得写成一篇文章。

（不要脸的小插曲：我关注这个主题的原因是@Aaron1011为Miri实现了展开的支持。我一直都很期待在Miri中看到这样的实现，但一直没有时间亲手去做。所以看到有人突然提交了PR，我真的很高兴。经过了一轮轮的review，它在最近终于落地了。尽管还有一些地方[比较粗糙](https://github.com/rust-lang/miri/issues?q=is%3Aissue+is%3Aopen+label%3AA-panics)，但基础部分还是很牢固的。）

这篇文章的目的是记录Rust panic方面的高层级结构和与此相关的接口。实际的展开机制完全是另一回事（并且我也没有资格谈论这个主题）。

*注*：这篇文章描述的panic机制是基于[这个提交](https://github.com/rust-lang/rust/commit/4007d4ef26eab44bdabc2b7574d032152264d3ad)的（译注：该提交位于2019年12月1日，相关版本号是1.41.0）。描述的许多接口都是libstd不稳定的内部细节，随时都有可能发生更改。

## 高层结构

当试图通过阅读libstd的源码来弄清“panicking”的工作原理时，很容易迷失在迷宫之中。源码中有多层只有链接器才能拼凑出来的间接关系，有`#[panic_handler]`属性和“panic运行时”（由panic策略控制）和“panic钩子”，并且在`#[no_std]`上下文中进行`panic`操作时需要完全不同的代码路径……真是千头万绪。更糟糕的是，描述panic钩子的RFC将其称为“panic handler”，但这个术语后来又被重新使用了。

我认为最好从控制两次间接跳转的接口入手：

- libstd使用panic*运行时*来控制在panic信息打印到stderr之后会发生什么。它由panic策略决定：要么终止（`-C panic=abort`），要么展开（`-C panic=unwind`）。（panic*运行时*还提供了[`catch_unwind`](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html)的实现，但我们这里并不太需要关心这个）
- libcore使用*panic handler*来实现：
    - 代码生成插入的panic，例如算术溢出或越界数组/切片索引所引起的panic
    - `core::panic!`宏（这是libcore本身和`#[no_std]`上下文中的`panic!`）

> 译注：这里提到了libstd和libcore。std是我们平时使用的Rust标准库，而core是`no_std`环境下的“核心库”。与core对应的是alloc库，其中包含`Box`、`String`等依赖堆内存分配的库。可以简单（但不准确）地理解为core就是不需要堆内存的Rust标准库，这对没有接触过`no_std`编程的读者理解后文很有帮助。

这两个接口都是通过`extern`块来实现的：libstd/libcore分别只是引入了一些它们委托的函数，然后在crate树中的某个完全不同的地方，这些函数得到了实现。这种引入只在链接时解析，如果你在本地查看代码的话，根本无法知道这些接口的实际实现在哪里。难怪我一路迷路了好几次。

> 译注：“委托”这个概念在后文中还会出现很多次。委托这个概念在C#中很常见，不过这里的委托似乎和C#中的委托类型并不相同。这里的委托以我个人的理解应该是“代码执行的责任的分配”，或者说某个函数或操作将其功能的实现责任“委托”给了另一个位置或函数来处理。

在下文中，这两个接口会经常出现；当感觉困惑时，你首先要检查的是你是否混淆了“panic handler”和“panic*运行时*”（并且记住还有一个叫做panic*钩子*的东西，我们之后会提到）。我经常遇到这种情况。

此外，`core::panic!`和`std::panic!`并不是一个东西；正如我们所看到的，它们的代码路径截然不同：

- libcore中的`core::panic!`做的事情很少。它基本上只是立即委托给panic handler。
- libstd中的`std::panic!`（也就是Rust的“普通”`panic!`宏）会触发一个功能齐全的panic机制，并提供一个由用户控制的panic*钩子*。默认钩子会把恐慌信息打印到stderr。钩子运行完成后，libstd将委托给panic*运行时*。

  libstd还提供了一个调用相同机制的panic *handler*，因此`core::panic!`也会在这里结束。


现在让我们更深入地了解一下这些东西吧。

## Panic运行时

[panic运行时的接口](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libstd/panicking.rs#L53)（在[这个RFC](https://github.com/rust-lang/rfcs/blob/master/text/1513-less-unwinding.md)中被引入）是一个签名为`__rust_start_panic(payload: usize) -> u32`的函数，它被libstd导入，并在稍后被链接器解析。

其中的`usize`参数实际上是一个`*mut &mut dyn core::panic::BoxMeUp`——这是panic的“负载”（当panic被捕获时的可用信息）被传递的地方。`BoxMeUp`是一个不稳定的内部实现细节，但[从这个trait](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libcore/panic.rs#L268-L281)我们可以发现它实际上只是包装了一个`dyn Any + Send`，也就是`catch_unwind`和`thread::spawn`返回的[panic负载类型](https://doc.rust-lang.org/std/thread/type.Result.html)。`BoxMeUp::take_box`返回的是`Box<dyn Any + Send>`，但是以原始指针的形式（因为在这个trait定义的上下文中`Box`尚不可用）；而`BoxMeUp::get`只是借用了内容。

> 译注：`catch_unwind`函数接受一个闭包，并可以“捕捉”其中发生的panic。在从外部调用Rust代码时`catch_unwind`很有用（将Rust的panic展开和其他语言隔离开来），Rust程序的`main`实际上也是在一个`catch_unwind`里被调用的。

Rust自带这个接口的两个实现：`libpanic_unwind`（对应`-C panic=unwind`，大多数平台上的默认实现）和`libpanic_abort`（对应`-C panic=abort`）。

### `std::panic!`

在panic*运行时*接口上，libstd在[`std::panicking`](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libstd/panicking.rs)模块内实现了Rust默认的panic处理机制。

#### `rust_panic_with_hook`

[`rust_panic_with_hook`](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libstd/panicking.rs#L439-L441)是几乎所有程序都要经过的关键函数：

```rust
fn rust_panic_with_hook(
    payload: &mut dyn BoxMeUp,
    message: Option<&fmt::Arguments<'_>>,
    file_line_col: &(&str, u32, u32),
) -> !
```

这个函数接受触发panic的源代码位置、一个可选的panic消息（未格式化的fmt数据，参见[fmt::Arguments](https://doc.rust-lang.org/std/fmt/struct.Arguments.html)的文档）和一个负载。

> 译注：panic钩子是Rust标准库提供的一种错误处理机制。我们可以使用`std::panic::set_hook`来将一个`Fn(&PanicInfo<'_>)`函数设置为“panic钩子”，接下来当panic发生时，就会首先调用这个钩子，然后再进行展开（或中断）。

它的主要工作是调用当前的panic钩子，无论它实际上是什么。panic钩子拥有一个`PanicInfo`参数，所以我们为了调用它就需要panic代码位置、panic消息的格式化数据和一个负载。这和`rust_panic_with_hook`实际具有的参数相当匹配！`file_line_col`和`message`可以被直接用作组成`PanicInfo`的前两个元素；而`payload`则通过`BoxMeUp`的接口转换为`&(dyn Any + Send)`。

有趣的是，*默认*的panic钩子是完全忽略`message`的；你实际上看到的输出是[从`payload`向下转换到`&str`或`String`的](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libstd/panicking.rs#L176-L182)（但也能工作）。按理来说，调用者应该确保格式化信息`message`（如果存在的话）给出相同的结果。（并且我们接下来讨论的内容也确实能确保这一点）

最终，`rust_panic_with_hook`将panic分配给当前的panic*运行时*。在这一步时，只有`payload`参数还有后续的关联——这很重要：`message`（正如其`'_`生命周期所示）可能含有生命周期较短的引用，但panic负载将会沿着堆栈向上传播，因此这些负载必须是`'static`的。`'static`限制在这里相当隐蔽，我过了好一会儿才意识到[`Any`](https://doc.rust-lang.org/std/any/trait.Any.html)早已暗示了`'static`（并记住了`dyn BoxMeUp`仅仅用于获取一个`Box<dyn Any + Send>`）。（译注：`Any`trait拥有`'static`的生命周期约束。）

#### libstd pacnicking入口点

`rust_panic_with_hook`是`std::panicking`模块的一个私有函数；该模块在它的基础上提供了三个入口点，以及一个绕过它的入口点：

- [`begin_panic_handler`](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libstd/panicking.rs#L332)，默认的panic handler实现，支持了（我们马上会看到）来自`core::panic!`和内置代码（来自算术溢出、索引越界等）的panic。它的输入是[`PanicInfo`](https://doc.rust-lang.org/core/panic/struct.PanicInfo.html)，因此必须将它转换为`rust_panic_with_hook`的参数。有趣的是，尽管`PanicInfo`的组成部分和`rust_panic_with_hook`的参数很像，并且看起来可以直接透传，但实际的实现并没有这样干。libstd反而完全*忽略*了`PanicInfo`的内容，并构建实际的负载（传递给`rust_panic_with_hook`），以使其包含[格式化后的`message`](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libstd/panicking.rs#L350)。

  特别地，这意味着panic*运行时*和`no_std`的程序是无关的。它仅仅当libstd的panic handler实现被采用时才会起作用。（不过，即使在`no_std`下，通过`-C panic`设置的panic策略仍然会生效，因为它还是能影响到代码生成的过程。例如，当使用了`-C panic=abort`时，代码会变得更简单，因为它不需要支持展开了。）
- [`begin_panic_fmt`](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libstd/panicking.rs#L316)，支持了[`std::panic!`宏的格式化字符串版本](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libstd/macros.rs#L22-L23)（也就是当你往这个宏里传入多个参数时使用的版本）。这个函数基本上只是把格式字符串参数打包成一个`PanicInfo`（并带有一个[假的负载](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libcore/panic.rs#L56)），然后调用我们刚刚讨论过的默认panic handler。（译注：这里提到的“假的负载”，是一个叫作`NoPayload`的空结构体。）
- [`begin_panic`](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libstd/panicking.rs#L392)，支持了[`std::panic!`宏的单参数版本](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libstd/macros.rs#L16)。有趣的是，这个入口点的代码路径和前面两个入口点非常地不同！特别地，这是唯一一个允许传入*任意负载*的入口点。这个负载只是[被转换为`Box<dyn Any + Send>`](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libstd/panicking.rs#L419)，使得其可以被传递给`rust_panic_with_hook`，然后就没有然后了。

  特别地，当panic钩子查看`PanicData`的`message`字段时，将无法在`std::panic!("do panic")`中看到消息；但在`std::panic!("panic with data: {}", data)`中，它可以看到消息。这是因为后者是通过`begin_panic_fmt`传递的。这看起来挺令人吃惊的。（不过也请注意，`PanicData::message()`方法还没有稳定下来。）（译注：本文翻译时，Rust仓库中已经搜索不到`PanicData`了。）
- [`rust_panic_without_hook`](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libstd/panicking.rs#L496)是个怪胎：这个入口点为[`resume_unwind`](https://doc.rust-lang.org/nightly/std/panic/fn.resume_unwind.html)提供支持（译注：`resume_unwind`函数用于继续向上传播一个已经开始，且被`catch_unwind`捕获的恐慌），它实际上不会调用panic钩子。相反，它立即向panic运行时发出委托。像`begin_panic`一样，它允许调用者指定任意负载。和`begin_panic`不同的是，调用者需要自行负责对负载的装箱和调整大小；`rust_panic_without_hook`几乎逐字转发这些内容到panic运行时。

## Panic Handler

`std::panic!`的所有机制都很有用，但它们都依赖于通过`Box`完成的堆内存分配，而这样的条件并非总是具备的。为了给libcore一条引发panic的路，[Rust引入了panic handler](https://github.com/rust-lang/rfcs/blob/master/text/2070-panic-implementation.md)。正如我们所看到的，如果libstd可用，那么它将会为`core::panic!`提供一个连接其和libstd的panic机制的接口。

[panic handler的接口](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libcore/panicking.rs#L78)是一个被libcore导入的函数`fn panic(info:&core::panic::PanicInfo)->!`，并且在稍后由链接器解析。[`PanicInfo`](https://doc.rust-lang.org/core/panic/struct.PanicInfo.html)类型和panic钩子中使用的一样：它包括一个panic源位置，一条panic消息，以及一个负载（`dyn Any + Send`）。panic消息以[`fmt::Arguments`](https://doc.rust-lang.org/std/fmt/struct.Arguments.html)的类型呈现，或者说一条还没有被格式化的格式化字符串及其参数。

### `core::panic!`

在panic handler的接口之上，libcore提供了一套[最小化的panic API](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libcore/panicking.rs)。`core::panic!`宏创建了一个`fmt::Arguments`，并在稍后[传递给panic handler](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libcore/panicking.rs#L82)。这里并没有实际的格式化发生，因为进行格式化可能需要进行堆内存分配；这也就是为什么`PanicInfo`携带的是一个“没有被解释”的格式化字符串及其参数。

有趣的是，传递给panic handler的`PanicInfo`的`payload`字段总是被设置为一个[虚假的值](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libcore/panic.rs#L56)。这就解释了为什么libstd的panic handler会忽略`PanicInfo`中的负载（而是从消息中重新构建一个新的负载），但这也使得我想知道为什么这个字段是panic handler API的一部分。这种做法的另一个结果是`core::panic!("message")`和`std::panic!("message")`（没有格式化的变体）实际上会产生截然不同的panic：前者会变成`fmt::Arguments`，通过panic handler接口传递。之后libstd会格式化它并得到一个`String`负载。不过，后者直接将`&str`作为负载，并且将`PanicInfo`的`message`字段留空为`None`（正如我们在上文已经讲过的那样）。

libcore的panic API中的一些元素是语言项，因为编译器需要在生成代码期间插入对这些函数的调用：

- [`panic`语言项](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libcore/panicking.rs#L39)在编译器需要引发一个无需任何格式化的panic时被调用（例如算术溢出）；这个语言项也是在幕后支持了单参数`core::panic!`的函数。
- [`panic_bounds_check`](https://github.com/rust-lang/rust/blob/4007d4ef26eab44bdabc2b7574d032152264d3ad/src/libcore/panicking.rs#L55)语言项在数组或切片的索引越界检查失败时被调用。它调用了和`core::panic!`相同的方法，并进行了格式化。

> 译注：语言项（lang item）是Rust中一个非常核心的概念，它们是编译器内部特殊处理的功能或特性的标识。语言项机制允许开发者直接为编译器提供某些必须的底层实现细节。语言项通常用于实现Rust标准库中的核心功能，例如`Box`、`Trait`对象等等。语言项不是普通的库函数和特性，它们在编译阶段拥有特殊的意义，并由编译器特殊识别和处理。

## 结语

我们已经走过了4层API，其中2层是通过函数导入和链接器解析来间接实现的。这真是一场意义非凡的旅行！但是现在我们已经接近终点了。我希望你在这一路上[没有把自己panic](https://en.wikipedia.org/wiki/Phrases_from_The_Hitchhiker%27s_Guide_to_the_Galaxy#Don't_Panic)了;)。（译注：原句为“I hope you didn't panic yourself”，Don't Panic是《银河系漫游指南》中的一个梗，或者说其中引申出的一句俗语，详情可以点击链接查看。）

我提到的有些东西听起来可能比较令人惊讶。事实证明，panic钩子和panic handler的接口共享`PanicInfo`结构体，这个结构体中都包括一个可选的、未格式化的`message`和一个擦出了类型的`payload`：

- panic*钩子*总是能在`payload`中找到已经格式化的消息，因此`message`看起来对钩子函数的作用并不大。事实上，即使`payload`中包含一条消息，`message`也可能会是空的（例如`std::panic!("message")`）。
- panic *handler*永远不会真正收到一个有用处的`payload`，因此这个字段对handler们的用处看起来并不大。

根据这个[panic handle的RFC](https://github.com/rust-lang/rfcs/blob/master/text/2070-panic-implementation.md)，看起来有计划让`core::panic!`也支持任意有效载荷，但到目前（2019年）为止还没有实现。不过，即使未来有了这样的扩展，我认为我们也会有一个不变量，即当`message`是`Some`时，要么`payload`是假的（`payload == &NoPayload`，负载是无用的），要么`payload`是一个格式化后的消息（此时消息是无用的）。我想知道会不会有一种让两个字段都有用的情况——如果没有的话，我们难道不能直接把这两个变量合并成一个`enum`吗？可能有一些很好的理由来反对这个提议和支持现有设计，如果能把它们记录在某处就太棒了:)。

还有很多内容可以讲述，但在这里，我邀请你根据我在文中提供的链接去看看源代码。只要你记得高层次的结构，你应该就能理解那些代码。如果人们认为这些概述值得放在更持久的地方，我很乐于将这篇博文整理成某种形式的文档——尽管我不确定放在哪里比较合适。如果你发现我写的有任何错误，请告诉我！


