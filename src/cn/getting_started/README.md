# 开始使用

开始使用 Leptos 有两种基本途径：

1. **使用 [Trunk](https://trunkrs.dev/) 进行客户端渲染 (CSR)** - 如果你只是想用 Leptos 创建一个简洁的网站，或者使用现有的服务器或 API，这是一个很好的选择。
在 CSR 模式下，Trunk 将你的 Leptos 应用程序编译为 WebAssembly (WASM)，并在浏览器中运行，就像典型的 Javascript 单页应用程序 (SPA) 一样。Leptos CSR 的优势包括更快的构建时间和更快的迭代开发周期，以及更简单的思维模型和更多的应用程序部署选项。CSR 应用程序也有一些缺点：与服务器端渲染方法相比，最终用户的初始加载时间较慢，并且使用 JS 单页应用程序模型带来的常见 SEO 挑战也适用于 Leptos CSR 应用程序。另外请注意，在底层，使用自动生成的 JS 代码段来加载 Leptos WASM 包，因此客户端设备上*必须*启用 JS 才能使 CSR 应用程序正确显示。与所有软件工程一样，这里也需要权衡利弊。

2. **使用 [`cargo-leptos`](https://github.com/leptos-rs/cargo-leptos) 的全栈、服务器端渲染 (SSR)** - 如果你希望 Rust 为你的前端和后端提供支持，那么 SSR 是构建 CRUD 风格网站和自定义 Web 应用程序的绝佳选择。
使用 Leptos SSR 选项，你的应用程序将在服务器上渲染为 HTML 并发送到浏览器；然后，使用 WebAssembly 来检测 HTML，使你的应用程序变得具有交互性 - 这个过程称为“hydration”。在服务器端，Leptos SSR 应用程序与你选择的 [Actix-web](https://docs.rs/leptos_actix/latest/leptos_actix/index.html) 或 [Axum](https://docs.rs/leptos_axum/latest/leptos_axum/index.html) 服务器库紧密集成，因此你可以利用这些社区的 crates 来帮助构建你的 Leptos 服务器。
选择 Leptos SSR 路线的优势包括帮助你获得最佳的初始加载时间和最佳的 Web 应用程序 SEO 分数。SSR 应用程序还可以通过 Leptos 的一项称为“服务器函数”的功能极大地简化跨服务器/客户端边界的工作，该功能允许你从客户端代码透明地调用服务器上的函数（稍后将详细介绍此功能）。然而，全栈 SSR 并非完美无缺 - 缺点包括较慢的开发者迭代循环（因为在进行 Rust 代码更改时需要重新编译服务器和客户端），以及 hydration 带来的额外复杂性。

到本书结束时，你应该能够根据项目的需求，很好地了解需要做出哪些权衡，以及选择哪条路线 - CSR 还是 SSR。


在本书的第一部分，我们将从客户端渲染 Leptos 网站开始，并使用 `Trunk` 将我们的 JS 和 WASM 包提供给浏览器，构建响应式 UI。

我们将在本书的第二部分介绍 `cargo-leptos`，该部分将全面介绍如何在全栈 SSR 模式下使用 Leptos 的全部功能。

```admonish note
如果你来自 JavaScript 世界，并且不熟悉客户端渲染 (CSR) 和服务器端渲染 (SSR) 等术语，那么理解它们之间区别的最简单方法是通过类比：

Leptos 的 CSR 模式类似于使用 React（或基于“信号”的框架，如 SolidJS），专注于生成客户端 UI，你可以将其与服务器上的任何技术栈一起使用。

使用 Leptos 的 SSR 模式类似于在 React 世界中使用全栈框架，如 Next.js（或 Solid 的“SolidStart”框架） - SSR 帮助你构建在服务器上渲染然后发送到客户端的网站和应用程序。SSR 可以帮助提高网站的加载性能和可访问性，还可以让一个人更容易在*客户端*和*服务器端*工作，而无需在前端和后端的不同语言之间进行上下文切换。

Leptos 框架既可以在 CSR 模式下使用，仅用于制作 UI（如 React），也可以在全栈 SSR 模式下使用（如 Next.js），以便你可以使用一种语言（Rust）构建 UI 和服务器。

```

## Hello World! Leptos CSR 开发环境搭建

首先，确保已安装 Rust 且为最新版本（[如果需要说明，请参见此处](https://www.rust-lang.org/tools/install)）。

如果你尚未安装 “Trunk” 工具，可以通过在命令行中运行以下命令来安装它，以便运行 Leptos CSR 网站：

```bash
cargo install trunk
```

然后创建一个基本的 Rust 项目

```bash
cargo init leptos-tutorial
```

`cd` 进入你的新 `leptos-tutorial` 项目，并将 `leptos` 添加为依赖项

```bash
cargo add leptos --features=csr,nightly
```

或者，如果你使用的是稳定的 Rust 版本，可以省略 `nightly`

```bash
cargo add leptos --features=csr
```

> 在 Leptos 中使用 `nightly` Rust 和 `nightly` 特性可以启用本书大多数部分使用的信号 getter 和 setter 的函数调用语法。
>
> 要使用 nightly Rust，你可以通过运行以下命令选择为所有 Rust 项目使用 nightly 版本
>
> ```bash
> rustup toolchain install nightly
> rustup default nightly
> ```
>
> 或者只针对此项目
>
> ```bash
> rustup toolchain install nightly
> cd <进入你的项目>
> rustup override set nightly
> ```
>
> [更多详细信息请参见此处。](https://doc.rust-lang.org/book/appendix-07-nightly-rust.html)
>
> 如果你更愿意在 Leptos 中使用稳定的 Rust 版本，你也可以这样做。在本指南和示例中，你只需使用 [`ReadSignal::get()`](https://docs.rs/leptos/latest/leptos/struct.ReadSignal.html#impl-SignalGet%3CT%3E-for-ReadSignal%3CT%3E) 和 [`WriteSignal::set()`](https://docs.rs/leptos/latest/leptos/struct.WriteSignal.html#impl-SignalGet%3CT%3E-for-ReadSignal%3CT%3E) 方法，而不是将信号 getter 和 setter 作为函数调用。

确保你已添加 `wasm32-unknown-unknown` 目标，以便 Rust 可以将你的代码编译为 WebAssembly 以在浏览器中运行。

```bash
rustup target add wasm32-unknown-unknown
```

在 `leptos-tutorial` 目录的根目录下创建一个简单的 `index.html` 文件

```html
<!DOCTYPE html>
<html>
  <head></head>
  <body></body>
</html>
```

并在你的 `main.rs` 中添加一个简单的 “Hello, world!”

```rust
use leptos::*;

fn main() {
    mount_to_body(|| view! { <p>"Hello, world!"</p> })
}
```

你的目录结构现在应该如下所示

```
leptos_tutorial
├── src
│   └── main.rs
├── Cargo.toml
├── index.html
```

现在从 `leptos-tutorial` 目录的根目录运行 `trunk serve --open`。
Trunk 应该会自动编译你的应用程序并在你的默认浏览器中打开它。
如果你对 `main.rs` 进行编辑，Trunk 将重新编译你的源代码并实时重新加载页面。

欢迎来到由 Leptos 和 Trunk 支持的 Rust 和 WebAssembly (WASM) UI 开发世界！

```admonish note
如果你使用的是 Windows 系统，请注意 `trunk serve --open` 可能无法正常工作。 如果你在使用 `--open` 时遇到问题，
只需使用 `trunk serve` 并手动打开浏览器标签页即可。
```

---

在开始使用 Leptos 构建你的第一个真正的 UI 之前，你需要了解一些事情，这些事情可以帮助你更轻松地使用 Leptos。
