# 定义路由

## 入门

路由器很容易上手。

首先，请确保你已将 `leptos_router` 包添加到你的依赖项中。与 `leptos` 一样，路由器依赖于激活 `csr`、`hydrate` 或 `ssr` 功能。例如，如果你要将路由器添加到客户端渲染的应用程序中，你将要运行
```sh 
cargo add leptos_router --features=csr 
```

> leptos_router 是一个单独的包，这一点很重要。这意味着路由器中的所有内容都可以在用户代码中定义。如果你想创建自己的路由器，或者不使用路由器，你完全可以这样做！

并从路由器中导入相关的类型，可以使用类似以下的内容

```rust
use leptos_router::{Route, RouteProps, Router, RouterProps, Routes, RoutesProps};
```

或者简单地使用

```rust
use leptos_router::*;
```

## 提供 `<Router/>`

路由行为由 [`<Router/>`](https://docs.rs/leptos_router/latest/leptos_router/fn.Router.html) 组件提供。这通常应该在你的应用程序的根目录附近，包装应用程序的其余部分。

> 你不应该尝试在你的应用程序中使用多个 `<Router/>`。请记住，路由器驱动着全局状态：如果你有多个路由器，当 URL 发生变化时，由谁来决定做什么？

让我们从一个使用路由器的简单 `<App/>` 组件开始：

```rust
use leptos::*;
use leptos_router::*;

#[component]
pub fn App() -> impl IntoView {
  view! {
    <Router>
      <nav>
        /* ... */
      </nav>
      <main>
        /* ... */
      </main>
    </Router>
  }
}
```

## 定义 `<Routes/>`

[`<Routes/>`](https://docs.rs/leptos_router/latest/leptos_router/fn.Routes.html) 组件是你定义用户可以在你的应用程序中导航到的所有路由的地方。每个可能的路由都由一个 [`<Route/>`](https://docs.rs/leptos_router/latest/leptos_router/fn.Route.html) 组件定义。

你应该将 `<Routes/>` 组件放置在你的应用程序中希望渲染路由的位置。`<Routes/>` 之外的所有内容都将出现在每个页面上，因此你可以将导航栏或菜单之类的内容留在 `<Routes/>` 之外。

```rust
use leptos::*;
use leptos_router::*;

#[component]
pub fn App() -> impl IntoView {
  view! {
    <Router>
      <nav>
        /* ... */
      </nav>
      <main>
        // 我们所有的路由都将出现在 <main> 内部
        <Routes>
          /* ... */
        </Routes>
      </main>
    </Router>
  }
}
```

通过使用 `<Route/>` 组件为 `<Routes/>` 提供子级来定义单个路由。`<Route/>` 接受一个 `path` 和一个 `view`。当当前位置与 `path` 匹配时，将创建并显示 `view`。

`path` 可以包括

- 一个静态路径（`/users`），
- 以冒号开头的动态命名参数（`/:id`），
- 和/或以星号开头的通配符（`/user/*any`）

`view` 是一个返回视图的函数。任何没有 props 的组件都可以在这里工作，返回某个视图的闭包也可以。

```rust
<Routes>
  <Route path="/" view=Home/>
  <Route path="/users" view=Users/>
  <Route path="/users/:id" view=UserProfile/>
  <Route path="/*any" view=|| view! { <h1>"Not Found"</h1> }/>
</Routes>
```

> `view` 接受一个 `Fn() -> impl IntoView`。如果一个组件没有 props，它可以直接传递到 `view` 中。在这种情况下，`view=Home` 只是 `|| view! { <Home/> }` 的简写。

现在，如果你导航到 `/` 或 `/users`，你将获得主页或 `<Users/>`。如果你访问 `/users/3` 或 `/blahblah`，你将获得用户配置文件或你的 404 页面（`<NotFound/>`）。在每次导航中，路由器都会确定应该匹配哪个 `<Route/>`，从而确定应该在 `<Routes/>` 组件定义的位置显示什么内容。

请注意，你可以按任何顺序定义你的路由。路由器会对每个路由进行评分，以查看它的匹配程度，而不是简单地尝试从上到下进行匹配。

够简单吧？

## 条件路由

`leptos_router` 基于你的应用程序中只有一个 `<Routes/>` 组件的假设。它使用它在服务器端生成路由，通过缓存计算的分支优化路由匹配，并渲染你的应用程序。

你不应该使用 `<Show/>` 或 `<Suspense/>` 之类的其他组件有条件地渲染 `<Routes/>`。

```rust
// ❌ 不要这样做！
view! {
  <Show when=|| is_loaded() fallback=|| view! { <p>"Loading"</p> }>
    <Routes>
      <Route path="/" view=Home/>
    </Routes>
  </Show>
}
```

相反，你可以使用嵌套路由来渲染一次 `<Routes/>`，并有条件地渲染路由器出口：

```rust
// ✅ 改为这样做！
view! {
  <Routes>
    // 父路由
    <Route path="/" view=move || {
      view! {
        // 仅在数据加载完成后显示出口
        <Show when=|| is_loaded() fallback=|| view! { <p>"Loading"</p> }>
          <Outlet/>
        </Show>
      }
    }>
      // 嵌套子路由
      <Route path="/" view=Home/>
    </Route>
  </Routes>
}
```

如果这看起来很奇怪，不要担心！本书的下一节将介绍这种嵌套路由。
