# 控制流

在大多数应用程序中，你有时需要做出决定：我应该渲染视图的这一部分吗？我应该渲染 `<ButtonA/>` 还是 `<WidgetB/>`？这就是**控制流**。

## 一些技巧

在考虑如何使用 Leptos 来做到这一点时，记住以下几点很重要：

1. Rust 是一种面向表达式的语言：像 `if x() { y } else { z }` 和 `match x() { ... }` 这样的控制流表达式会返回它们的值。这使得它们对于声明式用户界面非常有用。
2. 对于任何实现了 `IntoView` 的 `T`——换句话说，对于 Leptos 知道如何渲染的任何类型——`Option<T>` 和 `Result<T, impl Error>` _也_ 实现了 `IntoView`。正如 `Fn() -> T` 渲染一个响应式的 `T` 一样，`Fn() -> Option<T>` 和 `Fn() -> Result<T, impl Error>` 也是响应式的。
3. Rust 有很多方便的辅助函数，比如 [Option::map](https://doc.rust-lang.org/std/option/enum.Option.html#method.map)、[Option::and_then](https://doc.rust-lang.org/std/option/enum.Option.html#method.and_then)、[Option::ok_or](https://doc.rust-lang.org/std/option/enum.Option.html#method.ok_or)、[Result::map](https://doc.rust-lang.org/std/result/enum.Result.html#method.map)、[Result::ok](https://doc.rust-lang.org/std/result/enum.Result.html#method.ok) 和 [bool::then](https://doc.rust-lang.org/std/primitive.bool.html#method.then)，它们允许你以声明式的方式在几种不同的标准类型之间进行转换，所有这些类型都可以被渲染。特别是在 `Option` 和 `Result` 文档中花费时间是提升你的 Rust 水平的最佳方法之一。
4. 永远记住：要成为响应式的，值必须是函数。你会看到我在下面不断地将东西包装在一个 `move ||` 闭包中。这是为了确保当它们依赖的信号发生变化时，它们能够实际重新运行，从而保持 UI 的响应性。

## 那又怎样？

为了把这些点联系起来：这意味着你实际上可以使用原生 Rust 代码实现大部分的控制流，而无需任何控制流组件或特殊知识。

例如，让我们从一个简单的信号和派生信号开始：

```rust
let (value, set_value) = create_signal(0);
let is_odd = move || value() & 1 == 1;
```

> 如果你不认识 `is_odd` 发生了什么，不要太担心。这只是通过对 `1` 进行按位 `AND` 来测试整数是否为奇数的一种简单方法。

我们可以使用这些信号和普通的 Rust 来构建大多数控制流。

### `if` 语句

假设我想在数字为奇数时渲染一些文本，在数字为偶数时渲染其他一些文本。那么，这样如何？

```rust
view! {
    <p>
    {move || if is_odd() {
        "Odd"
    } else {
        "Even"
    }}
    </p>
}
```

`if` 表达式返回它的值，并且 `&str` 实现了 `IntoView`，所以 `Fn() -> &str` 实现了 `IntoView`，所以这... 就行了！

### `Option<T>`

假设我们想在数字为奇数时渲染一些文本，在数字为偶数时什么也不渲染。

```rust
let message = move || {
    if is_odd() {
        Some("Ding ding ding!")
    } else {
        None
    }
};

view! {
    <p>{message}</p>
}
```

这很好用。如果我们愿意，我们可以使用 `bool::then()` 使它更短一些。

```rust
let message = move || is_odd().then(|| "Ding ding ding!");
view! {
    <p>{message}</p>
}
```

你甚至可以内联它，如果你愿意的话，虽然我个人有时喜欢通过将东西从 `view` 中拉出来获得更好的 `cargo fmt` 和 `rust-analyzer` 支持。

### `match` 语句

我们仍然只是在编写普通的 Rust 代码，对吧？所以你拥有 Rust 模式匹配的所有能力。

```rust
let message = move || {
    match value() {
        0 => "Zero",
        1 => "One",
        n if is_odd() => "Odd",
        _ => "Even"
    }
};
view! {
    <p>{message}</p>
}
```

为什么不呢？YOLO，对吧？

## 避免过度渲染

不要太 YOLO。

我们刚刚做的所有事情基本上都没问题。但是有一件事你应该记住并尽量小心。到目前为止，我们创建的每个控制流函数基本上都是一个派生信号：每次值发生变化时它都会重新运行。在上面的例子中，值在每次变化时都会从偶数切换到奇数，这很好。

但是考虑下面的例子：

```rust
let (value, set_value) = create_signal(0);

let message = move || if value() > 5 {
    "Big"
} else {
    "Small"
};

view! {
    <p>{message}</p>
}
```

这当然_可以_。但是如果你添加一个日志，你可能会感到惊讶

```rust
let message = move || if value() > 5 {
    logging::log!("{}: rendering Big", value());
    "Big"
} else {
    logging::log!("{}: rendering Small", value());
    "Small"
};
```

当用户点击一个按钮时，你会看到类似这样的内容：

```
1: rendering Small
2: rendering Small
3: rendering Small
4: rendering Small
5: rendering Small
6: rendering Big
7: rendering Big
8: rendering Big
... 无限循环
```

每次 `value` 发生变化时，它都会重新运行 `if` 语句。这在响应性工作原理中是有道理的。但它有一个缺点。对于一个简单的文本节点，重新运行 `if` 语句并重新渲染没什么大不了的。但是想象一下它是这样的：

```rust
let message = move || if value() > 5 {
    <Big/>
} else {
    <Small/>
};
```

这会重新渲染 `<Small/>` 五次，然后无限重新渲染 `<Big/>`。如果它们正在加载资源、创建信号，或者仅仅是创建 DOM 节点，这就是不必要的工作。

### `<Show/>`

[`<Show/>`](https://docs.rs/leptos/latest/leptos/fn.Show.html) 组件就是答案。你给它传递一个 `when` 条件函数，一个在 `when` 函数返回 `false` 时显示的 `fallback`，以及在 `when` 为 `true` 时渲染的子级。

```rust
let (value, set_value) = create_signal(0);

view! {
  <Show
    when=move || { value() > 5 }
    fallback=|| view! { <Small/> }
  >
    <Big/>
  </Show>
}
```

`<Show/>` 会记住 `when` 条件，因此它只渲染一次 `<Small/>`，并继续显示相同的组件，直到 `value` 大于 5；然后它渲染一次 `<Big/>`，并继续无限期地显示它，或者直到 `value` 小于 5 然后再次渲染 `<Small/>`。

当使用动态 `if` 表达式时，这是一个避免重新渲染的有用工具。与往常一样，这会有一些开销：对于一个非常简单的节点（比如更新单个文本节点，或者更新一个类或属性），`move || if ...` 会更有效率。但是，如果渲染任何一个分支的成本都很高，那就使用 `<Show/>`。

## 注意：类型转换

在本节中，最后还有一件重要的事情要说。

`view` 宏不会返回最通用的包装类型 [`View`](https://docs.rs/leptos/latest/leptos/enum.View.html)。相反，它返回类型为 `Fragment` 或 `HtmlElement<Input>` 的东西。如果从条件的不同分支返回不同的 HTML 元素，这可能会有点烦人：

```rust,compile_error
view! {
    <main>
        {move || match is_odd() {
            true if value() == 1 => {
                // 返回 HtmlElement<Pre>
                view! { <pre>"One"</pre> }
            },
            false if value() == 2 => {
                // 返回 HtmlElement<P>
                view! { <p>"Two"</p> }
            }
            // 返回 HtmlElement<Textarea>
            _ => view! { <textarea>{value()}</textarea> }
        }}
    </main>
}
```

这种强类型实际上非常强大，因为 [`HtmlElement`](https://docs.rs/leptos/0.1.3/leptos/struct.HtmlElement.html) 除了其他功能外，还是一个智能指针：每个 `HtmlElement<T>` 类型都为相应的底层 `web_sys` 类型实现了 `Deref`。换句话说，在浏览器中，你的 `view` 返回的是真正的 DOM 元素，你可以访问它们上的原生 DOM 方法。

但这在像这样的条件逻辑中可能会有点烦人，因为在 Rust 中你不能从条件的不同分支返回不同的类型。有两种方法可以让你摆脱这种情况：

1. 如果你有多个 `HtmlElement` 类型，可以使用 [`.into_any()`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.into_any) 将它们转换为 `HtmlElement<AnyElement>`
2. 如果你有各种各样的视图类型，而不仅仅是 `HtmlElement`，可以使用 [`.into_view()`](https://docs.rs/leptos/latest/leptos/trait.IntoView.html#tymethod.into_view) 将它们转换为 `View`。

以下是添加了转换的相同示例：

```rust,compile_error
view! {
    <main>
        {move || match is_odd() {
            true if value() == 1 => {
                // 返回 HtmlElement<Pre>
                view! { <pre>"One"</pre> }.into_any()
            },
            false if value() == 2 => {
                // 返回 HtmlElement<P>
                view! { <p>"Two"</p> }.into_any()
            }
            // returns HtmlElement<Textarea>
            _ => view! { <textarea>{value()}</textarea> }.into_any()
        }}
    </main>
}
```

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/6-control-flow-0-5-4yn7qz?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/6-control-flow-0-5-4yn7qz?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源码</summary>

```rust
use leptos::*;

#[component]
fn App() -> impl IntoView {
    let (value, set_value) = create_signal(0);
    let is_odd = move || value() & 1 == 1;
    let odd_text = move || if is_odd() { Some("How odd!") } else { None };

    view! {
        <h1>"Control Flow"</h1>

        // 用于更新和显示值的简单 UI
        <button on:click=move |_| set_value.update(|n| *n += 1)>
            "+1"
        </button>
        <p>"Value is: " {value}</p>

        <hr/>

        <h2><code>"Option<T>"</code></h2>
        // 对于任何实现了 `IntoView` 的 `T`，
        // `Option<T>` 也实现了 `IntoView`

        <p>{odd_text}</p>
        // 这意味着你可以对它使用 `Option` 方法
        <p>{move || odd_text().map(|text| text.len())}</p>

        <h2>"Conditional Logic"</h2>
        // 你可以通过几种方式进行动态条件 if-then-else
        // 逻辑
        //
        // a. 函数中的 "if" 表达式
        //    这将在每次值发生变化时重新渲染，这使得它适用于轻量级 UI
        <p>
            {move || if is_odd() {
                "Odd"
            } else {
                "Even"
            }}
        </p>

        // b. 切换某种类
        //    这对于经常切换的元素来说很聪明，因为它不会破坏
        //    它在不同状态之间的状态
        //    （你可以在 `index.html` 中找到 `hidden` 类）
        <p class:hidden=is_odd>"Appears if even."</p>

        // c. <Show/> 组件
        //    这只会渲染一次 fallback 和子级，并且是惰性的，并且在
        //    需要时在它们之间切换。这使得它在很多情况下比 {move || if ...} 块更有效率
        <Show when=is_odd
            fallback=|| view! { <p>"Even steven"</p> }
        >
            <p>"Oddment"</p>
        </Show>

        // d. 因为 `bool::then()` 将 `bool` 转换为
        //    `Option`，你可以使用它来创建一个显示/隐藏切换
        {move || is_odd().then(|| view! { <p>"Oddity!"</p> })}

        <h2>"Converting between Types"</h2>
        // e. 注意：如果分支返回不同的类型，
        //    你可以使用
        //    `.into_any()`（对于不同的 HTML 元素类型）
        //    或 `.into_view()`（对于所有视图类型）在它们之间进行转换
        {move || match is_odd() {
            true if value() == 1 => {
                // <pre> 返回 HtmlElement<Pre>
                view! { <pre>"One"</pre> }.into_any()
            },
            false if value() == 2 => {
                // <p> 返回 HtmlElement<P>
                // 所以我们转换为更通用的类型
                view! { <p>"Two"</p> }.into_any()
            }
            _ => view! { <textarea>{value()}</textarea> }.into_any()
        }}
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>