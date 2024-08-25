# 一个基础组件

那个“Hello, world!”是一个*非常*简单的例子。 让我们继续学习更像普通应用程序的内容。

首先，让我们编辑 `main` 函数，使其不再渲染整个应用程序，而只是渲染一个 `<App/>` 组件。 组件是大多数 Web 框架中组合和设计的基本单元，Leptos 也不例外。
从概念上讲，它们类似于 HTML 元素：它们代表 DOM 的一部分，具有独立的、定义的行为。 与 HTML 元素不同，它们采用 `PascalCase` 形式，因此大多数 Leptos 应用程序都将以 `<App/>` 组件之类的内容开头。

```rust
fn main() {
    leptos::mount_to_body(|| view! { <App/> })
}
```

现在让我们定义 `<App/>` 组件本身。 因为它相对简单，
我将先完整地展示它，然后逐行解释。

```rust
#[component]
fn App() -> impl IntoView {
    let (count, set_count) = create_signal(0);

    view! {
        <button
            on:click=move |_| {
                // 在稳定版本中，这是 set_count.set(3);
                set_count(3);
            }
        >
            "Click me: "
            // 在稳定版本中，这是 move || count.get();
            {move || count()}
        </button>
    }
}
```

## 组件签名

```rust
#[component]
```

与所有组件定义一样，这以 [`#[component]`](https://docs.rs/leptos/latest/leptos/attr.component.html) 宏开头。 `#[component]` 注释一个函数，以便它可以
在你的 Leptos 应用程序中用作组件。 我们将在接下来的几章中看到此宏的其他一些功能。

```rust
fn App() -> impl IntoView
```

每个组件都是具有以下特征的函数

1. 它接受零个或多个任何类型的参数。
2. 它返回 `impl IntoView`，这是一个不透明类型，包括
   你可以从 Leptos `view` 中返回的任何内容。

> 组件函数参数被收集到一个由 `view` 宏根据需要构建的 props 结构体中。

## 组件主体

组件函数的主体是一个只运行一次的设置函数，而不是一个多次重新运行的渲染函数。 你通常会使用它来创建一些响应式变量，定义任何响应这些值变化而运行的副作用，以及描述用户界面。

```rust
let (count, set_count) = create_signal(0);
```

[`create_signal`](https://docs.rs/leptos/latest/leptos/fn.create_signal.html)
创建一个信号，它是 Leptos 中响应式变化和状态管理的基本单元。
这将返回一个 `(getter, setter)` 元组。 要访问当前值，你将使用 `count.get()`（或者，在 `nightly` Rust 上，使用简写 `count()`）。 要设置当前值，你将调用 `set_count.set(...)`（或者 `set_count(...)`）。

> `.get()` 克隆值，`.set()` 覆盖它。 在许多情况下，使用 `.with()` 或 `.update()` 更有效率；如果你现在想了解更多关于这些权衡的信息，请查看 [`ReadSignal`](https://docs.rs/leptos/latest/leptos/struct.ReadSignal.html) 和 [`WriteSignal`](https://docs.rs/leptos/latest/leptos/struct.WriteSignal.html) 的文档。

## 视图

Leptos 使用类似 JSX 的格式通过 [`view`](https://docs.rs/leptos/latest/leptos/macro.view.html) 宏定义用户界面。

```rust
view! {
    <button
        // 使用 on: 定义事件监听器
        on:click=move |_| {
            set_count(3);
        }
    >
        // 文本节点用引号括起来
        "Click me: "
        // 代码块可以包含 Rust 代码
        {move || count()}
    </button>
}
```

这应该很容易理解：它看起来像 HTML，带有一个特殊的 `on:click` 来定义 `click` 事件监听器，一个格式化像 Rust 字符串的文本节点，然后是...

```rust
{move || count()}
```

无论那是什么。

人们有时会开玩笑说，他们在他们的第一个 Leptos 应用程序中使用的闭包比他们一生中使用的任何时候都多。 这很公平。 基本上，将一个函数传递给视图会告诉框架：“嘿，这是一个可能会改变的东西。”

当我们点击按钮并调用 `set_count` 时，`count` 信号会被更新。 这个
`move || count()` 闭包，它的值依赖于 `count` 的值，会重新运行，
框架会对那个特定的文本节点进行有针对性的更新，而不会触及应用程序中的任何其他内容。 这就是允许对 DOM 进行极其高效的更新的原因。

现在，如果你打开了 Clippy——或者你有一双特别敏锐的眼睛——你可能会注意到
这个闭包是多余的，至少在 `nightly` Rust 中是这样。 如果你在 `nightly` Rust 中使用
Leptos，信号已经是函数了，所以闭包是不必要的。
因此，你可以编写一个更简单的视图：

```rust
view! {
    <button /* ... */>
        "Click me: "
        // 与 {move || count()} 相同
        {count}
    </button>
}
```

记住——这*非常重要*——只有函数是响应式的。 这意味着
`{count}` 和 `{count()}` 在你的视图中做的事情非常不同。 `{count}` 传递
一个函数，告诉框架每次 `count` 改变时都要更新视图。
`{count()}` 只访问一次 `count` 的值，并将一个 `i32` 传递给视图，
渲染一次，非响应式。 你可以在下面的 CodeSandbox 中看到区别！

让我们做最后一个改变。 `set_count(3)` 对于点击处理程序来说是一个非常无用的操作。 让我们将“将此值设置为 3”替换为“将此值递增 1”：

```rust
move |_| {
    set_count.update(|n| *n += 1);
}
```

你可以在这里看到，虽然 `set_count` 只是设置值，但 `set_count.update()` 为我们提供了一个可变引用并在原地修改值。 两者都会触发我们 UI 中的响应式更新。

> 在整个教程中，我们将使用 CodeSandbox 来展示交互式示例。
> 将鼠标悬停在任何变量上以显示 Rust-Analyzer 详细信息
> 以及正在发生的事情的文档。 随意 fork 示例自己尝试！

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox。](https://codesandbox.io/p/sandbox/1-basic-component-3d74p3?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  请启用 JavaScript 以查看示例。
</noscript>

> 要在沙盒中显示浏览器，你可能需要点击 `添加开发者工具 >
其他预览 > 8080。`

<template>
  <iframe src="https://codesandbox.io/p/sandbox/1-basic-component-3d74p3?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源代码</summary>

```rust
use leptos::*;

// #[component] 宏将函数标记为可重用组件
// 组件是用户界面的构建块
// 它们定义了一个可重用的行为单元
#[component]
fn App() -> impl IntoView {
    // 在这里我们创建一个响应式信号
    // 并获取一个 (getter, setter) 对
    // 信号是框架中变化的基本单元
    // 我们稍后会详细讨论它们
    let (count, set_count) = create_signal(0);

    // `view` 宏是我们定义用户界面的方式
    // 它使用类似 HTML 的格式，可以接受某些 Rust 值
    view! {
        <button
            // on:click 将在每次 `click` 事件触发时运行
            // 每个事件处理程序都定义为 `on:{eventname}`

            // 我们能够将 `set_count` 移入闭包中
            // 因为信号是 Copy 和 'static
            on:click=move |_| {
                set_count.update(|n| *n += 1);
            }
        >
            // RSX 中的文本节点应该用引号括起来，
            // 就像普通的 Rust 字符串一样
            "Click me"
        </button>
        <p>
            <strong>"响应式: "</strong>
            // 你可以通过将 Rust 表达式括在大括号中来将它们作为值插入 DOM 中
            // 如果你传入一个函数，它将进行响应式更新
            {move || count()}
        </p>
        <p>
            <strong>"响应式简写: "</strong>
            // 信号是函数，所以我们可以移除包装闭包
            {count}
        </p>
        <p>
            <strong>"非响应式: "</strong>
            // 注意：如果你写 {count()}，这将*不会*是响应式的
            // 它只是获取 count 的值一次
            {count()}
        </p>
    }
}

// 这个 `main` 函数是应用程序的入口点
// 它只是将我们的组件挂载到 <body> 上
// 因为我们将其定义为 `fn App`，所以我们现在可以在
// 模板中将其用作 <App/>
fn main() {
    leptos::mount_to_body(|| view! { <App/> })
}
```
</details>