# 表单和输入

表单和表单输入是交互式应用程序的重要组成部分。在 Leptos 中与输入交互有两种基本模式，如果你熟悉 React、SolidJS 或类似的框架，你可能会认出它们：使用**受控**或**非受控**输入。

## 受控输入

在“受控输入”中，框架控制输入元素的状态。在每个 `input` 事件上，它都会更新一个保存当前状态的本地信号，而该信号又会更新输入的 `value` 属性。

有两件重要的事情需要记住：

1. `input` 事件在元素的（几乎）每次更改时触发，而 `change` 事件在（或多或少）你取消输入焦点时触发。你可能想要 `on:input`，但我们让你自由选择。
2. `value` _属性_ 只设置输入的初始值，即它只更新输入到你开始输入之前的点。之后，`value` _属性_ 会继续更新输入。由于这个原因，你通常想要设置 `prop:value`。（对于 `<input type="checkbox">` 上的 `checked` 和 `prop:checked` 也是如此。）

```rust
let (name, set_name) = create_signal("Controlled".to_string());

view! {
    <input type="text"
        on:input=move |ev| {
            // event_target_value 是一个 Leptos 辅助函数
            // 它的功能与 JavaScript 中的 event.target.value 相同
            // 但它简化了在 Rust 中使其工作所需的一些类型转换
            set_name(event_target_value(&ev));
        }

        // `prop:` 语法允许你更新 DOM 属性，
        // 而不是属性。
        prop:value=name
    />
    <p>"Name is: " {name}</p>
}
```

> #### 为什么需要 `prop:value`？
>
> Web 浏览器是现存最普遍和最稳定的图形用户界面渲染平台。在它们存在的三十年中，它们还保持了令人难以置信的向后兼容性。不可避免地，这意味着存在一些怪癖。
>
> 一个奇怪的怪癖是 HTML 属性和 DOM 元素属性之间存在区别，即在从 HTML 解析并可以使用 `.setAttribute()` 在 DOM 元素上设置的称为“属性”的东西与解析后的 HTML 元素的 JavaScript 类表示形式的字段称为“属性”的东西之间存在区别。
>
> 在 `<input value=...>` 的情况下，设置 `value` _属性_ 被定义为设置输入的初始值，而设置 `value` _属性_ 设置其当前值。通过打开 `about:blank` 并在浏览器控制台中逐行运行以下 JavaScript，也许最容易理解这一点：
>
> ```js
> // 创建一个输入并将其附加到 DOM
> const el = document.createElement("input");
> document.body.appendChild(el);
>
> el.setAttribute("value", "test"); // 更新输入
> el.setAttribute("value", "another test"); // 再次更新输入
>
> // 现在去输入框中输入：删除一些字符，等等。
>
> el.setAttribute("value", "one more time?");
> // 什么都没有改变。现在设置“初始值”没有任何作用
>
> // 但是...
> el.value = "But this works";
> ```
>
> 许多其他前端框架将属性和属性混为一谈，或者为正确设置值的输入创建了一个特殊情况。也许 Leptos 也应该这样做；但就目前而言，我更喜欢让用户最大程度地控制他们是在设置属性还是属性，并尽我所能教育人们了解实际的底层浏览器行为，而不是掩盖它。

## 非受控输入

在“非受控输入”中，浏览器控制输入元素的状态。我们不使用不断更新的信号来保存它的值，而是使用 [`NodeRef`](https://docs.rs/leptos/latest/leptos/struct.NodeRef.html) 在我们想要获取它的值时访问输入。

在这个例子中，我们只在 `<form>` 触发 `submit` 事件时通知框架。注意 [`leptos::html`](https://docs.rs/leptos/latest/leptos/html/index.html#) 模块的使用，它为每个 HTML 元素提供了一堆类型。

```rust
let (name, set_name) = create_signal("Uncontrolled".to_string());

let input_element: NodeRef<html::Input> = create_node_ref();

view! {
    <form on:submit=on_submit> // on_submit 在下面定义
        <input type="text"
            value=name
            node_ref=input_element
        />
        <input type="submit" value="Submit"/>
    </form>
    <p>"Name is: " {name}</p>
}
```

到目前为止，视图应该很容易理解。注意两件事：

1. 与受控输入示例不同，我们使用 `value`（而不是 `prop:value`）。这是因为我们只是设置输入的初始值，并让浏览器控制其状态。（我们可以使用 `prop:value` 代替。）
2. 我们使用 `node_ref=...` 来填充 `NodeRef`。（较旧的示例有时使用 `_ref`。它们是一回事，但 `node_ref` 具有更好的 rust-analyzer 支持。）

`NodeRef` 是一种响应式智能指针：我们可以使用它来访问底层的 DOM 节点。它的值将在元素渲染时设置。

```rust
let on_submit = move |ev: leptos::ev::SubmitEvent| {
    // 阻止页面重新加载！
    ev.prevent_default();

    // 在这里，我们将从输入中提取值
    let value = input_element()
        // 事件处理程序只能在视图
        // 被挂载到 DOM 后触发，因此 `NodeRef` 将是 `Some`
        .expect("<input> should be mounted")
        // `leptos::HtmlElement<html::Input>` 实现了 `Deref`
        // 到 `web_sys::HtmlInputElement`。
        // 这意味着我们可以调用 `HtmlInputElement::value()`
        // 来获取输入的当前值
        .value();
    set_name(value);
};
```

我们的 `on_submit` 处理程序将访问输入的值并使用它来调用 `set_name`。要访问存储在 `NodeRef` 中的 DOM 节点，我们可以简单地将其作为函数调用（或使用 `.get()`）。这将返回 `Option<leptos::HtmlElement<html::Input>>`，但我们知道该元素已经挂载（否则你如何触发此事件！），因此在这里解包是安全的。

然后我们可以调用 `.value()` 从输入中获取值，因为 `NodeRef` 允许我们访问正确类型的 HTML 元素。

查看 [`web_sys` 和 `HtmlElement`](../web_sys.md) 以了解有关使用 `leptos::HtmlElement` 的更多信息。另请参阅本页末尾的完整 CodeSandbox 示例。

## 特殊情况：`<textarea>` 和 `<select>`

两个表单元素往往会以不同的方式引起一些混淆。

### `<textarea>`

与 `<input>` 不同，`<textarea>` 元素不支持 `value` 属性。相反，它将其值作为纯文本节点接收在其 HTML 子级中。

在当前版本的 Leptos 中（实际上在 Leptos 0.1-0.6 中），创建动态子级会插入注释标记节点。如果你尝试使用它来显示动态内容，这可能会导致不正确的 `<textarea>` 渲染（以及 hydration 期间的问题）。

相反，你可以将非响应式的初始值作为子级传递，并使用 `prop:value` 来设置其当前值。（`<textarea>` 不支持 `value` **属性**，但 _确实_ 支持 `value` **属性**...）

```rust
view! {
    <textarea
        prop:value=move || some_value.get()
        on:input=/* etc */
    >
        /* 纯文本初始值，如果信号发生变化，则不会改变 */
        {some_value.get_untracked()}
    </textarea>
}
```

### `<select>`

同样，`<select>` 元素可以通过 `<select>` 本身的 `value` 属性来控制，这将选择具有该值的任何 `<option>`。

```rust
let (value, set_value) = create_signal(0i32);
view! {
  <select
    on:change=move |ev| {
      let new_value = event_target_value(&ev);
      set_value(new_value.parse().unwrap());
    }
    prop:value=move || value.get().to_string()
  >
    <option value="0">"0"</option>
    <option value="1">"1"</option>
    <option value="2">"2"</option>
  </select>
  // 一个循环选择选项的按钮
  <button on:click=move |_| set_value.update(|n| {
    if *n == 2 {
      *n = 0;
    } else {
      *n += 1;
    }
  })>
    "Next Option"
  </button>
}
```

```admonish sandbox title="受控与非受控表单 CodeSandbox" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/5-forms-0-5-rf2t7c?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/5-forms-0-5-rf2t7c?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源码</summary>

```rust
use leptos::{ev::SubmitEvent, *};

#[component]
fn App() -> impl IntoView {
    view! {
        <h2>"Controlled Component"</h2>
        <ControlledComponent/>
        <h2>"Uncontrolled Component"</h2>
        <UncontrolledComponent/>
    }
}

#[component]
fn ControlledComponent() -> impl IntoView {
    // 创建一个信号来保存值
    let (name, set_name) = create_signal("Controlled".to_string());

    view! {
        <input type="text"
            // 每当输入发生变化时触发事件
            on:input=move |ev| {
                // event_target_value 是一个 Leptos 辅助函数
                // 它的功能与 JavaScript 中的 event.target.value 相同
                // 但它简化了在 Rust 中使其工作所需的一些类型转换
                set_name(event_target_value(&ev));
            }

            // `prop:` 语法允许你更新 DOM 属性，
            // 而不是属性。
            //
            // 重要提示：`value` *属性* 只设置
            // 初始值，直到你进行更改。
            // `value` *属性* 设置当前值。
            // 这是 DOM 的一个怪癖；我并没有发明它。
            // 其他框架掩盖了这一点；我认为
            // 让你能够访问真正工作的浏览器
            // 更为重要。
            //
            // tl;dr：对表单输入使用 prop:value
            prop:value=name
        />
        <p>"Name is: " {name}</p>
    }
}

#[component]
fn UncontrolledComponent() -> impl IntoView {
    // 导入 <input> 的类型
    use leptos::html::Input;

    let (name, set_name) = create_signal("Uncontrolled".to_string());

    // 我们将使用 NodeRef 来存储对输入元素的引用
    // 这将在创建元素时填充
    let input_element: NodeRef<Input> = create_node_ref();

    // 在表单 `submit` 事件发生时触发
    // 这会将 <input> 的值存储在我们的信号中
    let on_submit = move |ev: SubmitEvent| {
        // 阻止页面重新加载！
        ev.prevent_default();

        // 在这里，我们将从输入中提取值
        let value = input_element()
            // 事件处理程序只能在视图
            // 被挂载到 DOM 后触发，因此 `NodeRef` 将是 `Some`
            .expect("<input> to exist")
            // `NodeRef` 为 DOM 元素类型实现了 `Deref`
            // 这意味着我们可以调用 `HtmlInputElement::value()`
            // 来获取输入的当前值
            .value();
        set_name(value);
    };

    view! {
        <form on:submit=on_submit>
            <input type="text"
                // 在这里，我们使用 `value` *属性* 只设置
                // 初始值，之后让浏览器维护
                // 状态。
                value=name

                // 在 `input_element` 中存储对此输入的引用
                node_ref=input_element
            />
            <input type="submit" value="Submit"/>
        </form>
        <p>"Name is: " {name}</p>
    }
}

// 这个 `main` 函数是应用程序的入口点
// 它只是将我们的组件挂载到 <body>
// 因为我们将其定义为 `fn App`，我们现在可以在
// 模板中将其用作 <App/>
fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
