# 父子组件通信

你可以将你的应用程序视为一个嵌套的组件树。每个组件都处理自己的本地状态并管理用户界面的一部分，因此组件往往是相对独立的。

但是，有时你需要在父组件与其子组件之间进行通信。例如，假设你定义了一个 `<FancyButton/>` 组件，它为 `<button/>` 添加了一些样式、日志记录或其他内容。你想在你的 `<App/>` 组件中使用 `<FancyButton/>`。但是你如何在两者之间进行通信呢？

将状态从父组件传递到子组件很容易。我们在 [组件和 props](./03_components.md) 的材料中介绍了一些这方面的内容。基本上，如果你希望父组件与子组件通信，你可以传递一个 [`ReadSignal`](https://docs.rs/leptos/latest/leptos/struct.ReadSignal.html)、一个 [`Signal`](https://docs.rs/leptos/latest/leptos/struct.Signal.html)，甚至一个 [`MaybeSignal`](https://docs.rs/leptos/latest/leptos/enum.MaybeSignal.html) 作为 prop。

但是反过来呢？子组件如何将有关事件或状态更改的通知发送回父组件？

在 Leptos 中，有四种基本的父子组件通信模式。

## 1. 传递一个 [`WriteSignal`](https://docs.rs/leptos/latest/leptos/struct.WriteSignal.html)

一种方法是简单地将 `WriteSignal` 从父组件传递到子组件，并在子组件中更新它。这使你可以从子组件操作父组件的状态。

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = create_signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        <ButtonA setter=set_toggled/>
    }
}

#[component]
pub fn ButtonA(setter: WriteSignal<bool>) -> impl IntoView {
    view! {
        <button
            on:click=move |_| setter.update(|value| *value = !*value)
        >
            "Toggle"
        </button>
    }
}
```

这种模式很简单，但你应该小心使用它：传递 `WriteSignal` 会使你的代码难以推理。在这个例子中，当你阅读 `<App/>` 时，很明显你正在交出改变 `toggled` 的能力，但根本不清楚它何时或如何改变。在这个小的、局部的例子中很容易理解，但是如果你发现你在整个代码中都像这样传递 `WriteSignal`，你应该认真考虑这是否会让编写意大利面条式代码变得太容易。

## 2. 使用回调函数

另一种方法是将一个回调函数传递给子组件：例如 `on_click`。

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = create_signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        <ButtonB on_click=move |_| set_toggled.update(|value| *value = !*value)/>
    }
}


#[component]
pub fn ButtonB(#[prop(into)] on_click: Callback<MouseEvent>) -> impl IntoView
{
    view! {
        <button on:click=on_click>
            "Toggle"
        </button>
    }
}
```

你会注意到，`<ButtonA/>` 被赋予了一个 `WriteSignal` 并决定如何改变它，而 `<ButtonB/>` 只是触发一个事件：改变发生在 `<App/>` 中。这样做的好处是将局部状态保持在局部，防止了意大利面条式修改的问题。但这也意味着修改该信号的逻辑需要存在于 `<App/>` 中，而不是 `<ButtonB/>` 中。这些是真正的权衡，而不是简单的对错选择。

> 注意我们使用 `Callback<In, Out>` 类型的方式。这基本上是一个围绕闭包 `Fn(In) -> Out` 的包装器，它也是 `Copy` 的，并且易于传递。
>
> 我们还使用了 `#[prop(into)]` 属性，以便我们可以将普通的闭包传递给 `on_click`。请参阅[章节 “`into` Props”](./03_components.md#into-props) 了解更多详细信息。

### 2.1 使用闭包而不是 `Callback`

你可以直接使用 Rust 闭包 `Fn(MouseEvent)` 而不是 `Callback`：

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = create_signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        <ButtonB on_click=move |_| set_toggled.update(|value| *value = !*value)/>
    }
}


#[component]
pub fn ButtonB<F>(on_click: F) -> impl IntoView
where
    F: Fn(MouseEvent) + 'static
{
    view! {
        <button on:click=on_click>
            "Toggle"
        </button>
    }
}
```

在这种情况下，代码非常相似。在更高级的用例中，使用闭包可能需要一些克隆，而使用 `Callback` 则不需要。

> 注意我们在这里为回调函数声明泛型类型 `F` 的方式。如果你感到困惑，请回顾一下关于组件的章节中的 [泛型 props](./03_components.html#generic-props) 部分。

## 3. 使用事件监听器

你实际上可以用稍微不同的方式编写选项 2。如果回调函数直接映射到原生 DOM 事件，你可以直接在 `<App/>` 的 `view` 宏中使用组件的地方添加 `on:` 监听器。

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = create_signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        // 注意 on:click 而不是 on_click
        // 这与 HTML 元素事件监听器的语法相同
        <ButtonC on:click=move |_| set_toggled.update(|value| *value = !*value)/>
    }
}


#[component]
pub fn ButtonC() -> impl IntoView {
    view! {
        <button>"Toggle"</button>
    }
}
```

这让你在 `<ButtonC/>` 中编写的代码比在 `<ButtonB/>` 中少得多，并且仍然为监听器提供了一个正确类型的事件。这是通过为 `<ButtonC/>` 返回的每个元素添加一个 `on:` 事件监听器来实现的：在本例中，只有一个 `<button>`。

当然，这只适用于你直接传递给组件中渲染的元素的实际 DOM 事件。对于不直接映射到元素的更复杂的逻辑（例如，你创建了 `<ValidatedForm/>` 并想要一个 `on_valid_form_submit` 回调函数），你应该使用选项 2。

## 4. 提供一个上下文

这个版本实际上是选项 1 的一个变体。假设你有一个深度嵌套的组件树：

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = create_signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        <Layout/>
    }
}

#[component]
pub fn Layout() -> impl IntoView {
    view! {
        <header>
            <h1>"My Page"</h1>
        </header>
        <main>
            <Content/>
        </main>
    }
}

#[component]
pub fn Content() -> impl IntoView {
    view! {
        <div class="content">
            <ButtonD/>
        </div>
    }
}

#[component]
pub fn ButtonD<F>() -> impl IntoView {
    todo!()
}
```

现在 `<ButtonD/>` 不再是 `<App/>` 的直接子级，因此你不能简单地将你的 `WriteSignal` 传递给它的 props。你可以做一些有时被称为“prop drilling”的事情，在两者之间的每一层添加一个 prop：

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = create_signal(false);
    view! {
        <p>"Toggled? " {toggled}</p>
        <Layout set_toggled/>
    }
}

#[component]
pub fn Layout(set_toggled: WriteSignal<bool>) -> impl IntoView {
    view! {
        <header>
            <h1>"My Page"</h1>
        </header>
        <main>
            <Content set_toggled/>
        </main>
    }
}

#[component]
pub fn Content(set_toggled: WriteSignal<bool>) -> impl IntoView {
    view! {
        <div class="content">
            <ButtonD set_toggled/>
        </div>
    }
}

#[component]
pub fn ButtonD<F>(set_toggled: WriteSignal<bool>) -> impl IntoView {
    todo!()
}
```

这真是一团糟。`<Layout/>` 和 `<Content/>` 不需要 `set_toggled`；它们只是将其传递给 `<ButtonD/>`。但我需要声明三次这个 prop。这不仅烦人，而且难以维护：想象一下，我们添加了一个“half-toggled”选项，`set_toggled` 的类型需要更改为一个 `enum`。我们必须在三个地方更改它！

有没有办法跳过层级？

有！

### 4.1 上下文 API

你可以使用 [`provide_context`](https://docs.rs/leptos/latest/leptos/fn.provide_context.html) 和 [`use_context`](https://docs.rs/leptos/latest/leptos/fn.use_context.html) 来提供跳过层级的数据。上下文由你提供的数据类型（在本例中为 `WriteSignal<bool>`）标识，并且它们存在于一个自上而下的树中，该树遵循你的 UI 树的轮廓。在这个例子中，我们可以使用上下文来跳过不必要的 prop drilling。

```rust
#[component]
pub fn App() -> impl IntoView {
    let (toggled, set_toggled) = create_signal(false);

    // 与此组件的所有子组件共享 `set_toggled`
    provide_context(set_toggled);

    view! {
        <p>"Toggled? " {toggled}</p>
        <Layout/>
    }
}

// <Layout/> 和 <Content/> 省略
// 要在此版本中工作，请删除它们对 set_toggled 的引用

#[component]
pub fn ButtonD() -> impl IntoView {
    // use_context 向上搜索上下文树，希望
    // 找到一个 `WriteSignal<bool>`
    // 在这种情况下，我使用 .expect() 因为我知道我提供了它
    let setter = use_context::<WriteSignal<bool>>()
        .expect("to have found the setter provided");

    view! {
        <button
            on:click=move |_| setter.update(|value| *value = !*value)
        >
            "Toggle"
        </button>
    }
}
```

与 `<ButtonA/>` 相同的注意事项也适用于此：传递 `WriteSignal` 应该谨慎行事，因为它允许你从代码的任意部分修改状态。但是，如果小心谨慎地进行，这可能是 Leptos 中最有效的全局状态管理技术之一：只需在你需要它的最高级别提供状态，并在你需要它的较低级别使用它。

请注意，这种方法没有性能方面的缺点。因为你传递的是一个细粒度的响应式信号，所以在更新它时，中间组件（`<Layout/>` 和 `<Content/>`）_什么也不会发生_。你直接在 `<ButtonD/>` 和 `<App/>` 之间进行通信。事实上——这就是细粒度响应式的强大之处——你直接在 `<ButtonD/>` 中的按钮点击和 `<App/>` 中的单个文本节点之间进行通信。就好像这些组件本身根本不存在一样。而且，嗯... 在运行时，它们确实不存在。一直到底都只是信号和效果。

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/8-parent-child-0-5-7rz7qd?file=%2Fsrc%2Fmain.rs%3A1%2C2)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/8-parent-child-0-5-7rz7qd?file=%2Fsrc%2Fmain.rs%3A1%2C2" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源码</summary>

```rust
use leptos::{ev::MouseEvent, *};

// 这突出了子组件与父组件通信的四种不同方式：
// 1) <ButtonA/>：将 WriteSignal 作为子组件 props 之一传递，
//    供子组件写入和父组件读取
// 2) <ButtonB/>：将闭包作为子组件 props 之一传递，供
//    子组件调用
// 3) <ButtonC/>：向组件添加 `on:` 事件监听器
// 4) <ButtonD/>：提供一个在组件中使用的上下文（而不是 prop drilling）

#[derive(Copy, Clone)]
struct SmallcapsContext(WriteSignal<bool>);

#[component]
pub fn App() -> impl IntoView {
    // 只是一些用于切换 <p> 上三个类的信号
    let (red, set_red) = create_signal(false);
    let (right, set_right) = create_signal(false);
    let (italics, set_italics) = create_signal(false);
    let (smallcaps, set_smallcaps) = create_signal(false);

    // newtype 模式在这里不是*必需的*，但这是一个好习惯
    // 它避免了与其他可能的未来 `WriteSignal<bool>` 上下文的混淆
    // 并使其更容易在 ButtonC 中引用
    provide_context(SmallcapsContext(set_smallcaps));

    view! {
        <main>
            <p
                // class: 属性接受 F: Fn() => bool，并且这些信号都实现了 Fn()
                class:red=red
                class:right=right
                class:italics=italics
                class:smallcaps=smallcaps
            >
                "Lorem ipsum sit dolor amet."
            </p>

            // 按钮 A：传递信号设置器
            <ButtonA setter=set_red/>

            // 按钮 B：传递一个闭包
            <ButtonB on_click=move |_| set_right.update(|value| *value = !*value)/>

            // 按钮 B：使用常规事件监听器
            // 像这样在组件上设置事件监听器会将其应用于
            // 组件返回的每个顶级元素
            <ButtonC on:click=move |_| set_italics.update(|value| *value = !*value)/>

            // 按钮 D 从上下文而不是 props 获取其设置器
            <ButtonD/>
        </main>
    }
}

/// 按钮 A 接收一个信号设置器并更新信号本身
#[component]
pub fn ButtonA(
    /// 单击按钮时将切换的信号。
    setter: WriteSignal<bool>,
) -> impl IntoView {
    view! {
        <button
            on:click=move |_| setter.update(|value| *value = !*value)
        >
            "Toggle Red"
        </button>
    }
}

/// 按钮 B 接收一个闭包
#[component]
pub fn ButtonB<F>(
    /// 单击按钮时将调用的回调。
    on_click: F,
) -> impl IntoView
where
    F: Fn(MouseEvent) + 'static,
{
    view! {
        <button
            on:click=on_click
        >
            "Toggle Right"
        </button>
    }

    // 只是一个注释：在普通函数中，ButtonB 可以接受 on_click: impl Fn(MouseEvent) + 'static
    // 并让你免于输入泛型
    // 组件宏实际上扩展为定义一个
    //
    // struct ButtonBProps<F> where F: Fn(MouseEvent) + 'static {
    //   on_click: F
    // }
    //
    // 这就是允许我们在组件调用中使用命名 props 的原因，
    // 而不是有序的函数参数列表
    // 如果 Rust 曾经有命名的函数参数，我们可以放弃这个要求
}

/// 按钮 C 是一个虚拟按钮：它渲染一个按钮，但不处理
/// 它的点击。相反，父组件添加了一个事件监听器。
#[component]
pub fn ButtonC() -> impl IntoView {
    view! {
        <button>
            "Toggle Italics"
        </button>
    }
}

/// 按钮 D 与按钮 A 非常相似，但不是将设置器作为 prop 传递，
/// 而是从上下文中获取它
#[component]
pub fn ButtonD() -> impl IntoView {
    let setter = use_context::<SmallcapsContext>().unwrap().0;

    view! {
        <button
            on:click=move |_| setter.update(|value| *value = !*value)
        >
            "Toggle Small Caps"
        </button>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
