---
title: Rust的From和Into特征：可能是最常用的转换类特征
zhihu-url: https://zhuanlan.zhihu.com/p/684663427
---

说到`From`和`Into`，以及从他们中衍生出的`TryFrom`和`TryInto`，想必大家都不会陌生。它们不像`Borrow`、`AsRef`、`ToOwned`这些默默工作在泛型里的特征，是绝大多数Rust开发者每天都会使用到的东西。今天我们就来加深一下对这四个特征的了解吧~



## From和Into

如果说`AsRef`和`AsMut`的功能是做“引用到引用”的转换的话，那么`From`和`Into`做的就是“值到值”的转换了:

```rust
pub trait From<T>: Sized {
    // Required method
    fn from(value: T) -> Self;
}

pub trait Into<T>: Sized {
    // Required method
    fn into(self) -> T;
}
```

纵观两个特征的签名，它们都消耗掉一个值来产生另一个值；这就是`From`和`Into`的第一个小特点了：它们会立即把参数消耗掉。

对实现了`From<T>`的类型`U`，标准库为`T`提供了`Into<U>`的实现；也就是说，在为`U`实现了`From<T>`之后，就可以直接使用`T::into()`来构造`U`了：

```rust
use std::fmt;

struct BeautifulString(String);

impl From<String> for BeautifulString {
    fn from(mut value: String) -> Self {
        value.push_str("(✪ω✪)");
        Self(value)
    }
}

impl fmt::Display for BeautifulString {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

fn main() {
    let string = String::from("I am beautiful!");
    let beautiful: BeautifulString = string.into();	// 看这里
    println!("{}", beautiful); 						// I am beautiful!(✪ω✪)
}
```

除此以外，`From`是反身的，也就是说对任何类型`T`，都有`T: From<T>`。

### 我们该实现From还是Into？

既然`impl From<T> for U`之后可以自动获得`impl Into<U> for T`，那么我们自然应该优先实现`From`而不是`Into`了；仅仅当转换的一方不是当前crate的成员时，才应当考虑实现`Into`。最直观的例子就是我们可以为`T`实现`Into<String>`，但肯定不能为`String`实现`From<T>`，这违反了Rust的孤儿原则。

### 使用From和Into的原则

Rust文档中对`From`和`Into`的使用提出了以下的几条原则；它们中的一部分在技术角度上并无强制性，但遵循这些原则可以满足一般的用户预期：

- 转换应当是**万无一失**的：如果转换可能失败，那么应该使用`TryFrom`代替，而不是在`From`和`Into`的实现中埋下隐患，甚至产生panic。
- 转换应当是**无损**的：从语义上来讲，转换过程中不应该丢失或丢弃信息。例如，对`i32: From<u16>`来说，使用`u16: TryFrom<i32>`可以将前一个过程的转换结果恢复原始值；但是对`u16`和`u32`来说，从`u32`转换为`u16`便不是无损的，因此也不该实现`u16: From<u32>`。
- 转换应当是**保值**的：将`i8`转换为`u8`是无损的——被转换为255的-1_i8可以被毫不费力地转换回255，但是我们不能因此就允许`u8: From<i8>`的存在——毕竟-1和255是截然不同的两个数字，转换过程不保值。又比如说，`String: From<u32>`是不存在的，因为身为`1`的数字和身为`"1"`的文本差别过大；而`String: From<char>`便是可以接受的，因为`'1'`和`"1"`都是文本。
- 转换应当是**显而易见**的：转换应当是两种类型之间唯一合理的选择。例如，从`[u8;4]`转换成`u32`的过程可以有多种选择：使用小字序、大字序和本地字序，所以应当分别为每种字节序实现不同的转换方法，而不是实现`u32: From<[u8;4]>`。



## TryFrom和TryInto

`TryFrom`和`TryInto`的功能和上文中介绍过的`From`/`Into`相同，但是它们可能会受控地失败：

```rust
pub trait TryFrom<T>: Sized {
    type Error;

    // Required method
    fn try_from(value: T) -> Result<Self, Self::Error>;
}

pub trait TryInto<T>: Sized {
    type Error;

    // Required method
    fn try_into(self) -> Result<T, Self::Error>;
}
```

### 通用实现

实现`U: TryFrom<T>`会自动实现`T: TryInto<U>`；`TryFrom`也是反身的，任何类型`T`都自动实现了`TryFrom<T>`。

> 小插曲：`T: TryFrom<T>`是永远不会失败的，它的返回类型是`Result<Self, Infallible>`。正如`Infallible`的名字所言，它用于表示永远不会发生的错误。`Infallible`将来会被`!`替代。



---

这篇文章不算很长，简单地梳理了`From`、`Into`、`TryFrom`和`TryInto`四个用于“值到值”的转换的特征。下篇文章将会介绍`Display`和`ToString`特征。

