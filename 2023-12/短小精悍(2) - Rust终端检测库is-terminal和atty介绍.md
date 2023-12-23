---
title: 短小精悍(2) - Rust终端检测库is-terminal和atty介绍
zhihu-url: https://zhuanlan.zhihu.com/p/673841498
---

今天给大家介绍的是Rust中非常常用的两个用于检测终端的库`is-terminal`和`atty`。这两个库都是千万级别的下载量，大多数和日志、流、交互相关的库都会依赖它们，而我们在开发基础工具时可能也会用到。



由于两个包的功能都大差不差，因此接下来主要介绍`is-terminal`。

## 简介

两个库的文档都很简单：该库用于回答一个简单的问题：这是不是终端（tty）？

![image-20231223192224840](https://s.c.accr.cc/picgo/1703330547-fcf0f9.png)

但是，可能有读者还在疑惑：到底什么是终端？程序不在终端运行，还能在哪里运行？关于tty和终端究竟是什么，可以参考这个[知乎问题](https://www.zhihu.com/question/21711307)；但是在这里，终端是相对于普通文件流的一个概念，可以参考下面的图片来理解：

![is-terminal(1)](https://s.c.accr.cc/picgo/1703332156-7bdd75.png)

我们可以看到，在右边的测试中，我们将输出**重定向**到了一个txt文件里，`is-terminal`就对`stdout`给出了“不是终端”的判断。类似的测试对管道也是成立的，如果我们把输入或输出使用管道进行了重定向，`is-terminal`也会给出“不是终端”的判断。

当程序被其他程序调用并作为新进程运行时，`is-terminal`的判断则取决于我们怎么处理它的stdout。比如下面是另一个Rust程序的代码，它创建了一个检查stdout是否在终端内的进程，并收集其输出（位于stderr）：

```rust
use std::{process::Command, io, fs::File};

fn main() {
    Command::new("./rust-is-terminal").stderr(io::stdout()).spawn().unwrap();
}
```

我们并没有收集它的stdout，因此它判断它的stdout位于终端内：

![](https://s.c.accr.cc/picgo/1703334212-a15641.png)

稍加修改：我们将它的stdout重定向到一个文件中。

```rust
use std::{process::Command, io, fs::File};

fn main() {
    let stdout = File::create("out.txt").unwrap();
    Command::new("./rust-is-terminal").stdout(stdout).stderr(io::stdout()).spawn().unwrap();
}
```

这次运行，它判断它的stdout不在终端内：

![](https://s.c.accr.cc/picgo/1703334272-b9e2a4.png)

## 用途

所以这玩意儿有什么用呢？其实它的用处还真挺大。首先，我们知道终端和普通的文件流都是计算机中用于数据输入和输出的接口，但它们在功能和用途上存在显著差异。例如，终端可以解释特殊的控制序列，像Linux终端里大名鼎鼎的`\033`（`\x1b`），便可以移动光标、清屏、设置颜色等。因此，程序在运行时就可以通过判断流是否来自终端，来决定自己要不要提供仅在终端下可用的功能。

比如下面的代码，它在终端内运行时可以获得非常漂亮的输出效果：

```rust
fn main() {
    println!("\x1b[0;30;41m WARN \x1b[0m The target is not found \n")
}
```

![](https://s.c.accr.cc/picgo/1703335113-0259d1.png)

但是假如这个程序需要被定期自动执行，管理员在检查程序输出的时候就会被难以阅读的控制序列所困扰：

![](https://s.c.accr.cc/picgo/1703335309-99243e.png)

因此，这段代码更加合理的写法便是：

```rust
use is_terminal::IsTerminal;
use std::io::stdout;

fn main() {
    if stdout().is_terminal() {
        println!("\x1b[0;30;41m WARN \x1b[0m The target is not found \n");
    }else{
        println!("[ WARN ] The target is not found \n");
    }
}
```

![](https://s.c.accr.cc/picgo/1703335596-609ff3.png)

## 扩展：实现原理

如果你是因为想知道`is-terminal`和`atty`怎么用/有什么用而点进这篇文章的话，那么读到这里对你来说应该已经差不多了。不过，如果你对它的实现原理感兴趣的话，我们接下来可以一起研究一下。

### Windows平台

在Windows上，`is-terminal`使用了一个叫做`GetConsoleMode`的Windows API来判断一个流是否属于终端。这个API的作用是检索控制台的输入模式/屏幕缓冲区的输出模式。具体的文档可以参见[这里](https://learn.microsoft.com/zh-cn/windows/console/getconsolemode)。

我们简单地读一下这个API的文档，就会发现这个API实际上本意并非用于判断终端，而是用于获取当前终端的各种属性，例如终端当前是否位于插入模式、是否位于快速编辑模式等等。那么如果提供API的流不属于终端呢？这就会导致`GetConsoleMode`出错，并返回一个代表错误的0。

以下摘自`is-terminal`的源码：

```rust
#[cfg(windows)]
fn handle_is_console(handle: BorrowedHandle<'_>) -> bool {
    use windows_sys::Win32::System::Console::{
        GetConsoleMode, GetStdHandle, STD_ERROR_HANDLE, STD_INPUT_HANDLE, STD_OUTPUT_HANDLE,
    };

    let handle = handle.as_raw_handle();

    unsafe {
        // A null handle means the process has no console.
        if handle.is_null() {
            return false;
        }

        let mut out = 0;
        if GetConsoleMode(handle as HANDLE, &mut out) != 0 {
            // False positives aren't possible. If we got a console then we definitely have a console.
            return true;
        }

        // At this point, we *could* have a false negative. We can determine that this is a true
        // negative if we can detect the presence of a console on any of the standard I/O streams. If
        // another stream has a console, then we know we're in a Windows console and can therefore
        // trust the negative.
        for std_handle in [STD_INPUT_HANDLE, STD_OUTPUT_HANDLE, STD_ERROR_HANDLE] {
            let std_handle = GetStdHandle(std_handle);
            if std_handle != 0
                && std_handle != handle as HANDLE
                && GetConsoleMode(std_handle, &mut out) != 0
            {
                return false;
            }
        }

        // Otherwise, we fall back to an msys hack to see if we can detect the presence of a pty.
        msys_tty_on(handle as HANDLE)
    }
}
```

值得一提的是，`is-terminal`的作者注意到了在Windows上利用这个API来判断流是否来自终端并不总是可靠，比如当程序在msys或cygwin的环境中运行时，即使它确确实实位于msys的终端下，Windows也会认为它并没有在终端内运行。作者在注释中称这种特性为假阴性。因此，为了排除这种假阴性，作者首先检查了另外两个标准流是否来自终端，只要它们中有一个被检查出来自终端，就说明程序确实是运行在终端而不是msys下。如果另外两个标准流也并非来自于终端，那么程序就会检查流是否来自于msys的终端。

检查一个流是否来自msys终端的原理也很简单：作者调用了另一个API `GetFileInformationByHandleEx`，这个API的作用是检索句柄对应的文件信息。作者利用这个API来获取流的文件名，如果文件名带有`msys-`、`cygwin-`的前缀和`-pty`的后缀，就能够证明流来自于msys或cygwin的终端。

以下是这一过程的源码：

```rust
#[cfg(windows)]
unsafe fn msys_tty_on(handle: HANDLE) -> bool {
    use std::ffi::c_void;
    use windows_sys::Win32::{
        Foundation::MAX_PATH,
        Storage::FileSystem::{
            FileNameInfo, GetFileInformationByHandleEx, GetFileType, FILE_TYPE_PIPE,
        },
    };

    // Early return if the handle is not a pipe.
    if GetFileType(handle) != FILE_TYPE_PIPE {
        return false;
    }

    /// Mirrors windows_sys::Win32::Storage::FileSystem::FILE_NAME_INFO, giving
    /// it a fixed length that we can stack allocate
    #[repr(C)]
    #[allow(non_snake_case)]
    struct FILE_NAME_INFO {
        FileNameLength: u32,
        FileName: [u16; MAX_PATH as usize],
    }
    let mut name_info = FILE_NAME_INFO {
        FileNameLength: 0,
        FileName: [0; MAX_PATH as usize],
    };
    // Safety: buffer length is fixed.
    let res = GetFileInformationByHandleEx(
        handle,
        FileNameInfo,
        &mut name_info as *mut _ as *mut c_void,
        std::mem::size_of::<FILE_NAME_INFO>() as u32,
    );
    if res == 0 {
        return false;
    }

    // Use `get` because `FileNameLength` can be out of range.
    let s = match name_info
        .FileName
        .get(..name_info.FileNameLength as usize / 2)
    {
        None => return false,
        Some(s) => s,
    };
    let name = String::from_utf16_lossy(s);
    // Get the file name only.
    let name = name.rsplit('\\').next().unwrap_or(&name);
    // This checks whether 'pty' exists in the file name, which indicates that
    // a pseudo-terminal is attached. To mitigate against false positives
    // (e.g., an actual file name that contains 'pty'), we also require that
    // the file name begins with either the strings 'msys-' or 'cygwin-'.)
    let is_msys = name.starts_with("msys-") || name.starts_with("cygwin-");
    let is_pty = name.contains("-pty");
    is_msys && is_pty
}
```

### Unix/Linux平台

Linux和Unix这边就没那么多弯弯绕了。POSIX标准直接定义了接口`isatty`，任何兼容POSIX标准的操作系统都可以用这个接口来判断一个流是否来自于终端。

[isatty（定义）](https://pubs.opengroup.org/onlinepubs/007908799/xsh/isatty.html)

[isatty(3) - Linux manual page](https://www.man7.org/linux/man-pages/man3/isatty.3.html)

一个疑惑：Windows也是兼容POSIX接口的操作系统之一，为什么`is-terminal`的实现在`GetConsoleMode`的那一步里不使用Windows提供的`_isatty`接口呢？[_isatty | Microsoft Learn](https://learn.microsoft.com/zh-cn/cpp/c-runtime-library/reference/isatty)

