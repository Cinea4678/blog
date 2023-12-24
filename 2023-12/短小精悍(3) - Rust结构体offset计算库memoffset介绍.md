---
title: 短小精悍(3) - Rust结构体offset计算库memoffset介绍
zhihu-url: https://zhuanlan.zhihu.com/p/673961917
---

今天给大家带来的是另一个“短小精悍”的库：`memoffset`。经常和C语言打交道的同学肯定不会对C风格的结构体陌生，而在操作硬件设备、进行系统级编程时，直接从内存地址读/写结构体更是家常便饭。`memoffset`就是一个用于帮助我们“精细”操作结构体的工具，它可以计算指定字段在结构体中的偏移量，从而帮助我们手动操作结构体的内存。

## 简介

`memoffset`提供的宏`offset_of`和C的宏[offsetof](https://www.runoob.com/cprogramming/c-macro-offsetof.html)很像，都可以计算一个结构成员相对于结构开头的字节偏移量，例如以下这个来自文档的示例：

```rust
use memoffset::offset_of;

#[repr(C, packed)]
struct Foo {
    a: u32,
    b: u32,
    c: [u8; 5],
    d: u32,
}

fn main() {
    assert_eq!(offset_of!(Foo, b), 4);
    assert_eq!(offset_of!(Foo, d), 4+4+5);
}
```

`memoffset`还提供另一个用于获取内存范围的宏：`span_of`。顾名思义，`span_of`可以获取一个变量所位于的内存范围，例如这个来自于文档的示例：

```rust
use memoffset::span_of;

#[repr(C, packed)]
struct Foo {
    a: u32,
    b: u32,
    c: [u8; 5],
    d: u32,
}

fn main() {
    assert_eq!(span_of!(Foo, a),        0..4);
    assert_eq!(span_of!(Foo, a ..  c),  0..8);
    assert_eq!(span_of!(Foo, a ..= c),  0..13);
    assert_eq!(span_of!(Foo, ..= d),    0..17);
    assert_eq!(span_of!(Foo, b ..),     4..17);
}
```

## 案例

我们来看看`memoffset`的使用案例。

### NTFS库

`NTFS`是一个用 Rust 实现的低级NTFS文件系统库。像它这样的库可能并不太喜欢特别频繁地进行大块的读写，但又不太方便直接把整个结构体从裸指针加载上来（这样做有违Rust库的身份），就可以考虑用memoffset来指向性地读取部分内容。例如：

```rust
#[repr(C, packed)]
pub(crate) struct RecordHeader {
    signature: [u8; 4],
    update_sequence_offset: u16,
    update_sequence_count: u16,
    logfile_sequence_number: u64,
}

#[derive(Clone, Debug)]
pub(crate) struct Record {
    data: Vec<u8>,
    position: NtfsPosition,
}

impl Record {
    fn update_sequence_offset(&self) -> u16 {
        let start = offset_of!(RecordHeader, update_sequence_offset);
        LittleEndian::read_u16(&self.data[start..])
    }
}
```

这是节选自`ntfs/src/record.rs`的一段代码。可以看到，作者使用`offset_of`宏来直接“精准”地从内存中读取了一个u16。我水平有限，但是我认为这样做可能是出于减少磁盘读写的目的，如果读者了解个中缘由可以留言赐教。


