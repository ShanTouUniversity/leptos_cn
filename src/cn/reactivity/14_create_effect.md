# 使用 `create_effect` 响应变化

我们已经走到这一步，而没有提到响应式系统的一半：效果。

响应性工作分为两部分：更新单个响应式值（“信号”）会通知依赖于它们的代码片段（“效果”）它们需要再次运行。响应式系统的这两部分是相互依赖的。没有效果，信号可以在响应式系统内更改，但永远无法以与外部世界交互的方式观察到。没有信号，效果只运行一次，因为没有可订阅的可观察值。效果实际上是响应式系统的“副作用”：它们的存在是为了将响应式系统与其外部的非响应式世界同步。

到目前为止，我们看到的整个响应式 DOM 渲染器背后隐藏着一个名为 `create_effect` 的函数。

[`create_effect`](https://docs.rs/leptos_reactive/latest/leptos_reactive/fn.create_effect.html) 接受一个函数作为参数。它立即运行该函数。如果你在该函数内部访问任何响应式信号，它会向响应式运行时注册该效果依赖于该信号的事实。每当效果依赖的信号之一发生变化时，该效果就会再次运行。

```rust
let (a, set_a) = create_signal(0);
let (b, set_b) = create_signal(0);

create_effect(move |_| {
  // 立即打印“值：0”并订阅 `a`
  log::debug!("值：{}", a());
});
```

调用效果函数时，会传递一个参数，该参数包含它上次运行时返回的值。在初始运行时，这是 `None`。

默认情况下，效果**不会在服务器上运行**。这意味着你可以在效果函数中调用特定于浏览器的 API，而不会导致问题。如果你需要在服务器上运行效果，请使用 [`create_isomorphic_effect`](https://docs.rs/leptos_reactive/latest/leptos_reactive/fn.create_isomorphic_effect.html)。

## 自动跟踪和动态依赖

如果你熟悉像 React 这样的框架，你可能会注意到一个关键的区别。React 和类似的框架通常要求你传递一个“依赖数组”，一组显式变量，用于确定何时应该重新运行效果。

因为 Leptos 来自同步响应式编程的传统，所以我们不需要这个显式的依赖列表。相反，我们根据效果中访问的信号自动跟踪依赖关系。

这有两个效果（没有双关语）。依赖关系是：

1. **自动**：你不需要维护依赖列表，也不需要担心应该包含什么或不应该包含什么。框架只是跟踪哪些信号可能导致效果重新运行，并为你处理它。
2. **动态**：依赖列表在每次效果运行时都会被清除和更新。如果你的效果包含一个条件（例如），则只会跟踪当前分支中使用的信号。这意味着效果重新运行的次数绝对是最少的。

> 如果这听起来很神奇，如果你想深入了解自动依赖跟踪是如何工作的，[请查看此视频](https://www.youtube.com/watch?v=GWB3vTWeLd4)。（抱歉音量有点低！）

## 效果作为零成本抽象

虽然它们不是最严格意义上的“零成本抽象”——它们需要一些额外的内存使用，在运行时存在，等等——但在更高的层次上，从你正在其中进行的任何昂贵的 API 调用或其他工作的角度来看，效果是零成本抽象。考虑到你对它们的描述，它们只在绝对必要的次数内重新运行。

想象一下，我正在创建某种聊天软件，我希望人们能够显示他们的全名，或者只是他们的名字，并在他们的名字改变时通知服务器：

```rust
let (first, set_first) = create_signal(String::new());
let (last, set_last) = create_signal(String::new());
let (use_last, set_use_last) = create_signal(true);

// 这会在任何时候将名称添加到日志中
// 任何一个源信号发生变化
create_effect(move |_| {
    log(
        if use_last() {
            format!("{} {}", first(), last())
        } else {
            first()
        },
    )
});
```

如果 `use_last` 为 `true`，则每当 `first`、`last` 或 `use_last` 发生变化时，效果都应该重新运行。但是，如果我将 `use_last` 切换为 `false`，则 `last` 的更改永远不会导致全名更改。实际上，`last` 将从依赖列表中删除，直到 `use_last` 再次切换。如果我在 `use_last` 仍然为 `false` 的情况下多次更改 `last`，这将避免我们向 API 发送多个不必要的请求。

## 何时使用 `create_effect`，何时不使用？

效果旨在将响应式系统与其外部的非响应式世界同步，而不是在不同的响应式值之间同步。换句话说：使用效果从一个信号中读取值并将其设置到另一个信号中总是次优的。

如果你需要定义一个依赖于其他信号值的信号，请使用派生信号或 [`create_memo`](https://docs.rs/leptos_reactive/latest/leptos_reactive/fn.create_memo.html)。在效果内部写入信号并不是世界末日，它不会导致你的计算机着火，但派生信号或 memo 始终更好——不仅因为数据流清晰，而且因为性能更好。

```rust
let (a, set_a) = create_signal(0);

// ⚠️ 不太好
let (b, set_b) = create_signal(0);
create_effect(move |_| {
    set_b(a() * 2);
});

// ✅ 太棒了！
let b = move || a() * 2;
```

如果你需要将一些响应式值与外部的非响应式世界同步——例如 Web API、控制台、文件系统或 DOM——在效果中写入信号是一种很好的方法。然而，在许多情况下，你会发现你实际上是在事件监听器或其他东西内部写入信号，而不是在效果内部。在这种情况下，你应该查看 [`leptos-use`](https://leptos-use.rs/)，看看它是否已经提供了一个响应式包装原语来做到这一点！

> 如果你想了解更多关于何时应该以及何时不应该使用 `create_effect` 的信息，[请查看此视频](https://www.youtube.com/watch?v=aQOFJQ2JkvQ) 以获得更深入的了解！

## 效果和渲染

我们已经设法在不提及效果的情况下走到了这一步，因为它们内置于 Leptos DOM 渲染器中。我们已经看到，你可以创建一个信号并将其传递到 `view` 宏中，并且每当信号发生变化时，它都会更新相关的 DOM 节点：

```rust
let (count, set_count) = create_signal(0);

view! {
    <p>{count}</p>
}
```

这是有效的，因为框架本质上创建了一个包装此更新的效果。你可以想象 Leptos 将此视图转换为如下内容：

```rust
let (count, set_count) = create_signal(0);

// 创建一个 DOM 元素
let document = leptos::document();
let p = document.create_element("p").unwrap();

// 创建一个效果来响应式地更新文本
create_effect(move |prev_value| {
    // 首先，访问信号的值并将其转换为字符串
    let text = count().to_string();

    // 如果这与先前值不同，则更新节点
    if prev_value != Some(text) {
        p.set_text_content(&text);
    }

    // 返回此值，以便我们可以记住下次更新
    text
});
```

每次更新 `count` 时，此效果都会重新运行。这就是允许对 DOM 进行响应式、细粒度更新的原因。

## 使用 `watch` 进行显式、可取消的跟踪

除了 `create_effect`，Leptos 还提供了一个 [`watch`](https://docs.rs/leptos_reactive/latest/leptos_reactive/fn.watch.html) 函数，它可以用于两个主要目的：

1. 通过显式传入一组要跟踪的值来分离跟踪和响应更改。
2. 通过调用停止函数取消跟踪。

与 `create_resource` 一样，`watch` 接受第一个参数，它是响应式跟踪的，第二个参数则不是。每当其 `deps` 参数中的响应式值发生更改时，就会运行 `callback`。`watch` 返回一个函数，可以调用该函数来停止跟踪依赖项。

```rust
let (num, set_num) = create_signal(0);

let stop = watch(
    move || num.get(),
    move |num, prev_num, _| {
        log::debug!("Number: {}; Prev: {:?}", num, prev_num);
    },
    false,
);

set_num.set(1); // > "数字：1；上一个：Some(0)"

stop(); // 停止观察

set_num.set(2); // （什么都没有发生）
```

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/14-effect-0-5-d6hkch?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/14-effect-0-5-d6hkch?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源码</summary>

```rust
use leptos::html::Input;
use leptos::*;

#[derive(Copy, Clone)]
struct LogContext(RwSignal<Vec<String>>);

#[component]
fn App() -> impl IntoView {
    // 这里只是创建一个可见的日志
    // 你可以忽略它...
    let log = create_rw_signal::<Vec<String>>(vec![]);
    let logged = move || log().join("\n");

    // newtype 模式在这里不是*必需的*，但这是一个好习惯
    // 它避免了与其他可能的未来 `RwSignal<Vec<String>>` 上下文的混淆
    // 并使其更容易引用
    provide_context(LogContext(log));

    view! {
        <CreateAnEffect/>
        <pre>{logged}</pre>
    }
}

#[component]
fn CreateAnEffect() -> impl IntoView {
    let (first, set_first) = create_signal(String::new());
    let (last, set_last) = create_signal(String::new());
    let (use_last, set_use_last) = create_signal(true);

    // 这会在任何时候将名称添加到日志中
    // 任何一个源信号发生变化
    create_effect(move |_| {
        log(if use_last() {
            with!(|first, last| format!("{first} {last}"))
        } else {
            first()
        })
    });

    view! {
        <h1>
            <code>"create_effect"</code>
            " Version"
        </h1>
        <form>
            <label>
                "First Name"
                <input
                    type="text"
                    name="first"
                    prop:value=first
                    on:change=move |ev| set_first(event_target_value(&ev))
                />
            </label>
            <label>
                "Last Name"
                <input
                    type="text"
                    name="last"
                    prop:value=last
                    on:change=move |ev| set_last(event_target_value(&ev))
                />
            </label>
            <label>
                "Show Last Name"
                <input
                    type="checkbox"
                    name="use_last"
                    prop:checked=use_last
                    on:change=move |ev| set_use_last(event_target_checked(&ev))
                />
            </label>
        </form>
    }
}

#[component]
fn ManualVersion() -> impl IntoView {
    let first = create_node_ref::<Input>();
    let last = create_node_ref::<Input>();
    let use_last = create_node_ref::<Input>();

    let mut prev_name = String::new();
    let on_change = move |_| {
        log("      listener");
        let first = first.get().unwrap();
        let last = last.get().unwrap();
        let use_last = use_last.get().unwrap();
        let this_one = if use_last.checked() {
            format!("{} {}", first.value(), last.value())
        } else {
            first.value()
        };

        if this_one != prev_name {
            log(&this_one);
            prev_name = this_one;
        }
    };

    view! {
        <h1>"Manual Version"</h1>
        <form on:change=on_change>
            <label>"First Name" <input type="text" name="first" node_ref=first/></label>
            <label>"Last Name" <input type="text" name="last" node_ref=last/></label>
            <label>
                "Show Last Name" <input type="checkbox" name="use_last" checked node_ref=use_last/>
            </label>
        </form>
    }
}

#[component]
fn EffectVsDerivedSignal() -> impl IntoView {
    let (my_value, set_my_value) = create_signal(String::new());
    // 不要这样做。
    /*let (my_optional_value, set_optional_my_value) = create_signal(Option::<String>::None);

    create_effect(move |_| {
        if !my_value.get().is_empty() {
            set_optional_my_value(Some(my_value.get()));
        } else {
            set_optional_my_value(None);
        }
    });*/

    // 这样做
    let my_optional_value =
        move || (!my_value.with(String::is_empty)).then(|| Some(my_value.get()));

    view! {
        <input prop:value=my_value on:input=move |ev| set_my_value(event_target_value(&ev))/>

        <p>
            <code>"my_optional_value"</code>
            " is "
            <code>
                <Show when=move || my_optional_value().is_some() fallback=|| view! { "None" }>
                    "Some(\""
                    {my_optional_value().unwrap()}
                    "\")"
                </Show>
            </code>
        </p>
    }
}

#[component]
pub fn Show<F, W, IV>(
    /// Show 包装的组件
    children: Box<dyn Fn() -> Fragment>,
    /// 返回一个布尔值的闭包，用于确定此内容是否运行
    when: W,
    /// 在 when 语句为 false 时返回渲染内容的闭包
    fallback: F,
) -> impl IntoView
where
    W: Fn() -> bool + 'static,
    F: Fn() -> IV + 'static,
    IV: IntoView,
{
    let memoized_when = create_memo(move |_| when());

    move || match memoized_when.get() {
        true => children().into_view(),
        false => fallback().into_view(),
    }
}

fn log(msg: impl std::fmt::Display) {
    let log = use_context::<LogContext>().unwrap().0;
    log.update(|log| log.push(msg.to_string()));
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
