---
title: Rust的Cow类型有什么用？详解Cow及其用途   
zhihu-url: https://zhuanlan.zhihu.com/p/684402569
---
Rust的智能指针有哪些？大多数人都能马上答出`Box<T>`、`Rc<T>`和`Arc<T>`和在异步编程中很常见的`Pin<P>`等等。不过，有一个可能经常被大多数人遗忘的类型，它功能强大，利用好了可以节省很多复制开销；它就是这篇文章的主角：`Cow<B>`。



## 什么是COW（Copy-On-Write）？

在开始之前，有必要先介绍一下COW（Copy-On-Write，写时复制）的概念。COW是一种用于资源管理的优化策略，在操作系统中应用非常广泛。COW的核心思想是当多个任务需要读取同一个资源（比如内存中的数据、文件）的时候，它们会共享同一份资源副本，而不是为每个任务复制一份资源副本。只有当某个任务需要修改这个资源时，才会为这个任务创建一份资源副本。

需要注意的是，上述的整个过程对任务（也就是程序员编写的用户程序）来说都是不可见的；对程序员来说，他并不知道他所使用的资源在发生写操作时才被真正地复制了一份，自始至终他仿佛就像在独占整份资源一样。

COW在文件系统、虚拟内存管理中都有非常成熟的应用；在编程语言中，也被广泛应用于优化字符串、集合的处理。



## Cow：定义

Rust的`Cow<B>`是一个枚举类型，包含两个成员：`Borrowed`和`Owned`。不过，我们几乎不会直接用到它的成员，因为`Cow<B>`实现了`Deref`特征，这使得我们可以通过**Deref转换**这一语法糖来便捷地直接使用`Cow<B>`中的内容。有关Deref转换可以阅读我之前的文章。

```rust
pub enum Cow<'a, B>
where
    B: 'a + ToOwned + ?Sized,
{
    Borrowed(&'a B),
    Owned(<B as ToOwned>::Owned),
}
```

需要注意一下`Cow`的模板参数。`Cow`接受一个生命周期和一个类型`B`，其中类型`B`需要实现`ToOwned`特征；`ToOwned`特征的介绍可以看之前的文章，这里仅仅提一下所有实现了`Clone`的类型都会自动实现`ToOwned`自身。除此以外，成员`Owned`的内容类型不是类型`B`本身，而是类型`B`的`ToOwned`的目标类型（例如对`str`来说，这个类型是`String`）。



## 使用方法

这里是一段`Cow<B>`的简单使用范例：

```rust
use std::borrow::Cow;

fn main() {
    let foo = "Hello World";
    let mut bar: Cow<str> = Cow::from(foo);
    println!("{bar}");      // 这里没有发生复制
    
    bar.to_mut().push_str(" Rust");  // 这里发生了复制
    println!("{bar}");
    
    println!("{foo}");      // 原来的字符串foo仍然可用，而且没有变化
}

```

### Cow的构造

`Cow<B>`是一个枚举，所以首先它是可以直接从它的成员`Borrowed`和`Owned`来构造的：

```rust
use std::borrow::Cow;

fn main() {
    let str_ = "Hello World";
    let string = String::from("Hello World!");
    
    let foo: Cow<str> = Cow::Borrowed(str_);
    let bar: Cow<str> = Cow::Owned(string);
    
    // 这里string不再可用
    // println!("{string}");
}
```

除此以外，标准库中的五对实现了`ToOwned`的类型（`str`/`String`，`[T]`/`Vec<T>`，`CStr`/`CString`，`OsStr`/`OsString`，`Path`/`PathBuf`）也可以使用`From::from`来构造`Cow<B>`：

```rust
use std::borrow::Cow;

fn main() {
    let str_ = "Hello World";
    let string = String::from("Hello World!");
    
    let foo: Cow<str> = Cow::from(str_);	// from -> Borrowed
    let bar: Cow<str> = Cow::from(string);	// from -> Owned
    
    // 这里string不再可用
    // println!("{string}");
}
```

使用`From::from`时，Rust会自动为我们匹配正确的类型（`&'a str`/`String`等），一般情况下推荐使用`from`来构造`Cow`，而不是手动指定`Borrowed`/`Owned`。

### deref和to_mut

前面提到过，`Cow<B>`实现了`Deref<B>`特征，这意味着我们不需要做任何操作就可以享受Deref转换的语法糖：

```rust
use std::borrow::Cow;

fn main() {
    let str1 = "Hello World";
    let cow: Cow<str> = Cow::from(str1);
    let str2: &str = &cow;  // 注意看，我们把&Cow<str>赋给了&str
    
    println!("{str2}"); // Hello World
    println!("{cow}");  // Hello World
    println!("{str1}"); // Hello World
}
```

```rust
use std::borrow::Cow;

fn main() {
    let str1 = "Hello World";
    let cow: Cow<str> = Cow::from(str1);
    
    cow.split(" ").for_each(|s|println!("{s}"));	// 使用str的方法split也不在话下
}
```

不过，`Cow<B>`并没有实现`DerefMut`；这意味着我们对`Cow`的修改不会影响到底层的内容，相反地，当我们试图修改`Cow`时，`Cow`会生成一个副本，并且修改这个拥有所有权的副本：

```rust
use std::borrow::Cow;

fn main() {
    let str1 = "Hello";
    let mut cow: Cow<str> = Cow::from(str1);
    
    cow += " World";
    
    println!("cow = {cow}");	// cow = Hello World
    println!("str1 = {str1}");	// str1 = Hello 
}

```

我们可以多加一点输出代码，来看看具体发生了什么：

```rust
#![feature(cow_is_borrowed)]
use std::borrow::Cow;

fn main() {
    let str1 = "Hello";
    let mut cow: Cow<str> = Cow::from(str1);
    
    println!("cow = {cow}, borrowed = {}", cow.is_borrowed());	// cow = Hello, borrowed = true
    
    cow += " World";
    
    println!("cow = {cow}, borrowed = {}", cow.is_borrowed());	// cow = Hello World, borrowed = false
    println!("str1 = {str1}");									// str1 = Hello
}
```

修改了`cow`变量后，它不再处于借用状态，而是拥有了这段字符串的所有权——这也是它能够安全地修改这段字符串的关键。

---

除了直接对`Cow<str>`使用`str`中实现的方法来修改字符串之外，还可以使用`to_mut()`来获取`&String`来使用`String`中实现的方法来修改字符串：

```rust
use std::borrow::Cow;

fn main() {
    let str1 = "Hello";
    let mut cow: Cow<str> = Cow::from(str1);
    
    cow.to_mut().push_str(" World");
    
    println!("cow = {cow}");	// cow = Hello World
    println!("str1 = {str1}");	// str1 = Hello 
}
```

再重复一遍：使用`to_mut()`修改和直接修改`Cow<B>`的不同在于，`to_mut()`返回的是`&mut <B as ToOwned>::Owned`（例如`String`），可以使用`B`的`Owned`类型（例如`String`）中额外实现的方法（例如`String::push_str`）；修改`Cow<B>`的时候，只能使用`B`中实现的方法（例如上面的`+=`，也就是`str::add_assign`）。

### 消费Cow

在不再需要使用`Cow`，或者想要完整取得`Cow`中的对象的所有权的时候，我们可以使用`Cow::into_owned`方法来消费掉`Cow`。方法返回的是`B`的`Owned`类型（例如`String`）。

```rust
use std::borrow::Cow;

fn main() {
    let str1 = "Hello";
    let mut cow: Cow<str> = Cow::from(str1);
    
    cow.to_mut().push_str(" World");
    
    let owned: String = cow.into_owned();
    
    println!("{owned}");    // Hello World
    println!("{str1}");	    // Hello 
}
```

在消费掉`Cow`之后，`Cow`将不再可用，但它之前借用的原数据不受影响。



## 用途

说了这么多，`Cow`到底有什么用呢？少复制几次数据真的那么重要吗？让我们看看标准库中的`String::from_utf8_lossy`方法吧。

`String::from_utf8_lossy`是一个把一个字节切片（`&[u8]`）按照UTF-8转换成`&str`的方法，并且会用“�”字符来替换掉字节切片中UTF-8不支持的字符。举个例子：

```rust
// 不包含错误字节的情况
fn main() {
    let hello = vec![72, 69, 76, 76, 79];
    let hello = String::from_utf8_lossy(&hello);
    assert_eq!("HELLO", hello);
}
```

以及：

```rust
// 包含错误字节的情况
fn main() {
    let input = b"Hello \xF0\x90\x80World";
    let output = String::from_utf8_lossy(input);
    assert_eq!("Hello �World", output);
}
```

现在假设我们是Rust标准库API的设计师，我们要为`from_utf8_lossy`方法选择一个恰当的返回类型。

**返回`&str`可以吗？**

最直接的想法就是返回一个`&str`，就像这样：

```rust
fn from_utf8_lossy<'a>(v: &'a [u8]) -> &'a str {
   todo!()
}
```

   这种方案可以吗？仔细想想，当字节切片中有UTF-8中不支持的错误字符时，错误字符需要被替换成“�”；直接返回`&str`的话是做不了对字符串内容的修改的。

**返回`String`呢？**

顺着刚才的思路，因为我们可能需要修改字符串，所以我们就需要返回`&str`的栈上类型`String`，合情合理：

```rust
fn from_utf8_lossy(v: &[u8]) -> String {
   todo!()
}
```

   不过，另一个问题冒出来了：虽然返回`String`完美地解决了修改字符串之后会导致新字符串无处存放的问题，但是如果旧的字符串（字节切片）不需要修改的话，也需要被复制到`String`中，这无形中增加了很多不必要的消耗；而且，字节切片中有错误字符是概率很小的事件，为了小概率事件影响拖累大概率发生的正常情况的性能，这值得吗？

   这时，我一拍大腿：在需要修改时返回`String`，不需要修改时返回`&str`不就好了？

**返回`(Option<&str>, Option<String>)`（或者`Either<&str, String>`）**

这样，上面所描述的性能和功能矛盾就解决了：

```rust
fn from_utf8_lossy<'a>(v: &'a [u8]) -> (Option<&'a str>, Option<String>) {
   todo!()
}
```

   但这种解决方式也不是没问题的：太复杂了……而且需要用户判断返回的是`&str`还是`String`。不过，这个要么返回借用的`&str`、要么返回有所有权的`String`的东西，是不是感觉有点眼熟？

   这不就是`Cow<str>`嘛！

**最终方案：返回`Cow<str>`**

经过一番艰难而复杂的思考，我们最终得到了最恰当的结果：

```rust
fn from_utf8_lossy(v: &[u8]) -> Cow<'_, str> {
    todo!()
}
```

使用了`Cow<str>`之后，它不仅可以在需要修改字符串时克隆并返回新数据，更可以在绝大多数普通情况之下直接借用数据；更妙的是，它可以享受Deref转换的语法糖，可谓十分完美！



## 总结

`Cow`是Rust中非常有用的一个类型，虽然日常开发中几乎用不到它，但是某些性能敏感的场景下善用`Cow`说不定会有奇效喔~
