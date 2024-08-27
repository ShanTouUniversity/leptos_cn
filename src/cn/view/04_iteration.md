# 迭代

无论你是列出待办事项、显示表格还是显示产品图片，迭代项目列表都是 Web 应用程序中的常见任务。协调不断变化的项目集之间的差异也是框架需要妥善处理的最棘手的任务之一。

Leptos 支持两种不同的迭代项目模式：

1. 对于静态视图：`Vec<_>`
2. 对于动态列表：`<For/>`

## 使用 `Vec<_>` 的静态视图

有时你需要重复显示一个项目，但你从中绘制的列表并不经常更改。在这种情况下，重要的是要知道你可以在视图中插入任何 `Vec<IV> where IV: IntoView`。换句话说，如果你可以渲染 `T`，你就可以渲染 `Vec<T>`。

```rust
let values = vec![0, 1, 2];
view! {
    // 这将只渲染 "012"
    <p>{values.clone()}</p>
    // 或者我们可以将它们包装在 <li> 中
    <ul>
        {values.into_iter()
            .map(|n| view! { <li>{n}</li>})
            .collect::<Vec<_>>()}
    </ul>
}
```

Leptos 还提供了一个 `.collect_view()` 辅助函数，允许你将任何 `T: IntoView` 的迭代器收集到 `Vec<View>` 中。

```rust
let values = vec![0, 1, 2];
view! {
    // 这将只渲染 "012"
    <p>{values.clone()}</p>
    // 或者我们可以将它们包装在 <li> 中
    <ul>
        {values.into_iter()
            .map(|n| view! { <li>{n}</li>})
            .collect_view()}
    </ul>
}
```

_列表_ 是静态的并不意味着界面需要是静态的。你可以将动态项目渲染为静态列表的一部分。

```rust
// 创建一个包含 5 个信号的列表
let length = 5;
let counters = (1..=length).map(|idx| create_signal(idx));

// 每个项目管理一个响应式视图
// 但列表本身永远不会改变
let counter_buttons = counters
    .map(|(count, set_count)| {
        view! {
            <li>
                <button
                    on:click=move |_| set_count.update(|n| *n += 1)
                >
                    {count}
                </button>
            </li>
        }
    })
    .collect_view();

view! {
    <ul>{counter_buttons}</ul>
}
```

你 _也可以_ 响应式地渲染 `Fn() -> Vec<_>`。但请注意，每次它发生变化时，这都会重新渲染列表中的每个项目。这是非常低效的！幸运的是，有一种更好的方法。

## 使用 `<For/>` 组件进行动态渲染

[`<For/>`](https://docs.rs/leptos/latest/leptos/fn.For.html) 组件是一个带键的动态列表。它接受三个 props：

- `each`：一个函数（例如信号），返回要迭代的项目 `T`
- `key`：一个键函数，接受 `&T` 并返回一个稳定的、唯一的键或 ID
- `children`：将每个 `T` 渲染成一个视图

`key` 是，嗯，关键。你可以在列表中添加、删除和移动项目。只要每个项目的键随着时间的推移保持稳定，框架就不需要重新渲染任何项目，除非它们是新增的，并且它可以非常有效地添加、删除和移动项目，因为它们会发生变化。这允许在列表更改时对其进行极其有效的更新，而只需最少的额外工作。

创建一个好的 `key` 可能有点棘手。你通常 _不_ 想为此目的使用索引，因为它不稳定——如果你删除或移动项目，它们的索引会发生变化。

但是，在生成每一行时为其生成一个唯一的 ID，并将其用作键函数的 ID，这是一个好主意。

查看下面的 `<DynamicList/>` 组件以获取示例。

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/4-iteration-0-5-pwdn2y?file=%2Fsrc%2Fmain.rs%3A1%2C1)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/4-iteration-0-5-pwdn2y?file=%2Fsrc%2Fmain.rs%3A1%2C1" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源码</summary>

```rust
use leptos::*;

// 迭代是大多数应用程序中非常常见的任务。
// 那么如何获取数据列表并在 DOM 中渲染它呢？
// 此示例将向你展示两种方法：
// 1) 对于大多数静态列表，使用 Rust 迭代器
// 2) 对于增长、收缩或移动项目的列表，使用 <For/>

#[component]
fn App() -> impl IntoView {
    view! {
        <h1>"Iteration"</h1>
        <h2>"Static List"</h2>
        <p>"Use this pattern if the list itself is static."</p>
        <StaticList length=5/>
        <h2>"Dynamic List"</h2>
        <p>"Use this pattern if the rows in your list will change."</p>
        <DynamicList initial_length=5/>
    }
}

/// 计数器列表，无法
/// 添加或删除任何计数器。
#[component]
fn StaticList(
    /// 此列表中要包含的计数器数量。
    length: usize,
) -> impl IntoView {
    // 创建以递增数字开头的计数器信号
    let counters = (1..=length).map(|idx| create_signal(idx));

    // 当你有一个不变的列表时，你可以
    // 使用普通的 Rust 迭代器来操作它
    // 并将其收集到 Vec<_> 中以将其插入 DOM
    let counter_buttons = counters
        .map(|(count, set_count)| {
            view! {
                <li>
                    <button
                        on:click=move |_| set_count.update(|n| *n += 1)
                    >
                        {count}
                    </button>
                </li>
            }
        })
        .collect::<Vec<_>>();

    // 请注意，如果 `counter_buttons` 是一个响应式列表
    // 并且它的值发生了变化，这将非常低效：
    // 每次列表更改时，它都会重新渲染每一行。
    view! {
        <ul>{counter_buttons}</ul>
    }
}

/// 允许你添加或
/// 删除计数器的计数器列表。
#[component]
fn DynamicList(
    /// 开始时的计数器数量。
    initial_length: usize,
) -> impl IntoView {
    // 此动态列表将使用 <For/> 组件。
    // <For/> 是一个带键的列表。这意味着每一行
    // 都有一个定义的键。如果键没有改变，则该行
    // 不会重新渲染。当列表发生变化时，只有
    // 对 DOM 进行最少数量的更改。

    // `next_counter_id` 将让我们生成唯一的 ID
    // 我们通过在每次
    // 创建计数器时简单地将 ID 加一来做到这一点
    let mut next_counter_id = initial_length;

    // 我们生成一个初始列表，如 <StaticList/> 中所示
    // 但这次我们将 ID 与信号一起包含在内
    let initial_counters = (0..initial_length)
        .map(|id| (id, create_signal(id + 1)))
        .collect::<Vec<_>>();

    // 现在我们将该初始列表存储在一个信号中
    // 这样，我们将能够随着时间的推移修改列表，
    // 添加和删除计数器，它将以响应式的方式发生变化
    let (counters, set_counters) = create_signal(initial_counters);

    let add_counter = move |_| {
        // 为新的计数器创建一个信号
        let sig = create_signal(next_counter_id + 1);
        // 将此计数器添加到计数器列表中
        set_counters.update(move |counters| {
            // 因为 `.update()` 为我们提供了 `&mut T`
            // 我们可以使用普通的 Vec 方法，如 `push`
            counters.push((next_counter_id, sig))
        });
        // 增加 ID，使其始终唯一
        next_counter_id += 1;
    };

    view! {
        <div>
            <button on:click=add_counter>
                "Add Counter"
            </button>
            <ul>
                // <For/> 组件在这里是中心
                // 这允许高效、关键的列表渲染
                <For
                    // `each` 接受任何返回迭代器的函数
                    // 这通常应该是信号或派生信号
                    // 如果它不是响应式的，只需渲染 Vec<_> 而不是 <For/>
                    each=counters
                    // 键对于每一行应该是唯一的和稳定的
                    // 使用索引通常是一个坏主意，除非你的列表
                    // 只能增长，因为在列表中移动项目
                    // 意味着它们的索引会发生变化，并且它们都会重新渲染
                    key=|counter| counter.0
                    // `children` 接收来自你的 `each` 迭代器的每个项目
                    // 并返回一个视图
                    children=move |(id, (count, set_count))| {
                        view! {
                            <li>
                                <button
                                    on:click=move |_| set_count.update(|n| *n += 1)
                                >
                                    {count}
                                </button>
                                <button
                                    on:click=move |_| {
                                        set_counters.update(|counters| {
                                            counters.retain(|(counter_id, _)| counter_id != &id)
                                        });
                                    }
                                >
                                    "Remove"
                                </button>
                            </li>
                        }
                    }
                />
            </ul>
        </div>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
