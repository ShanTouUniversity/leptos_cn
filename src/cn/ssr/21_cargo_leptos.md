# `cargo-leptos` 简介

到目前为止，我们只是在浏览器中运行代码，并使用 Trunk 来协调构建过程和运行本地开发过程。如果我们要添加服务器端渲染，我们还需要在服务器上运行我们的应用程序代码。这意味着我们需要构建两个独立的二进制文件，一个编译为本机代码并在服务器上运行，另一个编译为 WebAssembly (WASM) 并在用户的浏览器中运行。此外，服务器需要知道如何将此 WASM 版本（以及初始化它所需的 JavaScript）提供给浏览器。

这不是一项不可逾越的任务，但它增加了一些复杂性。为了方便起见和更好的开发体验，我们构建了 [`cargo-leptos`](https://github.com/leptos-rs/cargo-leptos) 构建工具。`cargo-leptos` 基本上是为了协调你的应用程序的构建过程，在进行更改时处理服务器和客户端两部分的重新编译，并添加对 Tailwind、SASS 和测试等内容的内置支持。

入门非常简单。只需运行

```bash
cargo install cargo-leptos
```

然后要创建一个新项目，你可以运行以下任一命令

```bash
# 对于 Actix 模板
cargo leptos new --git leptos-rs/start
```

或

```bash
# 对于 Axum 模板
cargo leptos new --git leptos-rs/start-axum
```

确保你已添加 wasm32-unknown-unknown 目标，以便 Rust 可以将你的代码编译为 WebAssembly 以在浏览器中运行。
```bash
rustup target add wasm32-unknown-unknown
```

现在 `cd` 到你创建的目录并运行

```bash
cargo leptos watch
```

> **注意**：请记住，Leptos 有一个 `nightly` feature，这些启动器都使用了它。如果你使用的是稳定的 Rust 编译器，
> 那没关系；只需从你的新 `Cargo.toml` 中删除每个 Leptos 依赖项中的 `nightly` feature，你就应该可以开始了。

你的应用程序编译完成后，你可以打开浏览器访问 [`http://localhost:3000`](http://localhost:3000) 来查看它。

`cargo-leptos` 有很多额外的功能和内置工具。你可以 [在其 `README` 中](https://github.com/leptos-rs/cargo-leptos/blob/main/README.md) 了解更多信息。

但是，当你打开浏览器访问 `localhost:3000` 时，到底发生了什么呢？好吧，请继续阅读以找出答案。
