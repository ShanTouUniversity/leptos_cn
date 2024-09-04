# `<ActionForm/>`

[`<ActionForm/>`](https://docs.rs/leptos_router/latest/leptos_router/fn.ActionForm.html) 是一个特殊的 `<Form/>`，它接受一个服务器操作，并在表单提交时自动调度它。这允许你直接从 `<form>` 调用服务器函数，即使没有 JS/WASM。

过程很简单：

1. 使用 [`#[server]` 宏](https://docs.rs/leptos/latest/leptos/attr.server.html) 定义一个服务器函数（参见 [服务器函数](../server/25_server_functions.md)）。
2. 使用 [`create_server_action`](https://docs.rs/leptos/latest/leptos/fn.create_server_action.html) 创建一个操作，指定你定义的服务器函数的类型。
3. 创建一个 `<ActionForm/>`，在 `action` prop 中提供服务器操作。
4. 将命名参数作为具有相同名称的表单字段传递给服务器函数。

> **注意：** `<ActionForm/>` 仅适用于服务器函数的默认 URL 编码的 `POST` 编码，以确保作为 HTML 表单的优雅降级/正确行为。

```rust
#[server(AddTodo, "/api")]
pub async fn add_todo(title: String) -> Result<(), ServerFnError> {
    todo!()
}

#[component]
fn AddTodo() -> impl IntoView {
	let add_todo = create_server_action::<AddTodo>();
	// 保存从服务器返回的最新值
	let value = add_todo.value();
	// 检查服务器是否返回了错误
	let has_error = move || value.with(|val| matches!(val, Some(Err(_))));

	view! {
		<ActionForm action=add_todo>
			<label>
				"Add a Todo"
				// `title` 与 `add_todo` 的 `title` 参数匹配
				<input type="text" name="title"/>
			</label>
			<input type="submit" value="Add"/>
		</ActionForm>
	}
}
```

真的就这么简单。使用 JS/WASM，你的表单将在没有页面重新加载的情况下提交，将其最近提交的内容存储在操作的 `.input()` 信号中，将其待处理状态存储在 `.pending()` 中，等等。（如果需要，请参阅 [`Action`](https://docs.rs/leptos/latest/leptos/struct.Action.html) 文档以进行复习。）如果没有 JS/WASM，你的表单将在页面重新加载时提交。如果你调用一个 `redirect` 函数（来自 `leptos_axum` 或 `leptos_actix`），它将重定向到正确的页面。默认情况下，它会重定向回你当前所在的页面。HTML、HTTP 和同构渲染的强大功能意味着你的 `<ActionForm/>` 可以正常工作，即使没有 JS/WASM。

## 客户端验证

因为 `<ActionForm/>` 只是一个 `<form>`，所以它会触发一个 `submit` 事件。你可以在 `on:submit` 中使用 HTML 验证或你自己的客户端验证逻辑。只需调用 `ev.prevent_default()` 即可防止提交。

[`FromFormData`](https://docs.rs/leptos_router/latest/leptos_router/trait.FromFormData.html) 特征在这里可能会有所帮助，用于尝试从提交的表单中解析你的服务器函数的数据类型。

```rust
let on_submit = move |ev| {
	let data = AddTodo::from_event(&ev);
	// 愚蠢的验证示例：如果待办事项是“nope!”，则不执行
	if data.is_err() || data.unwrap().title == "nope!" {
		// ev.prevent_default() 将阻止表单提交
		ev.prevent_default();
	}
}
```

## 复杂输入

作为具有嵌套可序列化字段的结构体的服务器函数参数，应使用 `serde_qs` 的索引符号。

```rust
use leptos::*;
use leptos_router::*;

#[derive(serde::Serialize, serde::Deserialize, Debug, Clone)]
struct HeftyData {
    first_name: String,
    last_name: String,
}

#[component]
fn ComplexInput() -> impl IntoView {
    let submit = Action::<VeryImportantFn, _>::server();

    view! {
      <ActionForm action=submit>
        <input type="text" name="hefty_arg[first_name]" value="leptos"/>
        <input
          type="text"
          name="hefty_arg[last_name]"
          value="closures-everywhere"
        />
        <input type="submit"/>
      </ActionForm>
    }
}

#[server]
async fn very_important_fn(
    hefty_arg: HeftyData,
) -> Result<(), ServerFnError> {
    assert_eq!(hefty_arg.first_name.as_str(), "leptos");
    assert_eq!(hefty_arg.last_name.as_str(), "closures-everywhere");
    Ok(())
}

```
