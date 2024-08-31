# `<Form/>` 组件

链接和表单有时看起来完全无关。但事实上，它们的工作方式非常相似。

在纯 HTML 中，有三种方法可以导航到另一个页面：

1. 链接到另一个页面的 `<a>` 元素：使用 `GET` HTTP 方法导航到其 `href` 属性中的 URL。
2. 一个 `<form method="GET">`：使用 `GET` HTTP 方法导航到其 `action` 属性中的 URL，并将来自其输入的表单数据编码在 URL 查询字符串中。
3. 一个 `<form method="POST">`：使用 `POST` HTTP 方法导航到其 `action` 属性中的 URL，并将来自其输入的表单数据编码在请求的正文中。

由于我们有一个客户端路由器，我们可以在不重新加载页面的情况下进行客户端链接导航，即无需完全往返服务器。我们也可以用同样的方式进行客户端表单导航，这是有道理的。

路由器提供了一个 [`<Form>`](https://docs.rs/leptos_router/latest/leptos_router/fn.Form.html) 组件，它的工作方式类似于 HTML `<form>` 元素，但使用客户端导航而不是完整的页面重新加载。`<Form/>` 适用于 `GET` 和 `POST` 请求。使用 `method="GET"`，它将导航到表单数据中编码的 URL。使用 `method="POST"`，它将发出 `POST` 请求并处理服务器的响应。

`<Form/>` 为我们将在后面的章节中看到的一些组件（如 `<ActionForm/>` 和 `<MultiActionForm/>`）奠定了基础。但它本身也支持一些强大的模式。

例如，假设你想要创建一个搜索字段，在用户搜索时实时更新搜索结果，而无需重新加载页面，但也将搜索内容存储在 URL 中，以便用户可以复制粘贴它以与他人共享结果。

事实证明，我们迄今为止学到的模式使得这很容易实现。

```rust
async fn fetch_results() {
	// 一些获取我们搜索结果的异步函数
}

#[component]
pub fn FormExample() -> impl IntoView {
    // 对 URL 查询字符串的响应式访问
    let query = use_query_map();
	// 存储为 ?q= 的搜索
    let search = move || query().get("q").cloned().unwrap_or_default();
	// 由搜索字符串驱动的资源
	let search_results = create_resource(search, fetch_results);

	view! {
		<Form method="GET" action="">
			<input type="search" name="q" value=search/>
			<input type="submit"/>
		</Form>
		<Transition fallback=move || ()>
			/* 渲染搜索结果 */
		</Transition>
	}
}
```

每当你点击“提交”时，`<Form/>` 都会“导航”到 `?q={search}`。但由于此导航是在客户端完成的，因此页面不会闪烁或重新加载。URL 查询字符串发生变化，这会触发 `search` 更新。因为 `search` 是 `search_results` 资源的源信号，所以这会触发 `search_results` 重新加载其资源。`<Transition/>` 会继续显示当前的搜索结果，直到新的搜索结果加载完成。当它们完成后，它将切换到显示新的结果。

这是一个很棒的模式。数据流非常清晰：所有数据都从 URL 流向资源，再流向 UI。应用程序的当前状态存储在 URL 中，这意味着你可以刷新页面或将链接发送给朋友，它将完全按照你的预期显示。一旦我们引入服务器端渲染，这种模式也将被证明是非常容错的：因为它在底层使用 `<form>` 元素和 URL，所以它实际上即使没有在客户端加载你的 WASM 也能很好地工作。

我们实际上可以更进一步，做一些聪明的事情：

```rust
view! {
	<Form method="GET" action="">
		<input type="search" name="q" value=search
			oninput="this.form.requestSubmit()"
		/>
	</Form>
}
```

你可能会注意到，此版本删除了“提交”按钮。相反，我们在输入框中添加了一个 `oninput` 属性。注意，这不是 `on:input`，它会监听 `input` 事件并运行一些 Rust 代码。没有冒号，`oninput` 就是普通的 HTML 属性。所以这个字符串实际上是一个 JavaScript 字符串。`this.form` 为我们提供了输入框所附加的表单。`requestSubmit()` 会触发 `<form>` 上的 `submit` 事件，这会被 `<Form/>` 捕获，就像我们点击了“提交”按钮一样。现在，表单会在每次按键或输入时“导航”，以使 URL（以及因此搜索）与用户输入的内容完全同步。

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/20-form-0-5-9g7v9p?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/20-form-0-5-9g7v9p?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
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
            <h1><code>"<Form/>"</code></h1>
            <main>
                <Routes>
                    <Route path="" view=FormExample/>
                </Routes>
            </main>
        </Router>
    }
}

#[component]
pub fn FormExample() -> impl IntoView {
    // 对 URL 查询的响应式访问
    let query = use_query_map();
    let name = move || query().get("name").cloned().unwrap_or_default();
    let number = move || query().get("number").cloned().unwrap_or_default();
    let select = move || query().get("select").cloned().unwrap_or_default();

    view! {
        // 读出 URL 查询字符串
        <table>
            <tr>
                <td><code>"name"</code></td>
                <td>{name}</td>
            </tr>
            <tr>
                <td><code>"number"</code></td>
                <td>{number}</td>
            </tr>
            <tr>
                <td><code>"select"</code></td>
                <td>{select}</td>
            </tr>
        </table>
        // <Form/> 将在每次提交时进行导航
        <h2>"Manual Submission"</h2>
        <Form method="GET" action="">
            // 输入名称决定查询字符串键
            <input type="text" name="name" value=name/>
            <input type="number" name="number" value=number/>
            <select name="select">
                // `selected` 将设置哪个开始时被选中
                <option selected=move || select() == "A">
                    "A"
                </option>
                <option selected=move || select() == "B">
                    "B"
                </option>
                <option selected=move || select() == "C">
                    "C"
                </option>
            </select>
            // 提交应该会导致客户端
            // 导航，而不是完全重新加载
            <input type="submit"/>
        </Form>
        // 这个 <Form/> 使用一些 JavaScript 在
        // 每次输入时提交
        <h2>"Automatic Submission"</h2>
        <Form method="GET" action="">
            <input
                type="text"
                name="name"
                value=name
                // 这个 oninput 属性将导致
                // 表单在每次输入到字段时提交
                oninput="this.form.requestSubmit()"
            />
            <input
                type="number"
                name="number"
                value=number
                oninput="this.form.requestSubmit()"
            />
            <select name="select"
                onchange="this.form.requestSubmit()"
            >
                <option selected=move || select() == "A">
                    "A"
                </option>
                <option selected=move || select() == "B">
                    "B"
                </option>
                <option selected=move || select() == "C">
                    "C"
                </option>
            </select>
            // 提交应该会导致客户端
            // 导航，而不是完全重新加载
            <input type="submit"/>
        </Form>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
