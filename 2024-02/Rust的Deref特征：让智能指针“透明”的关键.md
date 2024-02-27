---
title: Rust的Deref特征：让智能指针“透明”的关键
zhihu-url: https://zhuanlan.zhihu.com/p/684274628
---
除了上篇文章中介绍过的`Borrow` 和`AsRef`外，Rust中还有一个很常见的和引用相关的特征：`Deref`。不过，和`Borrow`、`AsRef`两个特征不同，`Deref`其实是用于重载解引用运算符（也就是`*`）的特征；在为某个类实现了`Deref`特征后，对它使用`*`运算就会调用特征中重载的方法。

这篇文章不仅将介绍`Deref`特性，还将探讨Rust中一个极其重要的机制：Deref转换（Deref Coercion）。对于各领域的开发者来说，理解这一机制都大有裨益。



## Deref：定义

`Deref`特征被用于解引用操作。实现了`Deref`或`DerefMut`的类型被称为**智能指针**。通常，智能指针类型被用于改变所含值的所有权语义（如`Rc`和`Cow`）或所含值的存储语义（如`Box`）。

`Deref`的定义很简单，只需要提供一个方法：

```rust
pub trait Deref {
    type Target: ?Sized;

    // Required method
    fn deref(&self) -> &Self::Target;
}
```



## Deref 转换

如果`Deref`只有这点东西的话，看起来和`Add`和`Neg`这样的纯运算符trait也没什么区别；但是，`Deref`特殊就特殊在Rust的一种机制：Deref转换。具体来说，**Rust编译器会在许多时候隐式地使用Deref**。看看下面的例子吧：

```rust
fn main() {
    let foo = Box::new(5i32);
    let bar: &i32 = &foo;
    println!("{}", bar);
}
```

请你花五秒钟阅读这段代码，然后告诉我它能不能正常编译？如果能，会输出什么？

![](https://s.c.accr.cc/picgo/1709045474-16ec90.png)

答案是5，也就是被`Box`所包裹的值。这种情况看起来显然是不符合语法规则的：我们怎么能把一个类型为`&Box<i32>`的变量赋给`&i32`呢？其实，这就是Deref转换在悄悄发挥作用。回到我们的代码：

```rust
let bar: &i32 = &foo;
```

编译器注意到foo的类型`Box<i32>`和`i32`不符合，但是`Box<i32>`实现了`Deref<i32>`；于是它尝试在foo上插入了`Deref`：

```rust
let bar: &i32 = &(*(foo.deref()));
```

foo被执行了一次解引用后，类型由`Box<i32>`变为了`i32`，和bar的要求符合，于是编译没有失败，bar获得了foo中的值的引用。



回到Deref转换上来，根据Rust官方文档中的定义，如果一个类型`T`实现了`Deref<Target = U>`，那么对类型为`T`的变量`v`来说：

- 在不可变的上下文中，`*v`相当于`*Deref::deref(&v)`。
- 类型为`&T`的值会被转换为`&U`。
- `T`隐式地实现了`U`中的所有方法（以`&self`为接收者）。



不管读者之前有没有意识到，其实我们已经无数次享受过`Deref`转换带来的便利了；例如`split`方法实现于`str`而不是`String`，但是我们仍然可以对`String`使用`split`：

```rust
fn main() {
    let foo = String::from("Hello world");
    foo.split(" ").for_each(|s| println!("{s}"));
}
```

又或者`first`方法实现于`[T]`而不是`Vec<T>`，但我们仍然可以对`Vec`使用`first`：

```rust
fn main() {
    let foo = vec![10, 20, 30];
    println!("{:?}", foo.first());
}
```

或者是最明显的：当我们在使用`Mutex<T>`的时候，调用`lock`方法之后返回的明明是一个类型为`MutexGuard<T>`的变量，我们却可以像使用`T`本身一样使用它：

```rust
use std::sync::{Mutex, MutexGuard};

fn main() {
    let foo = Mutex::new("hello world");
    let foo_guard: MutexGuard<&str> = foo.lock().unwrap();
    foo_guard.split(" ").for_each(|s| println!("{s}"));
}
```



除此以外，Deref转换也会连续进行，直到无法再继续`Deref`或匹配到正确的类型：

```rust
fn main() {
    let foo = Box::pin(String::from("Hello world"));  // foo: Pin<Box<String>>
    foo.split(" ").for_each(|s| println!("{s}"));
}
```

这段代码中，foo按照`Pin<Box<String>> -> Box<String> -> String -> str`的顺序连续进行，直到拥有`split`方法的`str`类型为止。



感谢Deref转换的存在，我们不需要写这样的代码：

```rust
fn main() {
    let foo = String::from("Hello world");
    &(*foo)[..].split(" ").for_each(|s| println!("{s}"));
}
```

个人认为，Deref转换这个机制的存在，使得Rust在保障安全性的同时，将语言的易用程度提高到了一个全新的高度，是Rust语言中我个人最喜欢的一颗语法糖。



## 实现Deref时需要注意的

`Deref`也不是万能妙具，不能也不该被随意滥用。Rust文档对实现`Deref`的行为提出了这样的警告：

> **Warning:** Deref coercion is a powerful language feature which has far-reaching implications for every type that implements `Deref`. The compiler will silently insert calls to `Deref::deref`. For this reason, one should be careful about implementing `Deref` and only do so when deref coercion is desirable. See [below](https://doc.rust-lang.org/std/ops/trait.Deref.html#when-to-implement-deref-or-derefmut) for advice on when this is typically desirable or undesirable.
>
>
> **警告**：Deref 转换是一种强大的语言功能，对每个实现了 Deref 的类型都会造成深远的影响。编译器会默默插入对 Deref::deref 的调用。因此，在实现 Deref 时应小心谨慎，只有在需要 Deref 转换时才应该使用。请参阅下文，了解什么情况下需要或不需要使用 Deref。

文档也给出了这样的准则：何时可以实现`Deref`，何时不可以实现。

>一般来说，如果出现以下情况，就应该实现`Deref`特性：
>
>1. 该类型的值与目标类型的值行为透明；
>2. 实现 deref 函数的成本较低；并且
>3. 该类型的用户不会对任何 deref 转换行为感到惊讶。
>
>一般来说，如果出现以下情况，就不应该实现`Deref`特性：
>
>1. deref 实现可能意外失败；或
>2. 类型的某方法和目标类型的不一致；或
>3. 不希望将 deref 转换作为公共 API 的一部分。

`AsRef`和`Borrow`的签名和`Deref`也很相似，在大多数情况下也需要同时实现它们中的一个或两个。

此外，`Deref`的实现在任何情况下都不应该失败，因为Deref转换的机制会使得这样的失败难以排查和定位。



最后，在介绍完`Deref`和Deref转换之后，也推荐大家看一下经典著作中对`Deref`的介绍；我对Rust了解不是很深入，对相关概念的介绍想必也肯定不如这些老师，因此建议大家还是延伸阅读一下：

1. [Deref 解引用 - Rust语言圣经(Rust Course)](https://course.rs/advance/smart-pointer/deref.html)
2. [Treating Smart Pointers Like Regular References with the Deref Trait - The Rust Programming Language](https://doc.rust-lang.org/book/ch15-02-deref.html)

