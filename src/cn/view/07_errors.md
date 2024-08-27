# 错误处理

[在上一章中](./06_control_flow.md)，我们看到你可以渲染 `Option<T>`：在 `None` 的情况下，它什么也不会渲染，而在 `Some(T)` 的情况下，它会渲染 `T`（也就是说，如果 `T` 实现了 `IntoView`）。你实际上可以使用 `Result<T, E>` 做一些非常类似的事情。在 `Err(_)` 的情况下，它什么也不会渲染。在 `Ok(T)` 的情况下，它会渲染 `T`。

让我们从一个简单的组件开始，用于捕获数字输入。

```rust
#[component]
fn NumericInput() -> impl IntoView {
    let (value, set_value) = create_signal(Ok(0));

    // 当输入发生变化时，尝试从输入中解析一个数字
    let on_input = move |ev| set_value(event_target_value(&ev).parse::<i32>());

    view! {
        <label>
            "Type an integer (or not!)"
            <input type="number" on:input=on_input/>
            <p>
                "You entered "
                <strong>{value}</strong>
            </p>
        </label>
    }
}
```

每次你更改输入时，`on_input` 都会尝试将其值解析为 32 位整数 (`i32`)，并将其存储在我们的 `value` 信号中，该信号是 `Result<i32, _>`。如果你输入数字 `42`，UI 将显示

```
You entered 42
```

但是如果你输入字符串 `foo`，它会显示

```
You entered
```

这不太好。它避免了我们使用 `.unwrap_or_default()` 或其他类似的东西，但如果我们可以捕获错误并对其进行处理，那就更好了。

你可以使用 [`<ErrorBoundary/>`](https://docs.rs/leptos/latest/leptos/fn.ErrorBoundary.html) 组件来做到这一点。

```admonish note
人们经常试图指出 `<input type="number">` 会阻止你输入像 `foo` 这样的字符串，或任何其他不是数字的内容。这在某些浏览器中是正确的，但并非所有浏览器都如此！此外，还有各种各样的内容可以被输入到一个普通的数字输入框中，而这些内容并不是 `i32`：浮点数、大于 32 位的数字、字母 `e` 等等。可以告诉浏览器维护其中一些不变式，但浏览器的行为仍然会有所不同：自己进行解析很重要！
```

## `<ErrorBoundary/>`

`<ErrorBoundary/>` 有点像我们在上一章中看到的 `<Show/>` 组件。如果一切正常——也就是说，如果一切都是 `Ok(_)`——它会渲染它的子级。但是，如果在这些子级中渲染了 `Err(_)`，它将触发 `<ErrorBoundary/>` 的 `fallback`。

让我们在这个例子中添加一个 `<ErrorBoundary/>`。

```rust
#[component]
fn NumericInput() -> impl IntoView {
    let (value, set_value) = create_signal(Ok(0));

    let on_input = move |ev| set_value(event_target_value(&ev).parse::<i32>());

    view! {
        <h1>"Error Handling"</h1>
        <label>
            "Type a number (or something that's not a number!)"
            <input type="number" on:input=on_input/>
            <ErrorBoundary
                // fallback 接收一个包含当前错误的信号
                fallback=|errors| view! {
                    <div class="error">
                        <p>"Not a number! Errors: "</p>
                        // 我们可以将错误列表渲染为字符串，如果我们愿意的话
                        <ul>
                            {move || errors.get()
                                .into_iter()
                                .map(|(_, e)| view! { <li>{e.to_string()}</li>})
                                .collect_view()
                            }
                        </ul>
                    </div>
                }
            >
                <p>"You entered " <strong>{value}</strong></p>
            </ErrorBoundary>
        </label>
    }
}
```

现在，如果你输入 `42`，`value` 就是 `Ok(42)`，你会看到

```
You entered 42
```

如果你输入 `foo`，`value` 就是 `Err(_)`，`fallback` 将被渲染。我们选择将错误列表渲染为 `String`，因此你会看到类似这样的内容

```
Not a number! Errors:
- cannot parse integer from empty string
```

如果修复了错误，错误消息将消失，你用 `<ErrorBoundary/>` 包装的内容将再次出现。

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/7-errors-0-5-5mptv9?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/7-errors-0-5-5mptv9?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>
```

<details>
<summary>CodeSandbox 源码</summary>

```rust
use leptos::*;

#[component]
fn App() -> impl IntoView {
    let (value, set_value) = create_signal(Ok(0));

    // 当输入发生变化时，尝试从输入中解析一个数字
    let on_input = move |ev| set_value(event_target_value(&ev).parse::<i32>());

    view! {
        <h1>"Error Handling"</h1>
        <label>
            "Type a number (or something that's not a number!)"
            <input type="number" on:input=on_input/>
            // 如果在 <ErrorBoundary/> 内部渲染了 `Err(_)`,
            // 将显示 fallback。否则，将显示
            // <ErrorBoundary/> 的子级。
            <ErrorBoundary
                // fallback 接收一个包含当前错误的信号
                fallback=|errors| view! {
                    <div class="error">
                        <p>"Not a number! Errors: "</p>
                        // 我们可以将错误列表渲染为
                        // 字符串，如果我们愿意的话
                        <ul>
                            {move || errors.get()
                                .into_iter()
                                .map(|(_, e)| view! { <li>{e.to_string()}</li>})
                                .collect::<Vec<_>>()
                            }
                        </ul>
                    </div>
                }
            >
                <p>
                    "You entered "
                    // 因为 `value` 是 `Result<i32, _>`,
                    // 如果它是 `Ok`，它将渲染 `i32`，
                    // 如果它是 `Err`，它将渲染 nothing 并触发错误边界。
                    // 它是一个信号，因此当 `value` 发生变化时，它将动态更新
                    <strong>{value}</strong>
                </p>
            </ErrorBoundary>
        </label>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
