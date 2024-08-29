# 测试你的组件

测试用户界面可能相对棘手，但确实很重要。本文将讨论测试 Leptos 应用程序的一些原则和方法。

## 1. 使用普通的 Rust 测试来测试业务逻辑

在许多情况下，将逻辑从你的组件中提取出来并单独测试是有意义的。对于一些简单的组件，没有特别的逻辑需要测试，但对于许多组件来说，值得使用一个可测试的包装类型，并在普通的 Rust `impl` 块中实现逻辑。

例如，与其像这样直接在组件中嵌入逻辑：

```rust
#[component]
pub fn TodoApp() -> impl IntoView {
    let (todos, set_todos) = create_signal(vec![Todo { /* ... */ }]);
    // ⚠️ 这很难测试，因为它嵌入在组件中
    let num_remaining = move || todos.with(|todos| {
        todos.iter().filter(|todo| !todo.completed).sum()
    });
}
```

你可以将该逻辑提取到一个单独的数据结构中并对其进行测试：

```rust
pub struct Todos(Vec<Todo>);

impl Todos {
    pub fn num_remaining(&self) -> usize {
        self.0.iter().filter(|todo| !todo.completed).sum()
    }
}

#[cfg(test)]
mod tests {
    #[test]
    fn test_remaining() {
        // ...
    }
}

#[component]
pub fn TodoApp() -> impl IntoView {
    let (todos, set_todos) = create_signal(Todos(vec![Todo { /* ... */ }]));
    // ✅ 这有一个与之关联的测试
    let num_remaining = move || todos.with(Todos::num_remaining);
}
```

一般来说，你的组件本身包含的逻辑越少，你的代码就越容易理解，也越容易测试。

## 2. 使用端到端（`e2e`）测试来测试组件

我们的 [`examples`](https://github.com/leptos-rs/leptos/tree/leptos_0.6/examples) 目录中有几个示例，其中包含使用不同测试工具进行的广泛的端到端测试。

了解如何使用这些示例的最简单方法是查看测试示例本身：

### 使用 [`counter`](https://github.com/leptos-rs/leptos/blob/main/examples/counter/tests/web.rs) 的 `wasm-bindgen-test`

这是一个相当简单的手动测试设置，它使用 [`wasm-pack test`](https://rustwasm.github.io/wasm-pack/book/commands/test.html) 命令。

#### 示例测试

```rust
#[wasm_bindgen_test]
fn clear() {
    let document = leptos::document();
    let test_wrapper = document.create_element("section").unwrap();
    let _ = document.body().unwrap().append_child(&test_wrapper);

    mount_to(
        test_wrapper.clone().unchecked_into(),
        || view! { <SimpleCounter initial_value=10 step=1/> },
    );

    let div = test_wrapper.query_selector("div").unwrap().unwrap();
    let clear = test_wrapper
        .query_selector("button")
        .unwrap()
        .unwrap()
        .unchecked_into::<web_sys::HtmlElement>();

    clear.click();

assert_eq!(
    div.outer_html(),
    // 这里我们生成一个微型响应式系统来渲染测试用例
    run_scope(create_runtime(), || {
        // 就好像我们用值 0 创建它一样，对吧？
        let (value, set_value) = create_signal(0);

        // 我们可以删除事件监听器，因为它们不会渲染到 HTML 中
        view! {
            <div>
                <button>"Clear"</button>
                <button>"-1"</button>
                <span>"Value: " {value} "!"</span>
                <button>"+1"</button>
            </div>
        }
        // 返回的视图是 HtmlElement<Div>，它是 DOM 元素的智能指针。所以我们仍然可以调用 .outer_html()
        .outer_html()
    })
);
}
```

### [`wasm-bindgen-test` 和 `counters`](https://github.com/leptos-rs/leptos/tree/leptos_0.6/examples/counters/tests/web.rs)

这个更发达的测试套件使用了一个 fixtures 系统来重构 `counter` 测试的手动 DOM 操作，并轻松地测试各种情况。

#### 示例测试

```rust
use super::*;
use crate::counters_page as ui;
use pretty_assertions::assert_eq;

#[wasm_bindgen_test]
fn should_increase_the_total_count() {
    // 给定
    ui::view_counters();
    ui::add_counter();

    // 当
    ui::increment_counter(1);
    ui::increment_counter(1);
    ui::increment_counter(1);

    // 那么
    assert_eq!(ui::total(), 3);
}
```

### [Playwright 和 `counters`](https://github.com/leptos-rs/leptos/tree/main/examples/counters/e2e)

这些测试使用常见的 JavaScript 测试工具 Playwright 在同一个例子上运行端到端测试，使用许多以前做过前端开发的人熟悉的库和测试方法。

#### 示例测试

```js
import { test, expect } from "@playwright/test";
import { CountersPage } from "./fixtures/counters_page";

test.describe("Increment Count", () => {
  test("should increase the total count", async ({ page }) => {
    const ui = new CountersPage(page);
    await ui.goto();
    await ui.addCounter();

    await ui.incrementCount();
    await ui.incrementCount();
    await ui.incrementCount();

    await expect(ui.total).toHaveText("3");
  });
});
```

### [使用 `todo_app_sqlite` 的 Gherkin/Cucumber 测试](https://github.com/leptos-rs/leptos/blob/leptos_0.6/examples/todo_app_sqlite/e2e/README.md)

你可以将任何你喜欢的测试工具集成到这个流程中。这个例子使用 Cucumber，一个基于自然语言的测试框架。

```
@add_todo
Feature: Add Todo

    Background:
        Given I see the app

    @add_todo-see
    Scenario: Should see the todo
        Given I set the todo as Buy Bread
        When I click the Add button
        Then I see the todo named Buy Bread

    # @allow.skipped
    @add_todo-style
    Scenario: Should see the pending todo
        When I add a todo as Buy Oranges
        Then I see the pending todo
```

这些操作的定义在 Rust 代码中。

```rust
use crate::fixtures::{action, world::AppWorld};
use anyhow::{Ok, Result};
use cucumber::{given, when};

#[given("I see the app")]
#[when("I open the app")]
async fn i_open_the_app(world: &mut AppWorld) -> Result<()> {
    let client = &world.client;
    action::goto_path(client, "").await?;

    Ok(())
}

#[given(regex = "^I add a todo as (.*)$")]
#[when(regex = "^I add a todo as (.*)$")]
async fn i_add_a_todo_titled(world: &mut AppWorld, text: String) -> Result<()> {
    let client = &world.client;
    action::add_todo(client, text.as_str()).await?;

    Ok(())
}

// 等等。
```

### 了解更多

请随时查看 Leptos 仓库中的 CI 设置，以了解更多关于如何在你的应用程序中使用这些工具的信息。所有这些测试方法都会定期针对实际的 Leptos 示例应用程序运行。
