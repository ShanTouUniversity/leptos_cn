# 使用 resource 加载数据

[ Resource ](https://docs.rs/leptos/latest/leptos/struct.Resource.html) 是一种响应式数据结构，它反映了异步任务的当前状态，允许你将异步 `Future` 集成到同步响应式系统中。你无需使用 `.await` 等待其数据加载，而是将 `Future` 转换为一个信号，如果它已解析则返回 `Some(T)`，如果它仍在等待中则返回 `None`。

你可以使用 [`create_resource`](https://docs.rs/leptos/latest/leptos/fn.create_resource.html) 函数来做到这一点。它接受两个参数：

1. 一个源信号，每当它发生变化时，都会生成一个新的 `Future`
2. 一个获取器函数，它从该信号中获取数据并返回一个 `Future`

下面是一个例子

```rust
// 我们的源信号：一些同步的、本地状态
let (count, set_count) = create_signal(0);

// 我们的 resource
let async_data = create_resource(
    count,
    // 每次 `count` 发生变化时，这都会运行
    |value| async move {
        logging::log!("从 API 加载数据");
        load_data(value).await
    },
);
```

要创建一个只运行一次的 resource，你可以传递一个非响应式的、空的源信号：

```rust
let once = create_resource(|| (), |_| async move { load_data().await });
```

要访问该值，你可以使用 `.get()` 或 `.with(|data| /* */)`。这些方法的工作方式与信号上的 `.get()` 和 `.with()` 一样——`get` 克隆该值并返回它，`with` 对其应用一个闭包——但是对于任何 `Resource<_, T>`，它们总是返回 `Option<T>`，而不是 `T`：因为你的 resource 始终有可能仍在加载。

因此，你可以在视图中显示 resource 的当前状态：

```rust
let once = create_resource(|| (), |_| async move { load_data().await });
view! {
    <h1>"My Data"</h1>
    {move || match once.get() {
        None => view! { <p>"Loading..."</p> }.into_view(),
        Some(data) => view! { <ShowData data/> }.into_view()
    }}
}
```

Resource 还提供了一个 `refetch()` 方法，允许你手动重新加载数据（例如，响应按钮点击），以及一个 `loading()` 方法，该方法返回一个 `ReadSignal<bool>`，指示 resource 当前是否正在加载。

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/10-resources-0-5-x6h5j6?file=%2Fsrc%2Fmain.rs%3A2%2C3)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/10-resources-0-5-9jq86q?file=%2Fsrc%2Fmain.rs%3A2%2C3" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源码</summary>

```rust
use gloo_timers::future::TimeoutFuture;
use leptos::*;

// 这里我们定义一个异步函数
// 这可以是任何东西：网络请求、数据库读取等等。
// 这里，我们只是将一个数字乘以 10
async fn load_data(value: i32) -> i32 {
    // 模拟一秒钟的延迟
    TimeoutFuture::new(1_000).await;
    value * 10
}

#[component]
fn App() -> impl IntoView {
    // 这个计数是我们同步的、本地状态
    let (count, set_count) = create_signal(0);

    // create_resource 在其作用域后接受两个参数
    let async_data = create_resource(
        // 第一个是“源信号”
        count,
        // 第二个是加载器
        // 它以源信号的值作为参数
        // 并进行一些异步工作
        |value| async move { load_data(value).await },
    );
    // 每当源信号发生变化时，加载器都会重新加载

    // 你也可以创建只加载一次的 resource
    // 只需从源信号返回单元类型 () 即可
    // 这不依赖于任何东西：我们只加载一次
    let stable = create_resource(|| (), |_| async move { load_data(1).await });

    // 我们可以使用 .get() 访问 resource 值
    // 这将在 Future 解析之前以响应式方式返回 None
    // 并在解析后更新为 Some(T)
    let async_result = move || {
        async_data
            .get()
            .map(|value| format!("Server returned {value:?}"))
            // 此加载状态仅在首次加载之前显示
            .unwrap_or_else(|| "Loading...".into())
    };

    // resource 的 loading() 方法给了我们一个
    // 信号，指示它当前是否正在加载
    let loading = async_data.loading();
    let is_loading = move || if loading() { "Loading..." } else { "Idle." };

    view! {
        <button
            on:click=move |_| {
                set_count.update(|n| *n += 1);
            }
        >
            "Click me"
        </button>
        <p>
            <code>"stable"</code>": " {move || stable.get()}
        </p>
        <p>
            <code>"count"</code>": " {count}
        </p>
        <p>
            <code>"async_value"</code>": "
            {async_result}
            <br/>
            {is_loading}
        </p>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
