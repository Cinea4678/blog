---
title: 短小精悍(1) - Rust字符串搜索库memchr介绍
zhihu-url: https://zhuanlan.zhihu.com/p/673645289
---

## 前言

初入Rust的同学可能会时常被编译时动辄上百个的依赖所震撼，由于Cargo并不会像Maven Repository一样会在上传前就把包编译好，再加上每个Rust库的作者都喜欢再额外使用另外几个更底层的库，就导致了每次从零开始编译一个Rust项目都需要动辄五六分钟的长时间。

不过，如果你对相对更底层的开发感兴趣的话，你也许也会需要那些被反反复复间接依赖的库。它们有共同的特点：短小精悍、文档完善，并且每个库都只专注于做好一件事情。这个系列的文章主要为打算进军基础库、嵌入式应用、高性能应用开发，甚至是编写操作系统的读者介绍一些在Rust生态中常用的“短小精悍”的基础库，内容包括库的用途、特点，甚至是一些实现的细节等，供读者未来选用基础库时参考。

## 简介

`memchr`是一个高性能的字符串搜索库，提供了一系列经过高度优化的字符串搜索函数。它主要分为两个部分：顶层的模块提供了针对字节的搜索，而模块`memmem`提供了针对子串的搜索。`memchr`对字符串的所有操作都是编码无关的。

以下是官方的一个例子，用于在一个字符串中寻找某个字节第一次出现的位置：

```rust
use memchr::memchr;

let haystack = b"foo bar baz quuz";
assert_eq!(Some(10), memchr(b'z', haystack));
```

除了可以查找字节第一次出现的位置，`memchr`也提供了迭代器来遍历字节在字符串内的所有位置：

```rust
use memchr::memchr_iter;

let haystack = b"foo bar baz quuz";
let mut iter = memchr_iter(b'a', haystack);
assert_eq!(Some(5), iter.next());
assert_eq!(Some(9), iter.next());
assert_eq!(None, iter.next());
```

`memchr`还提供了一个很有趣的功能：同时查找感兴趣的2或3个字节在字符串内的位置。

```rust
use memchr::memchr3_iter;

let haystack = b"xyzaxyzbxyzc";

let mut it = memchr3_iter(b'a', b'b', b'c', haystack).rev();
assert_eq!(Some(11), it.next());
assert_eq!(Some(7), it.next());
assert_eq!(Some(3), it.next());
assert_eq!(None, it.next());
```

查找子字符串的函数位于`memmem`模块内，用法和查找字节的类似：

```rust
use memchr::memmem;

let haystack = b"foo bar foo baz foo";

let it = memmem::find(haystack, b"foo");
assert_eq!(Some(0), it);
```

特别地，在查找子字符串时，官方推荐复用`Finder`来进一步提高效率：

```rust
use memchr::memmem;

let finder = memmem::Finder::new("foo");

assert_eq!(Some(4), finder.find(b"baz foo quux"));
assert_eq!(None, finder.find(b"quux baz bar"));
```

## 特点

一句话：在处理长字符串时特别快。到底有多快呢，我们来尝试和标准库做一个对比。

### 基准测试

测试的规格如下：AMD Ryzen 7 5800H@3.20 GHz，开启优化。

#### 测试1：超长字符串，匹配字节

测试代码：

```rust
#![feature(test)]

extern crate test;

#[cfg(test)]
mod tests {
    use test::Bencher;

    #[bench]
    fn bench_std(b: &mut Bencher) {
        // 生成测试使用的字符串
        let test_str = "a".repeat(600000) + "b" + &"a".repeat(399999);
        b.iter(|| test_str.chars().position(|b| b == 'b'))
    }

    #[bench]
    fn bench_memchr(b: &mut Bencher) {
        // 生成测试使用的字符串
        let test_str = "a".repeat(600000) + "b" + &"a".repeat(399999);
        b.iter(|| memchr::memchr(b'b', test_str.as_bytes()))
    }
}

```

测试结果：

![](https://s.c.accr.cc/picgo/1703223883-f4328f.png)

#### 测试2：短字符串，匹配字节

测试代码：

```rust
#![feature(test)]

extern crate test;

#[cfg(test)]
mod tests {
    use test::Bencher;

    #[bench]
    fn bench_std(b: &mut Bencher) {
        let test_str = "Remember us...";
        b.iter(|| {
            for _ in 0..10000 {
                test_str.chars().position(|b| b == 'u');
            }
        })
    }

    #[bench]
    fn bench_memchr(b: &mut Bencher) {
        let test_str = "Remember us...";
        b.iter(|| {
            for _ in 0..10000 {
                memchr::memchr(b'u', test_str.as_bytes());
            }
        })

    }
}

```

测试结果：

![](https://s.c.accr.cc/picgo/1703225398-dd86ec.png)

#### 测试3：长字符串，匹配子串

测试代码：

```rust
#![feature(test)]

extern crate test;

#[cfg(test)]
mod tests {
    use test::Bencher;

    #[bench]
    fn bench_std(b: &mut Bencher) {
        // 生成测试使用的字符串
        let test_str = "a".repeat(600000) + "niddle" + &"a".repeat(399999);
        b.iter(|| test_str.find("niddle"))
    }

    #[bench]
    fn bench_memchr(b: &mut Bencher) {
        // 生成测试使用的字符串
        let test_str = "a".repeat(600000) + "niddle" + &"a".repeat(399999);
        b.iter(|| memchr::memmem::find(test_str.as_bytes(), b"niddle"))
    }
}

```

测试结果：

![](https://s.c.accr.cc/picgo/1703224372-3b81b0.png)

#### 测试4：短字符串，匹配子串

测试代码：

```rust
#![feature(test)]

extern crate test;

#[cfg(test)]
mod tests {
    use test::Bencher;

    #[bench]
    fn bench_std(b: &mut Bencher) {
        let test_str = "Remember us...";
        b.iter(|| {
            for _ in 0..10000 {
                test_str.find("us");
            }
        })
    }

    #[bench]
    fn bench_memchr(b: &mut Bencher) {
        let test_str = "Remember us...";
        b.iter(|| {
            for _ in 0..10000 {
                memchr::memmem::find(test_str.as_bytes(), b"us");
            }
        })
    }
}

```

测试结果：

![](https://s.c.accr.cc/picgo/1703225338-ebdb38.png)

#### 结果整理

我们可以把结果整理为这样的一张表格：

|                    | `memchr` | 标准库 |
| ------------------ | -------- | ------ |
| 长字符串，匹配字节 | 5494     | 290115 |
| 短字符串，匹配字节 | 57137    | 58545  |
| 长字符串，匹配子串 | 10545    | 45133  |
| 短字符串，匹配子串 | 65322    | 135526 |

![](https://s.c.accr.cc/picgo/1703227675-cb59d8.png)

### 性能特点

官方在文档中非常详细地说明了`memchr`在匹配字节时的性能特点：延迟稍差，但吞吐量很高。什么是延迟和吞吐量呢？简单来说，延迟是指一个字符串从输入到开始真正进行匹配之间的用时，而吞吐量是指在匹配过程中单位时间内可以被匹配的字节数量。我们可以看到，对短字符串而言，延迟的大小决定了匹配的时间，而对长字符串而言，则是吞吐量的大小决定匹配的时间。

因此，我们可以看到在长字符串中匹配字节时，`memchr`表现出了惊艳的性能；而在短字符串中匹配字节时，`memchr`和标准库差距不大。读者在决定是否需要使用`memchr`时，也可以将这个因素纳入考虑。

## no-std 支持

本库支持no_std。在`x86_64`平台上，禁用std时，将使用SSE2技术作为加速实现；在启用std时，如果可以在运行时确定CPU支持AVX2加速，则会使用AVX2加速实现。这可能会导致一定的性能差异。

