---
title: Rust的Cell、RefCell和OnceCell：灵活且安全的内部可变性
zhihu-url: https://zhuanlan.zhihu.com/p/686894377
---

> 这一系列文章的创作目的主要是帮助我自己深入学习Rust，同时也为已经具备一定Rust编程经验，但还没有深入研究过语言和标准库的朋友提供参考。对于正在入门Rust的同学，我更建议你们看[《Rust圣经》](course.rs)或者[《The Book》](https://doc.rust-lang.org/stable/book/)，而不是这种晦涩难懂的文章。

终于拿到了某量化公司的offer，继续系列文章的更新。今天没有复杂的开场白，在开始前我们先回顾一下Rust的内存安全机制对引用的限制：

## 数据争用和引用限制

首先，什么是数据争用？根据[《The Book》4.2章](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html)中的定义，数据争用是指下面的一些情况：

> - 两个或多个指针同时访问同一数据。
> - 至少有一个指针用于写入数据。
> - 没有同步访问数据的机制。

Rust用于解决数据争用的对策很粗暴：拒绝编译含有数据争用的代码。具体来说，Rust给出了这样的引用限制：

> - 在任意时间，都只允许存在一个对象的多个不可变引用或者一个可变引用。
> - 引用必须是有效的。

Rust的限制很好地在编译期就消灭了数据争用；但是，这样的限制有些时候反而可能会成为我们的绊脚石。

### 局限

我们都知道，人（在大多数时候）是比编译器更灵活、更聪明的。例如有些时候，我们写了一段不会发生数据争用的代码，但是编译器却死板地以“不满足引用限制”为由拒绝编译。看看这个例子：

```c++
/**
LeetCode 19. 删除链表的倒数第 N 个结点
给你一个链表，删除链表的倒数第 n 个结点，并且返回链表的头结点。
*/

算法 删除链表的倒数第N个节点(head, n)
输入：链表的头节点 head，整数 n
输出：删除倒数第n个节点后的链表的头节点

1. 创建一个哨兵节点 dummy，其next指向head
2. 初始化两个指针 first 和 second 都指向哨兵节点 dummy
3. 将 first 指针向前移动 n+1 次（因为dummy节点的存在，实际上是n+1而不是n）
4. 同时移动 first 和 second 指针，直到 first 指向链表末尾的null
5. 此时，second 指针的下一个节点就是要删除的节点，进行删除操作：second.next = second.next.next
6. 返回 dummy.next 作为新链表的头节点（因为可能删除的是头节点本身）
```

这道经典的链表题可以用双指针来轻松、高效地解决；那么如果用Rust来实现这个双指针算法呢？

```rust
impl Solution {
    pub fn remove_nth_from_end(head: Option<Box<ListNode>>, n: i32) -> Option<Box<ListNode>> {
        let mut head = Some(Box::new(ListNode {
            val: 0,
            next: head
        }));  // 哨兵

        let mut slow = &mut head;
        let mut fast = &head;

        // fast先走n步
        for _ in 0..n {
            fast = &fast.as_ref().unwrap().next;
        }

        while fast.as_ref().unwrap().next.is_some() {
            fast = &fast.as_ref().unwrap().next;
            slow = &mut slow.as_mut().unwrap().next;
        }
    
    	// 删除节点
        slow.as_mut().unwrap().next = slow.as_mut().unwrap().next.as_mut().unwrap().next.take();

        head.unwrap().next
    }
}
```

看起来不错？但是当我们按下了运行按钮时，我们就会发现Rust拒绝了我们的代码：

```
Line 30, Char 24: cannot borrow `head` as immutable because it is also borrowed as mutable (solution.rs)
   |
29 |         let mut slow = &mut head;
   |                        --------- mutable borrow occurs here
30 |         let mut fast = &head;
   |                        ^^^^^ immutable borrow occurs here
```

我们再次审视我们的Rust代码。诚然，它同时出现了`&mut T`和`&T`，但我们的代码确实没有数据争用。我们可以注意到，从我们第一次修改`slow`的时候起，我们就再也没有使用过`fast`。但是Rust是不会在乎的。它只知道我们没有遵守引用限制。

### 另一种局限

除了上面提到的“双指针”问题之外，当项目体量过大时，一些过大的结构体也会导致引用限制冲突的问题。最好的例子就是《Rust圣经》中给出的例子：Rust编译器中的`ctxt`结构体。这种结构体的字段的使用分散在了整个项目中，如果某次使用时需要对字段作修改，那么整个程序中其他的`&ctxt`都必须避开这个`&mut ctxt`，这对Rust编译器这种体量的项目来说显然是不现实的。

另一个例子是配置文件的管理。在项目中我们也许会使用一个`HashMap`来存储配置（例如HTTP的监听端口、监听地址，和数据库的地址、用户名、密码等等），并直接把这个map的引用传递给负责各种功能的类。例如，假设我有一个负责启动HTTP服务器的类`HttpServer`，为了读取配置方便，它将配置表的引用作为其中一项内部字段；另一个负责封装数据库连接的类`Database`，也在内部保存了配置表的引用以方便读取。

现在，假设`Database`类在初始化数据库连接时发现配置项不完整，需要往配置项里填充默认值。但是问题出现了：`Database`类持有的仅仅是`&HashMap`，它不能直接对配置表做修改。它很显然也不能持有`&mut HashMap`，因为这样`HttpServer`类就不能持有配置表的引用了。比较容易想到的解决方法是我们不再让`Database`和`HttpServer`持有同一张配置表的引用，而是让它们各自把需要用到的字段复制到一个独立的的`HashMap`中。但是，这种解决方法并不算特别完美（毕竟额外多了一次复制），配置的互相剥离也使得在运行时更改配置变得复杂很多。而如果想要继续持有引用的话，我们就需要将填充默认值的步骤放在创建`Database`和`HttpServer`之前，这种方法需要我们比较大地更改代码架构。想一想，如果有一种办法可以回避引用限制，是不是会让一切都方便很多？

> 这个例子在后文会用代码写出来，如果看不明白的话可以去后面看看

### 解决：内部可变性

Rust在设计之初就已经考虑到了这些问题；对于不可避免地需要“打破”引用限制的情况，Rust提出了一个概念：内部可变性。具体来说，和大多数类型不同，具有内部可变性的类型`T`可以直接用`&T`来变化。通过使用这些类型，用户可以无视Rust的引用限制——反正我都是用的`&T`，怎么限制都无所谓。

有的朋友可能会疑惑：既然开了这么大的口子，那么Rust还安全吗？数据争用岂不是说来就来？其实，和很多人想的不同，标准库中有内部可变性的类型都没有彻底放下对内存安全的追求————正相反，它们仍然会检查用户的操作会不会导致数据争用，并在用户的操作违反引用限制时将程序`panic`。后面会有具体的例子。

## RefCell：动态借用内部的值

最经典也最常用的“内部可变性”类型是`RefCell`，它支持对内部的值进行“动态借用”，也就是对内部值进行临时、独占、可变地访问。

### 用法

`RefCell`的用法很简单：它使用`new`来构造自己，使用`borrow`来获取内部值的不可变引用，使用`borrow_mut`来获取内部值的可变引用。

```rust
use std::cell::*;

fn main() {
    let a = RefCell::new(1);
    println!("{}", a.borrow());     // 1
    
    *a.borrow_mut() = 2;
    println!("{}", a.borrow());     // 2
}
```

此外，值得注意的是，`RefCell`还提供了一个更高效的API：`get_mut`，它需要`RefCell`的`&mut`引用，作用和`borrow_mut`相同。由于它所需的`&mut`引用保证了程序中没有其他该对象的引用存在，因此它不需要进行运行时检查，性能也比`borrow_mut`更高。

## Cell：来自移动（Move）的内部可变性

`RefCell`的兄弟`Cell`也是一个具有“内部可变性”的类型，但是它的内部可变性是通过在内存中移动值来实现的。`Cell`的使用也很简单：它使用`new`来构造自己，使用`get`来获得内部的值，使用`set`来改变内部的值。

```rust
use std::cell::*;

fn main() {
    let a = Cell::new(1);

    println!("{}", a.get());     // 1
    a.set(2);
    println!("{}", a.get());     // 2
}
```

和`RefCell`不同的是，`Cell`不提供内部值的引用，调用`get`和`set`时返回和提供的都是值本身。也正因此，`Cell`通常用于一些简单的类型，复制和移动值不会消耗太多的资源。并且，只有内部值实现了`Copy`特征后，才能使用`Cell`的`get`方法（未实现`Copy`特征的内部值可以使用`Cell`的`replace`等方法来获得内部值）。

`Cell`不占用额外的内存空间，性能也比`RefCell`更优（因为不需要检查引用限制），但是使用上比`RefCell`受限，在操作复杂类型时不如`RefCell`方便。建议读者根据使用场景来灵活判断使用`RefCell`还是`Cell`。

## OnceCell：一次性使用的RefCell

> 这里讨论的是Rust在1.70.0中引入标准库的类型，而不是包`once_cell`中的同名类型。

`OnceCell`是`Cell`和`RefCell`的混合体，它既可以在不移动和不复制的情况下获得内部值的引用（与`Cell`不同），又不需要在运行时进行引用限制检查（与`RefCell`不同）。但是，它的便利也有代价：一旦其中的值被设置了，就不能再被改变。

```rust
// 官方文档的示例
use std::cell::OnceCell;

let cell = OnceCell::new();
assert!(cell.get().is_none());

let value: &String = cell.get_or_init(|| {
    "Hello, World!".to_string()
});
assert_eq!(value, "Hello, World!");
assert!(cell.get().is_some());
```

`OnceCell`的用途相比`Cell`和`RefCell`都更加局限，它的内部可变性也仅仅体现在那一次性的`set`上。相对而言它的线程安全版本`OnceLock`就更常用也更有用，因为我们可以用它来取代`lazy_static`，保存程序的全局/静态变量。

## 内部可变性还能保证内存安全吗？

虽然直接使用`&T`来改变内部值的做法看起来很暴力，但标准库中提供的这些内部可变性类型都是内存安全的。

在`RefCell`中，当我们调用`borrow`和`borrow_mut`时，它会在运行时检查我们是否违反了Rust的引用限制（也就是仅允许同时存在一个可变或多个不可变）。例如，下面的几段代码都会panic：

```rust
use std::cell::*;

fn main() {
    let a = RefCell::new(1);

    let a_ref = a.borrow_mut();
    println!("{}", a.borrow());
    println!("{}", a_ref);
}

// thread 'main' panicked at src/main.rs:7:22:
// already mutably borrowed: BorrowError
```

```rust
use std::cell::*;

fn main() {
    let a = RefCell::new(1);

    let a_ref = a.borrow();
    println!("{}", a.borrow_mut());
    println!("{}", a_ref);
}

// thread 'main' panicked at src/main.rs:7:22:
// already borrowed: BorrowMutError
```

毫无疑问，这种在运行时panic的做法会**降低程序的稳定性**，因此这就要求使用`RefCell`的程序员小心谨慎，在写代码时**主动检查自己的用法是否满足引用限制**，不要把`RefCell`当作万能的银弹来用，更不要把`RefCell`当做逃避编译错误的手段。

相比`RefCell`还能拿到内部对象的可变引用，连可变引用都见不到的`Cell`和`OnceCell`就更安全了，它们主动放弃了很多灵活性，换取了无需运行时检查的性能。

## 线程安全

`std::cell`内的三个类型并不是线程安全的。不过，`RefCell`有线程安全版本的对应：`RwLock`，`OnceCell`也有线程安全的版本`OnceLock`。相信`RwLock`要比`RefCell`常用和常见很多，因为多线程间的数据同步是一个比引用限制更容易遇到的问题，这也导致了许多人在学习Rust之初就已经在接触`Mutex`和`RwLock`这样的锁类型了。

## 内部可变性的应用

内部可变性的应用就非常广泛了！除了之前提到的两个情况之外，我们也会在这样的场景下需要使用内部可变性：

### 在实现Clone等要求对象不可变的特征时改变对象

大家每天都在用的`Rc`和`Arc`在`.clone()`时会增加引用计数，但是大家想过`Rc`和`Arc`为什么可以在`Clone`这个接受`&self`的特征里改变引用计数的值吗？看到这里相信读者都能猜到原因了：它使用`Cell`来保存引用计数的值。下面是简化版的`Rc`实现，来自标准库的文档：

```rust
use std::cell::Cell;
use std::ptr::NonNull;
use std::process::abort;
use std::marker::PhantomData;

struct Rc<T: ?Sized> {
    ptr: NonNull<RcBox<T>>,
    phantom: PhantomData<RcBox<T>>,
}

struct RcBox<T: ?Sized> {
    strong: Cell<usize>,
    refcount: Cell<usize>,
    value: T,
}

impl<T: ?Sized> Clone for Rc<T> {
    fn clone(&self) -> Rc<T> {
        self.inc_strong();
        Rc {
            ptr: self.ptr,
            phantom: PhantomData,
        }
    }
}

trait RcBoxPtr<T: ?Sized> {

    fn inner(&self) -> &RcBox<T>;

    fn strong(&self) -> usize {
        self.inner().strong.get()
    }

    fn inc_strong(&self) {
        self.inner()
            .strong
            .set(self.strong()			// 重点在这一行
                     .checked_add(1)
                     .unwrap_or_else(|| abort() ));
    }
}

impl<T: ?Sized> RcBoxPtr<T> for Rc<T> {
   fn inner(&self) -> &RcBox<T> {
       unsafe {
           self.ptr.as_ref()
       }
   }
}
```

### 在不可变的“内部”引入内部可变性

继续以`Rc`和`Arc`为例，这样的共享指针类型提供了在不同的位置克隆和共享的容器；为了避免这种对同一对象的共享访问导致数据竞争，我们只能使用`&`而不是`&mut`来借用`Rc`和`Arc`的内部变量。如果没有支持内部可变性的容器的话，我们就几乎不可能改变`Rc`和`Arc`中的值了。

出于上面的原因，在`Rc`和`Arc`中使用内部可变性类型的用法————例如`Rc<RefCell<T>>`和`Arc<Mutex<T>>`）在Rust世界中就特别特别常见。例如，当我们需要维护一个在全局范围使用的配置列表时，比起直接传递`HashMap`，传递一个`Rc<RefCell<HashMap>>`显然会帮助我们降低很多心智负担和开发难度————毕竟它不用处理所有权（作为参数传给其他函数的时候直接`clone`就完事），更不用处理可变引用（需要改变内部值的时候`borrow_mut`一下就好了，不用特意避开其他`&`引用）。

## 实例：使用Rc<RefCell<T>>管理配置文件

在下面的文章中，我们将会以之前作为例子提到的“配置文件管理”来示范`Rc<RefCell<T>>`的强大作用。回顾一下“配置文件管理”案例的场景：

> 在项目中我们也许会使用一个`HashMap`来存储配置（例如HTTP的监听端口、监听地址，和数据库的地址、用户名、密码等等），并直接把这个map的引用传递给负责各种功能的类。例如，假设我有一个负责启动HTTP服务器的类`HttpServer`，为了读取配置方便，它将配置表的引用作为其中一项内部字段；另一个负责封装数据库连接的类`Database`，也在内部保存了配置表的引用以方便读取。

### 不使用Rc<RefCell<T>>

首先，如果不使用`Rc<RefCell<T>>`，而是使用传统的`&`引用的话，代码像什么样子呢？

```rust
use std::collections::*;

/// 从配置文件读取配置
fn load_configs(config: &mut HashMap<String, String>) {
    // 假设我们从文件里面读取了配置，这里为了模拟演示需要就不实际读取了
    config.insert("host".to_owned(), "0.0.0.0".to_owned());
    config.insert("port".to_owned(), "8080".to_owned());
    config.insert("db_url".to_owned(), "mysql://localhost:3306".to_owned());
    config.insert("db_username".to_owned(), "root".to_owned());
}

/// 用于处理Http服务器的类
struct HttpServer<'a> {
    config: &'a HashMap<String, String>,
}

impl<'a> HttpServer<'a> {
    /// 在配置表中为缺失的配置项填充默认值
    fn fill_defaults(config: &mut HashMap<String, String>) {
        // 这里为了举例方便，只放一条
        if !config.contains_key("port") {
            config.insert("port".to_owned(), "8080".to_owned());
        }
    }

    fn new(config: &'a HashMap<String, String>) -> Self {
        Self { config }
    }

    fn listen(&self) {
        println!("Listening on {}:{}", self.config.get("host").unwrap(), self.config.get("port").unwrap())
    }
}

/// 用于处理数据库连接的类
struct Database<'a> {
    config: &'a HashMap<String, String>,
}

impl<'a> Database<'a> {
    /// 在配置表中为缺失的配置项填充默认值
    fn fill_defaults(config: &mut HashMap<String, String>) {
        // 这里为了举例方便，只放一条
        if !config.contains_key("db_password") {
            config.insert("db_password".to_owned(), "admin".to_owned());
        }
    }

    fn new(config: &'a HashMap<String, String>) -> Self {
        Self { config }
    }

    fn connect(&self) {
        println!("Connected to Database: {}, user:{}, password:{}", self.config.get("db_url").unwrap(), self.config.get("db_username").unwrap(), self.config.get("db_password").unwrap())
    }
}


fn main() {
    let mut config = HashMap::new();

    // 读取配置
    load_configs(&mut config);

    // 填充默认值
    HttpServer::fill_defaults(&mut config);
    Database::fill_defaults(&mut config);

    let db = Database::new(&config);
    let http = HttpServer::new(&config);

    db.connect();
    http.listen();
}
```

最终输出是这样的：

```
Connected to Database: mysql://localhost:3306, user:root, password:admin
Listening on 0.0.0.0:8080
```

现在让我们分析一下不使用`Rc<RefCell<T>>`带来的不便之处：

首先，最明显的就是因为我们使用了引用，因此我们必须显式地管理引用的声明周期。例如代码的这几行：

```rust
struct Database<'a> {
    config: &'a HashMap<String, String>,
}

impl<'a> Database<'a> {
    fn new(config: &'a HashMap<String, String>) -> Self {
        // ...
    }
}
```

我们为了保证配置表的生命周期比`Database`长，需要作很多额外的声明。当然，针对这个问题，我们可以做一层改良：

```rust
struct Database {
    config: Rc<HashMap<String, String>>,	// 换成Rc
}

impl Database {
    fn new(config: Rc<HashMap<String, String>>) -> Self {
        Self { config: config.clone() }
    }
}
```

但是这次改良并不能解决另一个问题：我们在`new`之前就已经在调用`Database`和`HttpServer`的静态方法了。其实常理上讲，我们应该在`new`中调用这些静态方法的。

那么改成这样可以吗？我们也许可以试试给`&mut`降级：

```rust
fn new(config: &'a mut HashMap<String, String>) -> Self {
    // 填充默认值
    Self::fill_defaults(config);

    Self { config: &*config }
}
```

但是Rust编译器似乎认为这种用法下，在`Database`和`HttpServer`的整个生命周期内都在持有`config`的可变引用：

![](https://s.c.accr.cc/picgo/1710339242-b27128.png)

如果说上述的问题都还仅仅属于“不优雅”和“不方便”的范畴话，接下来的问题就很麻烦了：在`Database`和`HttpServer`初始化之后，我们就不能修改`config`了。这看起来似乎不是很严重的问题，但如果项目后期有不停机修改配置的需求的话，那么就真的彻底无从下手了————因为`Database`和`HttpServer`持有了`config`的不可变引用，这就导致获取`config`的可变引用会直接导致违反引用限制，编译失败。

### 使用Rc<RefCell<T>>

带着上面提到的各种各样的问题，我们将代码改造为使用`Rc<RefCell<T>>`的版本：

```rust
use std::cell::*;
use std::rc::*;
use std::collections::*;

/// 从配置文件读取配置
fn load_configs(config: Rc<RefCell<HashMap<String, String>>>) {
    // 假设我们从文件里面读取了配置，这里为了模拟演示需要就不实际读取了
    let mut map = config.borrow_mut();
    map.insert("host".to_owned(), "0.0.0.0".to_owned());
    map.insert("port".to_owned(), "8080".to_owned());
    map.insert("db_url".to_owned(), "mysql://localhost:3306".to_owned());
    map.insert("db_username".to_owned(), "root".to_owned());
}

/// 用于处理Http服务器的类
struct HttpServer {
    config: Rc<RefCell<HashMap<String, String>>>,
}

impl HttpServer {
    /// 在配置表中为缺失的配置项填充默认值
    fn fill_defaults(config: &Rc<RefCell<HashMap<String, String>>>) {
        // 这里为了举例方便，只放一条
        let mut map = config.borrow_mut();
        if !map.contains_key("port") {
            map.insert("port".to_owned(), "8080".to_owned());
        }
    }

    fn new(config: Rc<RefCell<HashMap<String, String>>>) -> Self {
        // 填充默认值
        Self::fill_defaults(&config);

        Self { config }
    }

    fn listen(&self) {
        let map = self.config.borrow();
        println!("Listening on {}:{}", map.get("host").unwrap(), map.get("port").unwrap())
    }
}

/// 用于处理数据库连接的类
struct Database {
    config: Rc<RefCell<HashMap<String, String>>>,
}

impl Database {
    /// 在配置表中为缺失的配置项填充默认值
    fn fill_defaults(config: &Rc<RefCell<HashMap<String, String>>>) {
        // 这里为了举例方便，只放一条
        let mut map = config.borrow_mut();
        if !map.contains_key("db_password") {
            map.insert("db_password".to_owned(), "admin".to_owned());
        }
    }

    fn new(config: Rc<RefCell<HashMap<String, String>>>) -> Self {
        // 填充默认值
        Self::fill_defaults(&config);

        Self { config }
    }

    fn connect(&self) {
        let map = self.config.borrow();
        println!("Connected to Database: {}, user:{}, password:{}", map.get("db_url").unwrap(), map.get("db_username").unwrap(), map.get("db_password").unwrap())
    }
}


fn main() {
    let config = Rc::new(RefCell::new(HashMap::new()));

    // 读取配置
    load_configs(config.clone());

    let db = Database::new(config.clone());
    let http = HttpServer::new(config.clone());

    db.connect();
    http.listen();
}
```

输出还是一样的：

```
Connected to Database: mysql://localhost:3306, user:root, password:admin
Listening on 0.0.0.0:8080
```

大家可以看看代码的实现，我们不再需要考虑`&mut`和`&T`谁先谁后的问题，更不用担心引用限制对程序造成的影响。此外，代码的结构也更合理了，我们将配置项检查和默认值填充放在了`Database`和`HttpServer`对象的构造过程中，这是比刚刚的做法更符合常理的。此外，这样的设计使得我们在`Database`和`HttpServer`对象构造之后仍然可以修改`config`的内容，为将来的开发留出了很大空间。

## 番外：链表和二叉树

总是有人说Rust不能写链表和二叉树，读到这里之后屏幕前的读者可以转身向这些人大喊一声“Naive”了~如果读者之前没有使用`RefCell`的经验的话，可以考虑用`Rc<RefCell<T>>`实现一个链表来锻炼一下，以下是链表节点的参考定义：

```rust
pub struct ListNode {
    val: i32,
    next: Option<Rc<RefCell<ListNode>>>
}
```

（你也可以加上一个`prev`，然后写个双向链表）

进阶：完成力扣146题[《LRU缓存》](https://leetcode.cn/problems/lru-cache/description/)，实现一个`LinkedHashMap`，具体算法可以参考用Java的题解（算法不是难点），然后用Rust实现出来。这题做完之后，你一定就能自信地告诉别人自己已经掌握了`Rc<RefCell<T>>`，乃至Rust的大部分内容。

关于二叉树也是一样的，力扣上有很多Rust的二叉树题，也是用`Rc<RefCell<TreeNode>>`作为指针，大家意犹未尽 的话也可以去试试。

## 总结

这些具有内部可变性的类型，例如`Cell`和`RefCell`，扮演着在Rust安全模型中绕过不可变性规则的角色，但是它们总体上仍然是安全且受控地。在使用这些工具时，用户必须仔细地权衡它们提供的灵活性与潜在的运行时成本和错误风险。正确地应用`Cell`和`RefCell`可以在保持代码安全性和性能的同时，提供灵活的可变性以满足特定的编程需求。

