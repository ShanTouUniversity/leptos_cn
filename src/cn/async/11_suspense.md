# `<Suspense/>`

在上一章中，我们展示了如何创建一个简单的加载屏幕，以便在资源加载时显示一些回退内容。

```rust
let (count, set_count) = create_signal(0);
let once = create_resource(count, |count| async move { load_a(count).await });

view! {
    <h1>"My Data"</h1>
    {move || match once.get() {
        None => view! { <p>"Loading..."</p> }.into_view(),
        Some(data) => view! { <ShowData data/> }.into_view()
    }}
}
```

但是，如果我们有两个 resources ，并且想要等待它们都加载完成怎么办？

```rust
let (count, set_count) = create_signal(0);
let (count2, set_count2) = create_signal(0);
let a = create_resource(count, |count| async move { load_a(count).await });
let b = create_resource(count2, |count| async move { load_b(count).await });

view! {
    <h1>"My Data"</h1>
    {move || match (a.get(), b.get()) {
        (Some(a), Some(b)) => view! {
            <ShowA a/>
            <ShowA b/>
        }.into_view(),
        _ => view! { <p>"Loading..."</p> }.into_view()
    }}
}
```

这并不_太_糟糕，但有点烦人。如果我们可以反转控制流呢？

[`<Suspense/>`](https://docs.rs/leptos/latest/leptos/fn.Suspense.html) 组件可以让我们做到这一点。你给它一个 `fallback` prop 和子级，其中一个或多个通常涉及从 resource 中读取数据。从 `<Suspense/>`“下”（即它的一个子级中）读取 resource 会将该 resource 注册到 `<Suspense/>`。如果它仍在等待资源加载，它会显示 `fallback`。当它们都加载完成后，它会显示子级。

```rust
let (count, set_count) = create_signal(0);
let (count2, set_count2) = create_signal(0);
let a = create_resource(count, |count| async move { load_a(count).await });
let b = create_resource(count2, |count| async move { load_b(count).await });

view! {
    <h1>"My Data"</h1>
    <Suspense
        fallback=move || view! { <p>"Loading..."</p> }
    >
        <h2>"My Data"</h2>
        <h3>"A"</h3>
        {move || {
            a.get()
                .map(|a| view! { <ShowA a/> })
        }}
        <h3>"B"</h3>
        {move || {
            b.get()
                .map(|b| view! { <ShowB b/> })
        }}
    </Suspense>
}
```

每当其中一个 resource 重新加载时，`"Loading..."` 回退内容将再次显示。

这种控制流的反转使得添加或删除单个 resource 变得更容易，因为你不需要自己处理匹配。它还解锁了服务器端渲染期间的一些巨大性能改进，我们将在后面的章节中讨论这些内容。

## `<Await/>`

如果你只是想在渲染之前等待某个 `Future` 解析完成，你可能会发现 `<Await/>` 组件有助于减少样板代码。`<Await/>` 本质上是将一个带有源参数 `|| ()` 的 resource 与一个没有回退内容的 `<Suspense/>` 组合在一起。

换句话说：

1. 它只轮询一次 `Future`，并且不响应任何响应式更改。
2. 在 `Future` 解析完成之前，它不会渲染任何内容。
3. 在 `Future` 解析完成后，它将其数据绑定到你选择的任何变量名，然后使用该变量在作用域内渲染其子级。

```rust
async fn fetch_monkeys(monkey: i32) -> i32 {
    // 也许这不需要是异步的
    monkey * 2
}
view! {
    <Await
        // `future` 提供要解析的 `Future`
        future=|| fetch_monkeys(3)
        // 数据绑定到你提供的任何变量名
        let:data
    >
        // 你通过引用接收数据，并可以在此处在你的视图中使用它
        <p>{*data} " little monkeys, jumping on the bed."</p>
    </Await>
}
```

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/11-suspense-0-5-qzpgqs?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/11-suspense-0-5-qzpgqs?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源码</summary>

```rust
use gloo_timers::future::TimeoutFuture;
use leptos::*;

async fn important_api_call(name: String) -> String {
    TimeoutFuture::new(1_000).await;
    name.to_ascii_uppercase()
}

#[component]
fn App() -> impl IntoView {
    let (name, set_name) = create_signal("Bill".to_string());

    // 每次 `name` 更改时，这都会重新加载
    let async_data = create_resource(

        name,
        |name| async move { important_api_call(name).await },
    );

    view! {
        <input
            on:input=move |ev| {
                set_name(event_target_value(&ev));
            }
            prop:value=name
        />
        <p><code>"name:"</code> {name}</p>
        <Suspense
            // 每当在 suspense“下”读取的 resource
            // 正在加载时，都会显示回退内容
            fallback=move || view! { <p>"Loading..."</p> }
        >
            // 子级将在初始时渲染一次，
            // 然后每当任何 resource 解析完成后都会渲染一次
            <p>
                "Your shouting name is "
                {move || async_data.get()}
            </p>
        </Suspense>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
