# 第一部分：构建用户界面

在本书的第一部分中，我们将介绍如何使用 Leptos 在客户端构建用户界面。 在底层，Leptos 和 Trunk 将打包一小段 JavaScript 代码，用于加载 Leptos UI，该 UI 已被编译为 WebAssembly，以驱动 CSR（客户端渲染）网站中的交互性。

第一部分将向你介绍构建由 Leptos 和 Rust 支持的响应式用户界面所需的基本工具。 到第一部分结束时，你应该能够构建一个快速的同步网站，该网站在浏览器中渲染，并且可以部署在任何静态网站托管服务上，例如 Github Pages 或 Vercel。

```admonish info
为了充分利用本书，我们鼓励你跟随提供的示例进行编码。
在 [入门](/cn/getting_started/) 和 [Leptos DX](/cn/getting_started/leptos_dx.html) 章节中，我们向你展示了如何使用 Leptos 和 Trunk 设置一个基本项目，包括在浏览器中处理 WASM 错误。
这个基本设置足以让你开始使用 Leptos 进行开发。

如果你更愿意使用功能更全面的模板来开始，该模板演示了如何设置你在真实的 Leptos 项目中会看到的一些基本内容，例如路由（将在本书后面介绍）、将 `<Title>` 和 `<Meta>` 标签注入页面头部以及其他一些细节，那么请随意使用 [leptos-rs `start-trunk`](https://github.com/leptos-rs/start-trunk) 模板仓库来启动并运行。

`start-trunk` 模板要求你安装了 `Trunk` 和 `cargo-generate`，你可以通过运行 `cargo install trunk` 和 `cargo install cargo-generate` 来获取它们。

要使用该模板设置你的项目，只需运行

`cargo generate --git https://github.com/leptos-community/start-csr`

然后在新建的应用程序目录中运行

`trunk serve --port 3000 --open`

即可开始开发你的应用程序。
Trunk 服务器将在文件更改时重新加载你的应用程序，从而使开发相对无缝。
```