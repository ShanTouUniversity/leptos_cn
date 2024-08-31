# 嵌套路由

我们刚刚定义了以下一组路由：

```rust
<Routes>
  <Route path="/" view=Home/>
  <Route path="/users" view=Users/>
  <Route path="/users/:id" view=UserProfile/>
  <Route path="/*any" view=NotFound/>
</Routes>
```

这里有一定的重复：`/users` 和 `/users/:id`。这对于一个小型应用程序来说很好，但你可能已经知道它不能很好地扩展。如果我们可以嵌套这些路由，不是很好吗？

嗯... 你可以！

```rust
<Routes>
  <Route path="/" view=Home/>
  <Route path="/users" view=Users>
    <Route path=":id" view=UserProfile/>
  </Route>
  <Route path="/*any" view=NotFound/>
</Routes>
```

但是等等。我们刚刚微妙地改变了应用程序的功能。

下一节是本指南整个路由部分中最重要的部分之一。请仔细阅读，如果你有任何不明白的地方，请随时提问。

# 嵌套路由作为布局

嵌套路由是布局的一种形式，而不是路由定义的一种方法。

换句话说：定义嵌套路由的主要目的不是为了在输入路由定义中的路径时避免重复输入。它实际上是为了告诉路由器同时在页面上并排显示多个 `<Route/>`。

让我们回顾一下我们的实际例子。

```rust
<Routes>
  <Route path="/users" view=Users/>
  <Route path="/users/:id" view=UserProfile/>
</Routes>
```

这意味着：

- 如果我访问 `/users`，我会得到 `<Users/>` 组件。
- 如果我访问 `/users/3`，我会得到 `<UserProfile/>` 组件（参数 `id` 设置为 `3`；稍后会详细介绍）

假设我改为使用嵌套路由：

```rust
<Routes>
  <Route path="/users" view=Users>
    <Route path=":id" view=UserProfile/>
  </Route>
</Routes>
```

这意味着：

- 如果我访问 `/users/3`，该路径会匹配两个 `<Route/>`：`<Users/>` 和 `<UserProfile/>`。
- 如果我访问 `/users`，则该路径不匹配。

我实际上需要添加一个回退路由

```rust
<Routes>
  <Route path="/users" view=Users>
    <Route path=":id" view=UserProfile/>
    <Route path="" view=NoUser/>
  </Route>
</Routes>
```

现在：

- 如果我访问 `/users/3`，该路径会匹配 `<Users/>` 和 `<UserProfile/>`。
- 如果我访问 `/users`，该路径会匹配 `<Users/>` 和 `<NoUser/>`。

换句话说，当我使用嵌套路由时，每个 **路径** 可以匹配多个 **路由**：每个 URL 可以同时在同一页面上渲染由多个 `<Route/>` 组件提供的视图。

这可能与直觉相悖，但它非常强大，原因希望你在几分钟内就能看到。

## 为什么使用嵌套路由？

为什么要这么麻烦？

大多数 Web 应用程序都包含与布局不同部分相对应的导航级别。例如，在一个电子邮件应用程序中，你可能会有一个像 `/contacts/greg` 这样的 URL，它在屏幕左侧显示联系人列表，在屏幕右侧显示 Greg 的联系方式。联系人列表和联系方式应该始终同时出现在屏幕上。如果没有选择联系人，你可能希望显示一些说明性文字。

你可以使用嵌套路由轻松定义这一点

```rust
<Routes>
  <Route path="/contacts" view=ContactList>
    <Route path=":id" view=ContactInfo/>
    <Route path="" view=|| view! {
      <p>"Select a contact to view more info."</p>
    }/>
  </Route>
</Routes>
```

你甚至可以走得更深。假设你想为每个联系人的地址、电子邮件/电话以及你与他们的对话设置选项卡。你可以在 `:id` 内部添加*另一组*嵌套路由：

```rust
<Routes>
  <Route path="/contacts" view=ContactList>
    <Route path=":id" view=ContactInfo>
      <Route path="" view=EmailAndPhone/>
      <Route path="address" view=Address/>
      <Route path="messages" view=Messages/>
    </Route>
    <Route path="" view=|| view! {
      <p>"Select a contact to view more info."</p>
    }/>
  </Route>
</Routes>
```

> [Remix 网站](https://remix.run/)的主页（React 路由器的创建者创建的 React 框架）如果你向下滚动，会有一个很好的可视化示例，其中包含三级嵌套路由：Sales > Invoices > an invoice.

## `<Outlet/>`

父路由不会自动渲染它们的嵌套路由。毕竟，它们只是组件；它们不知道它们应该在哪里渲染它们的子级，而“把它放在父组件的末尾”并不是一个很好的答案。

相反，你可以使用 `<Outlet/>` 组件告诉父组件在哪里渲染任何嵌套组件。`<Outlet/>` 只渲染两件事之一：

- 如果没有匹配的嵌套路由，它什么也不显示
- 如果有一个匹配的嵌套路由，它会显示它的 `view`

就是这样！但重要的是要知道并记住这一点，因为它是一个常见的“为什么这不起作用？”的挫折来源。如果你没有提供一个 `<Outlet/>`，嵌套路由将不会被显示。

```rust
#[component]
pub fn ContactList() -> impl IntoView {
  let contacts = todo!();

  view! {
    <div style="display: flex">
      // 联系人列表
      <For each=contacts
        key=|contact| contact.id
        children=|contact| todo!()
      />
      // 嵌套子级，如果有的话
      // 不要忘记这个！
      <Outlet/>
    </div>
  }
}
```

## 重构路由定义

如果你不想的话，你不需要在一个地方定义所有路由。你可以将任何 `<Route/>` 及其子级重构到一个单独的组件中。

例如，你可以重构上面的例子，使用两个独立的组件：

```rust
#[component]
fn App() -> impl IntoView {
  view! {
    <Router>
      <Routes>
        <Route path="/contacts" view=ContactList>
          <ContactInfoRoutes/>
          <Route path="" view=|| view! {
            <p>"Select a contact to view more info."</p>
          }/>
        </Route>
      </Routes>
    </Router>
  }
}

#[component(transparent)]
fn ContactInfoRoutes() -> impl IntoView {
  view! {
    <Route path=":id" view=ContactInfo>
      <Route path="" view=EmailAndPhone/>
      <Route path="address" view=Address/>
      <Route path="messages" view=Messages/>
    </Route>
  }
}
```

第二个组件是 `#[component(transparent)]`，这意味着它只返回它的数据，而不是视图：在这种情况下，它是一个 [`RouteDefinition`](https://docs.rs/leptos_router/latest/leptos_router/struct.RouteDefinition.html) 结构体，这是 `<Route/>` 返回的内容。只要它被标记为 `#[component(transparent)]`，这个子路由就可以定义在你想要的任何地方，并作为组件插入到你的路由定义树中。

## 嵌套路由和性能

从概念上讲，所有这些都很不错，但再次强调——有什么大不了的？

性能。

在像 Leptos 这样的细粒度响应式库中，始终重要的是尽可能减少渲染工作。因为我们处理的是真实的 DOM 节点，而不是对虚拟 DOM 进行差异化处理，所以我们希望尽可能少地“重新渲染”组件。嵌套路由使得这变得非常容易。

想象一下我的联系人列表示例。如果我从 Greg 导航到 Alice 再到 Bob，然后返回 Greg，则每次导航时都需要更改联系信息。但 `<ContactList/>` 永远不应该重新渲染。这不仅可以节省渲染性能，还可以维护 UI 中的状态。例如，如果我在 `<ContactList/>` 的顶部有一个搜索栏，则从 Greg 导航到 Alice 再到 Bob 不会清除搜索内容。

实际上，在这种情况下，我们甚至不需要在联系人之间移动时重新渲染 `<Contact/>` 组件。路由器只会随着我们的导航而响应式地更新 `:id` 参数，从而允许我们进行细粒度更新。当我们在联系人之间导航时，我们将更新单个文本节点以更改联系人的姓名、地址等，而无需进行_任何_额外的重新渲染。

> 此沙盒包含本节和上一节中讨论的几个功能（如嵌套路由），以及本章其余部分将介绍的几个功能。路由器是一个如此集成的系统，以至于提供一个单独的示例是有意义的，所以如果你有任何不明白的地方，请不要感到惊讶。

```admonish sandbox title="实时示例" collapsible=true

[点击打开 CodeSandbox.](https://codesandbox.io/p/sandbox/16-router-0-5-4xp4zz?file=%2Fsrc%2Fmain.rs%3A102%2C2)

<noscript>
  请启用 JavaScript 来查看示例。
</noscript>

<template>
  <iframe src="https://codesandbox.io/p/sandbox/16-router-0-5-4xp4zz?file=%2Fsrc%2Fmain.rs%3A102%2C2" width="100%" height="1000px" style="max-height: 100vh"></iframe>
</template>

```

<details>
<summary>CodeSandbox 源码</summary>

```rust
use leptos::*;
use leptos_router::*;

#[component]
fn App() -> impl IntoView {
    view! {
        <Router>
            <h1>"Contact App"</h1>
            // 这个 <nav> 将显示在每个路由上，
            // 因为它在 <Routes/> 之外
            // 注意：我们可以只使用普通的 <a> 标签
            // 路由器将使用客户端导航
            <nav>
                <h2>"Navigation"</h2>
                <a href="/">"Home"</a>
                <a href="/contacts">"Contacts"</a>
            </nav>
            <main>
                <Routes>
                    // / 只有一个未嵌套的 "Home"
                    <Route path="/" view=|| view! {
                        <h3>"Home"</h3>
                    }/>
                    // /contacts 有嵌套路由
                    <Route
                        path="/contacts"
                        view=ContactList
                      >
                        // 如果没有指定 id，则回退
                        <Route path=":id" view=ContactInfo>
                            <Route path="" view=|| view! {
                                <div class="tab">
                                    "(Contact Info)"
                                </div>
                            }/>
                            <Route path="conversations" view=|| view! {
                                <div class="tab">
                                    "(Conversations)"
                                </div>
                            }/>
                        </Route>
                        // 如果没有指定 id，则回退
                        <Route path="" view=|| view! {
                            <div class="select-user">
                                "Select a user to view contact info."
                            </div>
                        }/>
                    </Route>
                </Routes>
            </main>
        </Router>
    }
}

#[component]
fn ContactList() -> impl IntoView {
    view! {
        <div class="contact-list">
            // 这是我们的联系人列表组件本身
            <div class="contact-list-contacts">
                <h3>"Contacts"</h3>
                <A href="alice">"Alice"</A>
                <A href="bob">"Bob"</A>
                <A href="steve">"Steve"</A>
            </div>

            // <Outlet/> 将显示嵌套的子路由
            // 我们可以将此出口放置在布局中的任何位置
            <Outlet/>
        </div>
    }
}

#[component]
fn ContactInfo() -> impl IntoView {
    // 我们可以使用 `use_params_map` 以响应式方式访问 :id 参数
    let params = use_params_map();
    let id = move || params.with(|params| params.get("id").cloned().unwrap_or_default());

    // 假设我们在这里从 API 加载数据
    let name = move || match id().as_str() {
        "alice" => "Alice",
        "bob" => "Bob",
        "steve" => "Steve",
        _ => "User not found.",
    };

    view! {
        <div class="contact-info">
            <h4>{name}</h4>
            <div class="tabs">
                <A href="" exact=true>"Contact Info"</A>
                <A href="conversations">"Conversations"</A>
            </div>

            // 这里的 <Outlet/> 是嵌套在
            // /contacts/:id 路由下的选项卡
            <Outlet/>
        </div>
    }
}

fn main() {
    leptos::mount_to_body(App)
}
```

</details>
</preview>
