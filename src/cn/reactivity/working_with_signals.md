# 使用信号

到目前为止，我们已经使用了一些 [`create_signal`](https://docs.rs/leptos/latest/leptos/fn.create_signal.html) 的简单示例，它返回一个 [`ReadSignal`](https://docs.rs/leptos/latest/leptos/struct.ReadSignal.html) getter 和一个 [`WriteSignal`](https://docs.rs/leptos/latest/leptos/struct.WriteSignal.html) setter。

## 获取和设置

有四种基本的信号操作：

1. [`.get()`](https://docs.rs/leptos/latest/leptos/struct.ReadSignal.html#impl-SignalGet%3CT%3E-for-ReadSignal%3CT%3E) 克隆信号的当前值，并以响应式方式跟踪对该值的任何未来更改。
2. [`.with()`](https://docs.rs/leptos/latest/leptos/struct.ReadSignal.html#impl-SignalWith%3CT%3E-for-ReadSignal%3CT%3E) 接受一个函数，该函数通过引用 (`&T`) 接收信号的当前值，并跟踪任何未来更改。
3. [`.set()`](https://docs.rs/leptos/latest/leptos/struct.WriteSignal.html#impl-SignalSet%3CT%3E-for-WriteSignal%3CT%3E) 替换信号的当前值，并通知任何订阅者他们需要更新。
4. [`.update()`](https://docs.rs/leptos/latest/leptos/struct.WriteSignal.html#impl-SignalUpdate%3CT%3E-for-WriteSignal%3CT%3E) 接受一个函数，该函数接收信号当前值的 mutable 引用 (`&mut T`)，并通知任何订阅者他们需要更新。（`.update()` 不返回闭包返回的值，但如果需要，你可以使用 [`.try_update()`](https://docs.rs/leptos/latest/leptos/trait.SignalUpdate.html#tymethod.try_update)；例如，如果你要从 `Vec<_>` 中删除一个项目并想要这个被删除的项目。）

将 `ReadSignal` 作为函数调用是 `.get()` 的语法糖。将 `WriteSignal` 作为函数调用是 `.set()` 的语法糖。所以

```rust
let (count, set_count) = create_signal(0);
set_count(1);
logging::log!(count());
```

与以下代码相同

```rust
let (count, set_count) = create_signal(0);
set_count.set(1);
logging::log!(count.get());
```

你可能会注意到 `.get()` 和 `.set()` 可以用 `.with()` 和 `.update()` 来实现。换句话说，`count.get()` 与 `count.with(|n| n.clone())` 相同，而 `count.set(1)` 是通过 `count.update(|n| *n = 1)` 实现的。

但是当然，`.get()` 和 `.set()`（或者普通的函数调用形式！）是更好的语法。

然而，`.with()` 和 `.update()` 有一些非常好的用例。

例如，考虑一个保存 `Vec<String>` 的信号。

```rust
let (names, set_names) = create_signal(Vec::new());
if names().is_empty() {
	set_names(vec!["Alice".to_string()]);
}
```

从逻辑上讲，这很简单，但它隐藏了一些明显的低效之处。记住，`names().is_empty()` 是 `names.get().is_empty()` 的语法糖，它克隆了值（它是 `names.with(|n| n.clone()).is_empty()`）。这意味着我们克隆了整个 `Vec<String>`，运行 `is_empty()`，然后立即丢弃克隆。

同样，`set_names` 用一个全新的 `Vec<_> `替换了该值。这很好，但我们不妨直接原地修改原始的 `Vec<_>`。

```rust
let (names, set_names) = create_signal(Vec::new());
if names.with(|names| names.is_empty()) {
	set_names.update(|names| names.push("Alice".to_string()));
}
```

现在我们的函数只是通过引用获取 `names` 来运行 `is_empty()`，避免了克隆。

如果你打开了 Clippy，或者你目光敏锐，你可能会注意到我们可以让它更简洁：

```rust
if names.with(Vec::is_empty) {
	// ...
}
```

毕竟，`.with()` 只是接受一个通过引用获取值的函数。因为 `Vec::is_empty` 接受 `&self`，我们可以直接传入它，避免不必要的闭包。

有一些辅助宏可以使 `.with()` 和 `.update()` 更易于使用，尤其是在使用多个信号时。

```rust
let (first, _) = create_signal("Bob".to_string());
let (middle, _) = create_signal("J.".to_string());
let (last, _) = create_signal("Smith".to_string());
```

如果你想将这 3 个信号连接在一起而不需要不必要的克隆，你必须编写如下内容：

```rust
let name = move || {
	first.with(|first| {
		middle.with(|middle| last.with(|last| format!("{first} {middle} {last}")))
	})
};
```

这写起来很长很烦人。

相反，你可以使用 `with!` 宏同时获取所有信号的引用。

```rust
let name = move || with!(|first, middle, last| format!("{first} {middle} {last}"));
```

这与上面的展开相同。查看 [`with!`](https://docs.rs/leptos/latest/leptos/macro.with.html) 文档了解更多信息，以及相应的宏 [`update!`](https://docs.rs/leptos/latest/leptos/macro.update.html)、[`with_value!`](https://docs.rs/leptos/latest/leptos/macro.with_value.html) 和 [`update_value!`](https://docs.rs/leptos/latest/leptos/macro.update_value.html)。

## 使信号相互依赖

人们经常会问一些信号需要根据其他信号的值而改变的情况。有三种好方法可以做到这一点，还有一种不太理想但可以在可控情况下使用的方法。

### 好的选择

**1）B 是 A 的函数。**为 A 创建一个信号，为 B 创建一个派生信号或 memo 。

```rust
let (count, set_count) = create_signal(1); // A
let derived_signal_double_count = move || count() * 2; // B 是 A 的函数
let memoized_double_count = create_memo(move |_| count() * 2); // B 是 A 的函数  
```

> 有关何时使用派生信号或 memo 的指导，请参阅 [`create_memo`](https://docs.rs/leptos/latest/leptos/fn.create_memo.html) 的文档

**2）C 是 A 和其他事物 B 的函数。**为 A 和 B 创建信号，为 C 创建派生信号或  memo 。

```rust
let (first_name, set_first_name) = create_signal("Bridget".to_string()); // A
let (last_name, set_last_name) = create_signal("Jones".to_string()); // B
let full_name = move || with!(|first_name, last_name| format!("{first_name} {last_name}")); // C 是 A 和 B 的函数
```

**3）A 和 B 是独立的信号，但有时同时更新。**当你调用更新 A 时，进行单独的调用来更新 B。

```rust
let (age, set_age) = create_signal(32); // A
let (favorite_number, set_favorite_number) = create_signal(42); // B
// 使用它来处理对 `Clear` 按钮的点击
let clear_handler = move |_| {
  // 同时更新 A 和 B
  set_age(0);
  set_favorite_number(0);
};
```

### 如果你真的必须...

**4) 创建一个效果，每当 A 发生变化时写入 B。**这在官方上是不鼓励的，原因有以下几点：
a) 它总是效率较低，因为这意味着每次 A 更新时，你都要完整地执行两次响应式过程。（你设置 A，这会导致效果运行，以及任何其他依赖于 A 的效果。然后你设置 B，这会导致任何依赖于 B 的效果运行。）
b) 它增加了你意外创建无限循环或过度运行效果的可能性。这是一种乒乓球式的、响应式意大利面条式代码，在 2010 年代初期很常见，我们试图通过读写隔离和不鼓励从效果中写入信号来避免这种情况。

在大多数情况下，最好重写代码，使其基于派生信号或 memo 具有清晰的自上而下的数据流。但这并不是世界末日。

> 我故意在这里没有提供示例。阅读 [`create_effect`](https://docs.rs/leptos/latest/leptos/fn.create_effect.html) 文档以了解它是如何工作的。
