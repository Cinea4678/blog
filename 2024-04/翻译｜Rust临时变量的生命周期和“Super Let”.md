---
title: 翻译｜Rust临时变量的生命周期和“super let”
zhihu-tags: rust
zhihu-url: https://zhuanlan.zhihu.com/p/693766112
---

> 本文原作者：[Mara Bos](https://blog.m-ou.se/format-args/)，原文链接：[https://blog.m-ou.se/super-let/](https://blog.m-ou.se/super-let/)

Rust临时变量的生命周期是一个复杂但经常被忽略的话题。在简单情况下，Rust将临时变量存在的时间控制得恰到好处，使我们不必过多考虑它们。然而，也有很多我们可能不会马上完全得到我们想要结果的情况。

在这篇文章中，我们会（重新）发现Rust对临时变量生命周期的规则，审视一些*延长临时变量生命周期*的案例，并探索一个新的语言概念——`super let`——来让我们更好地控制这一切。

## 临时变量

这是一条没有上下文的Rust语句，使用了一个临时的`String`：

```rust
f(&String::from('🦀'));
```

这个临时的`String`会生存多久呢？如果我们正在设计Rust，我们基本上可以在这两个选项中做选择：

1. 这条字符串在调用`f`前马上就被丢弃了，或者，
2. 这条字符串仅在调用`f`后才被丢弃。

如果我们选择选项1，那么上面的语句就会总是导致一个借用检查错误，因为我们不能让`f`借用已经不复存在的变量。

所以，Rust选择了选项2：这个`String`首先被构造，然后一个对它的引用被传递给`f`，并且，仅仅在`f`返回之后，我们才丢弃这个临时的`String`。

### 在let语句中

现在看看这个稍微难一点的：

```rust
let a = f(&String::from('🦀'));
…
g(&a);
```

再来一次：这个临时的`String`会生存多久？

1. 这条字符串在`let`语句结束后被丢弃：也就是`f`返回之后，`g`调用之前。或者，
2. 这条字符串在调用`g`之后，和`a`同时被丢弃。

这一次，选项1可能会正常工作，这取决于`f`的签名。如果`f`被定义为`fn f(s: &str) -> usize`（就像`str::len`），那么在`let`语句后马上丢弃`String`就是完全OK的。

不过，如果`f`被定义为`fn f(s: &str) -> &[u8]`（就像`str::as_bytes`），那么`a`就会借用这个临时的`String`，因此当我们继续持有`a`时就会收到一个借用检查的错误。

对选项2来说，它在上述两种情况下都能正常编译，但是我们可能会将临时变量持有一段比实际需要更长得多的时间。这可能会浪费资源，或者产生不明显的bug（例如，一个因为`MutexGuard`被丢弃得比预期更晚而导致的死锁）。

这听起来像是我们需要第三种选择：让这一切由`f`的签名来决定。

不过，Rust的借用检查器只进行*检查*；它不会影响代码的行为。这是一个重要而有用的特性，并有很多原因。作为例子，一个从`fn f(s: &str) -> &[u8]`（返回值借用参数）到`fn f(s: &str) -> &'static [u8]`（返回值不借用参数）的改变在调用位置不会改变任何事情，例如临时变量被丢弃的时间点。

所以，在唯二的两个选项中，Rust选择了选项1：在`let`语句结束时立即丢弃临时变量。如果需要`String`存在更长时间，可以很容易地将`String`移到一个独立的`let`语句内。

```rust
let s = String::from('🦀'); // 移动到了它自己的`let`内，以给它更长的生命周期
let a = f(&s);
…
g(&a);
```

### 在嵌套调用中

OK，另一个案例：

```rust
g(f(&String::from('🦀')));
```

还是有两个选项：

1. 这个字符串在`f`调用后、`g`调用前被丢弃，或者，
2. 这个字符串在整个语句结束，也就是`g`调用后被丢弃。

这个代码片段和之前几乎相同：一个临时`String`的引用被传递给`f`，然后它的返回值被传递给`g`。不过这一次，通过嵌套调用表达式，一切都被放到一个语句里了。

上面的推论依然是成立的：根据`f`的签名，选项1可能会也可能不会正常工作；选项2可能会比实际需要持有临时变量更长的时间。

不过，这一次，选项1可能会得到让程序员更加意外的结果。举个例子，哪怕是简单如`String::from('🦀').as_bytes().contains(&0x80)`的代码也不能通过编译，因为`String`在`as_bytes`（`f`）后、`contains`（`g`）前就会被丢弃。

同样可以争论的是，让这些临时变量稍微存在得更久一点并没有太大的坏处，因为在语句结束后它们还是会被丢弃。

所以，Rust选择了选项2：不考虑`f`的签名，`String`被保留到了语句结束，直到`g`被调用才被丢弃。

### 在`if`语句中

现在让我们移步一个简单的`if`语句：

```rust
if f(&String::from('🦀')) {
    …
}
```

同样的问题：`String`什么时候被丢弃？

1. 在`if`的条件被评估之后，`if`体被执行之前（也就是在`{`处）。或者，
2. 在`if`体之后（也就是在`}`处。

在这个案例里，没有理由在`if`体内保留临时变量存活。`if`的条件总是一个布尔值（仅`true`或`false`），根据定义它什么都不会借用。

所以，Rust选择了选项1。

这在使用`Mutex::lock`的例子里十分有用。`Mutex::lock`返回一个临时的`MutexGuard`变量，这个临时变量会在其被丢弃时解锁`Mutex`：

```rust
fn example(m: &Mutex<String>) {
    if m.lock().unwrap().is_empty() {
        println!("the string is empty!");
    }
}
```

在这里，来自`m.lock().unwrap()`的临时变量`MutexGuard`在`.is_empty()`后立即被丢弃，这使得`Mutex`不会在`println`期间被不必要地锁住。

### 在`if let`语句中

不过，对`if let`（和`match`）来说情况有所不同，因为此时我们的语句不需要被评估为布尔值：

```rust
if let … = f(&String::from('🦀')) {
    …
}
```

还是有两个选项：

1. 这条字符串在模式匹配后、`if let`体前（也就是在`{`处）被丢弃。或者，
2. 这条字符串在`if let`体后（也就是在`}`处）被丢弃。

这一次，有理由选择选项2而不是1。对`if let`或`match`分支里的模式来说，发生借用再正常不过了。

所以，在这种情况下，Rust选择了选项2。

举个例子，如果我们有一个`Mutex<Vec<T>>`类型的变量`vec`，这段代码是可以正常编译的：

```rust
if let Some(x) = vec.lock().unwrap().first() {
    // `Mutex`在这里仍然被锁着 :)
    // 这是有必要的，因为我们正在从`Vec`中借用`x`。（`x`是一个`&T`）
    println!("first item in vec: {x}");
}
```

我们从`m.lock().unwrap()`中获取了一个临时的`MutexGuard`，并使用`first()`方法来借用第一个元素。这个借用贯穿了整个`if let`体，因为`MutexGuard`直到最后的`}`才被丢弃。

不过，也会有非我们所愿的情况出现。举个例子，如果我们不使用返回引用的`first`，而是使用返回值的`pop`的话：

```rust
if let Some(x) = vec.lock().unwrap().pop() {
    // `Mutex`在这里仍然被锁着 :(
    // 这是不必要的，因为我们并没有从`Vec`中借用任何东西。（`x`是一个`T`）
    println!("popped item from the vec: {x}");
}
```

这会是[令人吃惊的](https://marabos.nl/atomics/basics.html#lifetime-of-mutexguard)，并引发不明显的bug或降低性能。

也许这是Rust做了错误选择的论据，又或者是未来版本的Rust做出改变的论据。关于这些规则可以被如何改变的想法，可以看看[Niko关于这个主题的博文](https://smallcultfollowing.com/babysteps/blog/2023/03/15/temporary-lifetimes/)。

就现在而言，变通的办法是用一个独立的`let`来将临时变量的生命周期限制到一条语句以内：

```rust
let x = vec.lock().unwrap().pop(); // MutexGuard在这条语句后就被丢弃了
if let Some(x) = x {
    …
}
```

## 临时变量生命周期延长

这种情况如何呢？

```rust
let a = &String::from('🦀');
…
f(&a);
```

两个选项：

1. 这条字符串在`let`语句结束后就被丢弃。或者，
2. 这条字符串和`a`被同时，也就是在`f`调用后丢弃。

选项1总是会导致一个借用检查的错误，所以选项2可能更正确一些。并且这就是Rust如今所做的：临时变量的生命周期被*扩展*了，以使得上面的代码片段可以正常编译。

临时变量生存得比它出现的语句更久了，这个现象叫作*临时变量生命周期延长*。

临时变量生命周期延长并不会应用到所有出现在`let`的语句中的临时变量上，正如我们已经见到的：`let a = f(&String::from('🦀'));`中的临时字符串并不会延伸到`let`语句之外。

在`let a = &f(&String::from('🦀'));`（注意多出来的`&`），临时变量生命周期延长确实应用到了最外层的`&`上，它借用了`f`返回的临时变量；但扩展没有应用到内层的、借用了临时`String`的`&`上。

举个例子，用`str::len`代替`f`：

```rust
let a: &usize = &String::from('a').len();
```

在这里，这个字符串在`let`语句结束后就被丢弃了，但来自`.len()`的`&usize`生存得和`a`一样久。

这并不局限于`let _ = &…;`语法。举例来说：

```rust
let a = Person {
    name: &String::from('🦀'), // 扩展了！
    address: &String::from('🦀'), // 扩展了！
};
```

在上面的这段代码中，临时字符串的生命周期被扩展了，因为哪怕我们对`Person`类型一无所知，我们也能确定为了继续使用这个对象，需要扩展它们的生命周期。

有关`let`语句中哪些临时变量的声明周期会被扩展的规则在[Rust的参考文档中有说明](https://doc.rust-lang.org/stable/reference/destructors.html#temporary-lifetime-extension)，但实际上可以归结为那些你从*语法上*就能看出来有必要延长生命周期的表达式，这与任何类型、函数签名或特质实现无关：

```rust
let a = &temporary().field; // 扩展了！
let a = MyStruct { field: &temporary() }; // 扩展了！
let a = &MyStruct { field: &temporary() }; // 都扩展了！
let a = [&temporary()]; // 扩展了！
let a = { …; &temporary() }; // 扩展了！

let a = f(&temporary()); // 没有扩展，因为可能没有必要 
let a = temporary().f(); // 没有扩展，因为可能没有必要 
let a = temporary() + temporary(); // 没有扩展，因为可能没有必要 
```

尽管这看起来很合理，但当我们考虑到构建元组结构或元组变量也是一个函数调用时，还是难免让人感到意外：`Some(123)`是，语法上的，一个对`Some`函数的调用。

举例来说：

```rust
let a = Some(&temporary()); // 没有扩展！（因为 `Some` 可以拥有任何函数签名……）
let a = Some { 0: &temporary() }; // 扩展了！（我赌你从来没有用过这种语法）
```

并且这确实非常令人困惑。:(

这也是值得考虑[重新修订规则](https://smallcultfollowing.com/babysteps/blog/2023/03/15/temporary-lifetimes/#design-principles)的原因之一。

---

**常量提升**

临时变量生命周期延长很容易和另一个称作*常量提升*的东西混淆起来，它是另一种让临时变量比预期生存得更久的另一种方式。

在类似`&123`和`&None`的表达式里，值被识别为一个常数（[不具备内部可变性，也没有析构函数](https://doc.rust-lang.org/stable/reference/destructors.html#constant-promotion)），并因此被自动提升为*永远*存活。这意味着这些引用将会拥有一个`'static`生命周期。

举例来说：

```rust
let x = f(&3); // 这里的&3是 'static 的，不论对 `f()` 来说是否有必要
···

这甚至应用到了简单的表达式上：

```rust
let x = f(&(1 + 2)); // 这里的&3是'static的
```

在临时变量生命周期延长和常量提升都可以应用的情况下，后者一般会被优先采用，因为它将生命周期扩展得更远一些：

```rust
let x = &1; // 常量提升，而不是临时变量生命周期延长
```

这就是说，在上面的代码片段中，`x`是一个`'static`的引用。数值`1`生存得甚至比`x`本身还要久。

---


### 代码块中的临时变量生命周期延长

假设我们有一种`Writer`类型，它持有了需要写入的`File`的引用：

```rust
pub struct Writer<'a> {
    pub file: &'a File
}
```

并且有一些代码，创建了一个写入新创建文件的`Writer`：

```rust
println!("opening file...");
let filename = "hello.txt";
let file = File::create(filename).unwrap();
let writer = Writer { file: &file };
```

现在，作用域中含有`filename`, `file`和`writer`。不过，后面的代码只应该通过`Writer`来进行写入。理想情况下，`filename`与（特别是）`file`在作用域内是不可见的。

因为临时变量生命周期延长对代码块的最终表达式也是有效的，我们可以像这样达成目标：

```rust
let writer = {
    println!("opening file...");
    let filename = "hello.txt";
    Writer { file: &File::create(filename).unwrap() }
};
```

现在，`Writer`的创建被整洁地包装在了它自己的作用域中，除了`writer`之外没有什么可以被外部作用域看到。得益于临时变量生命周期提升，被内部作用域创建为临时变量的`File`可以和`writer`生存得一样久。

### 临时变量生命周期延长的局限

现在假设我们将`Writer`的`file`字段改为了*私有*：

```rust
pub struct Writer<'a> {
    file: &'a File
}

impl<'a> Writer<'a> {
    pub fn new(file: &'a File) -> Self {
        Self { file }
    }
}
```

然后我们不需要过多修改原来的代码：

```rust
println!("opening file...");
let filename = "hello.txt";
let file = File::create(filename).unwrap();
let writer = Writer::new(&file); // 只有这行变动了
```

我们只需要调用`Writer::new()`而不是使用`Writer {}`语法来进行构造。

不过，使用了作用域的版本就行不通了：

```rust
let writer = {
    println!("opening file...");
    let filename = "hello.txt";
    Writer::new(&File::create(filename).unwrap()) // 错误：生存得不够久！
};

writer.something(); // 错误：在这里File已经没有生存了
```

正如我们之前见到的，尽管临时变量生命周期延长通过`Writer {}`构造语法传播，但它不会通过`Writer::new()`函数调用语法传播。（因为函数签名可以是`fn new(&File) -> Self<'static>`，也可以是`fn new(&File) -> i32`，这些例子不需要延长临时变量的生命周期。）

不幸的是，现在没有显式*指定*延长临时变量生命周期的方式。我们不得不在最外层作用域放置一个`let file`。我们现在能做到的最好，就是采用[延迟初始化](https://doc.rust-lang.org/nomicon/checked-uninit.html)：

```rust
let file;
let writer = {
    println!("opening file...");
    let filename = "hello.txt";
    file = File::create(filename).unwrap();
    Writer::new(&file)
};
```

但那又把`file`带回到作用域里了，这是我们刚刚还在试图避免的。:(

尽管把`let file`放在作用域外部是不是一个大问题还有待商榷，这种变通方式对大多数Rust程序员来说都不够清晰。延迟初始化并不是一项常用的特性，并且编译器目前在给出临时变量的生命周期错误时也不会推荐这种变通方式。

在一定程度上，如果能修复这个问题就好了。

### 宏

如果有一个既能创建文件，又能返回它的`Writer`的函数的话，可能会很有用。比如：

```rust
let writer = Writer::new_file("hello.txt");
```

但是，因为`Writer`只是借用`File`，这将要求`new_file`将`File`存储在某个地方。它可以将`File`[泄露](https://doc.rust-lang.org/stable/std/boxed/struct.Box.html#method.leak)出去或以某种方式将其保存在`static`中，但是（当前）它还是不能让`File`活得和返回的`Writer`一样久。

所以，不如让我们用宏来在调用的地方同时定义文件和writer：

```rust
macro_rules! let_writer_to_file {
    ($writer:ident, $filename:expr) => {
        let file = std::fs::File::create($filename).unwrap();
        let $writer = Writer::new(&file);
    };
}
```

使用起来大概像这样：

```rust
let_writer_to_file!(writer, "hello.txt");

writer.something();
```

得益于[宏卫生](https://veykril.github.io/tlborm/decl-macros/minutiae/hygiene.html)，`file`在这个作用域内是不可见的。

这样做已经可行了，但如果它看起来更像一个普通的函数调用，就像下面一样，不是更好吗？

```rust
let writer = writer_to_file!("hello.txt");

writer.something();
```

正如我们之前见到过的，在`let writer = ...;`语句*内部*创建活得足够久的临时变量`File`的方法，是使用临时变量生命周期延长：

```rust
macro_rules! writer_to_file {
    ($filename:expr) => {
        Writer { file: &File::create($filename).unwrap() }
    };
}

let writer = writer_to_file!("hello.txt");
```

这将会扩展为：

```rust
let writer = Writer { file: &File::create("hello.txt").unwrap() };
```

这段代码将会按需延长`File`临时变量的生命周期。

如果`file`字段不是公开的，我们就不能简单地这样做了，而是应当使用`Writer::new`。这个宏将会需要在调用它的`let writer = ...`*前*插入`let file;`。这是不可能做到的。

#### `format_args!()`

这个问题也是（当今的）`format_args!()`的结果不能被保存到一个`let`表达式中的原因：

```rust
let f = format_args!("{}", 1); // Error!
something.write_fmt(f);
```

原因是`format_args!()`扩展到了一些类似`fmt::Arguments::new(&Argument::display(&arg), …)`的代码，其中的部分参数是对临时变量的引用。

临时变量生命周期延长并不会对函数调用中的参数生效，因此`fmt::Arguments`对象的使用只能被限制在同一条语句中。

如果能修复这个问题那就太好了。

`pin!()`

另一个经常被通过宏来创建的类型是[`Pin`](https://doc.rust-lang.org/stable/std/pin/struct.Pin.html)。大致来说，它持有一个永远不会被移动的值的引用。（确切的详情很复杂，但不是非常和现在的主题相关。）

它被通过一个名为`Pin::new_unchecked`的`unsafe`函数创建，因为你需要保证哪怕`Pin`本身都不存在了，它引用的值也不会被移动。

使用这个函数的最好方式，是利用[遮蔽](https://doc.rust-lang.org/rust-by-example/variable_bindings/scope.html)机制：

```rust
let mut thing = Thing { … };
let thing = unsafe { Pin::new_unchecked(&mut thing) };
```

因为第二个`thing`遮蔽了第一个，第一个`thing`（仍然存在）就不能被按名访问了。既然它不再能被按名访问，我们就可以确认它不会被移动了（哪怕第二个`thing`被丢弃了也是），这正是我们向`unsafe`代码块所保证的。

因为这是一种常见模式，这种模式通常在宏中捕获。

举例来说，人们可能会这样定义一个`let_pin`宏：

```rust
macro_rules! let_pin {
    ($name:ident, $init:expr) => {
        let mut $name = $init;
        let $name = unsafe { Pin::new_unchecked(&mut $name) };
    };
}
```

使用方式看起来和之前我们的`let_writer_to_file`宏类似：

```rust
let_pin!(thing, Thing { … });

thing.something();
```

这是可以正常工作的，并且很好地压缩和隐藏了不安全的代码。

但是，就像我们之前的`Writer`例子一样，如果它能像下面这样工作的话难道不会好很多吗？

```rust
let thing = pin!(Thing { … });
```

我们早已知道，只有我们能利用临时变量延长机制让`Thing`生存得足够久，我们才有可能实现这个目标。并且这也仅仅在我们能用`Pin {}`语法构建`Pin`的时候才有可能做到：`Pin { pinned: &mut Thing { ... } }`可以使用临时变量生命周期延长，但`Pin::new_unchecked(&mut Thing { ... })`不行。

那甚至意味着要把`Pin`的字段公开，而这违背了`Pin`的设计意图。仅当字段私有时，它才能提供有意义的保证。

这就意味着，很不幸地，你（如今）还不可能自己写出这样的`pin!()`宏。

但标准库还是[这样干了](https://doc.rust-lang.org/stable/std/pin/macro.pin.html)，它犯下了可怕的罪行👿：`Pin`的“私有字段”其实是被定义为`pub`的，但也被标记成了“unstable”以使得在你尝试使用它时编译器能发出警告。

如果不用这样hack的话就太好了。

## super let

我们现在已经见过几个被临时变量生存周期延长的限制性规则约束的案例了：

- 我们让`let writer = { ... };`良好地保持作用域的失败尝试，
- 我们让`let writer = writer_to_file!(…);`工作的失败尝试，
- 对执行`let f = format_args!(…);`的无能为力，以及
- 为了让`pin!()`工作所做的糟糕hack。

如果我们能*显式选择*去延长变量的生命周期的话，上面的这些问题都能各自得到很棒的解决方案。

如果我们能发明一种特殊的`let`语句，让其中涉及到的变量生存得比常规的`let`语句更久一些，会怎么样呢？就像超能力一样（或者把变量定义在“super”作用域的`let`）？把它叫作`super let`怎么样？

在我的想象中，它会像这样工作：

```rust
let writer = {
    println!("opening file...");
    let filename = "hello.txt";
    super let file = File::create(filename).unwrap();
    Writer::new(&file)
};
```

`super`关键字将会让`file`的生命周期和`writer`一样长，和周围代码块产生的`Writer`一样长。

`super let`工作的具体规则还需要继续研究，但主要目标是它允许对临时变量生命周期延长的“解语法糖”：

- `let a = &temporary();`和`let a = { super let t = temporary(); &t };`应该是等价的。

这个特性使得在不使用任何hack的前提下定义`pin!()`宏成为可能：

```rust
macro_rules! pin {
    ($init:expr) => {
        {
            super let pinned = $init;
            unsafe { Pin::new_unchecked(&pinned) }
        }
    };
}

let thing = pin!(Thing { … });
```

类似地，这样的新设计也会允许`format_args!()`宏为其中的临时变量使用`super let`，以使得宏的结果可以被作为`let a = format_args!()`语句的一部分被保存。

### 用户体验和诊断信息

同时存在`let`和`super let`两种仅有细微语义差异的语法，听起来也许并不是很棒。它解决了一些问题，尤其是和宏相关的，但它真的值得在厘清`let`和`super let`的差异时给人带来的潜在困扰吗？

我觉得是的，只要我们确保编译器能够在可能时提出建议，让用户添加或删除`let`中的`super`。

想象一下：有人写了这样的代码：

```rust
let output: Option<&mut dyn Write> = if verbose {
    let mut file = std::fs::File::create("log")?;
    Some(&mut file)
} else {
    None
};
```

在今天，它会产生这样的错误：

```
error[E0597]: `file` does not live long enough
  --> src/main.rs:16:14
   |
14 |     let output: Option<&mut dyn Write> = if verbose {
   |         ------ borrow later stored here
15 |         let mut file = std::fs::File::create("log")?;
   |             -------- binding `file` declared here
16 |         Some(&mut file)
   |              ^^^^^^^^^ borrowed value does not live long enough
17 |     } else {
   |     - `file` dropped here while still borrowed
```

尽管问题相对清晰，但这并没有实际地给出一个解决方案。我经常遇到带着类似例子来向我求助的Rust程序员，结果是我为他们解释延迟初始化的模式，并给出这样的解决方案：

```rust
let mut file;
let output: Option<&mut dyn Write> = if verbose {
    file = std::fs::File::create("log")?;
    Some(&mut file)
} else {
    None
};
```

对很多Rust程序员来说，这个解决方案不是非常清晰，也许是因为让`file`在一个分支下保持未初始化太奇怪了。

相反，如果错误信息变成这样的话，会不会感觉它变得更好了呢？

```
error[E0597]: `file` does not live long enough
  --> src/main.rs:16:14
   |
15 |         let mut file = std::fs::File::create("log")?;
   |             --------
   |
help: try using `super let`
   |
15 |         super let mut file = std::fs::File::create("log")?;
   |         +++++
```

即便对“super let”或它的语义了解不多，程序员们也能获得一个清晰和简单的、解决他们的问题的方案，并让他们学到`super`会让变量生存得更久一些。

类似的，当不必要地使用了`super let`，编译器应该建议删掉它：

```
warning: unnecessary use of `super let`
  --> src/main.rs:16:14
   |
15 |         super let mut file = std::fs::File::create("log")?;
   |         ^^^^^ help: remove this
   |
   = note: `file` would live long enough with a regular `let`
```

我相信这些诊断信息会让`super let`对全体Rust程序员有益，哪怕他们此前从未见过这个特性。

加上`pin`和`format_args`宏中人体工程学特性的增强，我认为`super let`在用户（程序员）的体验上取得了全胜。

---

**潜在的扩展**

让`super let`可以出现在函数作用域中是一项未来潜在的扩展。也就是说，此时的“super”指的是函数的调用者。

正如[@lorepozo@tech.lgbt在Mastodon上提到的](https://hachyderm.io/@lorepozo@tech.lgbt/111499621692587962)，那将会允许`pin!()`成为一个函数，而不是一个宏。类似的，它也会使得`Writer::new_file(…)`在不使用宏的前提下可以被实现。

让这得以有效工作的方式是，允许特定函数将对象放入调用者的栈帧中，然后稍后可以从返回值中引用这些对象。这在所有常规的旧函数中都不能运行；正常情况下，调用者不会给被调用的函数预留放入对象的空间。这需要成为函数签名的一部分。

也许像这样？

```rust
pub placing fn new_file(filename: &str) -> Writer {
    super let mut file = File::create(filename).unwrap(); // 放入了调用者的栈帧
    Writer::new(&file) // 所以我们就可以在返回值内借用它了！
}
```

这不是我现在提出的建议的一部分，但想象也很有趣。:)

---

## 临时变量生命周期2024的RFC

我和[Niko Matsakis](https://github.com/nikomatsakis)与[Ding Xiang Fei](https://github.com/dingxiangfei2009)在几个月前分享了我对`super let`的想法，他们对临时变量生命周期延长的“解语法糖”感到很激动。他们已经在努力确定`super let`的定义和具体的规则，以及下一版Rust的临时变量生命周期的一些新规则。

这个组合起来的[“临时变量生命周期2024”的努力](https://rust-lang.zulipchat.com/#narrow/stream/403629-t-lang.2Ftemporary-lifetimes-2024)正在为一项RFC作铺垫。该RFC基本上提议在可能的情况下减少临时变量生命周期，以防止在`if let`或`match`中由临时变量`MutexGuard`造成的死锁，并加入`super let`作为选择延长生命周期的一种方式。

