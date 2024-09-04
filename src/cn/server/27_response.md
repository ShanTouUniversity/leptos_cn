# 响应和重定向

提取器提供了一种简单的方法来访问服务器函数内部的请求数据。Leptos 还提供了一种使用 `ResponseOptions` 类型（请参阅 [Actix](https://docs.rs/leptos_actix/latest/leptos_actix/struct.ResponseOptions.html) 或 [Axum](https://docs.rs/leptos_axum/latest/leptos_axum/struct.ResponseOptions.html) 的文档）和 `redirect` 帮助函数（请参阅 [Actix](https://docs.rs/leptos_actix/latest/leptos_actix/fn.redirect.html) 或 [Axum](https://docs.rs/leptos_axum/latest/leptos_axum/fn.redirect.html) 的文档）来修改 HTTP 响应的方法。

## `ResponseOptions`

`ResponseOptions` 在初始服务器渲染响应期间和任何后续服务器函数调用期间通过上下文提供。它允许你轻松地设置 HTTP 响应的状态码，或向 HTTP 响应添加标头，例如设置 cookie。

```rust
#[server(TeaAndCookies)]
pub async fn tea_and_cookies() -> Result<(), ServerFnError> {
	use actix_web::{cookie::Cookie, http::header, http::header::HeaderValue};
	use leptos_actix::ResponseOptions;

	// 从上下文中提取 ResponseOptions
	let response = expect_context::<ResponseOptions>();

	// 设置 HTTP 状态码
	response.set_status(StatusCode::IM_A_TEAPOT);

	// 在 HTTP 响应中设置一个 cookie
	let mut cookie = Cookie::build("biscuits", "yes").finish();
	if let Ok(cookie) = HeaderValue::from_str(&cookie.to_string()) {
		response.insert_header(header::SET_COOKIE, cookie);
	}
}
```

## `redirect`

对 HTTP 响应的一种常见修改是重定向到另一个页面。Actix 和 Axum 集成提供了一个 `redirect` 函数来简化此操作。`redirect` 只需设置 HTTP 状态码 `302 Found` 并设置 `Location` 标头即可。

以下是一个来自我们的 [`session_auth_axum` 示例](https://github.com/leptos-rs/leptos/blob/a5f73b441c079f9138102b3a7d8d4828f045448c/examples/session_auth_axum/src/auth.rs#L154-L181) 的简化示例。

```rust
#[server(Login, "/api")]
pub async fn login(
    username: String,
    password: String,
    remember: Option<String>,
) -> Result<(), ServerFnError> {
	// 从上下文中提取数据库池和身份验证提供程序
    let pool = pool()?;
    let auth = auth()?;

	// 检查用户是否存在
    let user: User = User::get_from_username(username, &pool)
        .await
        .ok_or_else(|| {
            ServerFnError::ServerError("User does not exist.".into())
        })?;

	// 检查用户是否提供了正确的密码
    match verify(password, &user.password)? {
		// 如果密码正确...
        true => {
			// 登录用户
            auth.login_user(user.id);
            auth.remember_user(remember.is_some());

			// 并重定向到主页
            leptos_axum::redirect("/");
            Ok(())
        }
		// 如果不正确，则返回错误
        false => Err(ServerFnError::ServerError(
            "Password does not match.".to_string(),
        )),
    }
}
```

然后可以从你的应用程序中使用此服务器函数。此 `redirect` 与渐进增强的 `<ActionForm/>` 组件配合良好：如果没有 JS/WASM，服务器响应将因状态码和标头而重定向。使用 JS/WASM，`<ActionForm/>` 将检测服务器函数响应中的重定向，并使用客户端导航重定向到新页面。
