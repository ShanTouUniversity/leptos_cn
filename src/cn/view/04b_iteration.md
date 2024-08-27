# 使用 `<For/>` 迭代更复杂的数据

本章将更深入地介绍嵌套数据结构的迭代。它属于迭代的另一章，但如果你现在想坚持使用更简单的主题，请随时跳过它，稍后再回来。

## 问题

我刚才说过，除非键发生了变化，否则框架不会重新渲染任何行中的任何项目。这乍一看可能很有道理，但它很容易让你绊倒。

让我们考虑一个例子，其中我们行中的每一项都是某种数据结构。例如，假设这些项目来自某个 JSON 键和值数组：

```rust
#[derive(Debug, Clone)]
struct DatabaseEntry {
    key: String,
    value: i32,
}
```

让我们定义一个简单的组件，它将迭代这些行并显示每一行：

```rust
#[component]
pub fn App() -> impl IntoView {
	// 从一组三行开始
    let (data, set_data) = create_signal(vec![
        DatabaseEntry {
            key: "foo".to_string(),
            value: 10,
        },
        DatabaseEntry {
            key: "bar".to_string(),
            value: 20,
        },
        DatabaseEntry {
            key: "baz".to_string(),
            value: 15,
        },
    ]);
    view! {
		// 当我们点击时，更新每一行，
		// 将其值加倍
        <button on:click=move |_| {
            set_data.update(|data| {
                for row in data {
                    row.value *= 2;
                }
            });
			// 记录信号的新值
            logging::log!("{:?}", data.get());
        }>
            "Update Values"
        </button>
		// 迭代这些行并显示每个值
        <For
            each=data
            key=|state| state.key.clone()
            let:child
        >
            <p>{child.value}</p>
        </For>
    }
}
```

> 请注意这里的 `let:child` 语法。在上一章中，我们介绍了带有 `children` prop 的 `<For/>`。我们实际上可以直接在 `<For/>` 组件的子组件中创建这个值，而无需跳出 `view` 宏：上面的 `let:child` 与 `<p>{child.value}</p>` 的组合相当于
>
> ```rust
> children=|child| view! { <p>{child.value}</p> }
> ```

当你点击“更新值”按钮时......什么也没有发生。或者更确切地说：信号已更新，新值已记录，但每行的 `{child.value}` 不会更新。

让我们看看：这是因为我们忘记添加闭包以使其具有响应式吗？让我们试试 `{move || child.value}`。

...不。仍然没有。

问题在于：正如我所说，只有当键发生变化时，才会重新渲染每一行。我们已经更新了每一行的值，但没有更新任何行的键，所以没有任何内容重新渲染。如果你查看 `child.value` 的类型，它是一个普通的 `i32`，而不是一个响应式的 `ReadSignal<i32>` 或其他什么。这意味着即使我们用一个闭包将其包裹起来，此行中的值也永远不会更新。

我们有三种可能的解决方案：

1. 更改 `key`，使其在数据结构发生更改时始终更新
2. 更改 `value`，使其具有响应式
3. 获取数据结构的响应式切片，而不是直接使用每一行

## 选项 1：更改键

只有当键发生变化时，才会重新渲染每一行。我们上面的行没有重新渲染，因为键没有改变。那么：为什么不强制更改键呢？

```rust
<For
	each=data
	key=|state| (state.key.clone(), state.value)
	let:child
>
	<p>{child.value}</p>
</For>
```

现在我们将键和值都包含在 `key` 中。这意味着只要行的值发生变化，`<For/>` 就会将其视为一个全新的行，并替换前一行。

### 优点

这很容易。我们可以通过在 `DatabaseEntry` 上派生 `PartialEq`、`Eq` 和 `Hash` 来使其更容易，在这种情况下，我们可以只使用 `key=|state| state.clone()`。

### 缺点

**这是三种选择中效率最低的。** 每当行的值发生变化时，它都会丢弃之前的 `<p>` 元素，并用一个全新的元素替换它。换句话说，它不是对文本节点进行细粒度的更新，而是在每次更改时都会重新渲染整个行，这与行的 UI 的复杂程度成正比。

你还会注意到，我们最终会克隆整个数据结构，以便 `<For/>` 可以保存键的副本。对于更复杂的结构，这很快就会变成一个坏主意！

## 选项 2：嵌套信号

如果我们确实希望该值具有细粒度的响应式，一种选择是将每行的 `value` 包装在一个信号中。

```rust
#[derive(Debug, Clone)]
struct DatabaseEntry {
    key: String,
    value: RwSignal<i32>,
}
```

`RwSignal<_>` 是一个“读写信号”，它将 getter 和 setter 合并到一个对象中。我在这里使用它是因为它比单独的 getter 和 setter 更容易存储在结构体中。

```rust
#[component]
pub fn App() -> impl IntoView {
	// 从一组三行开始
    let (data, set_data) = create_signal(vec![
        DatabaseEntry {
            key: "foo".to_string(),
            value: create_rw_signal(10),
        },
        DatabaseEntry {
            key: "bar".to_string(),
            value: create_rw_signal(20),
        },
        DatabaseEntry {
            key: "baz".to_string(),
            value: create_rw_signal(15),
        },
    ]);
    view! {
		// 当我们点击时，更新每一行，
		// 将其值加倍
        <button on:click=move |_| {
            data.with(|data| {
                for row in data {
                    row.value.update(|value| *value *= 2);
                }
            });
			// 记录信号的新值
            logging::log!("{:?}", data.get());
        }>
            "Update Values"
        </button>
		// 迭代这些行并显示每个值
        <For
            each=data
            key=|state| state.key.clone()
            let:child
        >
            <p>{child.value}</p>
        </For>
    }
}
```

这个版本有效！如果你在浏览器的 DOM 检查器中查看，你会看到与之前的版本不同，在这个版本中只有单个文本节点被更新。将信号直接传递到 `{child.value}` 中是有效的，因为如果你将信号传递到视图中，信号确实会保持它们的响应性。

请注意，我将 `set_data.update()` 更改为 `data.with()`。`.with()` 是访问信号值的非克隆方式。在这种情况下，我们只更新内部值，而不更新值列表：因为信号维护它们自己的状态，我们实际上根本不需要更新 `data` 信号，所以这里使用不可变的 `.with()` 就可以了。

> 事实上，这个版本并没有更新 `data`，所以 `<For/>` 本质上是一个静态列表，如上一章所示，这可能只是一个普通的迭代器。但是，如果我们将来想要添加或删除行，`<For/>` 就会很有用。

### 优点

这是最有效的选择，并且与框架的其余心智模型直接吻合：随时间变化的值被包装在信号中，以便界面可以对它们做出响应。

### 缺点

如果你从 API 或你无法控制的其他数据源接收数据，并且你不想创建不同的结构体来将每个字段包装在信号中，那么嵌套的响应式可能会很麻烦。

## 选项 3：记忆切片

Leptos 提供了一个名为 [`create_memo`](https://docs.rs/leptos/latest/leptos/fn.create_memo.html) 的原语，它创建一个派生计算，仅在其值发生变化时才触发响应式更新。

这允许你为较大数据结构的子字段创建响应式值，而无需将该结构体的字段包装在信号中。

大多数应用程序可以保持与初始（已损坏）版本相同，但 `<For/>` 将更新为：

```rust
<For
    each=move || data().into_iter().enumerate()
    key=|(_, state)| state.key.clone()
    children=move |(index, _)| {
        let value = create_memo(move |_| {
            data.with(|data| data.get(index).map(|d| d.value).unwrap_or(0))
        });
        view! {
            <p>{value}</p>
        }
    }
/>
```

你会注意到这里有一些区别：

- 我们将 `data` 信号转换为枚举迭代器
- 我们显式使用 `children` prop，以便更容易运行一些非 `view` 代码
- 我们定义了一个  memo `value` 并在视图中使用它。这个 `value` 字段实际上并没有使用传递到每一行的 `child`。相反，它使用索引并返回到原始的 `data` 中以获取值。

现在，每次 `data` 发生变化时，每个 memo 都会重新计算。如果它的值发生了变化，它将更新它的文本节点，而不会重新渲染整个行。

### 优点

我们获得了与信号包装版本相同的细粒度响应性，而无需将数据包装在信号中。

### 缺点

在 `<For/>` 循环内设置这个逐行 memo 比使用嵌套信号要复杂一些。例如，你会注意到我们必须通过使用 `data.get(index)` 来防止 `data[index]` 发生 panic 的可能性，因为这个 memo 可能在行被删除后立即被触发重新运行一次。（这是因为每行的 memo 和整个 `<For/>` 都依赖于相同的 `data` 信号，并且依赖于相同信号的多个响应式值的执行顺序无法得到保证。）

还要注意，虽然 memo 会记住它们的响应式变化，但每次都需要重新运行相同的计算来检查值，因此嵌套的响应式信号对于此处的精确更新仍然更有效。
