# `<Transition/>`

你可能会注意到，在 `<Suspense/>` 示例中，如果你不断重新加载数据，它会一直闪烁回到 `"Loading..."`。有时这很好。对于其他时候，可以使用 [`<Transition/>`](https://docs.rs/leptos/latest/leptos/fn.Transition.html)。

`<Transition/>` 的行为与 `<Suspense/>` 完全相同，但它不是每次都回退，而只是在第一次显示回退内容。在所有后续加载中，它会继续显示旧数据，直到新数据准备就绪。这对于防止闪烁效果以及允许用户继续与你的应用程序交互非常方便。

此示例显示了如何使用 `<Transition/>` 创建一个简单的选项卡式联系人列表。当你选择一个新选项卡时，它会继续显示当前联系人，直到新数据加载完成。这比不断回退到加载消息的用户体验要好得多。

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/12-transition-0-5-2jg5lz?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/12-transition-0-5-2jg5lz?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源码</summary>

```rust
use gloo_timers::future::TimeoutFuture;
use leptos::*;

async fn important_api_call(id: usize) -> String {
    TimeoutFuture::new(1_000).await;
    match id {
        0 => "Alice",
        1 => "Bob",
        2 => "Carol",
        _ => "User not found",
    }
    .to_string()
}

#[component]
fn App() -> impl IntoView {
    let (tab, set_tab) = create_signal(0);

    // 每次 `tab` 更改时，这都会重新加载
    let user_data = create_resource(tab, |tab| async move { important_api_call(tab).await });

    view! {
        <div class="buttons">
            <button
                on:click=move |_| set_tab(0)
                class:selected=move || tab() == 0
            >
                "Tab A"
            </button>
            <button
                on:click=move |_| set_tab(1)
                class:selected=move || tab() == 1
            >
                "Tab B"
            </button>
            <button
                on:click=move |_| set_tab(2)
                class:selected=move || tab() == 2
            >
                "Tab C"
            </button>
            {move || if user_data.loading().get() {
                "Loading..."
            } else {
                ""
            }}
        </div>
        <Transition
            // 回退内容将初始显示
            // 在后续重新加载中，当前子级将
            // 继续显示
            fallback=move || view! { <p>"Loading..."</p> }
        >
            <p>
                {move || user_data.read()}
            </p>
        </Transition>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
