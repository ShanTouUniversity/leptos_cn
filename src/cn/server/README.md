# 与服务器协同工作

上一节描述了服务器端渲染的过程，使用服务器生成一个 HTML 版本的页面，该页面将在浏览器中变得有交互性。到目前为止，一切都“同构（isomorphic）”；换句话说，你的应用程序在客户端和服务器上具有“相同的（_iso_）形状（_morphe_）”。

> 译注： 引号中的内容比较难以说明，参看原文： `So far, everything has been “isomorphic”; in other words, your app has had the “same (iso) shape (morphe)” on the client and the server.`

但服务器的功能远不止渲染 HTML！事实上，服务器可以做很多你的浏览器_不能_做的事情，比如从 SQL 数据库读取和写入数据。

如果你习惯于构建 JavaScript 前端应用程序，你可能习惯于调用某种 REST API 来完成这种服务器端工作。如果你习惯于使用 PHP 或 Python 或 Ruby（或 Java 或 C# 或...）构建网站，那么这种服务器端工作是你的主要工作，而客户端交互性往往是事后才想到的。

使用 Leptos，你两者都可以做到：不仅使用相同的语言，不仅共享相同的类型，甚至在同一个文件中！

本节将讨论如何构建应用程序中独特的服务器端部分。
