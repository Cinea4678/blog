---
title: 翻译｜Rust字符串格式化的幕后：format_args!() 
zhihu-tags: rust 
zhihu-url: https://zhuanlan.zhihu.com/p/693641844
---

> 本文原作者：[Mara Bos](https://blog.m-ou.se/format-args/)，原文链接：[https://blog.m-ou.se/format-args/](https://blog.m-ou.se/format-args/)

[`fmt::Arguments`](https://doc.rust-lang.org/stable/std/fmt/struct.Arguments.html)是Rust标准库中我最喜欢的类型之一。虽然它并不算特别惊艳，但它却是一个几乎所有Rust程序都在使用的优秀构件。这个类型，连同[`format_args!()`宏](https://doc.rust-lang.org/stable/std/macro.format_args.html)一起，在背后支撑了`print!()`、`format!()`、`log::info!()`和其他众多的文本格式化宏。这些宏既有来自于标准库的，也有来自于社区包的。

在这篇博客里，我们会学习它是怎样工作的、它现在是如何被实现的，以及它未来可能会如何改变。

> 译注：原文发布于2023年12月5日。

## `format_args!()`

当你写下类似这样的代码时：

```rust
print!("Hello {}!\n", name);
```

`print`宏稍后会展开为这样的代码：

```rust
std::io::_print(format_args!("Hello, {}!\n", name));
```

`_print`是一个将`fmt::Arguments`作为唯一参数的内部函数。`fmt::Arguments`则由内建的`format_args!()`宏产生，这是一个能够读懂[Rust的字符串格式化语法](https://doc.rust-lang.org/stable/std/fmt/index.html)（使用`{}`作为占位符等）的宏。生成的`fmt::Arguments`对象既代表*字符串模板*，即带有占位符的（解析后的）格式字符串（在本例中为：`"Hello, <argument 1 here>\n"`），又代表对参数的引用（在本例中仅有一个：`&name`）。

因为它是一个宏，所以对格式化字符串的解析*在编译期*就已经完成了。（这里和C中的`printf`等函数不一样，它们是在运行时解析和处理`%`占位符的。）

这意味着`format_args!()`宏可以给出编译错误，例如占位符和参数不匹配。

这也意味着它可以将字符串模板转换成一种更易于在运行时处理的表现形式。例如：`[Str("Hello, "), Arg(0), Str("!\n")]`。在我们的`print`例子中，展开`format_args`会得到类似这样的结果：

```rust
std::io::_print(
    // 简化版的 format_args!() 展开:
    std::fmt::Arguments {
        template: &[Str("Hello, "), Arg(0), Str("!\n")],
        arguments: &[&name as &dyn Display],
    }
);
```

当涉及不同格式化特性（例如`Display`、`Debug`、`LowerHex`等）或标志（例如`{:02x}`、`{:.9}`、`{:#?}`等）的混合
时，情况会变得更复杂一些，但总体思路是不变的。

`_print`函数对`fmt::Arguments`类型的了解并不多。它仅仅包含一个`fn write_str(&mut self, &str)`的实现，以将
一个`&str`写到标准输出。并且[`fmt::Write`特性](https://doc.rust-lang.org/stable/std/fmt/trait.Write.html)会方便地添加一个`fn write_fmt(&mut self, fmt::Arguments)`方法。使用`fmt::Arguments`对象调用这个`write_fmt`方法将会产生一系列对`write_str`的调用以产生格式化的输出。

提供的`write_fmt`方法只是简单地调用`std::fmt::write()`，这是唯一一个知道怎么“执行”`fmt::Arguments`类型中的格式化指令的函数。它为模板中的静态部分调用`write_str`，为参数调用正确的`Display::fmt`（或`LowerHex::fmt`等）函数（也会产生对`write_str`的调用）。

## 使用范例

我们刚刚了解到，要想利用Rust字符串格式化的强大功能，只需要为你的类型提供`Write::write_str`实现即可。

例如：

```rust
pub struct Terminal;

impl std::fmt::Write for Terminal {
    fn write_str(&mut self, s: &str) -> std::fmt::Result {
        write_to_terminal(s.as_bytes());
        Ok(())
    }
}
```

这就是为了让它工作所需要做的一切。接下来请看：

```rust
Terminal.write_fmt(format_args!("Hello, {name}!\n"));
```

这会产生对你的`write_str`函数的一系列调用：`write_str("Hello, ")`、`write_str(name)`和`write_str("!\n")`。

并且，感谢[`write`宏](https://doc.rust-lang.org/stable/std/macro.write.html)，上面那行代码可以被方便地写作：

```rust
write!(Terminal, "Hello, {name}!\n");
```

换句话说，你甚至不需要知道`fmt::Arguments`或着`format_args!()`的存在，甚至不需要为自己的类型添加格式化功能：只需要实现`Write::write_str`，`write!()`宏就能正常工作！

## 实现细节

让我感到兴奋的是，`fmt::Arguments`的实现细节是完全私有的。作为Rust标准库的开发者，我们可以改变`fmt::Arguments`内部所有表示数据的方式，只要我们相应地更新`format_args!()`和`std::fmt::Write`的实现。所有正在使用格式化的代码，从`dbg!(x)`到`log::info!(n = {n}")`，以及我们上文中的例子，甚至都不会注意到它们依赖的构建组件发生过变化，并与此同时从潜在的优化中受益。

所以困扰我多年的问题现在变成了：什么才是`fmt::Arguments`的最高效、最轻便、性能最好、编译最快、总体来说最好的实现？

我还不知道答案。

我认为没有唯一的答案，因为其中涉及很多权衡。我可以肯定的是，我们*目前*的实现并不是这种实现。它还不错，但还有很多地方可以改进。

## 当前的实现

今天的实现看起来像这样：

```rust
pub struct Arguments<'a> {
    pieces: &'a [&'static str],
    placeholders: Option<&'a [Placeholder]>,
    args: &'a [Argument<'a>],
}
```

`piece`字段包含来自模板的字面字符串。举个例子，对`format_args!("a{}b{}c")`，这个字段是`&["a", "b", "c"]`。

这些字符串之间的占位符（`{}`）被列举在`placeholder`字段中，它包含了格式化的选项及将被格式化的参数：

```rust
pub struct Placeholder {
    argument: usize, // args 字段中的索引
    fill: char,
    align: Alignment,
    flags: u32,
    precision: Count,
    width: Count,
}
```
在一个例如`format_args!("a{1:>012}b{1:-5}c{0:#.1}")`的复杂例子中，所有这些字段都非常重要。

不过，在很多情况下，就像`format_args!("a{}b{}c{}")`，占位符仅仅只是参数的有序排列，所有标志和设置都是默认的。这就是为什么`placeholders`字段是个`Option`：一般情况下可以设为`None`，节省一些存储空间。

最后，`args`字段包含了将被格式化的参数。这些参数其实可以直接存储为`&dyn Display`，不这样做是因为我们还需要支持`Debug`、`LowerHex`和其它的显示类特性。

所以，相反地，我们使用了一个自定义的`Argument`类型，其行为和`&dyn Display`几乎完全一致。它以两个指针的形式实现：一个指向这个参数本身，另一个指向了对应的`Display::fmt`（或着`Debug::fmt`等）实现。

这意味着当一个参数被通过两个不同的特性使用时，它被存储到了`args`两次。举个例子，`format_args!("{0} {0:?} {1:x}", a, b)`的结果是：

```rust
fmt::Arguments {
    pieces: &["", " ", " "],
    placeholders: None,
    args: &[
        fmt::Argument::new(&a, Display::fmt),
        fmt::Argument::new(&a, Debug::fmt),
        fmt::Argument::new(&b, LowerHex::fmt),
    ],
}
```

不过，当一个参数被以相同的*特性*但不同的*标志*使用了两次时，它只会在`args`中出现一次。举个例子，`format_args!("{0:?} {0:#?}", a)`，其中`a`以`Debug`分别格式化了两次，一次打开了美化输出，另一次没有。展开的结果是：

```rust
fmt::Arguments {
    pieces: &["", " "],
    placeholders: Some(&[
        fmt::Placeholder { argument: 0, ..default() },
        fmt::Placeholder { argument: 0, flags: 4 /* alternate */, ..default() },
    ]),
    args: &[
        fmt::Argument::new(&a, Debug::fmt),
    ],
}
```

`fmt::Arguments`类型被设计为使尽可能多的数据能够*提升为常量*。`pieces`和`placeholders`字段仅引用可以放置在静态位置的常量数据，因此实际上前两个`&[]`就是`&'static []`。只有被有意保持为尽可能小的`args`字段，它的数组需要在运行时构建。这是唯一一个包含非静态数据，也就是对参数本身的引用，的数组。

尽管当前的设计有一些可圈可点的地方，`fmt::Arguments`如今的实现仍然有一些问题，或着至少是可供优化的机会。让我们聊聊这些问题中最有意思的那几个吧。

### 结构体体积

首先，`format_args!("a{}b{}c{}d")`扩展为了一个包含`&["a", "b", "c", "d"]`的结构体。这意味着这4个字节现在需要10个额外指针的空间：在64位的系统上，就是80个字节！（每个`&str`需要存储一个指针和一个长度，并且外层的`&[]`也被存储为一个指针和一个长度。）正如你能想象的，许多现实世界中对格式化宏的使用最终都归于小字符串碎片，像`" "`和`"\n"`，其中每个小字符串都会占据16字节的额外空间，而这仅仅只是为了1字节的数据！

在那之上，如果我们添加了任意一个标志或着其它格式化选项到任意一个占位符上，哪怕仅仅只是一个，`placeholders`字段就会从`None`切换为包含*所有*占位符信息的`Some(&[...])`。举个例子，`format_args!("{a}{b}{c}{d:#}{e}{f}{g}")`会扩展为：

```rust
fmt::Arguments {
    pieces: &["", " "],
    placeholders: Some(&[
        fmt::Placeholder { argument: 0, ..default() },
        fmt::Placeholder { argument: 1, ..default() },
        fmt::Placeholder { argument: 2, ..default() },
        fmt::Placeholder { argument: 3, flags: 4 /* alternate */, ..default() },
        fmt::Placeholder { argument: 4, ..default() },
        fmt::Placeholder { argument: 5, ..default() },
        fmt::Placeholder { argument: 6, ..default() },
    ]),
    args: &[
        fmt::Argument::new(&a, Debug::fmt),
        fmt::Argument::new(&b, Debug::fmt),
        fmt::Argument::new(&c, Debug::fmt),
        fmt::Argument::new(&d, Debug::fmt),
        fmt::Argument::new(&e, Debug::fmt),
        fmt::Argument::new(&f, Debug::fmt),
        fmt::Argument::new(&g, Debug::fmt),
    ],
}
```

如果第4个占位符没有`#`标志的话，`placeholders`就会变成`None`。这意味着一个标志的存储开销是一个占位符结构的7倍，在64位平台上，总计将近400字节！

哪怕我们不关心静态存储的大小，`fmt::Arguments`对象本身也是比实际的需要更大一些的。它包含了三个引用切片，指针加长度的大小乘以三，在64位平台上就是48字节的开销。如果`fmt::Arguments`的大小只有一个或两个指针，那么传递它的效率就会高得多。

### 代码体积

如果你在意（静态）存储的大小，那你肯定也会关心代码的大小。

设计显示特性的一个问题，就是一个单一的特性实现被用于许多不同的标志和选项。也就是说，虽然`Display for i32`不必支持十六进制格式化（留给`LowerHex for i32`来实现），它却必须支持诸如对齐、填充字符、正负号、填充0之类的选项。

所以，一个简单的`println!("{}", some_integer)`会创建一个`fmt::Arguments`，它含有一个指向`<i32 as Display>::fmt`函数的指针，而这个函数包含了对所有选项的支持，即使我们没有使用到。理想情况下，编译器足够智能，可以预见到Rust程序从未使用这些格式化选项中的任意一个，并把这些部分完全优化掉。

不过，由于`fmt::Arguments`，那件事情做起来可不简单：它内部有数层跨越了`&dyn Write`、`Argument`和函数指针的间接关系，有效地优化这一整根链条不是编译器能做到的。

这意味着`write!(x, "{}", some_str)`，这个可以被优化为一个`x.write_str(some_str)`的调用，还是会导致一个完整的`<str as Display>::fmt`实现被引入到代码中。这个实现引入了填充和对齐的支持，进而引入了对UTF-8码点和编码UTF-8的支持。真是一堆根本用不到的代码啊！

这对嵌入式项目来说是一个大问题，许多嵌入式Rust开发者因此完全不使用格式化。

### 运行时开销

即使你拥有了不在乎代码大小和静态存储大小的余裕，你可能还是会关心运行时的性能。

正如之前提到过的，`fmt::Arguments`结构体被设计为将尽可能多地数据放入静态存储中，以使得在运行时构建`fmt::Arguments`的成本尽可能低。如今，为了构建这样的一个对象，你必须构建包含参数指针和函数指针的`args`数组，紧接着是携带指向静态数据和`args`数组的引用的`fmt::Arguments`本身（其中的静态数据包括了字符串片段和占位符信息）。

除了指向参数的指针和`args`数组自己的地址可能会在运行时变化外，其它的一切都永远不会发生改变。举个例子，即使这些数组的长度是常数，它们仍然不得不在运行时作为三个`&[]`字段的组成部分被写入`fmt::Arguments`。在此之上，`args`数组内一半的数据都是常数：指向参数的指针可能会改变，但函数指针是不会改变的。

所以，作为例子，构建`format_args!("{a}{b}{c}")`目前意味着每次执行这条表达式时，需要初始化一个包含指向`a`,`b`,`c`及其格式化函数的指针的`args`数组和初始化包含三个宽指针（指针+长度）的`fmt::Arguments`，共计12字（指针或`usize`）的数据将被写入内存。

在理想情况下，`fmt::Arguments`可以只有两个指针大：一个指针指向所有的静态数据（字符串片段、占位符和函数指针），另一个指针指向仅包含参数指针的`args`数组。对我们的例子来说，仅仅有4个指针需要被写入。省下了75%！

## 一些想法

所以，我们可以怎样改进呢？

让我们从一些想法开始吧。

### 闭包

观察`fmt::Arguments`对象的一种方式，是简单地将其看做一个“指令组成的列表”。举个例子，`format_args!("Hello {}\n{:#}!)`可以看作：*写`"Hello"`*，*以默认标志显示第一个参数*，*写一个新行*，*以美化输出的标志显示第二个参数*，*写`"!"`*，*结束*。

在Rust中表示指令列表最明显的方式是什么？一系列命令或语句？没错，就是函数，或者闭包。

所以，如果我们把`format_args!("Hello {}\n{:#}!")`展开成闭包会得到什么？

```rust
fmt::Arguments::new(|w| {
    w.write_str("Hello ")?;
    Display::fmt(&arg1, Formatter::new(w))?;
    w.write_str("\n")?;
    Display::fmt(&arg2, Formatter::new(w).alternate(true))?;
    w.write_str("!")?;
    Ok(())
})
```

如果我们这样做，那么`std::fmt::write`将会变得很平凡：只要调用闭包就行了！

`fmt::Arguments`将会仅仅包含一个`&dyn Fn`，在体积上仅仅占两个指针的大小：一个指向函数自己，另一个指向它捕捉到的参数们。完美！

并且最重要的是：编译器现在可以轻易地内联和优化`Display::fmt`的实现，裁剪掉所有未使用标志的代码！

这听起来几乎好得很不真实。

我[实现](https://github.com/rust-lang/rust/pull/101568)了它，但不幸的是，虽然它可以[显著地优化](https://github.com/rust-lang/rust/pull/101568#issuecomment-1245327485)微型嵌入式程序的二进制体积，但也[灾难性地恶化](https://github.com/rust-lang/rust/pull/101568#issuecomment-1247057102)了大型程序的编译时间和二进制大小。

并且这也是有道理的：一个拥有大量print/write/format表达式的程序会突然增加大量需要优化的额外函数体。而内联`Display::fmt`函数虽然可以减少单个打印语句的开销，但当有很多print表达式时，会导致代码体积剧增。

### Display::simple_fmt

如果我们想避免引入不必要的格式化（对齐、填充等等）代码，而不想把一切都`#[inline]`的话，我们就需要采取更精确的方法。

在`fmt`方法旁，显示特质可以拥有一个额外的方法——让我们称它为`simple_fmt`——它会做同样的事情，但假定采用的是默认的格式化参数。举个例子，当`<&str as Display>::fmt`需要支持对齐和填充（并因此需要支持UTF-8解码和记数）时，`<&str as Display>::simple_fmt`可以被实现为仅仅一行：`f.write_str(s)`。

然后我们可以更新`format_args!()`来在没有设置标记的时候使用`simple_fmt`，而不是`fmt`，以避免引入不必要的代码。

我也[实现](https://github.com/rust-lang/rust/pull/104525)了这个想法，并且它工作得很棒：它将一个6KiB的基测程序成功[降低](https://github.com/rust-lang/rust/pull/104525#issuecomment-1318399783)到了不到3KiB！

[不幸的是](https://github.com/rust-lang/rust/pull/104525#issuecomment-1521546468)，如果你的程序使用在某处使用了`&dyn Display`，这个变化会导致事情稍稍变糟：显示特性的虚表（vtable）将多出一个容纳`simple_fmt`的条目。

有一些方法可以避免这种情况，但这些方法也有其他的复杂性和局限性。

### 合并片段和占位符

现在的`fmt::Arguments`结构体包含三个字段：字符串片段、占位符描述（含有标志等）和参数。前两个字段总是常量、静态的。我们可以把它们合并吗？

如果`fmt::Arguments`看起来像这样呢？

```rust
pub struct Arguments<'a> {
    template: &'a [Piece<'a>],
    argument: &'a [Argument<'a>],
}

enum Piece<'a> {
    String(&'static str),
    Placeholder {
        argument: usize,
        options: FormattingOptions,
    },
}
```

那么，`format_args!("> {a}{b} {c}!")`就会扩展成这样的代码：

```rust
Arguments {
    template: &[
        Piece::String("> "),
        Piece::Placeholder { argument: 0, options: FormattingOptions::default() },
        Piece::Placeholder { argument: 1, options: FormattingOptions::default() },
        Piece::String(" "),
        Piece::Placeholder { argument: 2, options: FormattingOptions::default() },
        Piece::String("!"),
    ],
    arguments: &[
        Argument::new(&a, Display::fmt),
        Argument::new(&b, Display::fmt),
        Argument::new(&c, Display::fmt),
    ],
}
```

这将`fmt::Arguments`的大小从3个宽指针减小到了2个（从6个字降到了4个），并避免了在相邻占位符之间出现空的字符串。

作为`placeholders::None`优化的替代，对于所有参数都按默认选项顺序格式化的情况（如上面的例子），我们可以增加一条规则，当两个`Piece::String`元素连续出现时，就代表着它们之间有一个隐式的占位符，因为如果不是这样的话它俩就没有必要分成两部分。

在这条规则下，`format_args!("> {a}{b} {c}!")`就会扩展成这样的代码：

```rust
Arguments {
    template: &[
        Piece::String("> "),
        Piece::String(""), // Implicit placeholder for argument 0 above.
        Piece::String(" "), // Implicit placeholder for argument 1 above.
        Piece::String("!"), // Implicit placeholder for argument 2 above.
    ],
    args: &[
        Argument::new(&a, Display::fmt),
        Argument::new(&b, Display::fmt),
        Argument::new(&c, Display::fmt),
    ],
}
```
第一眼望去*看起来*它和优化前的扩展结果效率相似（具有字段`pieces:&["> ", "", " ", "!"]`），但实际消耗了更多的空间。每个`Piece`元素都远远比单纯的`&str`更大，因为枚举需要空间来容纳一个也包括了所有格式化选项的`Piece::Placeholder`。

所以，即使这样做可能会在一定程度上降低运行时的开销（通过减小`fmt::Arguments`本身的体积），它可能也会导致静态数据增大，并最终造成更大的二进制体积。

### 指令列表

在之前的想法上迭代：我们*不必*让一个字符串片段占用和一个包含了所有可能标志的占位符一样多的空间。

我们可以将占位符分散到多个条目中，从而减小枚举的大小。

举例来说，与其这样：

```rust
[
    Piece::String("> "),
    Piece::Placeholder {
        argument: 0,
        options: FormattingOptions { alternate: true, … }
    },
]
```

我们可以这样写：

```rust
[
    Piece::String("> "),
    Piece::SetAlternateFlag,
    Piece::Placeholder { argument: 0 },
]
```

现在，设置了标记的占位符将会占用更多的条目，但具有默认设置的占位符并不会为存储所有标记付出代价。

我们在这里创建的实际上是一种格式化的小型“汇编语言”，只有几条指令：写入字符串、设置标志、调用参数的格式化函数。

```rust
Arguments {
    instructions: &[
        // A list of instructions in our imaginary 'formatting assembly language':
        Instruction::WriteString("> "),
        Instruction::DisplayArg(0),
        Instruction::WriteString(" "),
        Instruction::SetAlternateFlag,
        Instruction::SetSign(Sign::Plus),
        Instruction::DisplayArg(1),
    ],
    args: &[
        Argument::new(&a, Display::fmt),
        Argument::new(&b, Display::fmt),
        Argument::new(&c, Display::fmt),
    ],
}
```

如果我们稍微[在这条路上走远一点](https://github.com/m-ou-se/rust/blob/e89626fcf8d7868cba5fa5cf7d93fa602b58109c/library/core/src/fmt/mod.rs#L484-L494)，我们甚至可以为我们的格式化命令设计出一种更高效的“指令编码”，并由此引向许多有趣的设计决策和权衡。

### 静态函数指针

作为我们今天的最后一个技巧，让我们看看我们是否能减小`args`数组的体积。正如上面所提到过的，它既储存了指向参数本身的指针，又储存了（静态的）`Display::fmt`函数指针。

理想情况下，我们可以将函数指针从`args`移到`instructions`来降低运行时开销。（也许可以加入`Instruction::DisplayArg`中。）

所以，与其这样：

```rust
Arguments {
    instructions: &[
        Instruction::DisplayArg(0),
        Instruction::DisplayArg(1),
        Instruction::DisplayArg(2),
    ],
    args: &[
        Argument::new(&a, Display::fmt),
        Argument::new(&b, Display::fmt),
        Argument::new(&c, Display::fmt),
    ],
}
```

`format_args!("{a}{b}{c}")`会扩展到：

```rust
Arguments {
    instructions: &[
        Instruction::DisplayArg(0, <… as Display>::fmt),
        Instruction::DisplayArg(1, <… as Display>::fmt),
        Instruction::DisplayArg(2, <… as Display>::fmt),
    ],
    args: &[
        Argument::new(&a),
        Argument::new(&b),
        Argument::new(&c),
    ],
}
```

这会将`args`的存储体积降低一半！

看起来足够简单，但我们遇到了一个问题：扩展过程不能再依赖于`Argument::new`的泛型签名来神奇地为每个参数选择正确的`Display::fmt`，必须为每种参数类型明确指定`<T as Display>::fmt`。

但是`format_args!()`只是一个宏，它在扩展时不知道也不可能知道参数的类型，并且Rust目前也没有类似`<typeof(a) as Display>::fmt`的语法。

处理这个问题是[有可能的](https://github.com/rust-lang/rust/issues/44343)，但[令人意外地tricky！](https://github.com/rust-lang/rust/issues/99012#issuecomment-1177580114)

## 卡住了？
正如目前已经清晰的，有许多种可能的想法来进行优化。它们中的一些组合得很好，但是许多想法是互斥的。我们也跳过了一些
[棘手的细节](https://doc.rust-lang.org/stable/std/fmt/struct.Arguments.html#method.as_str)，比如对`Arguments::as_str()`的支持。

和一切设计方面的问题一样，任何可能的改变都是有利有弊的。正如我们看到的，一个在二进制大小或运行时性能上表现很棒的实现在编译期表现也会很灾难，举例来说。

但设计问题并不是唯一的问题。

真正使得`fmt::Arguments`极其难以优化的是改变其实现所需要的[精力](https://doc.rust-lang.org/stable/std/fmt/struct.Arguments.html#method.as_str)。你不仅需要改变`fmt::Arguments`类型，还需要重写内建的`format_args!()`宏（这涉及到改变`rustc`本身），并更新`fmt::write`的实现。并且由于Rust引导的方式，标准库被同时以上个版本和当前版本的编译器编译，所以你需要确保你对标准库做的改动仍然兼容上一版编译器中未修改的`format_args!()`宏，这会导致[`#[cfg(bootstrap)]`困境](https://jyn.dev/2023/01/12/Bootstrapping-Rust-in-2023.html)。更改内建的`format_args`宏是（或着曾是）[非常有挑战性的](https://github.com/rust-lang/rust/blob/1.63.0/compiler/rustc_builtin_macros/src/format.rs)，因为它不仅负责生成`fmt::Arguments`表达式，还负责生成因格式字符串无效而产生的诊断结果。在你克服了上面的所有的困难之后，你会发现Clipy随着你的更改中断了；它依赖于`fmt::Arguments`的实现细节，因为它是在宏展开*之后*才进行代码检查的。

所以，即使是对`fmt::Arguments`的微小改动，也不仅涉及修改标准库，还要求你精通Rustc内建宏、Rustc诊断、引导和Clippy的内部细节。你的改动将同时触及许多不同的部分，这些部分分布在两个代码仓库中，并且需要得到几位不同审阅者的批准。

这正是让事情永远卡住的秘诀。

## 小步骤

让事情变得不那么“卡住”的方法，就是[把这件事拆分成很多小步骤，把阻碍一一来解决](https://www.youtube.com/watch?v=DnYQKWs_7EA)。这可能会让人有些疲惫，因为回报（比如实际性能的提高）不会很快到来。但是如果你喜欢冗长的todo清单（我就是这样的人），你就会乐在其中了:)。

正如我在[std中的锁的工作中](https://github.com/rust-lang/rust/issues/93740)做的那样，我创建了一个issue来跟踪所有和优化`fmt::Arguments`相关的事情：[https://github.com/rust-lang/rust/issues/99012](https://github.com/rust-lang/rust/issues/99012)

正如你可以在那里的todo清单上看到的，尽管（目前！）还几乎没有改变`fmt::Arguments`工作方式的更改，但其它更改已经有很多了。迄今为止所做的更改，在一定程度上通过偿还了技术层面上欠下的债，为日后的改进提供了极大的便利。

举个例子，在[其中一项更改中](https://github.com/rust-lang/rust/pull/100996)，我重构了内建宏`format_args`来将解析（parseing）、解读（resolving）、生成诊断信息、展开步骤分开（就像一个迷你编译器），这样可以让别人在不了解其它步骤的情况下也能修改展开步骤。

然后，我[提议](https://github.com/rust-lang/compiler-team/issues/541)让`format_args`宏的展开变得更加神奇，实际上是将展开步骤推迟到稍后再进行，这解锁了[一些非常酷的优化](https://github.com/rust-lang/rust/issues/78356)（已经[作为Rust 1.71的一部分发布](https://github.com/rust-lang/rust/pull/109999s://github.com/rust-lang/rust/pull/109999)，同时也允许Clippy访问未展开的信息，这样它最终可以不再依赖于宏的实现细节，使得当展开方式更改的时候它不会因此停摆。

由于其它活动部件（如Clippy）所依赖的细节（大部分）保持不变，因此[这一更改](https://github.com/rust-lang/rust/pull/106745)在很大程度上又是自成一体的，而且可以由一个人进行review。在它被合并之后，下一步就是将Clippy由使用实现细节迁移到使用新可用的信息，这在另一个[issue](https://github.com/rust-lang/rust-clippy/issues/10233)中跟踪。

而在所有这些工作都完成后，`fmt::Arguments`甚至*还*没有被更改过！但我们现在已经把[*牦牛大部分的毛剃光了*](https://sketchplanations.com/yak-shaving)，*终于*不用再钻另一个兔子洞就可以进行改进了。

> 译注：给牦牛剃毛（Yak shaving）是一句习语，形容的是当你要开始做一件事情时，发现自己为了做这件事必须先做另一件事，如此循环，直到你最终发现自己在做给牦牛剃毛这种和最初的目标完全无关的事情。

## 接下来做什么？

由于大改进的更改还在不断慢慢推进（并且我也有[别的事](https://blog.m-ou.se/rust-standard/)要忙），因此大部令人激动的结果仍然在到来的路上。但是，已经有一些有趣的事情可以让我们感到激动了：

- 在Rust 1.71中，[嵌套的`format_args`调用会被扁平化了](https://github.com/rust-lang/rust/pull/106824)。

    举个例子，`format_args!("[{}] {}", 123, format_args!("error: {}", msg))`现在实际上等于`format_args!("[123] error: {}", msg)`。这意味着像`dbg!()`这样的宏现在会有小得多的开销。

    这会导致很大的提升，甚至对大型程序也如此。举例来说，`hyper`在[编译时间](https://perf.rust-lang.org/compare.html?start=0b90256ada21c6a81b4c18f2c7a23151ab5fc232&end=b19488dba41be7ad94ed1d2933dd71ff8a300b37&stat=instructions:u)和[二进制体积](https://perf.rust-lang.org/compare.html?start=0b90256ada21c6a81b4c18f2c7a23151ab5fc232&end=b19488dba41be7ad94ed1d2933dd71ff8a300b37&stat=size%3Alinked_artifact)上都提升了大约2%~3%。

- 对无标志的占位符的优化将会删除没有用到的格式化代码，这会为小型程序带来[超过50%的二进制体积降低](https://github.com/rust-lang/rust/pull/104525#issuecomment-1318399783)。
- [优化过后的`fmt::Arguments`表示](https://github.com/rust-lang/rust/pull/115129)将不仅降低运行时的开销，而且也继续兼容`fmt::Arguments::from_str()`方法。

但这与我们最终通过实施一些更大的想法而在未来获得的潜在改进相比，简直都不值一提。过去，实现这些想法异常困难，但现在情况已经有所改变，并且还在越变越好：

- 最重要的是，得益于重构，修改`format_args!()`和`fmt::Arguments`不再需要同步修改rustc的诊断信息或Clippy。尽管还需要修改一个rustc的内建宏，但现在做起来也简单很多了。
- 当前，标准库需要同时用旧版和新版的rustc编译，给修改`fmt::Arguments`造成了很多麻烦（带有大量的`#[cfg(bootstrap)]`），但[改变Rustc引导中的stage 0的计划](https://github.com/rust-lang/compiler-team/issues/619)将会，[一旦有人着手实现了](https://rust-lang.zulipchat.com/#narrow/stream/233931-t-compiler.2Fmajor-changes/topic/Redesign.20bootstrap.20stages.20compiler-team.23619)，完全地解决掉这个问题，让修改内建宏变得简单很多很多。
- Rust编译器的[性能追踪器](https://perf.rust-lang.org/)现在也追踪二进制体积的大小了，这使得来自修改`fmt::Arguments`的优化可以比以前更轻松地被追踪。
- 现在有一个正视的[二进制体积工作组](https://www.rust-lang.org/governance/teams/compiler#Binary%20size%20working%20group)，对缩小格式化代码非常感兴趣。

换句话说：我们可以期待未来会有许多令人兴奋的改进，而我们正在一步一步地实现它们。



> 译者后记
>
> 这篇文章翻译的体验很好，原文句式清晰简洁，没有复杂的从句和词藻，你也可以去读读看。
>
> Rust的fmt包是一个非常有意思的大宝库，纵使你对Rust有再多怨言，你也不应该否认std::fmt的设计非常地精巧，通过和Write宏的配合，使得开发者不论是使用Rust的format!等宏，还是在为自己的库适配format_args!，都能得到很舒适的体验。用我mentor的话来说，就是Rust的API帮我们抹平了很多fmt相关的细节。
>
> 作者对`core::fmt`及其局限的了解非常透彻，这也和他活跃开发者的身份相关。在嵌入式开发中，人们确实会在意他所提到的`fmt`的那些问题，并且社区中也出现了`ufmt`这样针对二进制大小和编译时间优化的、无动态派发的`fmt`替代。不过，作者也清晰地了解每一种解决方案的利弊，并且作为标准库这种牵一发而动全身的重要部位，作者也表现出了足够的谨慎和全局思维。
>
> 文章中作者花大量笔墨介绍了fmt::Arguments的实现细节及改进空间。对感兴趣的人来说是不可多得的财宝，因为你很难找到第二个对Rust的格式化如此熟悉和经验丰富的专家了，更何况他还为我们写下了这篇内容丰富的文章。
>
