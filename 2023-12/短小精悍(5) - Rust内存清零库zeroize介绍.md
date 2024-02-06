---
title: 短小精悍(5) - Rust内存清零库zeroize介绍
zhihu-url: https://zhuanlan.zhihu.com/p/674976137
---

今天带来的是一个“短小精悍”的库：`zeroize`。`zeroize`可以在确保不被编译器优化的前提下安全高效地清空一段内存，适合在保密应用内使用。

## 用法

`zeroize`的核心用法很简单：

```rust
use std::string::String;
use zeroize::Zeroize;

fn main() {
    let mut user_password = String::from("qwertyuiop");
    user_password.zeroize();
}
```

你也可以为自己的类型增加`zeroize`的支持（需要启用`derive`特性）：

```rust
use zeroize::Zeroize;

#[derive(Zeroize)]
struct MyStruct([u8; 32]);
```

甚至可以通过derive名为`ZeroizeOnDrop`的trait，来让自己的类型在被销毁时自动从内存擦除：

```rust
use zeroize::{Zeroize, ZeroizeOnDrop};

#[derive(Zeroize, ZeroizeOnDrop)]
struct MyStruct([u8; 32]);
```

接下来，我们来使用unsafe代码来检查一下`zeroize`的正确性，即安全清零、不被优化：

```rust
use zeroize::Zeroize;

#[derive(Zeroize)]
struct SecretInfomation([u8; 4]);

fn main() {
    let secret = SecretInfomation([2u8, 4, 6, 8]);

    let ptr: usize = secret.0.as_ptr() as usize;

    // 读取一次secret中的数据，现在secret还可以被正常访问
    unsafe {
        let read_secret = std::slice::from_raw_parts(ptr as *const u8, 4);
        println!("Secret before zeroize: {:?}", read_secret);
    }

    // secret.zeroize();
    drop(secret);

    // 再读取一次secret中的数据，现在secret没有被置零
    unsafe {
        let read_secret = std::slice::from_raw_parts(ptr as *const u8, 4);
        println!("Secret without zeroize: {:?}", read_secret);
    }

}
```

实验结果是：

![](https://s.c.accr.cc/picgo/1703733035-da2d5c.png)

现在我们在`drop`对象之前调用一次`zeroize`：

```rust
use zeroize::Zeroize;

#[derive(Zeroize)]
struct SecretInfomation([u8; 4]);

fn main() {
    let mut secret = SecretInfomation([2u8, 4, 6, 8]);

    let ptr: usize = secret.0.as_ptr() as usize;

    // 读取一次secret中的数据，现在secret还可以被正常访问
    unsafe {
        let read_secret = std::slice::from_raw_parts(ptr as *const u8, 4);
        println!("Secret before zeroize: {:?}", read_secret);
    }

    secret.zeroize();
    drop(secret);

    // 再读取一次secret中的数据，现在secret已经被置零
    unsafe {
        let read_secret = std::slice::from_raw_parts(ptr as *const u8, 4);
        println!("Secret after zeroize: {:?}", read_secret);
    }

}
```

![](https://s.c.accr.cc/picgo/1703733127-bf5973.png)

这次，`secret`的值被置0，并且在未优化和已优化两种情况下都持续有效。

## 用途

既然是即将被drop的数据，为什么我们还要特意清零它呢？在一个理想的完美世界里，这样做确实是无意义的。但是如果黑客能够利用漏洞读取计算机未初始化的内存（这不是如果，这是不可避免的事实），那么仍然留在内存里的敏感数据便会直接成为黑客的囊中之物。对一些加密应用、保密应用、处理各类机密的程序来说，这种风险是致命、不容忽视的。在[低级加密软件指南](https://github.com/veorq/cryptocoding#clean-memory-of-secret-data)里，有更为详细和专业的说明。

遗憾的是，编译器并不能保证我们直接使用`memset`等接口执行的清零操作总是不被优化。在C中，我们可以使用`volatile`关键字来保证清零操作不会被优化，例如这是一个载于[How to zero a buffer](http://www.daemonology.net/blog/2014-09-04-how-to-zero-a-buffer.html)的例子：

```c
static void * (* const volatile memset_ptr)(void *, int, size_t) = memset;

static void secure_memzero(void * p, size_t len)
{
  (memset_ptr)(p, 0, len);
}
```

Rust当然也提供了类似的关键字。在标准库函数`core::ptr::write_volatile`中，我们可以易失性地写入指定的内存地址。`zeroize`便为我们封装了这样的操作，并向我们提供了Rust风格的、高级的、易用的`Zeroize`trait。

此外，`zeroize`是Rust Crypto项目的一个组成部分，这个项目为Rust实现了大量的加密算法。同样的，这个项目中的算法实现也都使用了`zeroize`来清除使用完成的内存。

