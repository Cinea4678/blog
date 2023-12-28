---
title: 短小精悍(4) - Rust操作系统随机数getrandom库介绍
zhihu-url: https://zhuanlan.zhihu.com/p/674931614
---

今天带来的是另一个“短小精悍”的库：`getrandom`。它的作用是从操作系统提供的随机数源获得一段随机数。

## 用法

`getrandom`的用法很简单，唯一需要了解的就是它内部的同名函数：

```rust
pub fn getrandom(dest: &mut [u8]) -> Result<(), Error>
```

它将会向dest中填充来自操作系统的随机数。

示例：
```rust
fn main() {
    let mut data = [0u8; 8];
    getrandom::getrandom(&mut data).unwrap();
    println!("{:?}", data);
}

// $ ./playground
// [186, 180, 148, 20, 13, 30, 186, 46]
```

## 用途

首先，读者应当了解一点：`getrandom`在性能上并不如`rand`库的`thread_rng`，它并不适合被放置在热点路径上反复调用。`getrandom`应当被用来生成那种“一次性使用”的随机数，例如在其他随机数算法中使用的种子。

例如，在默认的feature flags下，著名的加密库`ahash`便使用`getrandom`库来为Hasher在运行时提供种子：

```rust
// aHash/src/random_state.rs:73
if #[cfg(all(feature = "runtime-rng", not(fuzzing)))] {
    #[inline]
    fn get_fixed_seeds() -> &'static [[u64; 4]; 2] {
        use crate::convert::Convert;

        static SEEDS: OnceBox<[[u64; 4]; 2]> = OnceBox::new();

        SEEDS.get_or_init(|| {
            let mut result: [u8; 64] = [0; 64];
            getrandom::getrandom(&mut result).expect("getrandom::getrandom() failed.");
            Box::new(result.convert())
        })
    }
}
```

库`uuid`对`getrandom`库的态度也能佐证这一点：

```rust
// uuid/src/rng.rs:1
#[cfg(any(feature = "v4", feature = "v7"))]
pub(crate) fn bytes() -> [u8; 16] {
    #[cfg(not(feature = "fast-rng"))]
    {
        let mut bytes = [0u8; 16];

        getrandom::getrandom(&mut bytes).unwrap_or_else(|err| {
            // NB: getrandom::Error has no source; this is adequate display
            panic!("could not retrieve random bytes for uuid: {}", err)
        });

        bytes
    }

    #[cfg(feature = "fast-rng")]
    {
        rand::random()
    }
}
```

在`uuid`默认开启的特性`fast-rng`下，`uuid`将会使用`rand`库来生成随机数，因为这比直接使用`getrandom`快得多。

所以，对于大多数情况来说，仅仅使用`rand`的`thread_rng`就已经完全足够了；当读者计划使用`getrandom`时，建议读者最好能够确定自己真的需要它。

## 原理和细节

`getrandom`库使用了来自操作系统的接口来生成随机数，具体来说，在Linux上使用`getrandom`（[man7.org](https://www.man7.org/linux/man-pages/man2/getrandom.2.html)），在Windows上使用`BCryptGenRandom`（[Windows Learn](https://learn.microsoft.com/en-us/windows/win32/api/bcrypt/nf-bcrypt-bcryptgenrandom)），在macOS上使用`getentropy`（[unix.com](https://www.unix.com/man-page/mojave/2/getentropy/)）。

`getrandom`库在遇到错误时会优先返回`Error`，而不是不可信的结果。

`getrandom`库在系统启动的早期可能会因为操作系统还没有安全地为RNG设置种子而阻塞。



