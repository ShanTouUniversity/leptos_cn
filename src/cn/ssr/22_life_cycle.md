# 页面加载的过程

在我们深入探讨之前，先进行高级概述可能会有所帮助。从你输入服务器端渲染的 Leptos 应用程序的 URL 到你点击按钮并增加计数器之间到底发生了什么？

我假设你在这里有一些关于互联网如何工作的基本知识，并且不会深入探讨 HTTP 或其他任何内容。相反，我将尝试展示 Leptos API 的不同部分如何映射到该过程的每个部分。

此描述还从你的应用程序正在为两个单独的目标编译的前提开始：

1. 服务器版本，通常在 Actix 或 Axum 上运行，使用 Leptos `ssr` 功能编译
2. 浏览器版本，使用 Leptos `hydrate` 功能编译为 WebAssembly (WASM)

[`cargo-leptos`](https://github.com/leptos-rs/cargo-leptos) 构建工具用于协调为这两个不同目标编译应用程序的过程。

# 页面加载的过程

在我们深入探讨之前，先进行高级概述可能会有所帮助。从你输入服务器端渲染的 Leptos 应用程序的 URL 到你点击按钮并增加计数器之间到底发生了什么？

我假设你在这里有一些关于互联网如何工作的基本知识，并且不会深入探讨 HTTP 或其他任何内容。相反，我将尝试展示 Leptos API 的不同部分如何映射到该过程的每个部分。

此描述还从你的应用程序正在为两个单独的目标编译的前提开始：

1. 服务器版本，通常在 Actix 或 Axum 上运行，使用 Leptos `ssr` 功能编译
2. 浏览器版本，使用 Leptos `hydrate` 功能编译为 WebAssembly (WASM)

[`cargo-leptos`](https://github.com/leptos-rs/cargo-leptos) 构建工具用于协调为这两个不同目标编译应用程序的过程。

## 在服务器上

- 你的浏览器向你的服务器发出对该 URL 的 `GET` 请求。此时，浏览器几乎不知道要渲染的页面。（“浏览器如何知道在哪里请求页面？”这个问题很有趣，但超出了本教程的范围！）
- 服务器收到该请求，并检查它是否能够在该路径上处理 `GET` 请求。这就是 [`leptos_axum`](https://docs.rs/leptos_axum/0.2.5/leptos_axum/trait.LeptosRoutes.html) 和 [`leptos_actix`](https://docs.rs/leptos_actix/0.2.5/leptos_actix/trait.LeptosRoutes.html) 中的 `.leptos_routes()` 方法的用途。当服务器启动时，这些方法会遍历你在 `<Routes/>` 中提供的路由结构，生成你的应用程序可以处理的所有可能路由的列表，并告诉服务器的路由器“对于这些路由中的每一个，如果你收到一个请求... 将其交给 Leptos。”
- 服务器看到此路由可以由 Leptos 处理。因此它会渲染你的根组件（通常称为 `<App/>` 之类的东西），为它提供正在请求的 URL 以及其他一些数据，例如 HTTP 标头和请求元数据。
- 你的应用程序在服务器上运行一次，构建将在该路由上渲染的组件树的 HTML 版本。（下一章将详细介绍 resource 和 `<Suspense/>`。）
- 服务器返回此 HTML 页面，还注入有关如何加载已编译为 WASM 的应用程序版本的信息，以便它可以在浏览器中运行。

> 返回的 HTML 页面本质上是你的应用程序，“脱水”或“冻干”版本：它是 HTML，没有任何你添加的响应式或事件监听器。浏览器将通过添加响应式系统并将事件监听器附加到该服务器渲染的 HTML 来“重新水合”此 HTML 页面。因此，有两个功能标志适用于此过程的两部分：服务器端的 `ssr` 用于“服务器端渲染”，浏览器端的 `hydrate` 用于重新水合过程。

## 在浏览器中

- 浏览器从服务器接收此 HTML 页面。它立即返回到服务器，开始加载运行应用程序的交互式客户端版本所需的 JS 和 WASM。
- 同时，它会渲染 HTML 版本。
- 当 WASM 版本重新加载完成后，它会执行与服务器相同的路由匹配过程。因为 `<Routes/>` 组件在服务器和客户端上是相同的，所以浏览器版本将读取 URL 并渲染与服务器已返回的页面相同的页面。
- 在这个初始的“水合”阶段，你的应用程序的 WASM 版本不会重新创建构成你的应用程序的 DOM 节点。相反，它会遍历现有的 HTML 树，“拾取”现有元素并添加必要的交互性。

> 请注意，这里有一些权衡。在此水合过程完成之前，该页面将*看起来*是交互式的，但实际上不会响应交互。例如，如果你有一个计数器按钮，并在 WASM 加载完成之前点击它，则计数将不会增加，因为必要的事件监听器和响应式尚未添加。我们将在后面的章节中介绍一些构建“优雅降级”的方法。

## 客户端导航

下一步非常重要。想象一下，用户现在点击一个链接来导航到你的应用程序中的另一个页面。

浏览器将_不会_再次往返服务器，重新加载整个页面，就像它在纯 HTML 页面之间导航，或者使用服务器端渲染（例如使用 PHP）但没有客户端部分的应用程序时那样。

相反，你的应用程序的 WASM 版本将在浏览器中加载新页面，而无需从服务器请求另一个页面。本质上，你的应用程序会将自身从服务器加载的“多页应用程序”升级为浏览器渲染的“单页应用程序”。这产生了两种技术的最佳组合：由于服务器端渲染的 HTML，初始加载时间很快，并且由于客户端路由，辅助导航很快。

在以下章节中将描述的一些内容——例如服务器函数、resource 和 `<Suspense/>` 之间的交互——可能看起来过于复杂。你可能会问自己，“如果我的页面在服务器上被渲染为 HTML，为什么我不能在服务器上 `.await` 它？如果我可以直接在服务器函数中调用库 X，为什么我不能在我的组件中调用它？”原因很简单：为了实现从服务器端渲染到客户端渲染的升级，你的应用程序中的所有内容都必须能够在服务器或浏览器上运行。

当然，这不是创建网站或 Web 框架的唯一方法。但它是_最常见_的方法，而且我们碰巧认为这是一种很好的方法，可以为你的用户创造最流畅的体验。
