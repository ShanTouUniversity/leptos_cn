# `view`：动态类、样式和属性

到目前为止，我们已经了解了如何使用 `view` 宏来创建事件监听器，以及如何通过将函数（例如信号）传递到视图中来创建动态文本。

但是当然你可能还想更新用户界面中的其他内容。
在本节中，我们将了解如何动态更新类、样式和属性，
并且我们将介绍**派生信号**的概念。

让我们从一个应该很熟悉的简单组件开始：点击一个按钮来增加计数器。

```rust
#[component]
fn App() -> impl IntoView {
    let (count, set_count) = create_signal(0);

    view! {
        <button
            on:click=move |_| {
                set_count.update(|n| *n += 1);
            }
        >
            "Click me: "
            {move || count()}
        </button>
    }
}
```

到目前为止，这只是上一章中的示例。

## 动态类

现在假设我想动态更新此元素上的 CSS 类列表。
例如，假设我想在计数为奇数时添加类 `red`。 我可以使用 `class:` 语法来做到这一点。

```rust
class:red=move || count() % 2 == 1
```

`class:` 属性接受

1. 冒号后面的类名 (`red`)
2. 一个值，可以是 `bool` 或返回 `bool` 的函数

当值为 `true` 时，添加该类。 当值为 `false` 时，删除该类。
如果该值是一个访问信号的函数，则该类将在信号更改时进行响应式更新。

现在，每次我点击按钮时，文本应该在红色和黑色之间切换，因为数字在偶数和奇数之间切换。

```rust
<button
    on:click=move |_| {
        set_count.update(|n| *n += 1);
    }
    // class: 语法响应式地更新单个类
    // 在这里，当 `count` 为奇数时，我们将设置 `red` 类
    class:red=move || count() % 2 == 1
>
    "Click me"
</button>
```

> 如果你正在跟随，请确保进入你的 `index.html` 并添加如下内容：
>
> ```html
> <style>
>   .red {
>     color: red;
>   }
> </style>
> ```

某些 CSS 类名不能被 `view` 宏直接解析，尤其是当它们包含破折号、数字或其他字符的混合时。 在这种情况下，你可以使用元组语法：`class=("name", value)` 仍然直接更新单个类。

```rust
class=("button-20", move || count() % 2 == 1)
```

可以使用类似的 `style:` 语法直接更新单个 CSS 属性。

```rust
    let (x, set_x) = create_signal(0);
        view! {
            <button
                on:click={move |_| {
                    set_x.update(|n| *n += 10);
                }}
                // 设置 `style` 属性
                style="position: absolute"
                // 并使用 `style:` 切换单个 CSS 属性
                style:left=move || format!("{}px", x() + 100)
                style:background-color=move || format!("rgb({}, {}, 100)", x(), 100)
                style:max-width="400px"
                // 设置一个 CSS 变量供样式表使用
                style=("--columns", x)
            >
                "Click to Move"
            </button>
    }
```

## 动态属性

这同样适用于普通属性。 将纯字符串或原始值传递给
属性会赋予它一个静态值。 将函数（包括信号）传递给
属性会使其响应式地更新其值。 让我们在我们的视图中添加另一个元素：

```rust
<progress
    max="50"
    // 信号是函数，所以 `value=count` 和 `value=move || count.get()`
    // 是可以互换的。
    value=count
/>
```

现在每次我们设置计数时，不仅 `<button>` 的 `class` 会被切换，
而且 `<progress>` 栏的 `value` 也会增加，这意味着我们的进度条会前进。

## 派生信号

让我们更深入一层，只是为了好玩。

你已经知道，我们只需将函数传递给 `view` 即可创建响应式界面。 这意味着我们可以轻松地更改我们的进度条。 例如，假设我们希望它移动速度快一倍：

```rust
<progress
    max="50"
    value=move || count() * 2
/>
```

但是想象一下，我们想在多个地方重用该计算。 你可以使用**派生信号**来做到这一点：一个访问信号的闭包。

```rust
let double_count = move || count() * 2;

/* 插入视图的其余部分 */
<progress
    max="50"
    // 我们在这里使用一次
    value=double_count
/>
<p>
    "Double Count: "
    // 在这里再次使用
    {double_count}
</p>
```

派生信号允许你创建响应式计算值，这些值可以在应用程序中的多个位置使用，并且开销最小。

注意：像这样使用派生信号意味着每次信号更改（当 `count()` 更改时）和每次我们访问 `double_count` 时，计算都会运行一次；换句话说，两次。 这是一个非常便宜的计算，所以没关系。
我们将在后面的章节中介绍 memo ，它们旨在解决昂贵计算的这个问题。

> #### 高级主题：注入原始 HTML
>
> `view` 宏支持一个额外的属性 `inner_html`，该属性
> 可用于直接设置任何元素的 HTML 内容，并擦除你赋予它的任何其他子元素。 请注意，这*不会*转义你提供的 HTML。 你
> 应确保它只包含受信任的输入或任何 HTML 实体都已转义，以防止跨站点脚本 (XSS) 攻击。
>
> ```rust
> let html = "<p>此 HTML 将被注入。</p>";
> view! {
>   <div inner_html=html/>
> }
> ```
>
> [点击此处查看完整的 `view` 宏文档](https://docs.rs/leptos/latest/leptos/macro.view.html)。

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox。](https://codesandbox.io/p/sandbox/2-dynamic-attributes-0-5-lwdrpm?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  请启用 JavaScript 以查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/2-dynamic-attributes-0-5-lwdrpm?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源代码</summary>

```rust
use leptos::*;

#[component]
fn App() -> impl IntoView {
    let (count, set_count) = create_signal(0);

    // “派生信号”是一个访问其他信号的函数
    // 我们可以使用它来创建依赖于
    // 一个或多个其他信号的值的响应式值
    let double_count = move || count() * 2;

    view! {
        <button
            on:click=move |_| {
                set_count.update(|n| *n += 1);
            }

            // class: 语法响应式地更新单个类
            // 在这里，当 `count` 为奇数时，我们将设置 `red` 类
            class:red=move || count() % 2 == 1
        >
            "Click me"
        </button>
        // 注意：像 <br> 这样的自闭合标签需要一个显式的 /
        <br/>

        // 每次 `count` 更改时，我们都会更新此进度条
        <progress
            // 静态属性的工作方式与 HTML 中相同
            max="50"

            // 将函数传递给属性
            // 响应式地设置该属性
            // 信号是函数，所以 `value=count` 和 `value=move || count.get()`
            // 是可以互换的。
            value=count
        ></progress>
        <br/>

        // 此进度条将使用 `double_count`
        // 所以它应该移动速度快一倍！
        <progress
            max="50"
            // 派生信号是函数，因此它们也可以
            // 响应式地更新 DOM
            value=double_count
        ></progress>
        <p>"Count: " {count}</p>
        <p>"Double Count: " {double_count}</p>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
