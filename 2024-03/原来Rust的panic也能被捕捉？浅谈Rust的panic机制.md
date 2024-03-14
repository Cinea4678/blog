---
title: 原来Rust的panic也能被捕捉？浅谈Rust的panic机制
zhihu-url: https://zhuanlan.zhihu.com/p/687092686
---

> 这一系列文章的创作目的主要是帮助我自己深入学习Rust，同时也为已经具备一定Rust编程经验，但还没有深入研究过语言和标准库的朋友提供参考。对于正在入门Rust的同学，我更建议你们看[《Rust圣经》](course.rs)或者[《The Book》](https://doc.rust-lang.org/stable/book/)，而不是这种晦涩难懂的文章。

你用过`panic!`宏吗？在Rust里，`panic!`宏可以用来触发一个panic，它会立即终止当前执行的函数，并开始展开（unwinding）当前线程的调用栈，清理每个栈帧中的数据（包括调用它们的`Drop`方法并释放资源）。清理完成后，它会终止当前运行的进程。

当然，上面描述的是panic的默认处理机制，也是大多数人印象里panic的行为。事实上，panic的行为和我们能对它做的处理可远远不止这些，本文将会带领大家浅尝Rust的panic机制，了解panic不为（大多数）人知的另一面。



## Panic的处理过程

从产生到线程终结，panic在这其中经历了什么？首先放一幅我自己画的大图镇楼：

![](https://s.c.accr.cc/picgo/1710425068-cdc2f6.png)

在用户视角，我们只是调用了`panic!()`宏，然后Rust为我们输出一堆信息；但是在Rust内部，其实程序经历了以下这些步骤：

1. 构造`PanicInfo`和调用`panic_impl`函数：`PanicInfo`是包括了panic的位置和信息的结构，`panic_impl`是在`std`内或自己定义的一个函数。在程序编译时，链接器会寻找`panic_impl`函数并链接，如果没有找到，那么编译就会失败。
2. 如果在非std环境下，那么现在处理panic的就是我们自己的函数，后续的处理逻辑也由我们自己定义。
3. 如果在std环境下，那么现在处理panic的就是标准库内的处理函数。首先，Rust会检查当前是否出现了双重panic。双重panic是一个类似于操作系统中的双重异常的概念，指的是在处理panic的过程中再次触发了panic。为了防止panic的处理陷入无尽循环，当Rust检测到发生了双重panic时，它会在屏幕上打印这样的信息并立即终止程序：

```
thread panicked while processing panic. aborting.
```

4. 如果没有出现双重panic，那么Rust就会进行下一步的处理。在`std::panic`模块内提供了`set_hook`方法，它允许程序在发生panic时在此处调用我们自定义的钩子。我们可以在这个钩子里释放一些特定资源，或者调用操作系统的接口向用户展示错误弹窗。

如果我们没有设置自定义钩子的话，Rust就会调用默认钩子，打印这种我们平时经常见到的信息：

```
thread 'main' panicked at src/main.rs:2:5:
System error
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

5. 从这里开始，程序将会切换到panic运行时。如果在`Cargo.toml`内设置了`panic = abort`的话，那么程序运行到这里后就会立即退出。反之，也就是在默认情况下，程序会开始展开函数调用栈，并释放栈帧内的资源，逐一调用其中变量的`Drop`实现。
6. 在清理完栈帧、完成展开之后，程序会把panic的信息通过`catch_unwind`函数返回。默认情况下，我们的`main`函数和整个程序会被包裹在std运行时提供的`catch_unwind`内；当然，在特殊场景内我们也会调用这个函数并自行处理。这段话读起来可能有点抽象，后文会详细介绍这个函数的用法。
7. 如果当前不在自定义的`catch_unwind`内的话，我们的程序就会从程序根部的`catch_unwind`返回。Rust会检查是否发生了panic；如果是的话，就会为程序设置值为101的返回代码，退出程序。

现在我们终于讲完了panic的处理机制，接下来我们来看看我们能对panic做什么吧。



## #[panic_handler]

使用Rust从事过嵌入式或操作系统等`#[no_std]`开发的读者一定不会对`#[panic_handler]`这个标注陌生；当我们不链接到标准库（也就是声明了`#[no_std]`）时，往往必须指定一个函数作为`#[panic_handler]`，也就是panic的处理函数。

前面介绍过我们为什么需要指定一个panic处理函数：`core`（也就是不链接std时使用的Rust语言核心库）中是不包含`panic_impl`函数的实现的；它在`core`的源码中被指定为了一个外部函数：

```rust
// rust/library/core/src/panicking.rs
pub const fn panic_fmt(fmt: fmt::Arguments<'_>) -> ! {
    // ......
    
	// NOTE This function never crosses the FFI boundary; it's a Rust-to-Rust call
    // that gets resolved to the `#[panic_handler]` function.
    extern "Rust" {
        #[lang = "panic_impl"]
        fn panic_impl(pi: &PanicInfo<'_>) -> !;
    }
    
    // ......
    
    // SAFETY: `panic_impl` is defined in safe Rust code and thus is safe to call.
    unsafe { panic_impl(&pi) }
}
```

`panic_impl`外部函数的身份意味着在链接时它必须以某种方式出现。在正常情况下，`std`会提供`panic_impl`的实现，不需要我们自己实现；但在无`std`环境下，我们就需要自己提供这个`panic_impl`。为了提供`panic_impl`的实现，我们需要定义一个签名为`fn(&PanicInfo) -> !`的函数，并对其使用`#[panic_handler]`标注：

```rust
#![no_std]

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

> 虽然`#[panic_handler]`的用法和它做的事情很像过程宏，但是实际上它是由编译器直接识别和处理的。

我们可以巧用`#[panic_handler]`来实现一些有趣的效果：

![大二时写的操作系统课设](https://s.c.accr.cc/picgo/1710415343-a68c1f.jpg)

## panic钩子

在拥有`std`的环境中，我们也可以指定panic的处理钩子，它会在进行栈展开（或终止进程）前被调用。在`std::panic`模块中，提供了用于操作panic钩子的两个函数：`set_hook`和`take_hook`。`set_hook`用于设置钩子，`take_hook`用于获取当前钩子并恢复原钩子。panic钩子的类型是`Box<dyn Fn(&PanicInfo<'_>) + Sync + Send + 'static>`。

panic钩子的类型很“Rust特色”，因为它是全局资源。下面是一个例子，它会在出现panic时向用户提供一个错误弹窗：

```rust
// native-dialog = {version = "0.7.0", features = ["windows_dpi_awareness"] }

use std::panic::PanicInfo;
use std::process::exit;
use native_dialog::{MessageDialog, MessageType};

pub fn panic_handler(_panic_info: &PanicInfo) {
    let panic_message = "A critical system failure occurred and the program will shutdown immediately.\n\n".to_owned();

    let _ = MessageDialog::new()
        .set_title("Sorry!")
        .set_text(&panic_message)
        .set_type(MessageType::Error)
        .show_alert()
        .unwrap();

    exit(1);
}

fn main() {
    std::panic::set_hook(Box::new(panic::panic_handler));
    
    panic!("Test");
}
```

效果如下图所示：

![](https://s.c.accr.cc/picgo/1710416635-f6bfdd.png)

当我们用`tauri`之类的框架开发本地应用时，通常在运行时是不会展示控制台的。比起不明不白地闪退，展示一个弹窗对用户会更友好一些。

## panic = "abort"

前面提到panic时Rust会进行栈展开，逐层释放栈帧中的资源。虽然这种做法很优雅，但在某些特殊场景下，也许直接让程序停止执行更好一些。这取决于你的实际需求。

如果你需要程序在panic时直接退出，可以在`Config.toml`中加入这样的配置：

```rust
[profile.dev]
panic = "abort"

// or

[profile.release]
panic = "abort"
```

## 使用catch_unwind捕获panic

这篇文章的最后一个话题是`std::panic::catch_unwind`函数。这个函数很神奇：它可以让我们像C++的try/catch一样捕获来自代码内部的panic，并自行处理。

### 用法

`catch_unwind`函数需要传入一个函数f，这个f就是需要执行（并捕捉其中的panic）的函数。`catch_unwind`返回一个`thread::Result`，其错误类型是`Box<dyn Any + Send + 'static>`，代表调用`panic!`宏时传入的参数。函数具体的定义如下所示：

```rust
pub fn catch_unwind<F: FnOnce() -> R + UnwindSafe, R>(f: F) -> Result<R>;
```

### 真实案例：Rust程序的main是怎么被调用的？

众所周知，我们的Rust程序从`main`开始执行；同样众所周知的是，`main`并不是程序真正的起始点。类似于C++的`_start`，Rust程序也有一个`start`和一套用于初始化的函数，它们在系统层面为为程序初始化，创建名为`main`的线程，并启动`main`函数。以下是`main`被调用的代码片段：

```rust
let ret_code = panic::catch_unwind(move || panic::catch_unwind(main).unwrap_or(101) as isize)
    .map_err(move |e| {
        mem::forget(e);
        rtabort!("drop of the panic payload panicked");
    });
```

（注意下面的四行`map_err`不是重点哦，重点是第一行的`unwrap_or`）

我们可以看到，这段代码用`catch_unwind`调用了`main`，并在`main`出现panic时将返回代码设置为101。这种做法规范地处理了Rust的panic，并且为我们提供了一种检测Rust进程发生panic的机制。

### catch_unwind在外部语言调用Rust时的应用

众所周知，Rust的代码可以被非常容易地暴露为C ABI并被其他语言链接和调用。但是，如果被外部调用期间发生了panic怎么办呢？因为其他语言没有Rust的panic机制，所以我们一般来说需要把提供给外部的接口用`catch_unwind`包装起来，防止Rust的panic影响其他程序运行。

### 注意事项

- `catch_unwind`不能检测和捕捉来自其他语言的异常和错误。不要妄图用`catch_unwind`来接收来自C++、Java等语言的Exception哦~
- `catch_unwind`在启用了`panic = "abort"`时是不会起作用的。
- 不推荐像用C++的try/catch一样频繁使用`catch_unwind`；panic展开的代价比try/catch大得多。

