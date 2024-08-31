# 元数据

到目前为止，我们渲染的所有内容都在 HTML 文档的 `<body>` 内部。这很有道理。毕竟，你在网页上看到的所有内容都位于 `<body>` 内部。

但是，有很多情况下，你可能希望使用与 UI 相同的响应式原语和组件模式来更新文档 `<head>` 中的内容。

这就是 [`leptos_meta`](https://docs.rs/leptos_meta/latest/leptos_meta/) 包的用武之地。

## 元数据组件

`leptos_meta` 提供了特殊的组件，让你可以将应用程序中任何地方的组件内部的数据注入到 `<head>` 中：

[`<Title/>`](https://docs.rs/leptos_meta/latest/leptos_meta/fn.Title.html) 允许你从任何组件设置文档的标题。它还接受一个 `formatter` 函数，该函数可用于将相同的格式应用于其他页面设置的标题。因此，例如，如果你在 `<App/>` 组件中放入 `<Title formatter=|text| format!("{text} — My Awesome Site")/>`，然后在你的路由上放入 `<Title text="Page 1"/>` 和 `<Title text="Page 2"/>`，你将获得 `Page 1 — My Awesome Site` 和 `Page 2 — My Awesome Site`。

[`<Link/>`](https://docs.rs/leptos_meta/latest/leptos_meta/fn.Link.html) 接受 `<link>` 元素的标准属性。

[`<Stylesheet/>`](https://docs.rs/leptos_meta/latest/leptos_meta/fn.Stylesheet.html) 使用你提供的 `href` 创建一个 `<link rel="stylesheet">`。

[`<Style/>`](https://docs.rs/leptos_meta/latest/leptos_meta/fn.Style.html) 使用你传入的子级（通常是一个字符串）创建一个 `<style>`。你可以使用它在编译时从另一个文件中导入一些自定义 CSS `<Style>{include_str!("my_route.css")}</Style>`。

[`<Meta/>`](https://docs.rs/leptos_meta/latest/leptos_meta/fn.Meta.html) 让你可以使用描述和其他元数据设置 `<meta>` 标签。

## `<Script/>` 和 `<script>`

`leptos_meta` 还提供了一个 [`<Script/>`](https://docs.rs/leptos_meta/latest/leptos_meta/fn.Script.html) 组件，在这里值得停顿一下。我们考虑过的所有其他组件都在 `<head>` 中注入了仅限 `<head>` 的元素。但是 `<script>` 也可以包含在 body 中。

有一种非常简单的方法可以确定你应该使用大写字母 S 的 `<Script/>` 组件还是小写字母 s 的 `<script>` 元素：`<Script/>` 组件将在 `<head>` 中渲染，而 `<script>` 元素将在你的用户界面 `<body>` 中你放置它的任何位置渲染，与其他普通 HTML 元素一起渲染。这些会导致 JavaScript 在不同的时间加载和运行，因此请使用适合你需求的任何一种。

## `<Body/>` 和 `<Html/>`

甚至还有一些元素旨在使语义 HTML 和样式更容易。[`<Html/>`](https://docs.rs/leptos_meta/latest/leptos_meta/fn.Html.html) 让你可以从你的应用程序代码中设置 `<html>` 标签上的 `lang` 和 `dir`。`<Html/>` 和 [`<Body/>`](https://docs.rs/leptos_meta/latest/leptos_meta/fn.Body.html) 都有 `class` props，让你可以设置它们各自的 `class` 属性，这有时是 CSS 框架用于样式设置所需要的。

`<Body/>` 和 `<Html/>` 都有 `attributes` props，可以使用 `attr:` 语法在它们上面设置任意数量的附加属性：

```rust
<Html
	lang="he"
	dir="rtl"
	attr:data-theme="dark"
/>
```

## 元数据和服务器端渲染

现在，其中一些内容在任何场景下都很有用，但其中一些内容对搜索引擎优化（SEO）尤其重要。确保你拥有适当的 `<title>` 和 `<meta>` 标签等内容至关重要。现代搜索引擎爬虫确实会处理客户端渲染，即作为空 `index.html` 发送并完全在 JS/WASM 中渲染的应用程序。但它们更喜欢接收你的应用程序已渲染为实际 HTML 的页面，并在 `<head>` 中包含元数据。

这正是 `leptos_meta` 的用途。事实上，在服务器端渲染期间，它正是这样做的：收集你通过在整个应用程序中使用其组件声明的所有 `<head>` 内容，然后将其注入到实际的 `<head>` 中。

但我跑题了。我们还没有真正讨论服务器端渲染。下一章将讨论与 JavaScript 库的集成。然后我们将结束对客户端的讨论，并转到服务器端渲染。
