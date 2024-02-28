---
title: Rust的ToOwned特征：泛型版的Clone
zhihu-url: https://zhuanlan.zhihu.com/p/684345334
---
`std::borrow::ToOwned`是Rust标准库中的一个特征，用于从借用的数据中创建一个具有所有权的副本。它的作用和`Clone`是一样的，但是相比`Clone`，它支持泛型；也就是说我们可以将一个类型`T`“Clone”为另一个类型`U`。这对处理一些特殊的类型来说很有用。



## ToOwned的签名

`ToOwned`提供了两个方法，其中一个是必须实现的：

```rust
pub trait ToOwned {
    type Owned: Borrow<Self>;

    // Required method
    fn to_owned(&self) -> Self::Owned;

    // Provided method
    fn clone_into(&self, target: &mut Self::Owned) { ... }
}
```



## 例子

说到例子，就不得不再次请出我文章里的常客——`str`和`[T]`。众所周知，因为`str`和`[T]`是切片类型，它们在编译时大小未知，因此Rust无法在栈上为它们分配空间；这就导致了我们几乎不会直接持有`str`和`[T]`变量：

```rust
fn main() {
    let foo: str;
    let bar: [i32];
}
```

这段代码会被编译器直接报错：

![](https://s.c.accr.cc/picgo/1709094764-dff322.png)

`str`和`[T]`是不能被持有了，但是Clone它们的需求是确实存在的；因此，Rust就提供了将这些切片类型“Clone”为栈上类型的方案：`ToOwned`。打开标准库文档和源码，我们可以看到`str`这样实现了`ToOwned`：

```rust
#[stable(feature = "rust1", since = "1.0.0")]
impl ToOwned for str {
    type Owned = String;
    #[inline]
    fn to_owned(&self) -> String {
        unsafe { String::from_utf8_unchecked(self.as_bytes().to_owned()) }
    }

    fn clone_into(&self, target: &mut String) {
        let mut b = mem::take(target).into_bytes();
        self.as_bytes().clone_into(&mut b);
        *target = unsafe { String::from_utf8_unchecked(b) }
    }
}
```

也可以看到`[T]`这样实现了`ToOwned`：

```rust
#[stable(feature = "rust1", since = "1.0.0")]
impl<T: Clone> ToOwned for [T] {
    type Owned = Vec<T>;
    #[cfg(not(test))]
    fn to_owned(&self) -> Vec<T> {
        self.to_vec()
    }

    #[cfg(test)]
    fn to_owned(&self) -> Vec<T> {
        hack::to_vec(self, Global)
    }

    fn clone_into(&self, target: &mut Vec<T>) {
        SpecCloneIntoVec::clone_into(self, target);
    }
}
```

于是，用户虽然不能直接Clone`str`和`[T]`，但却可以把它们用`ToOwned`“Clone”为`String`和`Vec<T>`。

---

小插曲：在日常开发中，我们经常使用的方法`String::from(&str)`，其实现正是依赖于`str`的`to_owned`特征：

```rust
#[stable(feature = "rust1", since = "1.0.0")]
impl From<&str> for String {
    #[inline]
    fn from(s: &str) -> String {
        s.to_owned()
    }
}
```



## 总结

对普通的开发者来说，对`ToOwned`有一些了解就已经完全足够了；而对API的设计者来说，巧用`ToOwned`可以让你的类型在一些特殊情况下更灵活、更高效。

---

下篇文章介绍的是Rust中经常被人忽视的智能指针`Cow`，我之前在学习Rust时就注意到无论是the book，还是Rust圣经，都没有讲到`Cow`，希望我的工作能为Rust增加一点可供新人参考的资料。
