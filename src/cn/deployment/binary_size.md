# 优化 WASM 二进制文件大小

部署 Rust/WebAssembly 前端应用程序的主要缺点之一是将 WASM 文件拆分成更小的块以动态加载比拆分 JavaScript 包要困难得多。在 Emscripten 生态系统中已经有一些实验，例如 [`wasm-split`](https://emscripten.org/docs/optimizing/Module-Splitting.html)，但目前还没有办法拆分和动态加载 Rust/`wasm-bindgen` 二进制文件。这意味着需要加载整个 WASM 二进制文件后，你的应用程序才能进行交互。由于 WASM 格式是为流式编译而设计的，因此与 JavaScript 文件相比，WASM 文件每千字节的编译速度要快得多。（要深入了解，你可以[阅读 Mozilla 团队的这篇精彩文章](https://hacks.mozilla.org/2018/01/making-webassembly-even-faster-firefoxs-new-streaming-and-tiering-compiler/)，了解流式 WASM 编译。）

尽管如此，将最小的 WASM 二进制文件发送给用户仍然很重要，因为它会减少他们的网络使用量并使你的应用程序尽快进行交互。

那么有哪些实际步骤呢？

## 要做的事情

1. 确保你正在查看发布版本。（调试版本要大得多。）
2. 为 WASM 添加一个发布配置文件，该配置文件针对大小进行优化，而不是速度。

例如，对于 `cargo-leptos` 项目，你可以将此添加到你的 `Cargo.toml` 中：

```toml
[profile.wasm-release]
inherits = "release"
opt-level = 'z'
lto = true
codegen-units = 1

# ....

[package.metadata.leptos]
# ....
lib-profile-release = "wasm-release"
```

这将针对大小对你的发布版本的 WASM 进行超优化，同时保持你的服务器版本针对速度进行优化。（对于没有服务器考虑的纯客户端渲染应用程序，只需使用 `[profile.wasm-release]` 块作为你的 `[profile.release]`。）

3. 始终在生产环境中提供压缩的 WASM。WASM 往往压缩得很好，通常会缩小到其未压缩大小的 50% 以下，并且为从 Actix 或 Axum 提供的静态文件启用压缩非常简单。

4. 如果你使用的是 nightly Rust，你可以使用相同的配置文件而不是随 `wasm32-unknown-unknown` 目标一起分发的预构建标准库来重建标准库。

为此，请在你的项目的 `.cargo/config.toml` 中创建一个文件

```toml
[unstable]
build-std = ["std", "panic_abort", "core", "alloc"]
build-std-features = ["panic_immediate_abort"]
```

请注意，如果你也将此用于 SSR，则将应用相同的 Cargo 配置文件。你需要明确指定你的目标：
```toml
[build]
target = "x86_64-unknown-linux-gnu" # 或其他任何内容
```

还要注意，在某些情况下，不会设置 cfg 功能 `has_std`，这可能会导致某些依赖项的构建错误，这些依赖项会检查 `has_std`。你可以通过添加以下内容来修复由此导致的任何构建错误：
```toml
[build]
rustflags = ["--cfg=has_std"]
```

你需要在 `Cargo.toml` 的 `[profile.release]` 中添加 `panic = "abort"`。请注意，这会将相同的 `build-std` 和恐慌设置应用于你的服务器二进制文件，这可能不是你想要的。这里可能需要进一步探索。

5. WASM 二进制文件中二进制大小的来源之一可能是 `serde` 序列化/反序列化代码。Leptos 默认使用 `serde` 来序列化和反序列化使用 `create_resource` 创建的资源。你可以尝试使用 `miniserde` 和 `serde-lite` 功能，它们允许你使用这些 crate 代替进行序列化和反序列化；它们都只实现了 `serde` 功能的一个子集，但通常针对大小而不是速度进行优化。

## 要避免的事情

有些 crate 往往会增加二进制文件的大小。例如，具有默认功能的 `regex` crate 会为 WASM 二进制文件增加约 500kb（主要是因为它必须引入 Unicode 表数据！）。在大小敏感的环境中，你可能会考虑一般避免使用正则表达式，甚至放弃并调用浏览器 API 来使用内置的正则表达式引擎。（这就是 `leptos_router` 在需要正则表达式的少数情况下所做的事情。）

通常，Rust 对运行时性能的承诺有时与对小型二进制文件的承诺不一致。例如，Rust 会对泛型函数进行单态化，这意味着它会为它调用的每个泛型类型创建一个不同的函数副本。这比动态调度快得多，但会增加二进制文件的大小。Leptos 尝试非常谨慎地平衡运行时性能和二进制文件大小的考虑；但你可能会发现，编写使用许多泛型的代码往往会增加二进制文件的大小。例如，如果你有一个在其主体中包含大量代码的泛型组件，并使用四种不同的类型调用它，请记住编译器可能包含该代码的四个副本。重构以使用具体的内部函数或辅助函数通常可以保持性能和人体工程学，同时减小二进制文件的大小。

## 最后的想法

请记住，在服务器端渲染的应用程序中，JS 包大小/WASM 二进制文件大小仅影响_一_件事：首次加载时的交互时间。这对良好的用户体验非常重要：没有人希望点击一个按钮三次而它什么也不做，因为交互式代码仍在加载——但这并不是唯一重要的指标。

特别值得记住的是，流式传输单个 WASM 二进制文件意味着所有后续导航几乎都是瞬时的，仅取决于任何额外的数据加载。正是因为你的 WASM 二进制文件*未*进行包拆分，所以导航到新路由不需要加载额外的 JS/WASM，这与几乎所有 JavaScript 框架中的情况一样。这是在自我安慰吗？也许。或者，这可能只是两种方法之间的一种诚实的权衡！

始终抓住机会优化应用程序中唾手可得的成果。并且在做出任何英勇的努力之前，始终在真实环境下使用真实的用户网络速度和设备来测试你的应用程序。
