# 插曲：样式

任何创建网站或应用程序的人很快都会遇到样式问题。对于小型应用程序，单个 CSS 文件可能足以设置用户界面的样式。但随着应用程序的增长，许多开发人员发现纯 CSS 越来越难以管理。

一些前端框架（如 Angular、Vue 和 Svelte）提供了将 CSS 范围限定到特定组件的内置方法，从而更容易管理整个应用程序的样式，而不会让用于修改一个小组件的样式产生全局影响。其他框架（如 React 或 Solid）不提供内置的 CSS 作用域，而是依赖生态系统中的库来为它们完成这项工作。Leptos 属于后者：框架本身对 CSS 没有任何看法，但提供了一些工具和原语，允许其他人构建样式库。

以下是一些为你的 Leptos 应用程序设置样式的不同方法，而不是纯 CSS。

## TailwindCSS：实用优先的 CSS

[TailwindCSS](https://tailwindcss.com/) 是一个流行的实用优先的 CSS 库。它允许你通过使用内联实用程序类来设置应用程序的样式，并使用自定义 CLI 工具扫描你的文件中的 Tailwind 类名并将必要的 CSS 打包在一起。

这允许你编写如下组件：

```rust
#[component]
fn Home() -> impl IntoView {
    let (count, set_count) = create_signal(0);

    view! {
        <main class="my-0 mx-auto max-w-3xl text-center">
            <h2 class="p-6 text-4xl">"Welcome to Leptos with Tailwind"</h2>
            <p class="px-10 pb-10 text-left">"Tailwind will scan your Rust files for Tailwind class names and compile them into a CSS file."</p>
            <button
                class="bg-sky-600 hover:bg-sky-700 px-5 py-3 text-white rounded-lg"
                on:click=move |_| set_count.update(|count| *count += 1)
            >
                {move || if count() == 0 {
                    "Click me!".to_string()
                } else {
                    count().to_string()
                }}
            </button>
        </main>
    }
}
```

最初设置 Tailwind 集成可能有点复杂，但你可以查看我们关于如何在 [客户端渲染的 `trunk` 应用程序](https://github.com/leptos-rs/leptos/tree/leptos_0.6/examples/tailwind_csr) 或 [服务器端渲染的 `cargo-leptos` 应用程序](https://github.com/leptos-rs/leptos/tree/leptos_0.6/examples/tailwind_actix) 中使用 Tailwind 的两个示例。`cargo-leptos` 还有一些 [内置的 Tailwind 支持](https://github.com/leptos-rs/cargo-leptos#site-parameters)，你可以将其用作 Tailwind CLI 的替代方案。

## Stylers：编译时 CSS 提取

[Stylers](https://github.com/abishekatp/stylers) 是一个编译时作用域 CSS 库，允许你在组件的主体中声明作用域 CSS。Stylers 将在编译时将此 CSS 提取到 CSS 文件中，然后你可以将其导入到你的应用程序中，这意味着它不会增加应用程序的 WASM 二进制文件大小。

这允许你编写如下组件：

```rust
use stylers::style;

#[component]
pub fn App() -> impl IntoView {
    let styler_class = style! { "App",
        ##two{
            color: blue;
        }
        div.one{
            color: red;
            content: raw_str(r#"\hello"#);
            font: "1.3em/1.2" Arial, Helvetica, sans-serif;
        }
        div {
            border: 1px solid black;
            margin: 25px 50px 75px 100px;
            background-color: lightblue;
        }
        h2 {
            color: purple;
        }
        @media only screen and (max-width: 1000px) {
            h3 {
                background-color: lightblue;
                color: blue
            }
        }
    };

    view! { class = styler_class,
        <div class="one">
            <h1 id="two">"Hello"</h1>
            <h2>"World"</h2>
            <h2>"and"</h2>
            <h3>"friends!"</h3>
        </div>
    }
}
```

## Stylance：用 CSS 文件编写的作用域 CSS

Stylers 允许你在 Rust 代码中内联编写 CSS，并在编译时提取它并限定其作用域。[Stylance](https://github.com/basro/stylance-rs) 允许你在组件旁边用 CSS 文件编写 CSS，将这些文件导入到你的组件中，并将 CSS 类的作用域限定到你的组件。

这与 `trunk` 和 `cargo-leptos` 的实时重新加载功能配合良好，因为编辑的 CSS 文件可以在浏览器中立即更新。

```rust
import_style!(style, "app.module.scss");

#[component]
fn HomePage() -> impl IntoView {
    view! {
        <div class=style::jumbotron/>
    }
}
```

你可以直接编辑 CSS，而无需进行 Rust 重新编译。

```css
.jumbotron {
  background: blue;
}
```

## Styled：运行时 CSS 作用域

[Styled](https://github.com/eboody/styled) 是一个运行时作用域 CSS 库，它与 Leptos 集成良好。它允许你在组件函数的主体中声明作用域 CSS，然后在运行时应用这些样式。

```rust
use styled::style;

#[component]
pub fn MyComponent() -> impl IntoView {
    let styles = style!(
      div {
        background-color: red;
        color: white;
      }
    );

    styled::view! { styles,
        <div>"This text should be red with white text."</div>
    }
}
```

## 欢迎贡献

Leptos 对你如何设置网站或应用程序的样式没有任何看法，但我们很乐意为你想创建的任何工具提供支持，以使这项工作变得更容易。如果你正在研究一种你想添加到此列表中的 CSS 或样式方法，请告诉我们！
