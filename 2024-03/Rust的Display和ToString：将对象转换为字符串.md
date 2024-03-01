---
title: Rust的Display和ToString：将对象转换为字符串   
zhihu-url: https://zhuanlan.zhihu.com/p/684722496
---
在写代码的时候，我们经常需要将对象输出到屏幕上，或者转换为字符串；在Python中，我们可以为类型定义魔法函数`__str__`，`print`和`str()`都会调用它；在C++中，我们可以为对象重载`ostream& operator<<(ostream& os)`函数，使用`ostringstream`、`fstream`和`cout`的时候会调用它。

在Rust中该实现什么，想必大家都能回答上来：`Display`。不过，你可能不知道，`Display`的责任并不仅仅局限于“显示”，还包括对象到字符串的转换；最有说服力的论据就是实现了`Display`之后，就会自动获得`ToString`的实现，这一点我们稍后会详细说明。



## Display

`Display`特征的签名比之前我们介绍过的特征要更复杂：

```rust
pub trait Display {
    // Required method
    fn fmt(&self, f: &mut Formatter<'_>) -> Result<(), Error>;
}
```

`Formatter`是一个用于获取`Display`上下文信息的结构体，并且我们的输出结果也应该写到这个`Formatter`中；另外，虽然这个方法返回的是一个`Result`，但这不等于转换过程可以失败。Rust文档中这样描述这个“Result”：

> 与函数签名所暗示的相反，字符串格式化是一个不会失败的操作。这个函数只是因为写入底层流可能会失败，并且它必须提供一种方式来传递错误发生的事实回到调用栈上，所以才会返回一个`Result`。

这是Rust文档上的一个案例：

```rust
use std::fmt;

#[derive(Debug)]
struct Vector2D {
    x: isize,
    y: isize,
}

impl fmt::Display for Vector2D {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
```



### Formatter

前面提到，`Formatter`充当了Rust和程序员之间的桥梁：我们通过`Formatter`来了解用户传入了哪些显示参数（例如对齐方向、填充字符、有/无符号等），并将生成好的结果写入`Formatter`内。除了`Display`，`fmt`模块中的其他用于格式化的特性也会使用`Formatter`。

以下是`Formatter`中用于获取用户传入参数的一些方法：

```rust
// 获取用户请求的对齐方式
fn align(&self) -> Option<Alignment>;

// 获取用户请求的输出宽度
fn width(&self) -> Option<usize>;

// 获取填充字符
fn fill(&self) -> char;

// 获取用户请求的输出精度
fn precision(&self) -> Option<usize>;

// 获取是否提供了“+”标志
fn sign_plus(&self) -> bool;

// 获取是否提供了“-”标志
fn sign_minus(&self) -> bool;

// 获取是否提供了“#”标志
fn alternate(&self) -> bool;
```

`Formatter`也提供了一些用于输出的方法：

```rust
fn write_fmt(&mut self, fmt: Arguments<'_>) -> Result<(), Error>;

fn write_str(&mut self, data: &str) -> Result<(), Error>;
```

**`Formatter`实现了`Write`特征**。这也就是说我们可以直接使用`write!()`这样的宏来向`Formatter`写入内容，就像上方的案例一样。



### 应该实现`Display`吗？

我们知道，除`Display`外Rust中还有一个类似的特征：`Debug`。`Debug`特征大伙都很熟悉，那什么时候该实现`Display`，什么时候该实现`Debug`呢？

- `Display`要求类型在任何时候都能够被可靠地表示为字符串；它并没有被硬性要求。
- `Debug`是**所有公共类型**都应当实现的，并且一般来说不需要手动实现，而是应当使用`#[derive(Debug)]`。`Debug`的输出通常会尽可能可靠地展示类型的内部状态。



## ToString

`ToString`特征来自`std::string`模块，用于将一个值转换为`String`：

```rust
pub trait ToString {
    // Required method
    fn to_string(&self) -> String;
}
```

`ToString`一眼望去和`Display`风马牛不相及，但是它却有一个重要的特点：只要类型实现了`Display`，那它就自动实现了`ToString`。

在实践中，如果我们需要把数字转换为字符串的话，可以直接使用`to_string()`来便捷转换。

> 如果性能需求特别高的话，则应该考虑使用三方库`itoa`或`ryu`。

但是，如果需要将`&str`转换为`String`的话，就应该用`to_owned()`而不是`to_string()`了；`to_string()`中构造`Formatter`的过程会造成性能浪费。这里我要批评一款Rust IDE，它总是为`String`类型的变量提供`"".to_string()`的默认值，在我年少懵懂的时候带来过不小的误导：

![](https://s.c.accr.cc/picgo/1709262256-3421ac.png)

---

总结：这篇文章介绍了`Display`和`ToString`，虽然没有提及其他`std::fmt`中的格式化特性，但它们和`Display`是基本相通的，除了功能并无太大区别；此外，我也没有介绍`format!()`之类的宏的用法，因为那是Rust新手教程该做的事。下篇文章将会聚焦`std::cell`中的几个类型：`Cell`、`RefCell`和`OnceCell`。
