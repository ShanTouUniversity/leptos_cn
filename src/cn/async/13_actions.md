# 使用 Action 修改数据

我们已经讨论了如何使用 resource 加载 `async` 数据。Resource 会立即加载数据，并与 `<Suspense/>` 和 `<Transition/>` 组件紧密合作，以显示你的应用程序中是否正在加载数据。但是，如果你只是想调用一些任意的 `async` 函数并跟踪它的执行情况，该怎么办？

好吧，你总是可以使用 [`spawn_local`](https://docs.rs/leptos/latest/leptos/fn.spawn_local.html)。这允许你通过将 `Future` 交给浏览器（或者在服务器上，是 Tokio 或任何你正在使用的其他运行时）来在同步环境中生成一个 `async` 任务。但是你如何知道它是否仍在等待中？好吧，你可以设置一个信号来显示它是否正在加载，另一个信号来显示结果...

所有这些都是真的。或者你可以使用最终的 `async` 原语：[`create_action`](https://docs.rs/leptos/latest/leptos/fn.create_action.html)。

Action 和 resource 看起来很相似，但它们代表着根本不同的东西。如果你试图通过运行一个 `async` 函数来加载数据，无论是运行一次还是在其他值发生变化时运行，你可能想使用 `create_resource`。如果你试图偶尔运行一个 `async` 函数来响应用户点击按钮之类的事情，你可能想使用 `create_action`。

假设我们有一些想要运行的 `async` 函数。

```rust
async fn add_todo_request(new_title: &str) -> Uuid {
    /* 在服务器上做一些添加新的待办事项的事情 */
}
```

`create_action` 接受一个 `async` 函数，该函数接受对单个参数的引用，你可以将其视为其“输入类型”。

> 输入始终是单个类型。如果要传入多个参数，可以使用结构体或元组。
>
> ```rust
> // 如果只有一个参数，就使用它
> let action1 = create_action(|input: &String| {
>    let input = input.clone();
>    async move { todo!() }
> });
>
> // 如果没有参数，则使用单元类型 `()`
> let action2 = create_action(|input: &()| async { todo!() });
>
> // 如果有多个参数，则使用元组
> let action3 = create_action(
>   |input: &(usize, String)| async { todo!() }
> );
> ```
>
> 因为 action 函数接受一个引用，但 `Future` 需要具有 `'static` 生命周期，所以你通常需要克隆该值才能将其传递给 `Future`。这确实很尴尬，但它解锁了一些强大的功能，如乐观 UI。我们将在后面的章节中看到更多相关内容。

所以在这种情况下，我们创建 action 所需要做的就是

```rust
let add_todo_action = create_action(|input: &String| {
    let input = input.to_owned();
    async move { add_todo_request(&input).await }
});
```

我们将使用 `.dispatch()` 调用它，而不是直接调用 `add_todo_action`，如下所示

```rust
add_todo_action.dispatch("Some value".to_string());
```

你可以从事件监听器、超时或任何地方执行此操作；因为 `.dispatch()` 不是一个 `async` 函数，所以可以从同步上下文中调用它。

Action 提供对一些信号的访问，这些信号在你要调用的异步 action 和同步响应式系统之间进行同步：

```rust
let submitted = add_todo_action.input(); // RwSignal<Option<String>>
let pending = add_todo_action.pending(); // ReadSignal<bool>
let todo_id = add_todo_action.value(); // RwSignal<Option<Uuid>>
```

这使得跟踪请求的当前状态、显示加载指示器或基于提交将成功的假设进行“乐观 UI”变得很容易。

```rust
let input_ref = create_node_ref::<Input>();

view! {
    <form
        on:submit=move |ev| {
            ev.prevent_default(); // 不要重新加载页面...
            let input = input_ref.get().expect("input to exist");
            add_todo_action.dispatch(input.value());
        }
    >
        <label>
            "What do you need to do?"
            <input type="text"
                node_ref=input_ref
            />
        </label>
        <button type="submit">"Add Todo"</button>
    </form>
    // 使用我们的加载状态
    <p>{move || pending().then("Loading...")}</p>
}
```

现在，有可能这一切看起来有点过于复杂，或者可能限制太多。我想在这里将 action 与 resource 一起包含进来，作为拼图中缺失的一块。在一个真实的 Leptos 应用程序中，你实际上最常将 action 与服务器函数 [`create_server_action`](https://docs.rs/leptos/latest/leptos/fn.create_server_action.html) 和 [`<ActionForm/>`](https://docs.rs/leptos_router/latest/leptos_router/fn.ActionForm.html) 组件一起使用，以创建真正强大的渐进增强表单。所以如果这个原语对你来说似乎毫无用处... 不要担心！也许以后会有意义。（或者现在就查看我们的 [`todo_app_sqlite`](https://github.com/leptos-rs/leptos/blob/main/examples/todo_app_sqlite/src/todo.rs) 示例。）


```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/13-actions-0-5-8xk35v?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/13-actions-0-5-8xk35v?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源码</summary>

```rust
use gloo_timers::future::TimeoutFuture;
use leptos::{html::Input, *};
use uuid::Uuid;

// 这里我们定义一个异步函数
// 这可以是任何东西：网络请求、数据库读取等等。
// 将其视为一个修改：你运行的某个命令式异步操作，
// 而 resource 将是你加载的一些异步数据
async fn add_todo(text: &str) -> Uuid {
    _ = text;
    // 模拟一秒钟的延迟
    TimeoutFuture::new(1_000).await;
    // 假装这是一个帖子 ID 或其他东西
    Uuid::new_v4()
}

#[component]
fn App() -> impl IntoView {
    // action 接受一个带有一个参数的异步函数
    // 它可以是一个简单类型、一个结构体或 ()
    let add_todo = create_action(|input: &String| {
        // 输入是一个引用，但我们需要 Future 拥有它
        // 这很重要：我们需要克隆并移动到 Future 中
        // 这样它就有一个 'static 生命周期
        let input = input.to_owned();
        async move { add_todo(&input).await }
    });

    // action 提供了一堆同步的、响应式变量
    // 这些变量告诉我们关于 action 状态的不同信息
    let submitted = add_todo.input();
    let pending = add_todo.pending();
    let todo_id = add_todo.value();

    let input_ref = create_node_ref::<Input>();

    view! {
        <form
            on:submit=move |ev| {
                ev.prevent_default(); // 不要重新加载页面...
                let input = input_ref.get().expect("input to exist");
                add_todo.dispatch(input.value());
            }
        >
            <label>
                "What do you need to do?"
                <input type="text"
                    node_ref=input_ref
                />
            </label>
            <button type="submit">"Add Todo"</button>
        </form>
        <p>{move || pending().then(|| "Loading...")}</p>
        <p>
            "Submitted: "
            <code>{move || format!("{:#?}", submitted())}</code>
        </p>
        <p>
            "Pending: "
            <code>{move || format!("{:#?}", pending())}</code>
        </p>
        <p>
            "Todo ID: "
            <code>{move || format!("{:#?}", todo_id())}</code>
        </p>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
