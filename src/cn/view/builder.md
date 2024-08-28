# 无宏：视图构建器语法

> 如果你对到目前为止描述的 `view!` 宏语法感到满意，欢迎跳过本章。本节中描述的构建器语法始终可用，但并非必需。

出于某种原因，许多开发人员更喜欢避免使用宏。也许你不喜欢有限的 `rustfmt` 支持。（不过，你应该看看 [`leptosfmt`](https://github.com/bram209/leptosfmt)，这是一个很棒的工具！）也许你担心宏对编译时间的影响。也许你更喜欢纯 Rust 语法的审美，或者你难以在类似 HTML 的语法和你的 Rust 代码之间进行上下文切换。或者，也许你希望在创建和操作 HTML 元素方面比 `view` 宏提供的更多灵活性。

如果你属于这些阵营中的任何一个，那么构建器语法可能适合你。

`view` 宏将类似 HTML 的语法扩展为一系列 Rust 函数和方法调用。如果你不想使用 `view` 宏，你可以简单地自己使用这种扩展语法。而且它实际上非常好！

首先，如果你愿意，你甚至可以放弃 `#[component]` 宏：组件只是一个创建视图的设置函数，因此你可以将组件定义为一个简单的函数调用：

```rust
pub fn counter(initial_value: i32, step: u32) -> impl IntoView { }
```

元素是通过调用与 HTML 元素同名的函数来创建的：

```rust
p()
```

你可以使用 [`.child()`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.child) 将子级添加到元素中，它接受一个子级或一个实现 [`IntoView`](https://docs.rs/leptos/latest/leptos/trait.IntoView.html) 类型的元组或数组。

```rust
p().child((em().child("Big, "), strong().child("bold "), "text"))
```

属性使用 [`.attr()`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.attr) 添加。这可以接受与你可以作为属性传递给视图宏的任何相同类型（实现 [`IntoAttribute`](https://docs.rs/leptos/latest/leptos/trait.IntoAttribute.html) 的类型）。

```rust
p().attr("id", "foo").attr("data-count", move || count().to_string())
```

类似地，`class:`、`prop:` 和 `style:` 语法直接映射到 [`.class()`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.class)、[`.prop()`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.prop) 和 [`.style()`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.style) 方法。

事件监听器可以使用 [`.on()`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.on) 添加。[`leptos::ev`](https://docs.rs/leptos/latest/leptos/ev/index.html) 中的类型化事件可以防止事件名称中的拼写错误，并允许在回调函数中进行正确的类型推断。

```rust
button()
    .on(ev::click, move |_| set_count.update(|count| *count = 0))
    .child("Clear")
```

> 许多其他方法可以在 [`HtmlElement`](https://docs.rs/leptos/latest/leptos/struct.HtmlElement.html#method.child) 文档中找到，包括一些在 `view` 宏中没有直接提供的方法。

如果你喜欢这种风格，所有这些加起来就是一个非常 Rust 的语法来构建功能齐全的视图。

```rust
/// 一个简单的计数器视图。
// 组件实际上只是一个函数调用：它运行一次以创建 DOM 和响应式系统
pub fn counter(initial_value: i32, step: u32) -> impl IntoView {
    let (count, set_count) = create_signal(0);
    div().child((
        button()
            // 在 leptos::ev 中找到的类型化事件
            // 1) 防止事件名称中的拼写错误
            // 2) 允许在回调中进行正确的类型推断
            .on(ev::click, move |_| set_count.update(|count| *count = 0))
            .child("Clear"),
        button()
            .on(ev::click, move |_| set_count.update(|count| *count -= 1))
            .child("-1"),
        span().child(("Value: ", move || count.get(), "!")),
        button()
            .on(ev::click, move |_| set_count.update(|count| *count += 1))
            .child("+1"),
    ))
}
```

这样做还有一个好处，那就是更加灵活：因为这些都是普通的 Rust 函数和方法，所以更容易在迭代器适配器等东西中使用它们，而无需任何额外的“魔法”：

```rust
// 获取一组属性名称和值
let attrs: Vec<(&str, AttributeValue)> = todo!();
// 你可以使用构建器语法将这些“扩展”到元素上，
// 这是视图宏无法实现的
let p = attrs
    .into_iter()
    .fold(p(), |el, (name, value)| el.attr(name, value));

```

> ## 性能说明
>
> 一个警告：`view` 宏在服务器端渲染（SSR）模式下应用了重大的优化，以显著提高 HTML 渲染性能（根据任何给定应用程序的特征，速度可提高 2-4 倍）。它通过在编译时分析你的 `view` 并将静态部分转换为简单的 HTML 字符串，而不是将它们扩展为构建器语法来做到这一点。
>
> 这意味着两件事：
>
> 1. 构建器语法和 `view` 宏不应该混合，或者应该非常小心地混合：至少在 SSR 模式下，`view` 的输出应该被视为一个“黑盒子”，不能对其应用额外的构建器方法，否则会导致不一致。
> 2. 使用构建器语法会导致 SSR 性能低于最佳水平。无论如何它都不会很慢（无论如何都值得运行你自己的基准测试），只是比 `view` 优化版本慢。
