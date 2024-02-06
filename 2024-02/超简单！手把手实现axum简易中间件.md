---
title: 超简单！手把手实现axum简易中间件
zhihu-url: https://zhuanlan.zhihu.com/p/681697965
---
axum是Rust语言tokio生态中的重要一环，以轻量、模块化、易用而闻名于世。它的中间件系统集成自另一个叫`tower`的框架，这就意味着如果我们要写axum的中间件的话，就得了解一下这个`tower`的各个核心概念，并学习它的用法。但是，很多时候我们可能只是想写一点简单的小工具，为了小需求去学习一个复杂的大库也许并不是很值得。

下图是`tower`文档里的中间件范例，可以在axum中使用……但是还是算了吧，让我们看看别的解决方法？

![](https://s.c.accr.cc/picgo/1707204714-52041b.png)

还好，axum贴心地为我们提供了一个小工具：[`middleware::from_fn`](https://docs.rs/axum/latest/axum/middleware/fn.from_fn.html) 。它可以让我们像写`Handler`一样地写中间件，就像这样：

![](https://s.c.accr.cc/picgo/1707204880-4b75a8.png)

没有乱七八糟的trait、生命周期和`Into<Box<dyn>>`，一切都是如此的自然和简单！

axum的文档这样描述这种构建中间件的方法：

> 当你满足这些条件时，使用`middleware::from_fn`：
>
> - 你不打算实现自己的Future，而更愿意用熟悉的async/await语法。
> - 你不打算将自己的中间件发布为一个crate供别人使用，因为像这样编写的中间件只能与axum兼容。

显然，我们没有上面的两种顾虑（自己的Future实现？这不是月薪三千还随时可能被裁的可怜rust程序员该考虑的）。我们的目标就是十分钟内搞定需求，所以我们就快乐地选择`middleware::from_fn`了。

如果读者想更加详细地了解axum的中间件生态，可以去看看官方文档：[axum::middleware](https://docs.rs/axum/latest/axum/middleware/index.html)



## 准备

接下来，我将会手把手带领你在10分钟内（根据熟悉axum的程度因人而异）写完一个简单的axum中间件。中间件的功能很简单：假设我们系统的登录接口没有设置验证码，为了防止系统内的用户密码被爆破，我们需要对登录接口加一层限流的中间件，限制每个用户每分钟能调用登录接口10次左右。



为了简化教程的复杂度，我们用`timedmap`库来代替Redis；在实际生产中，建议用Redis来保证服务的可扩展性。

---

首先，我们创建一个项目，并导入这样的依赖：

```toml
[dependencies]
anyhow = "1.0.79"
axum = "0.7.4"
axum-client-ip = "0.5.0"
lazy_static = "1.4.0"
timedmap = { version = "1.0.1", features = ["tokio"] }
tokio = { version = "1.36.0", features = ["full"] }
```

接下来，我们快速地构建一个简单的axum应用：

```rust
use anyhow::Result;
use axum::{routing::get, Router};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> Result<()> {
    let app = Router::new().route("/login", get(|| async {"Username or password invalid."}));
    let listener = TcpListener::bind("0.0.0.0:8080").await?;
    axum::serve(listener, app).await?;

    Ok(())
}
```

这个12行的程序可以在访问`/login`时返回一段文本，模拟一个成熟系统的登录功能。

![运行效果](https://s.c.accr.cc/picgo/1707217895-c48b64.png)

接下来，我们在它的基础上进行改造，为它编写一个中间件。


## 中间件

axum的`from_fn`中间件有这样的模板：

```rust
use axum::{
    response::Response,
    middleware::Next,
    extract::Request,
};

async fn my_middleware(
    request: Request,
    next: Next,
) -> Response {
    // do something with `request`...

    let response = next.run(request).await;

    // do something with `response`...

    response
}
```

简单介绍一下这个模板中的关键事项：

- 必须是`async fn`形式的异步函数
- 必须有一个或多个提取器作为函数参数（在上面的模板中，是`Request`）
- 最后一个参数必须是`Next`
- 返回值必须实现了`IntoResponse`

在了解了这些事项之后，我们就可以着手写中间件了。

### 最简单的中间件

话不多说，直接上代码：

```rust
use std::time::{self, SystemTime};

async fn hello_world(request: Request, next: Next) -> Response {
    let now = SystemTime::now()
        .duration_since(time::UNIX_EPOCH)
        .unwrap()
        .as_millis();
    if now % 2 == 0 {
        "Surprise!".into_response()
    } else {
        next.run(request).await
    }
}
```

这个中间件以二分之一的概率拦截请求并返回一个“Surprise”，虽然它并没有什么用 ，但是它很好地为我们的后续开发开了一个头。它就像一个真正的handler一样，接受各种各样的参数，并返回一个Response；它也像真正的handler一样，可以直接返回内容，而不管后续的中间件和handler。

搞定了中间件，接下来就是把它插入到路由里了：

```rust
let app = Router::new()
    .route("/login", get(|| async { "Username or password invalid." }))
    .route_layer(middleware::from_fn(hello_world)); // 看这一行
```

我们调用了`axum::middleware::from_fn`方法，这是我们之前所提到过的；它将我们的函数转换为一个真正的中间件，如下图所示，它用一个宏把我们的短短几行代码展开成了一个`tower`的`Service`。

![](https://s.c.accr.cc/picgo/1707218269-4f7482.png)

现在我们再启动程序，刷新刚刚的页面，可以看到新的结果：

![](https://s.c.accr.cc/picgo/1707218381-ab1047.png)

程序当前的代码如下：

```rust
use std::time::{self, SystemTime};

use anyhow::Result;
use axum::{
    extract::Request,
    middleware::{self, Next},
    response::{IntoResponse, Response},
    routing::get,
    Router,
};
use tokio::net::TcpListener;

async fn hello_world(request: Request, next: Next) -> Response {
    let now = SystemTime::now()
        .duration_since(time::UNIX_EPOCH)
        .unwrap()
        .as_millis();
    if now % 2 == 0 {
        "Surprise!".into_response()
    } else {
        next.run(request).await
    }
}

#[tokio::main]
async fn main() -> Result<()> {
    let app = Router::new()
        .route("/login", get(|| async { "Username or password invalid." }))
        .route_layer(middleware::from_fn(hello_world));
    let listener = TcpListener::bind("0.0.0.0:8080").await?;
    axum::serve(listener, app).await?;

    Ok(())
}
```

### 实战：登录限流

现在，我们可以继续我们的实战演练了。首先，我们准备一个数据结构来检查用户是否发出了过多请求：

```rust
struct RateLimiter {
    pub map: TimedMap<IpAddr, i32>,
}

impl RateLimiter {
    pub fn new() -> Self {
        Self {
            map: Default::default(),
        }
    }

    pub fn check(&self, ip_addr: &IpAddr) -> bool {
        let count = self.map.get(ip_addr).unwrap_or(0);
        if count > 10 {
            false
        }else{
            self.map.insert(ip_addr.clone(), count+1, Duration::from_secs(6));
            true
        }
    }
}
```

这里我们用了一个比较简单的算法来判断用户是否发送了过多请求：每次用户的请求到来时，我们从含有效期的哈希表中获取用户最近的连续请求次数；如果次数大于10次，我们判定用户请求量过大，返回false；否则，我们更新哈希表并返回true。这里可以注意到我们用的超时时长是6秒，这是因为每次更新数据时，有效期都会刷新，因此我们需要设置有效期为60秒/10次=6次/秒。

---

现在我们已经准备好了限流工具，那么我们要怎样将它放到中间件里呢？

也许有的读者刚接触axum，会说：

```rust
lazy_static!{
    pub static ref RATE_LIMITER: Mutex<RateLimiter> = Mutex::new(RateLimiter::new());
}
```

这种写法其实是不适合axum和异步编程的，具体的原因可以询问GPT；正确的做法是通过axum提供的State机制来将限流工具注入到中间件内：

```rust
#[derive(Clone)]
struct LimitState {
    pub limiter: Arc<RateLimiter>,
}

impl LimitState {
    pub fn new() -> Self {
        Self {
            limiter: Arc::new(RateLimiter::new()),
        }
    }
}
```

在这里，我们定义了一个用于中间件的State。应当注意，它内部的限流工具使用了Arc来包裹，并且State实现了Clone特质。这种写法可以保证`limiter`可以在多个线程/协程之间使用同一个`RateLimiter`实例。接下来，我们稍微修改中间件部分的`from_fn`，更换为`from_fn_with_state`：

```rust
let state = LimitState::new();
let app = Router::new()
    .route("/login", get(|| async { "Username or password invalid." }))
    .route_layer(middleware::from_fn_with_state(state, rate_limit));  // 变化在这里
```

---

拦在我们面前的是最后一道难关：怎么获取用户的IP地址？这里我们直接使用`axum-client-ip`库，它提供了提取用户真实IP的工具，详情和配置方法可以参阅它的[官方文档](https://docs.rs/axum-client-ip)。在这里，我们只需要在主函数里额外加一行配置就好了：

```rust
use axum_client_ip::SecureClientIpSource;

let app = Router::new()
    .route("/login", get(|| async { "Username or password invalid." }))
    .route_layer(middleware::from_fn_with_state(state, rate_limit))
    .layer(SecureClientIpSource::ConnectInfo.into_extension());  // 在这里
```

> 在生产环境中，不要忘了把`SecureClientIpSource`更换成你应用的部署平台/CDN/反向代理使用的HTTP头；如果没有用CDN或者反代，则保留这样不变即可

此外，`serve`一行也要做一些修改：

```rust
axum::serve(listener, app.into_make_service_with_connect_info::<SocketAddr>()).await?;
```

---

一切障碍都已经清除，我们现在直接手起刀落，写完我们的中间件：

```rust
async fn rate_limit(
    State(state): State<LimitState>,
    SecureClientIp(ip_addr): SecureClientIp,
    request: Request,
    next: Next,
) -> Response {
    if state.limiter.check(&ip_addr) {
        next.run(request).await
    } else {
        (
            StatusCode::TOO_MANY_REQUESTS,
            "Too many requests! Please wait for one minute.",
        )
            .into_response()
    }
}
```

接下来就是测试环节！

![](https://s.c.accr.cc/picgo/1707222248-9f636c.png)

![](https://s.c.accr.cc/picgo/1707222266-e9e60d.png)

可以看到，我们的中间件已经开始工作啦~



最后附上程序的完整代码：

```rust
use std::{
    net::{IpAddr, SocketAddr},
    sync::Arc,
    time::Duration,
};

use anyhow::Result;
use axum::{
    extract::{Request, State},
    http::StatusCode,
    middleware::{self, Next},
    response::{IntoResponse, Response},
    routing::get,
    Router,
};
use axum_client_ip::{SecureClientIp, SecureClientIpSource};
use timedmap::TimedMap;
use tokio::net::TcpListener;

struct RateLimiter {
    pub map: TimedMap<IpAddr, i32>,
}

impl RateLimiter {
    pub fn new() -> Self {
        Self {
            map: Default::default(),
        }
    }

    pub fn check(&self, ip_addr: &IpAddr) -> bool {
        let count = self.map.get(ip_addr).unwrap_or(0);
        if count > 10 {
            false
        } else {
            self.map
                .insert(ip_addr.clone(), count + 1, Duration::from_secs(6));
            true
        }
    }
}

#[derive(Clone)]
struct LimitState {
    pub limiter: Arc<RateLimiter>,
}

impl LimitState {
    pub fn new() -> Self {
        Self {
            limiter: Arc::new(RateLimiter::new()),
        }
    }
}

async fn rate_limit(
    State(state): State<LimitState>,
    SecureClientIp(ip_addr): SecureClientIp,
    request: Request,
    next: Next,
) -> Response {
    if state.limiter.check(&ip_addr) {
        next.run(request).await
    } else {
        (
            StatusCode::TOO_MANY_REQUESTS,
            "Too many requests! Please wait for one minute.",
        )
            .into_response()
    }
}

#[tokio::main]
async fn main() -> Result<()> {
    let state = LimitState::new();
    let app = Router::new()
        .route("/login", get(|| async { "Username or password invalid." }))
        .route_layer(middleware::from_fn_with_state(state, rate_limit))
        .layer(SecureClientIpSource::ConnectInfo.into_extension());
    let listener = TcpListener::bind("0.0.0.0:8080").await?;
    axum::serve(
        listener,
        app.into_make_service_with_connect_info::<SocketAddr>(),
    )
    .await?;

    Ok(())
}
```

