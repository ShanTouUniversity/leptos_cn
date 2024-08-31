# 全局状态管理

到目前为止，我们只处理过组件中的局部状态，并且我们已经了解了如何在父子组件之间协调状态。有时，人们会寻找更通用的全局状态管理解决方案，该解决方案可以在整个应用程序中使用。

一般来说，**你不需要本章**。典型的模式是将你的应用程序组合成组件，每个组件管理自己的局部状态，而不是将所有状态存储在全局结构中。但是，在某些情况下（例如主题设置、保存用户设置或在 UI 不同部分的组件之间共享数据），你可能希望使用某种全局状态管理。

三种最佳的全局状态方法是

1. 使用路由器通过 URL 驱动全局状态
2. 通过上下文传递信号
3. 创建一个全局状态结构体，并使用 `create_slice` 创建指向它的镜头

## 选项 #1：URL 作为全局状态

在很多方面，URL 实际上是存储全局状态的最佳方式。它可以从你的树中的任何组件、任何位置访问。有一些原生 HTML 元素（如 `<form>` 和 `<a>`）专门用于更新 URL。而且它可以在页面重新加载和不同设备之间持久存在；你可以与朋友共享 URL，或者将其从手机发送到笔记本电脑，其中存储的任何状态都将被复制。

本教程的接下来几节将介绍路由器，我们将深入探讨这些主题。

但现在，我们将只关注选项 #2 和 #3。

## 选项 #2：通过上下文传递信号

在[父子组件通信](view/08_parent_child.md)部分，我们看到你可以使用 `provide_context` 将信号从父组件传递给子组件，并使用 `use_context` 在子组件中读取它。但 `provide_context` 可以在任何距离上工作。如果你想创建一个全局信号来保存某个状态片段，你可以在你提供它的组件的后代中的任何地方提供它并通过上下文访问它。

通过上下文提供的信号只会在读取它的地方引起响应式更新，而不会在两者之间的任何组件中引起更新，因此它即使在远处也能保持细粒度响应式更新的能力。

我们首先在应用程序的根目录中创建一个信号，并使用 `provide_context` 将其提供给它的所有子级和后代。

```rust
#[component]
fn App() -> impl IntoView {
    // 这里我们在根目录中创建一个信号，可以在应用程序的任何地方使用它。
    let (count, set_count) = create_signal(0);
    // 我们将把设置器传递给特定的组件，
    // 但通过上下文将计数本身提供给整个应用程序
    provide_context(count);

    view! {
        // SetterButton 允许修改计数
        <SetterButton set_count/>
        // 这些消费者只能读取它
        // 但是如果我们愿意，我们可以通过传递 `set_count` 来赋予他们写权限
        <FancyMath/>
        <ListItems/>
    }
}
```

`<SetterButton/>` 是我们已经写过好几次的计数器类型。
（如果你不明白我的意思，请参阅下面的沙盒。）

`<FancyMath/>` 和 `<ListItems/>` 都使用我们通过 `use_context` 提供的信号并对其进行处理。

```rust
/// 使用全局计数进行一些“花哨”数学运算的组件
#[component]
fn FancyMath() -> impl IntoView {
    // 这里我们使用 `use_context` 使用全局计数信号
    let count = use_context::<ReadSignal<u32>>()
        // 我们知道我们刚刚在父组件中提供了它
        .expect("there to be a `count` signal provided");
    let is_even = move || count() & 1 == 0;

    view! {
        <div class="consumer blue">
            "The number "
            <strong>{count}</strong>
            {move || if is_even() {
                " is"
            } else {
                " is not"
            }}
            " even."
        </div>
    }
}
```

请注意，同样的模式可以应用于更复杂的状态。如果你有多个想要独立更新的字段，你可以通过提供一些信号结构体来做到这一点：

```rust
#[derive(Copy, Clone, Debug)]
struct GlobalState {
    count: RwSignal<i32>,
    name: RwSignal<String>
}

impl GlobalState {
    pub fn new() -> Self {
        Self {
            count: create_rw_signal(0),
            name: create_rw_signal("Bob".to_string())
        }
    }
}

#[component]
fn App() -> impl IntoView {
    provide_context(GlobalState::new());

    // 等等。
}
```

## 选项 #3：创建全局状态结构体和切片

你可能会觉得像这样将结构体的每个字段都包装在一个单独的信号中很麻烦。在某些情况下，创建一个具有非响应式字段的普通结构体，然后将其包装在一个信号中会很有用。

```rust
#[derive(Copy, Clone, Debug, Default)]
struct GlobalState {
    count: i32,
    name: String
}

#[component]
fn App() -> impl IntoView {
    provide_context(create_rw_signal(GlobalState::default()));

    // 等等。
}
```

但有一个问题：因为我们的整个状态都包装在一个信号中，所以更新一个字段的值会导致 UI 中仅依赖于另一个字段的部分发生响应式更新。

```rust
let state = expect_context::<RwSignal<GlobalState>>();
view! {
    <button on:click=move |_| state.update(|state| state.count += 1)>"+1"</button>
    <p>{move || state.with(|state| state.name.clone())}</p>
}
```

在这个例子中，点击按钮会导致 `<p>` 内部的文本被更新，再次克隆 `state.name`！因为信号是响应式的原子单元，所以更新信号的任何字段都会触发对其依赖的所有内容的更新。

有一种更好的方法。你可以使用 [`create_memo`](https://docs.rs/leptos/latest/leptos/fn.create_memo.html) 或 [`create_slice`](https://docs.rs/leptos/latest/leptos/fn.create_slice.html)（它使用 `create_memo` 但也提供了一个设置器）来获取细粒度的响应式切片。“记忆”一个值意味着创建一个新的响应式值，该值只有在它发生变化时才会更新。“记忆一个切片”意味着创建一个新的响应式值，该值只有在状态结构体的某个字段更新时才会更新。

在这里，我们不是直接从状态信号中读取，而是通过 `create_slice` 创建该状态的“切片”，并进行细粒度更新。每个切片信号仅在它访问的较大结构体的特定部分更新时才会更新。这意味着你可以创建一个单一的根信号，然后在不同的组件中获取它的独立的、细粒度的切片，每个切片都可以更新而不会通知其他切片更改。

```rust
/// 更新全局状态中计数的组件。
#[component]
fn GlobalStateCounter() -> impl IntoView {
    let state = expect_context::<RwSignal<GlobalState>>();

    // `create_slice` 让我们可以创建数据的一个“镜头”
    let (count, set_count) = create_slice(

        // 我们从 `state` 中获取一个切片
        state,
        // 我们的 getter 返回数据的“切片”
        |state| state.count,
        // 我们的 setter 描述了如何根据新值修改该切片
        |state, n| state.count = n,
    );

    view! {
        <div class="consumer blue">
            <button
                on:click=move |_| {
                    set_count(count() + 1);
                }
            >
                "Increment Global Count"
            </button>
            <br/>
            <span>"Count is: " {count}</span>
        </div>
    }
}
```

点击此按钮只会更新 `state.count`，因此如果我们在其他地方创建另一个仅获取 `state.name` 的切片，则点击该按钮不会导致该切片更新。这允许你结合自上而下的数据流和细粒度响应式更新的优点。

> **注意**：这种方法有一些明显的缺点。信号和 memo 都需要拥有它们的值，因此 memo 需要在每次更改时克隆字段的值。在像 Leptos 这样的框架中管理状态的最自然方法始终是提供尽可能局部作用域和细粒度的信号，而不是将所有东西都提升到全局状态。但是，当你*确实*需要某种全局状态时，`create_slice` 可能是一个有用的工具。

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/15-global-state-0-5-8c2ff6?file=%2Fsrc%2Fmain.rs%3A1%2C2)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/15-global-state-0-5-8c2ff6?file=%2Fsrc%2Fmain.rs%3A1%2C2" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源码</summary>

```rust
use leptos::*;

// 到目前为止，我们只处理过组件中的局部状态
// 我们只了解了如何在父子组件之间进行通信
// 但是还有一些更通用的方法来管理全局状态
//
// 全局状态的三种最佳方法是
// 1. 使用路由器通过 URL 驱动全局状态
// 2. 通过上下文传递信号
// 3. 创建一个全局状态结构体，并使用 `create_slice` 创建指向它的镜头
//
// 选项 #1：URL 作为全局状态
// 本教程的接下来几节将介绍路由器。
// 所以现在，我们将只关注选项 #2 和 #3。

// 选项 #2：通过上下文传递信号
//
// 在像 React 这样的虚拟 DOM 库中，使用 Context API 来管理全局
// 状态是一个坏主意：因为整个应用程序都存在于一棵树中，所以改变
// 树中高层提供的某些值会导致整个应用程序重新渲染。
//
// 在像 Leptos 这样的细粒度响应式库中，情况并非如此。
// 你可以在应用程序的根目录中创建一个信号，并使用 provide_context() 将其传递给
// 其他组件。更改它只会导致在实际使用它的特定位置重新渲染，
// 而不会导致整个应用程序重新渲染。
#[component]
fn Option2() -> impl IntoView {
    // 这里我们在根目录中创建一个信号，可以在应用程序的任何地方使用它。
    let (count, set_count) = create_signal(0);
    // 我们将把设置器传递给特定的组件，
    // 但通过上下文将计数本身提供给整个应用程序
    provide_context(count);

    view! {
        <h1>"Option 2: Passing Signals"</h1>
        // SetterButton 允许修改计数
        <SetterButton set_count/>
        // 这些消费者只能读取它
        // 但是如果我们愿意，我们可以通过传递 `set_count` 来赋予他们写权限
        <div style="display: flex">
            <FancyMath/>
            <ListItems/>
        </div>
    }
}

/// 增加我们全局计数器的按钮。
#[component]
fn SetterButton(set_count: WriteSignal<u32>) -> impl IntoView {
    view! {
        <div class="provider red">
            <button on:click=move |_| set_count.update(|count| *count += 1)>
                "Increment Global Count"
            </button>
        </div>
    }
}

/// 使用全局计数进行一些“花哨”数学运算的组件
#[component]
fn FancyMath() -> impl IntoView {
    // 这里我们使用 `use_context` 使用全局计数信号
    let count = use_context::<ReadSignal<u32>>()
        // 我们知道我们刚刚在父组件中提供了它
        .expect("there to be a `count` signal provided");
    let is_even = move || count() & 1 == 0;

    view! {
        <div class="consumer blue">
            "The number "
            <strong>{count}</strong>
            {move || if is_even() {
                " is"
            } else {
                " is not"
            }}
            " even."
        </div>
    }
}

/// 显示从全局计数生成的项目列表的组件。
#[component]
fn ListItems() -> impl IntoView {
    // 再次使用 `use_context` 使用全局计数信号
    let count = use_context::<ReadSignal<u32>>().expect("there to be a `count` signal provided");

    let squares = move || {
        (0..count())
            .map(|n| view! { <li>{n}<sup>"2"</sup> " is " {n * n}</li> })
            .collect::<Vec<_>>()
    };

    view! {
        <div class="consumer green">
            <ul>{squares}</ul>
        </div>
    }
}

// 选项 #3：创建一个全局状态结构体
//
// 你可以使用此方法来构建一个单一的全局数据结构
// 来保存整个应用程序的状态，然后通过
// 使用 `create_slice` 或 `create_memo` 获取细粒度切片来访问它，
// 这样更改状态的一部分不会导致你的
// 应用程序中依赖于状态其他部分的部分发生更改。

#[derive(Default, Clone, Debug)]
struct GlobalState {
    count: u32,
    name: String,
}

#[component]
fn Option3() -> impl IntoView {
    // 我们将提供一个保存整个状态的单一信号
    // 每个组件将负责创建自己的“镜头”来访问它
    let state = create_rw_signal(GlobalState::default());
    provide_context(state);

    view! {
        <h1>"Option 3: Passing Signals"</h1>
        <div class="red consumer" style="width: 100%">
            <h2>"Current Global State"</h2>
            <pre>
                {move || {
                    format!("{:#?}", state.get())
                }}
            </pre>
        </div>
        <div style="display: flex">
            <GlobalStateCounter/>
            <GlobalStateInput/>
        </div>
    }
}

/// 更新全局状态中计数的组件。
#[component]
fn GlobalStateCounter() -> impl IntoView {
    let state = use_context::<RwSignal<GlobalState>>().expect("state to have been provided");

    // `create_slice` 让我们可以创建数据的一个“镜头”
    let (count, set_count) = create_slice(

        // 我们从 `state` 中获取一个切片
        state,
        // 我们的 getter 返回数据的“切片”
        |state| state.count,
        // 我们的 setter 描述了如何根据新值修改该切片
        |state, n| state.count = n,
    );

    view! {
        <div class="consumer blue">
            <button
                on:click=move |_| {
                    set_count(count() + 1);
                }
            >
                "Increment Global Count"
            </button>
            <br/>
            <span>"Count is: " {count}</span>
        </div>
    }
}

/// 更新全局状态中计数的组件。
#[component]
fn GlobalStateInput() -> impl IntoView {
    let state = use_context::<RwSignal<GlobalState>>().expect("state to have been provided");

    // 这个切片完全独立于我们在另一个组件中创建的 `count` 切片
    // 它们都不会导致另一个重新运行
    let (name, set_name) = create_slice(
        // 我们从 `state` 中获取一个切片
        state,
        // 我们的 getter 返回数据的“切片”
        |state| state.name.clone(),
        // 我们的 setter 描述了如何根据新值修改该切片
        |state, n| state.name = n,
    );

    view! {
        <div class="consumer green">
            <input
                type="text"
                prop:value=name
                on:input=move |ev| {
                    set_name(event_target_value(&ev));
                }
            />
            <br/>
            <span>"Name is: " {name}</span>
        </div>
    }
}
// 这个 `main` 函数是应用程序的入口点
// 它只是将我们的组件挂载到 <body> 上
// 因为我们将其定义为 `fn App`，所以我们现在可以在
// 模板中将其用作 <App/>
fn main() {
    leptos::mount_to_body(|| view! { <Option2/><Option3/> })
}
```

</details>
</preview>
