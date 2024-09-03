# 第二部分：服务器端渲染

本书的第二部分是关于如何将你漂亮的 UI 变成全栈 Rust + Leptos 驱动的网站和应用程序。

正如你在上一章中读到的，使用客户端渲染的 Leptos 应用程序有一些限制——在接下来的几章中，你将看到我们如何克服这些限制，并从你的 Leptos 应用程序中获得最佳性能和 SEO。


```admonish info

在服务器端使用 Leptos 时，你可以自由选择 Actix-web 或 Axum 集成——Leptos 的全部功能集在任何一个选项中都可用。

但是，如果你需要部署到与 WinterCG 兼容的运行时（如 Deno、Cloudflare 等），那么请选择 Axum 集成，因为此部署选项仅适用于服务器上的 Axum。最后，如果你想使用全栈 WASM/WASI 并部署到基于 WASM 的无服务器运行时，那么 Axum 也是你的首选。

注意：这是 Web 框架本身的限制，而不是 Leptos 的限制。

```