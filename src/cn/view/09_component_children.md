# 组件子级

就像你可以将子级传递给 HTML 元素一样，将子级传递给组件也是很常见的。例如，假设我有一个 `<FancyForm/>` 组件，它增强了 HTML `<form>`。我需要某种方法来传递它的所有输入。

```rust
view! {
    <FancyForm>
        <fieldset>
            <label>
                "Some Input"
                <input type="text" name="something"/>
            </label>
        </fieldset>
        <button>"Submit"</button>
    </FancyForm>
}
```

在 Leptos 中，你如何做到这一点？基本上有两种方法可以将组件传递给其他组件：

1. **渲染 props**：返回视图的函数属性
2. **`children`** prop：一个特殊的组件属性，包含你作为子级传递给组件的任何内容。

事实上，你已经在 [`<Show/>`](/view/06_control_flow.html#show) 组件中看到了这两者的实际应用：

```rust
view! {
  <Show
    // `when` 是一个普通的 prop
    when=move || value() > 5
    // `fallback` 是一个“渲染 prop”：一个返回视图的函数
    fallback=|| view! { <Small/> }
  >
    // `<Big/>`（以及这里的任何其他内容）
    // 将被赋予 `children` prop
    <Big/>
  </Show>
}
```

让我们定义一个接受一些子级和一个渲染 prop 的组件。

```rust
#[component]
pub fn TakesChildren<F, IV>(
    /// 接受一个函数（类型 F），该函数返回任何可以
    /// 转换为视图（类型 IV）的内容
    render_prop: F,
    /// `children` 接受 `Children` 类型
    children: Children,
) -> impl IntoView
where
    F: Fn() -> IV,
    IV: IntoView,
{
    view! {
        <h2>"Render Prop"</h2>
        {render_prop()}

        <h2>"Children"</h2>
        {children()}
    }
}
```

`render_prop` 和 `children` 都是函数，所以我们可以调用它们来生成相应的视图。`children`，特别是，是 `Box<dyn FnOnce() -> Fragment>` 的别名。（你不高兴我们将其命名为 `Children` 而不是那个吗？）

> 如果你在这里需要一个 `Fn` 或 `FnMut`，因为你需要多次调用 `children`，我们还提供了 `ChildrenFn` 和 `ChildrenMut` 别名。

我们可以像这样使用组件：

```rust
view! {
    <TakesChildren render_prop=|| view! { <p>"Hi, there!"</p> }>
        // 这些被传递给 `children`
        "Some text"
        <span>"A span"</span>
    </TakesChildren>
}
```

## 操作子级

[`Fragment`](https://docs.rs/leptos/latest/leptos/struct.Fragment.html) 类型基本上是一种包装 `Vec<View>` 的方法。你可以将其插入到视图中的任何位置。

但是你也可以直接访问这些内部视图来操作它们。例如，这里有一个组件，它接受它的子级并将它们转换成一个无序列表。

```rust
#[component]
pub fn WrapsChildren(children: Children) -> impl IntoView {
    // Fragment 有一个 `nodes` 字段，其中包含一个 Vec<View>
    let children = children()
        .nodes
        .into_iter()
        .map(|child| view! { <li>{child}</li> })
        .collect_view();

    view! {
        <ul>{children}</ul>
    }
}
```

像这样调用它将创建一个列表：

```rust
view! {
    <WrapsChildren>
        "A"
        "B"
        "C"
    </WrapsChildren>
}
```

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/9-component-children-0-5-m4jwhp?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/9-component-children-0-5-m4jwhp?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源码</summary>

```rust
use leptos::*;

// 通常，你希望将某种子视图传递给另一个
// 组件。有两种基本模式可以做到这一点：
// - “渲染 props”：创建一个接受函数的组件 prop，
//   该函数创建一个视图
// - `children` prop：一个特殊的属性，其中包含
//   在你的视图中作为组件的子级传递的内容，而不是作为
//   属性传递的内容

#[component]
pub fn App() -> impl IntoView {
    let (items, set_items) = create_signal(vec![0, 1, 2]);
    let render_prop = move || {
        // items.with(...) 在不克隆的情况下对值做出反应
        // 通过应用一个函数。在这里，我们直接传递 `len` 方法
        // 在 `Vec<_>` 上
        let len = move || items.with(Vec::len);
        view! {
            <p>"Length: " {len}</p>
        }
    };

    view! {
        // 此组件仅显示两种类型的子级，
        // 将它们嵌入到其他一些标记中
        <TakesChildren
            // 对于组件 props，你可以简写
            // `render_prop=render_prop` => `render_prop`
            // （这不适用于 HTML 元素属性）
            render_prop
        >
            // 这些看起来就像 HTML 元素的子级
            <p>"Here's a child."</p>
            <p>"Here's another child."</p>
        </TakesChildren>
        <hr/>
        // 此组件实际上会迭代并包装子级
        <WrapsChildren>
            <p>"Here's a child."</p>
            <p>"Here's another child."</p>
        </WrapsChildren>
    }
}

/// 在标记内显示 `render_prop` 和一些子级。
#[component]
pub fn TakesChildren<F, IV>(
    /// 接受一个函数（类型 F），该函数返回任何可以
    /// 转换为视图（类型 IV）的内容
    render_prop: F,
    /// `children` 接受 `Children` 类型
    /// 这是 `Box<dyn FnOnce() -> Fragment>` 的别名
    /// ... 你不高兴我们将其命名为 `Children` 而不是那个吗？
    children: Children,
) -> impl IntoView
where
    F: Fn() -> IV,
    IV: IntoView,
{
    view! {
        <h1><code>"<TakesChildren/>"</code></h1>
        <h2>"Render Prop"</h2>
        {render_prop()}
        <hr/>
        <h2>"Children"</h2>
        {children()}
    }
}

/// 将每个子级包装在 `<li>` 中并将它们嵌入到 `<ul>` 中。
#[component]
pub fn WrapsChildren(children: Children) -> impl IntoView {
    // children() 返回一个 `Fragment`，它有一个
    // `nodes` 字段，其中包含一个 Vec<View>
    // 这意味着我们可以迭代子级
    // 来创建新的东西！
    let children = children()
        .nodes
        .into_iter()
        .map(|child| view! { <li>{child}</li> })
        .collect::<Vec<_>>();

    view! {
        <h1><code>"<WrapsChildren/>"</code></h1>
        // 将我们包装的子级包装在 UL 中
        <ul>{children}</ul>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
