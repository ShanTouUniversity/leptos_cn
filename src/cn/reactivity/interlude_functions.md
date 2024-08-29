# 插曲：响应式和函数

我们的一位核心贡献者最近对我说：“在开始使用 Leptos 之前，我从来没有这么频繁地使用闭包。” 这是真的。闭包是任何 Leptos 应用程序的核心。它有时看起来有点傻：

```rust
// 信号保存一个值，并且可以更新
let (count, set_count) = create_signal(0);

// 派生信号是一个访问其他信号的函数
let double_count = move || count() * 2;
let count_is_odd = move || count() & 1 == 1;
let text = move || if count_is_odd() {
    "odd"
} else {
    "even"
};

// 效果会自动跟踪它所依赖的信号
// 并在它们发生变化时重新运行
create_effect(move |_| {
    logging::log!("text = {}", text());
});

view! {
    <p>{move || text().to_uppercase()}</p>
}
```

到处都是闭包！

但为什么？

## 函数和 UI 框架

函数是每个 UI 框架的核心。这很有道理。创建用户界面基本上分为两个阶段：

1. 初始渲染
2. 更新

在 Web 框架中，框架进行某种初始渲染。然后它将控制权交还给浏览器。当某些事件触发（如鼠标单击）或异步任务完成（如 HTTP 请求完成）时，浏览器会唤醒框架来更新某些内容。框架运行某种代码来更新你的用户界面，然后再次休眠，直到浏览器再次唤醒它。

这里的关键词是“运行某种代码”。在任意时间点“运行某种代码”的自然方式——在 Rust 或任何其他编程语言中——是调用函数。事实上，每个 UI 框架都是基于一遍又一遍地重新运行某种函数：

1. 像 React、Yew 或 Dioxus 这样的虚拟 DOM（VDOM）框架一遍又一遍地重新运行组件或渲染函数，以生成一个虚拟 DOM 树，该树可以与之前的结果进行协调以修补 DOM
2. 像 Angular 和 Svelte 这样的编译框架将你的组件模板分为“创建”和“更新”函数，当它们检测到组件状态发生变化时，会重新运行更新函数
3. 在像 SolidJS、Sycamore 或 Leptos 这样的细粒度响应式框架中，*你* 定义了重新运行的函数

这就是我们所有组件正在做的事情。

以我们典型的 `<SimpleCounter/>` 示例的最简单形式为例：

```rust
#[component]
pub fn SimpleCounter() -> impl IntoView {
    let (value, set_value) = create_signal(0);

    let increment = move |_| set_value.update(|value| *value += 1);

    view! {
        <button on:click=increment>
            {value}
        </button>
    }
}
```

`SimpleCounter` 函数本身只运行一次。`value` 信号只创建一次。框架将 `increment` 函数作为事件监听器传递给浏览器。当你点击按钮时，浏览器会调用 `increment`，它通过 `set_value` 更新 `value`。这会更新在我们的视图中由 `{value}` 表示的单个文本节点。

闭包是响应式的关键。它们为框架提供了响应更改重新运行应用程序中最小可能单元的能力。

所以请记住两件事：

1. 你的组件函数是一个设置函数，而不是一个渲染函数：它只运行一次。
2. 为了使你的视图模板中的值具有响应性，它们必须是函数：要么是信号（实现 `Fn` 特征），要么是闭包。
