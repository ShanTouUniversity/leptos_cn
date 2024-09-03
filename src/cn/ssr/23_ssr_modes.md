# 异步渲染和 SSR“模式”

服务器端渲染仅使用同步数据的页面非常简单：你只需遍历组件树，将每个元素渲染为 HTML 字符串。但这是一个很大的警告：它没有回答我们应该如何处理包含异步数据的页面，即在客户端的 `<Suspense/>` 节点下渲染的内容。

当页面加载它需要渲染的异步数据时，我们应该怎么做？我们应该等待所有异步数据加载完成，然后一次渲染所有内容吗？（我们称之为“异步”渲染）我们应该走完全相反的方向，立即将我们拥有的 HTML 发送给客户端，并让客户端加载资源并填充它们吗？（我们称之为“同步”渲染）或者是否有一些中间解决方案可以以某种方式同时胜过它们？（提示：有。）

如果你曾经在线听过流媒体音乐或观看过视频，我确信你知道 HTTP 支持流式传输，允许单个连接一个接一个地发送数据块，而无需等待完整内容加载完成。你可能没有意识到浏览器也非常擅长渲染部分 HTML 页面。综上所述，这意味着你可以通过**流式传输 HTML** 来增强用户的体验：这是 Leptos 开箱即用支持的，根本无需配置。实际上，流式传输 HTML 的方法不止一种：你可以像视频帧一样按顺序流式传输构成你页面的 HTML 块，或者你可以流式传输它们... 好吧，乱序。

让我详细说明我的意思。

Leptos 支持所有主要的渲染包含异步数据的 HTML 的方法：

1. [同步渲染](#synchronous-rendering)
2. [异步渲染](#async-rendering)
3. [顺序流式传输](#in-order-streaming)
4. [乱序流式传输](#out-of-order-streaming)（以及部分阻塞的变体）

## 同步渲染

1. **同步**：为任何 `<Suspense/>` 提供一个包含 `fallback` 的 HTML 外壳。使用 `create_local_resource` 在客户端加载数据，并在加载资源后替换 `fallback`。

- _优点_：应用程序外壳出现得非常快：TTFB（首字节时间）很棒。
- _缺点_
  - 资源加载相对较慢；你需要等待 JS + WASM 加载完成后才能发出请求。
  - 无法在 `<title>` 或其他 `<meta>` 标签中包含来自异步资源的数据，这会损害 SEO 和社交媒体链接预览等内容。

如果你使用的是服务器端渲染，从性能的角度来看，同步模式几乎从来都不是你真正想要的。这是因为它错过了一个重要的优化。如果你在服务器端渲染期间加载异步资源，你实际上可以在服务器上开始加载数据。服务器端渲染实际上可以在客户端首次发出响应时开始加载资源，而不是等待客户端接收 HTML 响应，然后加载其 JS + WASM，*然后*意识到它需要资源并开始加载它们。从这个意义上说，在服务器端渲染期间，异步资源就像一个在服务器上开始加载并在客户端解析的 `Future`。只要资源实际上是可序列化的，这将始终导致更快的总加载时间。

> 这就是为什么 [`create_resource`](https://docs.rs/leptos/latest/leptos/fn.create_resource.html) 默认要求资源数据可序列化，以及为什么你需要为任何不可序列化的异步数据显式使用 [`create_local_resource`](https://docs.rs/leptos/latest/leptos/fn.create_local_resource.html)，因此这些数据只能在浏览器本身中加载。当你能够创建可序列化资源时创建本地资源始终是一种反优化。

## 异步渲染

<video controls>
	<source src="https://github.com/leptos-rs/leptos/blob/main/docs/video/async.mov?raw=true" type="video/mp4">
</video>

2. **`async`**：在服务器上加载所有资源。等待所有数据加载完成，然后一次性渲染 HTML。

- _优点_：更好地处理元标签（因为你甚至在渲染 `<head>` 之前就知道异步数据）。由于异步资源开始在服务器上加载，因此完成加载速度比**同步**快。
- _缺点_：加载时间/TTFB 较慢：你需要等待所有异步资源加载完成后才能在客户端上显示任何内容。在所有内容加载完成之前，页面完全空白。

## 顺序流式传输

<video controls>
	<source src="https://github.com/leptos-rs/leptos/blob/main/docs/video/in-order.mov?raw=true" type="video/mp4">
</video>

3. **顺序流式传输**：遍历组件树，渲染 HTML，直到遇到 `<Suspense/>`。将到目前为止你得到的所有 HTML 作为流中的一个块发送，等待 `<Suspense/>` 下访问的所有资源加载完成，然后将其渲染为 HTML 并继续遍历，直到遇到另一个 `<Suspense/>` 或页面末尾。

- _优点_：在数据准备好之前，至少显示_一些东西_，而不是空白屏幕。
- _缺点_
  - 加载外壳的速度比同步渲染（或乱序流式传输）慢，因为它需要在每个 `<Suspense/>` 处暂停。
  - 无法显示 `<Suspense/>` 的回退状态。
  - 在整个页面加载完成之前无法开始水合，因此页面的早期部分在暂停的块加载完成之前将不会是交互式的。

## 乱序流式传输

<video controls>
	<source src="https://github.com/leptos-rs/leptos/blob/main/docs/video/out-of-order.mov?raw=true" type="video/mp4">
</video>

4. **乱序流式传输**：与同步渲染类似，为任何 `<Suspense/>` 提供一个包含 `fallback` 的 HTML 外壳。但在**服务器**上加载数据，并在解析时将其流式传输到客户端，并为 `<Suspense/>` 节点流式传输 HTML，该节点将被交换以替换回退内容。

- _优点_：结合了**同步**和**`async`**的优点。
  - 由于它立即发送整个同步外壳，因此初始响应/TTFB 很快
  - 由于资源开始在服务器上加载，因此总时间很快。
  - 能够显示回退加载状态并动态替换它，而不是为未加载的数据显示空白部分。
- _缺点_：需要启用 JavaScript 才能使暂停的片段按正确顺序显示。（这小段 JS 与包含渲染的 `<Suspense/>` 片段的 `<template>` 标签一起流式传输到 `<script>` 标签中，因此它不需要加载任何额外的 JS 文件。）

5. **部分阻塞流式传输**：当你页面上有多个独立的 `<Suspense/>` 组件时，“部分阻塞”流式传输很有用。通过在路由上设置 `ssr=SsrMode::PartiallyBlocked` 并根据视图中的阻塞资源来触发它。如果 `<Suspense/>` 组件之一读取一个或多个“阻塞资源”（见下文），则不会发送回退内容；相反，服务器将等待该 `<Suspense/>` 解析完成，然后在服务器上将回退内容替换为已解析的片段，这意味着它包含在初始 HTML 响应中，即使禁用了 JavaScript 或不支持 JavaScript 也会出现。其他 `<Suspense/>` 以乱序流式传输，类似于 `SsrMode::OutOfOrder` 默认值。

当你页面上有多个 `<Suspense/>`，并且一个比另一个更重要时，这很有用：想想一篇博客文章和评论，或者产品信息和评论。如果只有一个 `<Suspense/>`，或者每个 `<Suspense/>` 都从阻塞资源中读取，则它*没有用处*。在这些情况下，它是 `async` 渲染的一种较慢的形式。

- _优点_：如果在用户的设备上禁用了 JavaScript 或不支持 JavaScript，则有效。
- _缺点_
  - 初始响应时间比乱序慢。
  - 由于服务器上的额外工作，总体响应略有延迟。
  - 不显示回退状态。

## 使用 SSR 模式

因为它提供了性能特征的最佳组合，所以 Leptos 默认使用乱序流式传输。但是选择这些不同的模式真的很简单。你可以通过在你的一个或多个 `<Route/>` 组件上添加 `ssr` 属性来实现，就像在 [`ssr_modes` 示例](https://github.com/leptos-rs/leptos/blob/main/examples/ssr_modes/src/app.rs) 中一样。

```rust
<Routes>
	// 我们将使用乱序流式传输和 `<Suspense/>` 加载主页
	<Route path="" view=HomePage/>

	// 我们将使用异步渲染加载帖子，以便它们可以在加载数据*后*设置
	// 标题和元数据
	<Route
		path="/post/:id"
		view=Post
		ssr=SsrMode::Async
	/>
</Routes>
```

对于包含多个嵌套路由的路径，将使用最严格的模式：即，如果即使单个嵌套路由请求 `async` 渲染，整个初始请求也将以 `async` 方式渲染。`async` 是最严格的要求，其次是顺序，然后是乱序。（如果你仔细想想，这可能是合理的。）

## 阻塞资源

任何晚于 `0.2.5` 的 Leptos 版本（即 git main 和 `0.3.x` 或更高版本）都引入了一个新的资源原语 `create_blocking_resource`。阻塞资源仍然像 Rust 中的任何其他 `async`/`.await` 一样异步加载；它不会阻塞服务器线程或任何东西。相反，在 `<Suspense/>` 下读取阻塞资源会阻止 HTML _流_ 返回任何内容，包括其初始同步外壳，直到该 `<Suspense/>` 解析完成。

现在从性能的角度来看，这并不理想。你的页面的任何同步外壳都不会加载，直到该资源准备就绪。但是，不渲染任何内容意味着你可以执行以下操作，例如在实际 HTML 的 `<head>` 中设置 `<title>` 或 `<meta>` 标签。这听起来很像 `async` 渲染，但有一个很大的区别：如果你有多个 `<Suspense/>` 部分，你可以阻塞其中_一个_，但仍然渲染一个占位符，然后流式传输另一个。

例如，想想一篇博客文章。为了 SEO 和社交分享，我肯定希望我的博客文章的标题和元数据出现在初始 HTML `<head>` 中。但我真的不关心评论是否已经加载；我想尽可能延迟加载它们。

使用阻塞资源，我可以执行以下操作：

```rust
#[component]
pub fn BlogPost() -> impl IntoView {
	let post_data = create_blocking_resource(/* 加载博客文章 */);
	let comments_data = create_resource(/* 加载博客评论 */);
	view! {
		<Suspense fallback=|| ()>
			{move || {
				post_data.with(|data| {
					view! {
						<Title text=data.title/>
						<Meta name="description" content=data.excerpt/>
						<article>
							/* 渲染帖子内容 */
						</article>
					}
				})
			}}
		</Suspense>
		<Suspense fallback=|| "Loading comments...">
			/* 在这里渲染评论数据 */
		</Suspense>
	}
}
```

第一个 `<Suspense/>`，包含博客文章的正文，将阻塞我的 HTML 流，因为它从阻塞资源中读取。元标签和其他等待阻塞资源的头部元素将在发送流之前渲染。

与以下路由定义相结合，该定义使用 `SsrMode::PartiallyBlocked`，阻塞资源将在服务器端完全渲染，从而使禁用 WebAssembly 或 JavaScript 的用户可以访问它。

```rust
<Routes>
	// 我们将使用乱序流式传输和 `<Suspense/>` 加载主页
	<Route path="" view=HomePage/>

	// 我们将使用异步渲染加载帖子，以便它们可以在加载数据*后*设置
	// 标题和元数据
	<Route
		path="/post/:id"
		view=Post
		ssr=SsrMode::PartiallyBlocked
	/>
</Routes>
```

第二个 `<Suspense/>`，包含评论，不会阻塞流。阻塞资源给了我优化页面 SEO 和用户体验所需的功能和粒度。
