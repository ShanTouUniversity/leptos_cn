# 使用 `async`

到目前为止，我们只处理过同步用户界面：你提供一些输入，应用程序立即处理它并更新界面。这很好，但只是 Web 应用程序功能的一小部分。特别是，大多数 Web 应用程序必须处理某种异步数据加载，通常是从 API 加载某些内容。

众所周知，异步数据很难与代码的同步部分集成。Leptos 提供了一个跨平台的 [`spawn_local`](https://docs.rs/leptos/latest/leptos/fn.spawn_local.html) 函数，可以轻松运行 `Future`，但除此之外还有更多内容。

在本章中，我们将看到 Leptos 如何帮助你简化此过程。
