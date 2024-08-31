# 参数和查询

静态路径对于区分不同的页面很有用，但几乎每个应用程序都希望在某些时候通过 URL 传递数据。

你可以通过两种方式来做到这一点：

1. 命名路由 **参数**，如 `/users/:id` 中的 `id`
2. 命名路由 **查询**，如 `/search?q=Foo` 中的 `q`

由于 URL 的构建方式，你可以从_任何_ `<Route/>` 视图中访问查询。你可以从定义它们的 `<Route/>` 或其任何嵌套子级访问路由参数。

使用几个钩子访问参数和查询非常简单：

- [`use_query`](https://docs.rs/leptos_router/latest/leptos_router/fn.use_query.html) 或 [`use_query_map`](https://docs.rs/leptos_router/latest/leptos_router/fn.use_query_map.html)
- [`use_params`](https://docs.rs/leptos_router/latest/leptos_router/fn.use_params.html) 或 [`use_params_map`](https://docs.rs/leptos_router/latest/leptos_router/fn.use_params_map.html)

每一个都有一个类型化选项（`use_query` 和 `use_params`）和一个非类型化选项（`use_query_map` 和 `use_params_map`）。

非类型化版本保存一个简单的键值映射。要使用类型化版本，请在结构体上派生 [`Params`](https://docs.rs/leptos_router/0.2.3/leptos_router/trait.Params.html) 特征。

> `Params` 是一个非常轻量级的特征，通过将 `FromStr` 应用于每个字段，将字符串的扁平键值映射转换为结构体。由于路由参数和 URL 查询的扁平结构，它远不如 `serde` 灵活；它也为你的二进制文件增加更少的重量。

```rust
use leptos::*;
use leptos_router::*;

#[derive(Params, PartialEq)]
struct ContactParams {
	id: usize
}

#[derive(Params, PartialEq)]
struct ContactSearch {
	q: String
}
```

> 注意：`Params` 派生宏位于 `leptos::Params`，`Params` 特征位于 `leptos_router::Params`。如果你避免使用像 `use leptos::*;` 这样的全局导入，请确保你为派生宏导入了正确的宏。
>
> 如果你没有使用 `nightly` 功能，你会收到以下错误
>
> ```
> no function or associated item named `into_param` found for struct `std::string::String` in the current scope
> ```
>
> 目前，支持 `T: FromStr` 和 `Option<T>` 作为类型化参数需要一个 nightly 功能。你可以通过简单地将结构体更改为使用 `q: Option<String>` 而不是 `q: String` 来解决此问题。

现在我们可以在组件中使用它们了。想象一个既有参数又有查询的 URL，如 `/contacts/:id?q=Search`。

类型化版本返回 `Memo<Result<T, _>>`。它是一个 Memo，因此它对 URL 的更改做出反应。它是一个 `Result`，因为需要从 URL 中解析参数或查询，并且可能有效也可能无效。

```rust
let params = use_params::<ContactParams>();
let query = use_query::<ContactSearch>();

// id: || -> usize
let id = move || {
	params.with(|params| {
		params.as_ref()
			.map(|params| params.id)
			.unwrap_or_default()
	})
};
```

非类型化版本返回 `Memo<ParamsMap>`。同样，它是一个 memo，以对 URL 的更改做出反应。[`ParamsMap`](https://docs.rs/leptos_router/0.2.3/leptos_router/struct.ParamsMap.html) 的行为与任何其他映射类型非常相似，它的 `.get()` 方法返回 `Option<&String>`。

```rust
let params = use_params_map();
let query = use_query_map();

// id: || -> Option<String>
let id = move || {
	params.with(|params| params.get("id").cloned())
};
```

这可能会有点混乱：派生一个包装 `Option<_>` 或 `Result<_>` 的信号可能需要几个步骤。但是这样做是值得的，原因有两个：

1. 它是正确的，即它迫使你考虑这些情况，“如果用户没有为此查询字段传递值怎么办？如果他们传递了一个无效的值怎么办？”
2. 它具有高性能。具体来说，当你导航到与同一个 `<Route/>` 匹配的不同路径时，只有参数或查询发生了变化，你可以在不重新渲染的情况下对应用程序的不同部分进行细粒度更新。例如，在我们联系人列表示例中，在不同联系人之间导航会对名称字段（最终是联系信息）进行有针对性的更新，而无需替换或重新渲染包装的 `<Contact/>`。这就是细粒度响应式的作用。

> 这与上一节的示例相同。路由器是一个如此集成的系统，以至于提供一个单独的示例来突出多个功能是有意义的，即使我们还没有解释所有功能。

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/16-router-0-5-4xp4zz?file=%2Fsrc%2Fmain.rs%3A102%2C2)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/16-router-0-5-4xp4zz?file=%2Fsrc%2Fmain.rs%3A102%2C2" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源码</summary>

```rust
use leptos::*;
use leptos_router::*;

#[component]
fn App() -> impl IntoView {
    view! {
        <Router>
            <h1>"Contact App"</h1>
            // 这个 <nav> 将显示在每个路由上，
            // 因为它在 <Routes/> 之外
            // 注意：我们可以只使用普通的 <a> 标签
            // 路由器将使用客户端导航
            <nav>
                <h2>"Navigation"</h2>
                <a href="/">"Home"</a>
                <a href="/contacts">"Contacts"</a>
            </nav>
            <main>
                <Routes>
                    // / 只有一个未嵌套的 "Home"
                    <Route path="/" view=|| view! {
                        <h3>"Home"</h3>
                    }/>
                    // /contacts 有嵌套路由
                    <Route
                        path="/contacts"
                        view=ContactList
                      >
                        // 如果没有指定 id，则回退
                        <Route path=":id" view=ContactInfo>
                            <Route path="" view=|| view! {
                                <div class="tab">
                                    "(Contact Info)"
                                </div>
                            }/>
                            <Route path="conversations" view=|| view! {
                                <div class="tab">
                                    "(Conversations)"
                                </div>
                            }/>
                        </Route>
                        // 如果没有指定 id，则回退
                        <Route path="" view=|| view! {
                            <div class="select-user">
                                "Select a user to view contact info."
                            </div>
                        }/>
                    </Route>
                </Routes>
            </main>
        </Router>
    }
}

#[component]
fn ContactList() -> impl IntoView {
    view! {
        <div class="contact-list">
            // 这是我们的联系人列表组件本身
            <div class="contact-list-contacts">
                <h3>"Contacts"</h3>
                <A href="alice">"Alice"</A>
                <A href="bob">"Bob"</A>
                <A href="steve">"Steve"</A>
            </div>

            // <Outlet/> 将显示嵌套的子路由
            // 我们可以将此出口放置在布局中的任何位置
            <Outlet/>
        </div>
    }
}

#[component]
fn ContactInfo() -> impl IntoView {
    // 我们可以使用 `use_params_map` 以响应式方式访问 :id 参数
    let params = use_params_map();
    let id = move || params.with(|params| params.get("id").cloned().unwrap_or_default());

    // 假设我们在这里从 API 加载数据
    let name = move || match id().as_str() {
        "alice" => "Alice",
        "bob" => "Bob",
        "steve" => "Steve",
        _ => "User not found.",
    };

    view! {
        <div class="contact-info">
            <h4>{name}</h4>
            <div class="tabs">
                <A href="" exact=true>"Contact Info"</A>
                <A href="conversations">"Conversations"</A>
            </div>

            // 这里的 <Outlet/> 是嵌套在
            // /contacts/:id 路由下的选项卡
            <Outlet/>
        </div>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
