---
title: Rust的Borrow和AsRef：让你的代码用起来像呼吸一样自然
zhihu-url: https://zhuanlan.zhihu.com/p/684078465
---
经常写Rust的朋友在日常开发中都能或多或少地见到`Borrow`和`AsRef`这两个trait，他们的出现总是和泛型编程相伴，例如`HashMap`的`get`方法接收的参数便是一个被`K`（键类型）实现了`Borrow`的类型：

![](https://s.c.accr.cc/picgo/1708950921-47d94d.png)

又或者`File`的`open`方法接收的参数是一个实现了`AsRef<Path>`的类型：

![](https://s.c.accr.cc/picgo/1708952063-2989fc.png)

实际上，正是因为这两个trait的存在，我们的Rust编程体验才如我们所体验到的一样灵活、自然，甚至在Rust不允许重载的前提下实现了类似重载的效果：

```rust
use std::{
    fs::File,
    path::{Path, PathBuf},
};

fn main() {
    let str_ = "./hello.txt";
    let string = String::from(str_);
    let path = Path::new(str_);
    let path_buf = PathBuf::from(".").join("hello.txt");

    File::open(str_).unwrap();		// 传入 &str
    File::open(string).unwrap();	// 传入 String
    File::open(path).unwrap();		// 传入 Path
    File::open(path_buf).unwrap();	// 传入 PathBuf
}

```

而如果读者计划开发一些支持泛型的类库的话，`Borrow`和`AsRef`更是能够让你的库变得更好用、更自然。接下来，我就来逐一介绍一下`Borrow`和`AsRef`两大神器吧。



## Borrow：强调逻辑一致的借用

`Borrow`的作用就像它的名字一样：借用。它需要实现者提供一个`borrow`方法的实现：

```rust
pub trait Borrow<Borrowed>
where
    Borrowed: ?Sized,
{
    // Required method
    fn borrow(&self) -> &Borrowed;
}
```

它为所有类型提供了全覆盖实现（Blanket Implementations），这意味着你能够在Rust中见到的所有类型都拥有一个可以用来借用自己的`borrow()`方法：

```rust
use std::borrow::Borrow;

struct Test;

fn main() {
    let t1 = Test {};
    t1.borrow();
}

```

作为一个trait，其全覆盖实现可以被重新实现，这一点提高了Rust中类型的灵活性。毕竟，当我们借用`Box<T>`时，我们实际上期望通过借用获得的是`&T`，而不是`&Box<T>`；同样，当我们借用`String`时，我们通常希望得到的是`&str`，而不是`&String`。

在日常编程中，我们几乎不会用到`Borrow`；毕竟如果我们想要某个类型的引用，我们可以直接用`&`；如果我们想从`String`和各种智能指针中获得内部类型的引用，我们也可以用`as_str`、`as_ref`之类的方法，而不是`borrow`；此外，`Borrow`也不位于`std::preclude`内，使用需要额外`use std::borrow::Borrow`。那么，它的存在还有什么意义呢？

答案是泛型编程。`Borrow`在泛型编程中有着不可替代的意义，它要求顶层类型和底层类型的**行为逻辑相同**，这也就是说，如果 `x==y`，那么`x.borrow()==y.borrow()`也一定要成立。具体而言，对借用值和拥有值而言，`Eq`、`Ord`和`Hash`的行为必须一致。这种借用值和拥有值的**等价性**在某些情况下很有意义，例如在上文中提到的`HashMap::get`中，我们当然希望借用值的Hash和拥有值的Hash是一致的，不然就会出现`insert`到map中的键不能被`get`到的情况。

标准库的文档对`Borrow`的等价性的应用举了一个例子：假如我们封装了一个`CaseInsensitiveString`的类型，它的特点是在做比较和Hash时对大小写不敏感，就像这样：

```rust
pub struct CaseInsensitiveString(String);

impl PartialEq for CaseInsensitiveString {
    fn eq(&self, other: &Self) -> bool {
        self.0.eq_ignore_ascii_case(&other.0)
    }
}

impl Eq for CaseInsensitiveString { }

impl Hash for CaseInsensitiveString {
    fn hash<H: Hasher>(&self, state: &mut H) {
        for c in self.0.as_bytes() {
            c.to_ascii_lowercase().hash(state)
        }
    }
}
```

我们可以为这个`CaseInsensitiveString`实现`Borrow<str>`吗？虽然实现`Borrow<str>`对它来说轻而易举，但是因为它的行为和`str`不同，因此我们无论如何也不能为它实现`Borrow<str>`。如果我们希望它用户能访问底层的`str`的话，我们应当为它实现没有等价性要求的`AsRef<str>`，而不是`Borrow<str>`。

回到文章一开始举出的例子：`HashMap::get`，`K: Borrow<Q>`保障了我们传入的类型不仅可以和map的键的类型（也就是K）在`borrow`之后得到相同的结果，而且`borrow`之后的行为和原类型的行为是一致的；并且，我们也可以更加灵活地使用`HashMap::get`：

```rust
use std::collections::HashMap;

fn main() {
    let mut map: HashMap<String, i32> = HashMap::new();
    map.insert("Test".into(), 0);
    map.get(&String::from("Test"));     // 用&String作为参数，但额外构造一个Test是否有点过于繁琐和浪费？
    map.get("Test");                    // 用&str做参数：既简单又清晰
}

```

因为`HashMap::get`接收的是`Borrow`泛型，因此我们不仅可以传入`&K`，也就是`&String`，更可以传入开销更低、也更方便更自然的`&str`了！并且，因为`Borrow`泛型自带的等价性要求，我们也不用担心`&String`和`&str`做比较的时候会不会行为不同~



## AsRef：强调转换结果的引用

`AsRef`，也正如其名，强调从引用到引用的低成本转换。它需要实现者提供一个`as_ref`方法的实现：

```rust
pub trait AsRef<T>
where
    T: ?Sized,
{
    // Required method
    fn as_ref(&self) -> &T;
}
```

Rust建议把`AsRef`仅仅用在廉价的、确定的转换上。如果转换的代价高昂，或者可能会产生错误，那么最好使用`From`trait或编写一个返回`Result<T,E>`的自定义函数。

`AsRef` trait使得对于类型`Foo`的变量`foo`，不论是`foo`、`&foo`、`&mut foo`还是`&&mut foo`，调用`foo.as_ref()`都可以获得相同类型的引用结果，因为Rust在调用方法时会根据需要自动应用解引用或引用。这意味着，即使是对引用的引用，`AsRef`方法调用也能够正确地工作，返回一个对应类型的引用：

```rust
fn main() {
    let mut foo = Box::new(5i32);  // Box<i32>实现了AsRef<i32>
    
    println!("{}", foo.as_ref()); 			// 5
    println!("{}", (&foo).as_ref());		// 5
    println!("{}", (&mut foo).as_ref());	// 5
    println!("{}", (&&mut foo).as_ref());	// 5
}

```

![](https://s.c.accr.cc/picgo/1708962877-b4bf61.png)

`AsRef`还有一个理想的属性——反身性。在反身性的要求中，对于任何类型`T`，都存在一个`AsRef<T>`的实现，使得`T`可以引用自己，类似于上文中讨论过的`Borrow`。不过，因为自动解引用功能已经为`AsRef`提供了一个全覆盖实现（`impl<T: AsRef<U>> AsRef<U> for &T`），Rust当前暂时不能提供`AsRef`的反身性的全覆盖实现，因为两个全覆盖实现可能会产生重叠，导致编译器不能选择出应该使用哪个实现。这个问题据说会在Rust的后续版本中尝试解决。因此，如果你需要这样的反身性，你应当自己为类型`T`手动实现`AsRef<T>`。

应用：和`Borrow`一样，`AsRef`在泛型编程中有非常广泛的用途，例如前面提到的`File::open`，它接受一个实现了`AsRef<Path>`的类型作为参数，而实现了`AsRef<Path>`的类型就很多了：

```rust
impl AsRef<Path> for Path {...}

impl AsRef<Path> for OsStr {...}

impl AsRef<Path> for Cow<'_, OsStr> {...}

impl AsRef<Path> for OsString {...}

impl AsRef<Path> for str {...}

impl AsRef<Path> for String {...}

impl AsRef<Path> for PathBuf {...}
```

这些类型的变量，全都可以被传入这个`File::open`中作为参数，从另一个角度实现了类似重载的便利效果：

```rust
use std::{
    fs::File,
    path::{Path, PathBuf},
};

fn main() {
    let str = "./hello.txt";
    let string = String::from(str);
    let path = Path::new(str);
    let path_buf = PathBuf::from(".").join("hello.txt");

    File::open(str).unwrap();		// 传入 &str
    File::open(string).unwrap();	// 传入 String
    File::open(path).unwrap();		// 传入 Path
    File::open(path_buf).unwrap();	// 传入 PathBuf
}
```



总结全文，对于类库的开发者来说，掌握并灵活运用`Borrow`和`AsRef`是非常有必要的，它们既可以保障类似`String`、`Box<T>`等类型的可以被自然地传入和处理，更能在最大的限度内为用户提供便利，让你的代码用起来像呼吸一样自然——毕竟用户肯定不愿意在明明可以直接`&foo`的时候写`&foo.into()`吧~

