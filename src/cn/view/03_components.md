# 组件和 Props

到目前为止，我们一直在单个组件中构建整个应用程序。这对于非常小的例子来说是可以的，但在任何实际的应用程序中，你都需要将用户界面分解成多个组件，这样你就可以将界面分解成更小、可重用、可组合的块。

让我们以进度条为例。假设你想要两个进度条而不是一个：一个每次点击前进一个刻度，一个每次点击前进两个刻度。

你 _可以_ 通过创建两个 `<progress>` 元素来做到这一点：

```rust
let (count, set_count) = create_signal(0);
let double_count = move || count() * 2;

view! {
    <progress
        max="50"
        value=count
    />
    <progress
        max="50"
        value=double_count
    />
}
```

但是当然，这不能很好地扩展。如果你想添加第三个进度条，你需要再次添加这段代码。如果你想编辑它的任何内容，你需要编辑三次。

相反，让我们创建一个 `<ProgressBar/>` 组件。

```rust
#[component]
fn ProgressBar() -> impl IntoView {
    view! {
        <progress
            max="50"
            // 嗯... 我们将从哪里获得这个？
            value=progress
        />
    }
}
```

只有一个问题：`progress` 没有定义。它应该从哪里来？当我们手动定义所有内容时，我们只使用了局部变量名。现在我们需要一些方法将参数传递给组件。

## 组件 Props

我们使用组件属性或“props”来做到这一点。如果你使用过其他的前端框架，这可能是一个熟悉的想法。基本上，属性之于组件就像属性之于 HTML 元素：它们允许你将额外的信息传递给组件。

在 Leptos 中，你可以通过给组件函数添加额外的参数来定义 props。

```rust
#[component]
fn ProgressBar(
    progress: ReadSignal<i32>
) -> impl IntoView {
    view! {
        <progress
            max="50"
            // 现在可以了
            value=progress
        />
    }
}
```

现在我们可以在主要的 `<App/>` 组件的视图中使用我们的组件。

```rust
#[component]
fn App() -> impl IntoView {
    let (count, set_count) = create_signal(0);
    view! {
        <button on:click=move |_| { set_count.update(|n| *n += 1); }>
            "Click me"
        </button>
        // 现在我们使用我们的组件!
        <ProgressBar progress=count/>
    }
}
```

在视图中使用组件看起来很像使用 HTML 元素。你会注意到你可以很容易地分辨元素和组件之间的区别，因为组件总是有 `PascalCase` 的名称。你像传递 HTML 元素属性一样传递 `progress` prop。很简单。

### 响应式和静态 Props

你会注意到在整个例子中，`progress` 接受一个响应式的 `ReadSignal<i32>`，而不是一个普通的 `i32`。这**非常重要**。

组件 props 没有附加任何特殊的含义。组件只是一个运行一次来设置用户界面的函数。告诉界面响应更改的唯一方法是传递一个信号类型。所以如果你有一个会随着时间变化的组件属性，比如我们的 `progress`，它应该是一个信号。

### `optional` Props

现在 `max` 设置是硬编码的。让我们也把它作为一个 prop。但是让我们添加一个条件：让我们通过使用 `#[prop(optional)]` 注释组件函数的特定参数来使这个 prop 成为可选的。

```rust
#[component]
fn ProgressBar(
    // 将此 prop 标记为可选
    // 当你使用 <ProgressBar/> 时，你可以指定它也可以不指定
    #[prop(optional)]
    max: u16,
    progress: ReadSignal<i32>
) -> impl IntoView {
    view! {
        <progress
            max=max
            value=progress
        />
    }
}
```

现在，我们可以使用 `<ProgressBar max=50 progress=count/>`，或者我们可以省略 `max` 来使用默认值（即 `<ProgressBar progress=count/>`）。`optional` 的默认值是它的 `Default::default()` 值，对于 `u16` 来说是 `0`。对于进度条来说，最大值为 `0` 并不是很有用。

所以让我们给它一个特定的默认值。

### `default` props

You can specify a default value other than `Default::default()` pretty simply
with `#[prop(default = ...)`.

```rust
#[component]
fn ProgressBar(
    #[prop(default = 100)]
    max: u16,
    progress: ReadSignal<i32>
) -> impl IntoView {
    view! {
        <progress
            max=max
            value=progress
        />
    }
}
```

### 泛型 Props

这很好。但我们从两个计数器开始，一个由 `count` 驱动，一个由派生信号 `double_count` 驱动。让我们通过使用 `double_count` 作为另一个 `<ProgressBar/>` 上的 `progress` prop 来重新创建它。

```rust
#[component]
fn App() -> impl IntoView {
    let (count, set_count) = create_signal(0);
    let double_count = move || count() * 2;

    view! {
        <button on:click=move |_| { set_count.update(|n| *n += 1); }>
            "Click me"
        </button>
        <ProgressBar progress=count/>
        // 添加第二个进度条
        <ProgressBar progress=double_count/>
    }
}
```

嗯... 这无法编译。应该很容易理解为什么：我们已经声明了 `progress` prop 接受 `ReadSignal<i32>`，而 `double_count` 不是 `ReadSignal<i32>`。正如 rust-analyzer 会告诉你的，它的类型是 `|| -> i32`，即，它是一个返回 `i32` 的闭包。

有几种方法可以处理这个问题。一种是说：“好吧，我知道 `ReadSignal` 是一个函数，而且我知道闭包是一个函数；也许我可以接受任何函数？” 如果你很精通，你可能知道这两个都实现了 trait `Fn() -> i32`。所以你可以使用一个泛型组件：

```rust
#[component]
fn ProgressBar(
    #[prop(default = 100)]
    max: u16,
    progress: impl Fn() -> i32 + 'static
) -> impl IntoView {
    view! {
        <progress
            max=max
            value=progress
        />
        // 添加一个换行符以避免重叠
        <br/>
    }
}
```

这是一种编写此组件的完全合理的方式：`progress` 现在接受任何实现此 `Fn()` trait 的值。

> 泛型 props 也可以使用 `where` 子句指定，或者使用内联泛型，如 `ProgressBar<F: Fn() -> i32 + 'static>`。请注意，对 `impl Trait` 语法的支持是在 0.6.12 版本中发布的；如果收到错误消息，你可能需要 `cargo update` 以确保你使用的是最新版本。

泛型需要在组件 props 中的某个地方使用。这是因为 props 被构建到一个结构体中，所以所有泛型类型都必须在结构体中的某个地方使用。这通常可以通过使用可选的 `PhantomData` prop 来轻松实现。然后你可以使用表达类型的语法在视图中指定泛型：`<Component<T>/>`（而不是使用 turbofish 风格的 `<Component::<T>/>`）。

```rust
#[component]
fn SizeOf<T: Sized>(#[prop(optional)] _ty: PhantomData<T>) -> impl IntoView {
    std::mem::size_of::<T>()
}

#[component]
pub fn App() -> impl IntoView {
    view! {
        <SizeOf<usize>/>
        <SizeOf<String>/>
    }
}
```

> 请注意，存在一些限制。例如，我们的视图宏解析器无法处理嵌套泛型，如 `<SizeOf<Vec<T>>/>`。

### `into` Props

还有另一种方法可以实现这一点，那就是使用 `#[prop(into)]`。此属性会自动对你作为 props 传递的值调用 `.into()`，这允许你轻松地传递具有不同值的 props。

在这种情况下，了解 [`Signal`](https://docs.rs/leptos/latest/leptos/struct.Signal.html) 类型会很有帮助。`Signal` 是一个枚举类型，表示任何类型的可读响应式信号。在为要在传递不同类型的信号时重复使用的组件定义 API 时，它会很有用。当你希望能够接受静态值或响应式值时，[`MaybeSignal`](https://docs.rs/leptos/latest/leptos/enum.MaybeSignal.html) 类型很有用。

```rust
#[component]
fn ProgressBar(
    #[prop(default = 100)]
    max: u16,
    #[prop(into)]
    progress: Signal<i32>
) -> impl IntoView
{
    view! {
        <progress
            max=max
            value=progress
        />
        <br/>
    }
}

#[component]
fn App() -> impl IntoView {
    let (count, set_count) = create_signal(0);
    let double_count = move || count() * 2;

    view! {
        <button on:click=move |_| { set_count.update(|n| *n += 1); }>
            "Click me"
        </button>
        // .into() 将 `ReadSignal` 转换为 `Signal`
        <ProgressBar progress=count/>
        // 使用 `Signal::derive()` 包装派生信号
        <ProgressBar progress=Signal::derive(double_count)/>
    }
}
```

### 可选泛型 Props

请注意，你不能为组件指定可选的泛型 props。让我们看看如果你尝试会发生什么：

```rust,compile_fail
#[component]
fn ProgressBar<F: Fn() -> i32 + 'static>(
    #[prop(optional)] progress: Option<F>,
) -> impl IntoView {
    progress.map(|progress| {
        view! {
            <progress
                max=100
                value=progress
            />
            <br/>
        }
    })
}

#[component]
pub fn App() -> impl IntoView {
    view! {
        <ProgressBar/>
    }
}
```

Rust 帮忙指出了错误

```
xx |         <ProgressBar/>
   |          ^^^^^^^^^^^ 无法推断函数 `ProgressBar` 上声明的类型参数 `F` 的类型
   |
help: 考虑指定泛型参数
   |
xx |         <ProgressBar::<F>/>
   |                     +++++
```

你可以使用 `<ProgressBar<F>/>` 语法（在 `view` 宏中没有 turbofish）在组件上指定泛型。在这里指定正确的类型是不可能的；闭包和函数通常是不可命名的类型。编译器可以用简写来显示它们，但你不能指定它们。

但是，你可以通过使用 `Box<dyn _>` 或 `&dyn _>` 提供具体类型来解决这个问题：

```rust
#[component]
fn ProgressBar(
    #[prop(optional)] progress: Option<Box<dyn Fn() -> i32>>,
) -> impl IntoView {
    progress.map(|progress| {
        view! {
            <progress
                max=100
                value=progress
            />
            <br/>
        }
    })
}

#[component]
pub fn App() -> impl IntoView {
    view! {
        <ProgressBar/>
    }
}
```

因为 Rust 编译器现在知道了 prop 的具体类型，因此即使在 `None` 的情况下，它也知道它在内存中的大小，所以这可以很好地编译。

> 在这种特殊情况下，`&dyn Fn() -> i32` 会导致生命周期问题，但在其他情况下，它可能是一种可能性。

## 记录组件

这是本书中最不重要但最重要的章节之一。严格来说，记录你的组件及其 props 并不是必要的。根据你的团队和应用程序的大小，这可能非常重要。但这非常容易，并且会立即产生效果。

要记录组件及其 props，你可以简单地在组件函数和每个 prop 上添加文档注释：

```rust
/// 显示目标进度。
#[component]
fn ProgressBar(
    /// 进度条的最大值。
    #[prop(default = 100)]
    max: u16,
    /// 应该显示多少进度。
    #[prop(into)]
    progress: Signal<i32>,
) -> impl IntoView {
    /* ... */
}
```

这就是你需要做的所有事情。这些行为就像普通的 Rust 文档注释，除了你可以记录单个组件 props，而这对于 Rust 函数参数是无法做到的。

这将自动为你的组件、它的 `Props` 类型以及用于添加 props 的每个字段生成文档。在将鼠标悬停在组件名称或 props 上并看到 `#[component]` 宏与 rust-analyzer 结合的强大功能之前，可能很难理解它的强大功能。

> #### 进阶主题：`#[component(transparent)]`
>
> 所有 Leptos 组件都返回 `-> impl IntoView`。但是，有些需要直接返回一些数据，而无需任何额外的包装。这些可以用 `#[component(transparent)]` 标记，在这种情况下，它们会完全返回它们返回的值，而渲染系统不会以任何方式转换它们。
>
> 这主要用于两种情况：
>
> 1. 创建 `<Suspense/>` 或 `<Transition/>` 的包装器，它们返回一个透明的 suspense 结构，以便与 SSR 和 hydration 正确集成。
> 2. 将 `leptos_router` 的 `<Route/>` 定义重构到单独的组件中，因为 `<Route/>` 是一个透明的组件，它返回一个 `RouteDefinition` 结构而不是一个视图。
>
> 通常，除非你正在创建属于这两种类别之一的自定义包装组件，否则你不需要使用透明组件。

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/3-components-0-5-5vvl69?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/3-components-0-5-5vvl69?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源码</summary>

```rust
use leptos::*;

// 将不同的组件组合在一起是我们构建
// 用户界面的方式。在这里，我们将定义一个可重用的 <ProgressBar/>。
// 你将看到如何使用文档注释来记录组件
// 及其属性。

/// 显示目标进度。
#[component]
fn ProgressBar(
    // 将此标记为可选 prop。它将默认为其类型的默认值，即 0。
    #[prop(default = 100)]
    /// 进度条的最大值。
    max: u16,
    // 将对传递到 prop 的值运行 `.into()`。
    #[prop(into)]
    // `Signal<T>` 是几个响应式类型的包装器。
    // 在像这样的组件 API 中，它会很有帮助，我们
    // 可能想要接受任何类型的响应式值
    /// 应该显示多少进度。
    progress: Signal<i32>,
) -> impl IntoView {
    view! {
        <progress
            max={max}
            value=progress
        />
        <br/>
    }
}

#[component]
fn App() -> impl IntoView {
    let (count, set_count) = create_signal(0);

    let double_count = move || count() * 2;

    view! {
        <button
            on:click=move |_| {
                set_count.update(|n| *n += 1);
            }
        >
            "Click me"
        </button>
        <br/>
        // 如果你在 CodeSandbox 或带有
        // rust-analyzer 支持的编辑器中打开了此文件，请尝试将鼠标悬停在 `ProgressBar`、
        // `max` 或 `progress` 上以查看我们在上面定义的文档
        <ProgressBar max=50 progress=count/>
        // 让我们在这个进度条上使用默认的最大值
        // 默认值为 100，所以它应该移动得慢一半
        <ProgressBar progress=count/>
        // Signal::derive 从我们的派生信号创建一个 Signal 包装器
        // 使用 double_count 意味着它应该移动得快两倍
        <ProgressBar max=50 progress=Signal::derive(double_count)/>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
