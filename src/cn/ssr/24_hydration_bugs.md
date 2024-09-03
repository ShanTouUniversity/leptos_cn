# 水合错误（以及如何避免它们）

## 一个思想实验

让我们尝试一个实验来测试你的直觉。打开你使用 `cargo-leptos` 进行服务器端渲染的应用程序。（如果你到目前为止一直在使用 `trunk` 来玩示例，为了这个练习，请去[克隆一个 `cargo-leptos` 模板](./21_cargo_leptos.md)。）

在你的根组件的某个地方放置一个日志。（我通常称之为 `<App/>`，但任何东西都可以。）

```rust
#[component]
pub fn App() -> impl IntoView {
	logging::log!("我在哪里运行？");
	// ... 任何内容
}
```

让我们启动它

```bash
cargo leptos watch
```

你希望 `我在哪里运行？` 记录在哪里？

- 在你运行服务器的命令行中？
- 在你加载页面时的浏览器控制台中？
- 两者都不是？
- 两者都是？

试一试。

...

...

...

好的，考虑一下剧透警报。

你当然会注意到它在两个地方都记录了，假设一切按计划进行。实际上，它在服务器上记录了两次——第一次是在初始服务器启动期间，当 Leptos 渲染你的应用程序一次以提取路由树时，然后在你发出请求时第二次。每次重新加载页面时，`我在哪里运行？` 应该在服务器上记录一次，在客户端上记录一次。

如果你回想一下最后几节中的描述，希望这很有道理。你的应用程序在服务器上运行一次，它在那里构建一个 HTML 树，然后发送到客户端。在此初始渲染期间，`我在哪里运行？` 在服务器上记录。

一旦 WASM 二进制文件在浏览器中加载完成，你的应用程序将第二次运行，遍历同一个用户界面树并添加交互性。

> 这听起来像是一种浪费吗？从某种意义上说，确实如此。但减少这种浪费是一个真正困难的问题。这就是像 Qwik 这样的一些 JS 框架旨在解决的问题，尽管现在判断它与其他方法相比是否能带来净性能提升还为时过早。

## 错误的可能性

好的，希望所有这些都有意义。但它与本章标题“水合错误（以及如何避免它们）”有什么关系？

请记住，应用程序需要在服务器和客户端上运行。这会产生几组不同的潜在问题，你需要知道如何避免它们。

### 服务器和客户端代码之间的不匹配

创建错误的一种方法是在服务器发送的 HTML 和客户端渲染的内容之间创建不匹配。我认为这样做是相当困难的（至少从我收到的错误报告来看）。但想象一下，我做了这样的事情

```rust
#[component]
pub fn App() -> impl IntoView {
    let data = if cfg!(target_arch = "wasm32") {
        vec![0, 1, 2]
    } else {
        vec![]
    };
    data.into_iter()
        .map(|value| view! { <span>{value}</span> })
        .collect_view()
}
```

换句话说，如果它被编译为 WASM，它有三个项目；否则它是空的。

当我在浏览器中加载页面时，我什么也看不到。如果我打开控制台，我会看到一堆警告：

```
找不到 id 为 0-3 的元素，忽略它进行水合
找不到 id 为 0-4 的元素，忽略它进行水合
找不到 id 为 0-5 的元素，忽略它进行水合
找不到 id 为 _0-6c 的组件，忽略它进行水合
找不到 id 为 _0-6o 的组件，忽略它进行水合
```

在浏览器中运行的你的应用程序的 WASM 版本希望找到三个项目；但 HTML 中没有。

#### 解决方案

你很少会故意这样做，但它可能会在服务器和浏览器中以某种方式运行不同的逻辑时发生。如果你看到这样的警告，并且你认为这不是你的错，那么更有可能是 `<Suspense/>` 或其他东西的错误。请随意在 GitHub 上打开 [问题](https://github.com/leptos-rs/leptos/issues) 或 [讨论](https://github.com/leptos-rs/leptos/discussions) 以获得帮助。

### 并非所有客户端代码都可以在服务器上运行

想象一下，你很高兴地导入了一个像 `gloo-net` 这样的依赖项，你已经习惯于在浏览器中使用它来发出请求，并在服务器端渲染的应用程序中的 `create_resource` 中使用它。

你可能会立即看到可怕的消息

```
panicked at 'cannot call wasm-bindgen imported functions on non-wasm targets'
```

哦，哦。

但当然，这很有道理。我们刚刚说过，你的应用程序需要在客户端和服务器上运行。

#### 解决方案

有几种方法可以避免这种情况：

1. 仅使用可以在服务器和客户端上运行的库。例如，[`reqwest`](https://docs.rs/reqwest/latest/reqwest/) 适用于在两种环境中发出 HTTP 请求。
2. 在服务器和客户端上使用不同的库，并使用 `#[cfg]` 宏来区分它们。([点击这里查看示例](https://github.com/leptos-rs/leptos/blob/main/examples/hackernews/src/api.rs)。)
3. 将仅限客户端的代码包装在 `create_effect` 中。因为 `create_effect` 仅在客户端运行，所以这可以是访问初始渲染不需要的浏览器 API 的有效方法。

例如，假设我想在信号发生变化时将某些内容存储在浏览器的 `localStorage` 中。

```rust
#[component]
pub fn App() -> impl IntoView {
    use gloo_storage::Storage;
	let storage = gloo_storage::LocalStorage::raw();
	logging::log!("{storage:?}");
}
```

这会恐慌，因为我无法在服务器端渲染期间访问 `LocalStorage`。

但如果我把它包装在一个效果中...

```rust
#[component]
pub fn App() -> impl IntoView {
    use gloo_storage::Storage;
    create_effect(move |_| {
        let storage = gloo_storage::LocalStorage::raw();
		logging::log!("{storage:?}");
    });
}
```

就好了！这将在服务器上正确渲染，忽略仅限客户端的代码，然后在浏览器上访问存储并记录一条消息。

### 并非所有服务器代码都可以在客户端上运行

在浏览器中运行的 WebAssembly 是一个相当有限的环境。你无法访问文件系统或标准库可能习惯使用的许多其他东西。并非每个 crate 都可以编译为 WASM，更不用说在 WASM 环境中运行了。

特别是，你有时会看到有关 crate `mio` 的错误或 `core` 中缺少的东西。这通常表明你正在尝试将某些不能编译为 WASM 的内容编译为 WASM。如果你要添加仅限服务器的依赖项，你将希望在你的 `Cargo.toml` 中将它们标记为 `optional = true`，然后在 `ssr` 功能定义中启用它们。（查看其中一个模板 `Cargo.toml` 文件以查看更多详细信息。）

你可以使用 `create_effect` 来指定某些内容只能在客户端运行，而不能在服务器上运行。有没有办法指定某些内容只能在服务器上运行，而不能在客户端上运行？

实际上，有。下一章将详细介绍服务器函数的主题。（同时，你可以 [在此处](https://docs.rs/leptos_server/latest/leptos_server/index.html) 查看它们的文档。）
