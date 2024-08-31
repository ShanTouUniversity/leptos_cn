# 投影子级

在构建组件时，你可能偶尔会发现自己想要通过多层组件“投影”子级。

## 问题

考虑以下内容：

```rust
pub fn LoggedIn<F, IV>(fallback: F, children: ChildrenFn) -> impl IntoView
where
    F: Fn() -> IV + 'static,
    IV: IntoView,
{
    view! {
        <Suspense
            fallback=|| ()
        >
            <Show
				// 通过从资源中读取来检查用户是否已验证
                when=move || todo!()
                fallback=fallback
            >
				{children()}
			</Show>
        </Suspense>
    }
}
```

这很简单：当用户登录时，我们想显示 `children`。如果用户未登录，我们想显示 `fallback`。在我们等待结果的时候，我们只渲染 `()`，也就是什么也不渲染。

换句话说，我们想将 `<LoggedIn/>` 的子级*通过* `<Suspense/>` 组件传递，以成为 `<Show/>` 的子级。这就是我所说的“投影”。

这无法编译。

```
error[E0507]: cannot move out of `fallback`, a captured variable in an `Fn` closure
error[E0507]: cannot move out of `children`, a captured variable in an `Fn` closure
```

这里的问题是 `<Suspense/>` 和 `<Show/>` 都需要能够多次构造它们的 `children`。第一次构造 `<Suspense/>` 的子级时，它会获取 `fallback` 和 `children` 的所有权，将它们移动到 `<Show/>` 的调用中，但随后它们将不可用于未来的 `<Suspense/>` 子级构造。

## 细节

> 可以随意跳到解决方案部分。

如果你想真正理解这里的问题，查看扩展后的 `view` 宏可能会有所帮助。这是一个清理后的版本：

```rust
Suspense(
    ::leptos::component_props_builder(&Suspense)
        .fallback(|| ())
        .children({
            // fallback 和 children 被移动到这个闭包中
            Box::new(move || {
                {
                    // fallback 和 children 在这里被捕获
                    leptos::Fragment::lazy(|| {
                        vec![
                            (Show(
                                ::leptos::component_props_builder(&Show)
                                    .when(|| true)
									// 但是 fallback 在这里被移动到 Show 中
                                    .fallback(fallback)
									// 并且 children 在这里被移动到 Show 中
                                    .children(children)
                                    .build(),
                            )
                            .into_view()),
                        ]
                    })
                }
            })
        })
        .build(),
)
```

所有组件都拥有自己的 props；所以在这种情况下，无法调用 `<Show/>`，因为它只捕获了对 `fallback` 和 `children` 的引用。

## 解决方案

然而，`<Suspense/>` 和 `<Show/>` 都接受 `ChildrenFn`，即它们的 `children` 应该实现 `Fn` 类型，这样它们就可以被多次调用，并且只使用一个不可变的引用。这意味着我们不需要拥有 `children` 或 `fallback`；我们只需要能够传递它们的 `'static` 引用。

我们可以使用 [`store_value`](https://docs.rs/leptos/latest/leptos/fn.store_value.html) 原语来解决这个问题。这实质上是将一个值存储在响应式系统中，将所有权交给框架，换取一个引用，该引用与信号一样是 `Copy` 和 `'static` 的，我们可以通过某些方法访问或修改该引用。

在这种情况下，它真的很简单：

```rust
pub fn LoggedIn<F, IV>(fallback: F, children: ChildrenFn) -> impl IntoView
where
    F: Fn() -> IV + 'static,
    IV: IntoView,
{
    let fallback = store_value(fallback);
    let children = store_value(children);
    view! {
        <Suspense
            fallback=|| ()
        >
            <Show
                when=|| todo!()
                fallback=move || fallback.with_value(|fallback| fallback())
            >
                {children.with_value(|children| children())}
            </Show>
        </Suspense>
    }
}
```

在顶层，我们将 `fallback` 和 `children` 都存储在 `LoggedIn` 拥有的响应式作用域中。现在我们可以简单地将这些引用向下传递到 `<Show/>` 组件中，并在那里调用它们。

## 最后一点说明

请注意，这是可行的，因为 `<Show/>` 和 `<Suspense/>` 只需要对其子级（`.with_value` 可以提供）的不可变引用，而不是所有权。

在其他情况下，你可能需要通过一个接受 `ChildrenFn` 的函数来投影拥有的 props，因此该函数需要多次调用。在这种情况下，你可能会发现 `view` 宏中的 `clone:` 帮助器很有用。

考虑这个例子

```rust
#[component]
pub fn App() -> impl IntoView {
    let name = "Alice".to_string();
    view! {
        <Outer>
            <Inner>
                <Inmost name=name.clone()/>
            </Inner>
        </Outer>
    }
}

#[component]
pub fn Outer(children: ChildrenFn) -> impl IntoView {
    children()
}

#[component]
pub fn Inner(children: ChildrenFn) -> impl IntoView {
    children()
}

#[component]
pub fn Inmost(name: String) -> impl IntoView {
    view! {
        <p>{name}</p>
    }
}
```

即使使用 `name=name.clone()`，也会出现以下错误

```
cannot move out of `name`, a captured variable in an `Fn` closure
```

它被捕获到需要多次运行的多层子级中，并且没有明显的方法将其克隆*到*子级中。

在这种情况下，`clone:` 语法就派上用场了。调用 `clone:name` 会在将 `name` 移动到 `<Inner/>` 的子级之前克隆 `name`，这解决了我们的所有权问题。

```rust
view! {
	<Outer>
		<Inner clone:name>
			<Inmost name=name.clone()/>
		</Inner>
	</Outer>
}
```

由于 `view` 宏的不透明性，这些问题可能有点难以理解或调试。但总的来说，它们总是可以解决的。
