# 与 JavaScript 集成：`wasm-bindgen`、`web_sys` 和 `HtmlElement`

Leptos 提供了各种工具，让你无需离开框架的世界就可以构建声明式的 Web 应用程序。诸如响应式系统、`component` 和 `view` 宏以及路由器之类的东西使你无需直接与浏览器提供的 Web API 交互即可构建用户界面。并且它们让你直接在 Rust 中完成所有这些工作，这很棒——假设你喜欢 Rust。（如果你已经读到本书的这一部分，我们假设你喜欢 Rust。）

像 [`leptos-use`](https://leptos-use.rs/) 提供的奇妙实用程序集之类的生态系统 crate 可以让你走得更远，通过为许多 Web API 提供特定于 Leptos 的响应式包装器。

但是，在许多情况下，你需要直接访问 JavaScript 库或 Web API。本章可以提供帮助。

## 使用 `wasm-bindgen` 使用 JS 库

你的 Rust 代码可以编译为 WebAssembly (WASM) 模块并加载到浏览器中运行。但是，WASM 无法直接访问浏览器 API。相反，Rust/WASM 生态系统依赖于从你的 Rust 代码到托管它的 JavaScript 浏览器环境生成绑定。

[`wasm-bindgen`](https://rustwasm.github.io/docs/wasm-bindgen/) crate 是该生态系统的核心。它提供了一个接口，用于使用注释标记 Rust 代码的各个部分，告诉它如何调用 JS，以及一个用于生成必要的 JS 粘合代码的 CLI 工具。你一直在不知不觉中使用它：`trunk` 和 `cargo-leptos` 都在底层依赖 `wasm-bindgen`。

如果你想从 Rust 中调用 JavaScript 库，你应该参考 `wasm-bindgen` 文档中关于 [从 JS 导入函数](https://rustwasm.github.io/docs/wasm-bindgen/examples/import-js.html) 的部分。从 JavaScript 导入单个函数、类或值以在你的 Rust 应用程序中使用相对容易。

将 JS 库直接集成到你的应用程序中并不总是那么容易。特别是，任何依赖于像 React 这样的特定 JS 框架的库都可能难以集成。还应谨慎使用以某种方式操作 DOM 状态的库（例如，富文本编辑器）：Leptos 和 JS 库都可能假设它们是应用程序状态的最终事实来源，因此你应该小心地分离它们的职责。

## 使用 `web-sys` 访问 Web API

如果你只需要访问一些浏览器 API 而无需引入单独的 JS 库，则可以使用 [`web_sys`](https://docs.rs/web-sys/latest/web_sys/) crate 来实现。它提供了浏览器提供的所有 Web API 的绑定，从浏览器类型和函数到 Rust 结构体和方法的 1:1 映射。

通常，如果你问“我如何使用 Leptos*执行 X*？”，其中*执行 X* 是访问某个 Web API，那么查找普通 JavaScript 解决方案并使用 [`web-sys` 文档](https://docs.rs/web-sys/latest/web_sys/) 将其翻译成 Rust 是一个好方法。


> 阅读完本节后，你可能会发现 [关于 `web-sys` 的 `wasm-bindgen` 指南章节](https://rustwasm.github.io/docs/wasm-bindgen/web-sys/index.html) 对于进一步阅读很有用。

### 启用功能

`web_sys` 的功能被大量分隔，以保持较低的编译时间。如果你想使用它的许多 API 之一，你可能需要启用一个功能才能使用它。

使用项目所需的功能始终在其文档中列出。
例如，要使用 [`Element::get_bounding_rect_client`](https://docs.rs/web-sys/latest/web_sys/struct.Element.html#method.get_bounding_client_rect)，你需要启用 `DomRect` 和 `Element` 功能。

Leptos 已经启用了 [一大堆](https://github.com/leptos-rs/leptos/blob/main/leptos_dom/Cargo.toml#L41) 功能——如果所需的功能已在此处启用，则你无需在自己的应用程序中启用它。
否则，将其添加到你的 `Cargo.toml` 中，就可以开始了！

```toml
[dependencies.web-sys]
version = "0.3"
features = ["DomRect"]
```

但是，随着 JavaScript 标准的演进和 API 的编写，你可能希望使用技术上尚不完全稳定的浏览器功能，例如 [WebGPU](https://docs.rs/web-sys/latest/web_sys/struct.Gpu.html)。
`web_sys` 将遵循（可能经常更改的）标准，这意味着不保证稳定性。

为了使用它，你需要添加 `RUSTFLAGS=--cfg=web_sys_unstable_apis` 作为环境变量。
这可以通过将其添加到每个命令中来完成，也可以将其添加到存储库中的 `.cargo/config.toml` 中来完成。

作为命令的一部分：
```sh
RUSTFLAGS=--cfg=web_sys_unstable_apis cargo # ...
```

在 `.cargo/config.toml` 中：
```toml
[env]
RUSTFLAGS = "--cfg=web_sys_unstable_apis"
```

## 从你的 `view` 中访问原始 `HtmlElement`

框架的声明式风格意味着你不需要直接操作 DOM 节点来构建你的用户界面。
但是，在某些情况下，你希望直接访问表示视图一部分的底层 DOM 元素。本书关于 [“不受控制的输入”](/view/05_forms.html?highlight=NodeRef#uncontrolled-inputs) 的部分介绍了如何使用 [`NodeRef`](https://docs.rs/leptos/latest/leptos/struct.NodeRef.html) 类型来实现此目的。

你可能会注意到 [`NodeRef::get`](https://docs.rs/leptos/latest/leptos/struct.NodeRef.html#method.get) 返回一个 `Option<leptos::HtmlElement<T>>`。这与 [`web_sys::HtmlElement`](https://docs.rs/web-sys/latest/web_sys/struct.HtmlElement.html) *不是*同一种类型，尽管它们是相关的。那么这个 [`HtmlElement<T>`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html) 类型是什么，你如何使用它呢？

### 概述

`web_sys::HtmlElement` 是浏览器 [`HTMLElement`](https://developer.mozilla.org/en-US/docs/Web/API/HTMLElement) 接口的 Rust 等效项，该接口为所有 HTML 元素实现。它提供了对保证可用于任何 HTML 元素的一组最少函数和 API 的访问权限。然后，每个特定的 HTML 元素都有自己的元素类，该类实现额外的功能。
`leptos::HtmlElement<T>` 的目标是弥合视图中的元素与这些更具体的 JavaScript 类型之间的差距，以便你可以访问这些元素的特定功能。

这是通过使用 Rust `Deref` 特征来实现的，该特征允许你将 `leptos::HtmlElement<T>` 解引用为该特定元素类型 `T` 的适当类型的 JS 对象。

### 定义

理解这种关系涉及理解一些相关的特征。

以下内容简单地定义了 `leptos::HtmlElement<T>` 的 `T` 中允许的类型，以及它如何链接到 `web_sys`。

```rust
pub struct HtmlElement<El> where El: ElementDescriptor { /* ... */ }

pub trait ElementDescriptor: ElementDescriptorBounds { /* ... */ }

pub trait ElementDescriptorBounds: Debug {}
impl<El> ElementDescriptorBounds for El where El: Debug {}

// 这为 `leptos::{html, svg, math}::*` 中的每个元素实现
impl ElementDescriptor for leptos::html::Div { /* ... */ }

// 与此相同，解引用到相应的 `web_sys::Html*Element`
impl Deref for leptos::html::Div {
    type Target = web_sys::HtmlDivElement;
    // ...
}
```

以下内容来自 `web_sys`：
```rust
impl Deref for web_sys::HtmlDivElement {
    type Target = web_sys::HtmlElement;
    // ...
}

impl Deref for web_sys::HtmlElement {
    type Target = web_sys::Element;
    // ...
}

impl Deref for web_sys::Element {
    type Target = web_sys::Node;
    // ...
}

impl Deref for web_sys::Node {
    type Target = web_sys::EventTarget;
    // ...
}
```

`web_sys` 使用长的解引用链来模拟 JavaScript 中使用的继承。
如果你在一种类型上找不到你要找的方法，请在解引用链中进一步查看。
`leptos::html::*` 类型都解引用到 `web_sys::Html*Element` 或 `web_sys::HtmlElement`。
通过调用 `element.method()`，Rust 将根据需要自动添加更多解引用以调用正确的方法！

但是，有些方法具有相同的名称，例如 [`leptos::HtmlElement::style`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.style) 和 [`web_sys::HtmlElement::style`](https://docs.rs/web-sys/latest/web_sys/struct.HtmlElement.html#method.style)。
在这种情况下，Rust 将选择需要最少解引用的方法，如果你直接从 `NodeRef` 获取元素，则为 `leptos::HtmlElement::style`。
如果你想改用 `web_sys` 方法，则可以使用 `(*element).style()` 手动解引用。

如果你想对从哪个类型调用方法进行更多控制，则为解引用链中所有类型都实现了 `AsRef<T>`，因此你可以明确说明你想要的类型。

> 另请参阅：[`wasm-bindgen` 指南：`web-sys` 中的继承](https://rustwasm.github.io/wasm-bindgen/web-sys/inheritance.html)。

### 克隆

`web_sys::HtmlElement`（以及扩展的 `leptos::HtmlElement`）实际上只存储对其影响的 HTML 元素的引用。
因此，调用 `.clone()` 实际上并不会创建一个新的 HTML 元素，它只是获取对同一个元素的另一个引用。
从其任何克隆中调用更改元素的方法将影响原始元素。

不幸的是，`web_sys::HtmlElement` 没有实现 `Copy`，因此你可能需要添加一堆克隆，尤其是在闭包中使用它时。
不过别担心，这些克隆很便宜！

### 转换

你可以通过 `Deref` 或 `AsRef` 获取不太具体的类型，因此尽可能使用它们。
但是，如果你需要转换为更具体的类型（例如，从 `EventTarget` 转换为 `HtmlInputElement`），则需要使用 `wasm_bindgen::JsCast` 提供的方法（通过 `web_sys::wasm_bindgen::JsCast` 重新导出）。
你可能只需要 [`dyn_ref`](https://docs.rs/wasm-bindgen/0.2.90/wasm_bindgen/trait.JsCast.html#method.dyn_ref) 方法。

```rust
use web_sys::wasm_bindgen::JsCast;

let on_click = |ev: MouseEvent| {
    let target: HtmlInputElement = ev.current_target().unwrap().dyn_ref().unwrap();
    // 或者，只需使用现有的 `leptos::event_target_*` 函数
}
```

> 如果你好奇的话，[请在此处查看 `event_target_*` 函数](https://docs.rs/leptos/latest/leptos/fn.event_target.html?search=event_target)。

### `leptos::HtmlElement`

[`leptos::HtmlElement`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html) 添加了一些额外的便捷方法，以便于操作常用属性。
这些方法是为 [构建器语法](./view/builder.md) 构建的，因此它接受并返回 `self`。
你可以只执行 `_ = element.clone().<method>()` 来忽略它返回的元素 - 它仍然会影响原始元素，即使它看起来不像（请参阅上一节关于 [克隆](#clones)）！

以下是一些你可能想使用的方法，例如在事件监听器或 `use:` 指令中。
- [`id`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.id)：*覆盖* 元素上的 id。
- [`classes`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.classes)：*添加* 元素的类。
    你可以使用空格分隔的字符串指定多个类。
    你还可以使用 [`class`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.class) 有条件地添加*单个*类：不要使用此方法添加多个类。
- [`attr`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.attr)：为元素设置一个 `key=value` 属性。
- [`prop`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.prop)：在元素上设置一个*属性*：请参阅 [此处属性和属性之间的区别](./view/05_forms.md#why-do-you-need-propvalue)。
- [`on`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.on)：向元素添加事件监听器。
    通过 [`leptos::ev::*`](https://docs.rs/leptos/latest/leptos/ev/index.html) 之一指定事件类型（它是所有小写的类型）。
- [`child`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.child)：将一个元素添加为该元素的最后一个子元素。

也请查看 [`leptos::HtmlElement`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html) 的其余方法。如果它们都不符合你的要求，还可以查看 [`leptos-use`](https://leptos-use.rs/)。否则，你将不得不使用 `web_sys` API。
