# 提取器

我们在上一章中看到的服务器函数展示了如何在服务器上运行代码，并将其与你在浏览器中渲染的用户界面集成。但它们并没有展示太多关于如何充分利用你的服务器的潜力。

## 服务器框架

我们称 Leptos 为“全栈”框架，但“全栈”始终是一个不当用词（毕竟，它从不意味着从浏览器到你所在电力公司的所有内容）。对我们来说，“全栈”意味着你的 Leptos 应用程序可以在浏览器中运行，可以在服务器上运行，并且可以将两者集成在一起，将每个环境中可用的独特功能融合在一起；正如我们到目前为止在本书中看到的那样，浏览器上的按钮点击可以驱动服务器上的数据库读取，两者都写在同一个 Rust 模块中。但 Leptos 本身不提供服务器（或数据库、操作系统、固件或电缆...）

相反，Leptos 为两个最流行的 Rust Web 服务器框架提供了集成，分别是 Actix Web ([`leptos_actix`](https://docs.rs/leptos_actix/latest/leptos_actix/)) 和 Axum ([`leptos_axum`](https://docs.rs/leptos_axum/latest/leptos_axum/))。我们已经构建了与每个服务器的路由器的集成，这样你就可以使用 `.leptos_routes()` 简单地将你的 Leptos 应用程序插入到现有的服务器中，并轻松处理服务器函数调用。

> 如果你还没有看过我们的 [Actix](https://github.com/leptos-rs/start) 和 [Axum](https://github.com/leptos-rs/start-axum) 模板，现在是查看它们的好时机。

## 使用提取器

Actix 和 Axum 处理程序都建立在相同的强大的**提取器**理念之上。提取器从 HTTP 请求中“提取”类型化数据，让你可以轻松访问特定于服务器的数据。

Leptos 提供了 `extract` 帮助程序函数，让你可以使用与每个框架的处理程序非常相似的便捷语法，直接在你的服务器函数中使用这些提取器。

### Actix 提取器

[`leptos_actix` 中的 `extract` 函数](https://docs.rs/leptos_actix/latest/leptos_actix/fn.extract.html) 接受一个处理程序函数作为参数。处理程序遵循与 Actix 处理程序类似的规则：它是一个异步函数，接收将从请求中提取的参数并返回一些值。处理程序函数接收提取的数据作为其参数，并可以在 `async move` 块的主体内对它们进行进一步的 `async` 工作。它将你返回的任何值返回到服务器函数中。

```rust
use serde::Deserialize;

#[derive(Deserialize, Debug)]
struct MyQuery {
    foo: String,
}

#[server]
pub async fn actix_extract() -> Result<String, ServerFnError> {
    use actix_web::dev::ConnectionInfo;
    use actix_web::web::{Data, Query};
    use leptos_actix::extract;

    let (Query(search), connection): (Query<MyQuery>, ConnectionInfo) = extract().await?;
    Ok(format!("search = {search:?}\nconnection = {connection:?}",))
}
```

### Axum 提取器

[`leptos_axum::extract`](https://docs.rs/leptos_axum/latest/leptos_axum/fn.extract.html) 函数的语法非常相似。

```rust
use serde::Deserialize;

#[derive(Deserialize, Debug)]
struct MyQuery {
    foo: String,
}

#[server]
pub async fn axum_extract() -> Result<String, ServerFnError> {
    use axum::{extract::Query, http::Method};
    use leptos_axum::extract;

    let (method, query): (Method, Query<MyQuery>) = extract().await?;

    Ok(format!("{method:?} and {query:?}"))
}
```

这些是从服务器访问基本数据的相对简单的示例。但你可以使用提取器来访问标头、cookie、数据库连接池等内容，使用完全相同的 `extract()` 模式。

Axum `extract` 函数仅支持状态为 `()` 的提取器。如果你需要一个使用 `State` 的提取器，你应该使用 [`extract_with_state`](https://docs.rs/leptos_axum/latest/leptos_axum/fn.extract_with_state.html)。这需要你提供状态。你可以通过使用 Axum `FromRef` 模式扩展现有的 `LeptosOptions` 状态来做到这一点，该模式在渲染和服务器函数期间使用自定义处理程序将状态作为上下文提供。

```rust
use axum::extract::FromRef;

/// 派生 FromRef 以允许状态中的多个项目，使用 Axum 的
/// SubStates 模式。
#[derive(FromRef, Debug, Clone)]
pub struct AppState{
    pub leptos_options: LeptosOptions,
    pub pool: SqlitePool
}
```

[点击这里查看在自定义处理程序中提供上下文的示例](https://github.com/leptos-rs/leptos/blob/19ea6fae6aec2a493d79cc86612622d219e6eebb/examples/session_auth_axum/src/main.rs#L24-L44)。

#### Axum 状态

Axum 的依赖注入的典型模式是提供一个 `State`，然后可以在你的路由处理程序中提取它。Leptos 通过上下文提供了自己的依赖注入方法。上下文通常可以用来代替 `State` 来提供共享的服务器数据（例如，数据库连接池）。

```rust
let connection_pool = /* 一些共享状态 */;

let app = Router::new()
    .leptos_routes_with_context(
        &app_state,
        routes,
        move || provide_context(connection_pool.clone()),
        App,
    )
    // 等等。
```

然后可以在你的服务器函数中使用简单的 `use_context::<T>()` 访问此上下文。

如果你*需要*在服务器函数中使用 `State`——例如，如果你有一个现有的 Axum 提取器需要 `State`——那么也可以使用 Axum 的 [`FromRef`](https://docs.rs/axum/latest/axum/extract/derive.FromRef.html) 模式和 [`extract_with_state`](https://docs.rs/leptos_axum/latest/leptos_axum/fn.extract_with_state.html)。本质上，你需要通过上下文和 Axum 路由器状态来提供状态：

```rust
#[derive(FromRef, Debug, Clone)]
pub struct MyData {
    pub value: usize,
    pub leptos_options: LeptosOptions,
}

let app_state = MyData {
    value: 42,
    leptos_options,
};

// 使用路由构建我们的应用程序
let app = Router::new()
    .leptos_routes_with_context(
        &app_state,
        routes,
        {
            let app_state = app_state.clone();
            move || provide_context(app_state.clone())
        },
        App,
    )
    .fallback(file_and_error_handler)
    .with_state(app_state);

// ... 
#[server] 
pub async fn uses_state() -> Result<(), ServerFnError> {
    let state = expect_context::<AppState>();
    let SomeStateExtractor(data) = extract_with_state(&state).await?;
    // 待办事项
}
```

## 关于数据加载模式的说明

因为 Actix 和（尤其是）Axum 建立在单次往返 HTTP 请求和响应的理念之上，所以你通常会在应用程序的“顶部”（即在你开始渲染之前）附近运行提取器，并使用提取的数据来确定应该如何渲染。在你渲染 `<button>` 之前，你会加载你的应用程序可能需要的所有数据。并且任何给定的路由处理程序都需要知道该路由需要提取的所有数据。

但 Leptos 集成了客户端和服务器，并且能够使用来自服务器的新数据刷新你的 UI 的小部分，而无需强制完全重新加载所有数据，这一点很重要。因此，Leptos 喜欢将数据加载“下推”到你的应用程序中，尽可能靠近你的用户界面的叶子节点。当你点击 `<button>` 时，它可以只刷新它需要的数据。这正是服务器函数的用途：它们让你可以粒度地访问要加载和重新加载的数据。

`extract()` 函数允许你通过在你的服务器函数中使用提取器来组合这两种模式。你可以访问路由提取器的全部功能，同时将需要提取的内容分散到你的各个组件中。这使得重构和重新组织路由变得更容易：你不需要预先指定路由需要的所有数据。
