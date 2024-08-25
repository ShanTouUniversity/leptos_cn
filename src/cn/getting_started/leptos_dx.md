# Leptos 开发体验改进

你可以做一些事情来改进使用 Leptos 开发网站和应用程序的体验。 你可能需要花几分钟时间设置你的环境以优化你的开发体验，特别是如果你想跟随本书中的示例进行编码。

## 1) 设置 `console_error_panic_hook`

默认情况下，在浏览器中运行 WASM 代码时发生的 panic 只会在浏览器中抛出一个错误，并显示一条无用的消息，例如 `Unreachable executed` 以及指向 WASM 二进制文件的堆栈跟踪。

使用 `console_error_panic_hook`，你可以获得一个实际的 Rust 堆栈跟踪，其中包含 Rust 源代码中的一行。

设置非常简单：

1. 在你的项目中运行 `cargo add console_error_panic_hook`
2. 在你的 `main` 函数中，添加 `console_error_panic_hook::set_once();`

> 如果不清楚，[点击此处查看示例](https://github.com/leptos-rs/leptos/blob/main/examples/counter/src/main.rs#L6)。

现在，你应该在浏览器控制台中看到更好的 panic 消息！

## 2) 在 `#[component]` 和 `#[server]` 中进行编辑器自动补全

由于宏的性质（它们可以从任何内容扩展到任何内容，但前提是输入在那一刻完全正确），rust-analyzer 很难进行正确的自动补全和其他支持。

如果你在编辑器中使用这些宏时遇到问题，你可以明确告诉 rust-analyzer 忽略某些过程宏。 尤其是对于 `#[server]` 宏，它注释函数体但实际上并不转换函数体内的任何内容，这可能非常有用。

从 Leptos 0.5.3 版本开始，添加了对 `#[component]` 宏的 rust-analyzer 支持，但如果你遇到问题，你可能也希望将 `#[component]` 添加到宏忽略列表中（见下文）。
请注意，这意味着 rust-analyzer 不知道你的组件 props，这可能会在 IDE 中生成它自己的一组错误或警告。

VSCode `settings.json`：

```json
"rust-analyzer.procMacro.ignored": {
	"leptos_macro": [
        // optional:
		// "component",
		"server"
	],
}
```

VSCode with cargo-leptos `settings.json`:
```json
"rust-analyzer.procMacro.ignored": {
	"leptos_macro": [
        // optional:
		// "component",
		"server"
	],
},
// 如果为 `ssr` 功能配置的代码显示为非活动状态，
// 你可能希望告诉 rust-analyzer 默认启用 `ssr` 功能
//
// 你也可以使用 `rust-analyzer.cargo.allFeatures` 来启用所有功能
"rust-analyzer.cargo.features": ["ssr"]
```

neovim with lspconfig:

```lua
require('lspconfig').rust_analyzer.setup {
  -- Other Configs ...
  settings = {
    ["rust-analyzer"] = {
      -- Other Settings ...
      procMacro = {
        ignored = {
            leptos_macro = {
                -- optional: --
                -- "component",
                "server",
            },
        },
      },
    },
  }
}
```

Helix, in `.helix/languages.toml`:

```toml
[[language]]
name = "rust"

[language-server.rust-analyzer]
config = { procMacro = { ignored = { leptos_macro = [
	# Optional:
	# "component",
	"server"
] } } }
```

Zed, in `settings.json`:

```json
{
  -- Other Settings ...
  "lsp": {
    "rust-analyzer": {
      "procMacro": {
        "ignored": [
          // optional:
          // "component",
          "server"
        ]
      }
    }
  }
}
```

SublimeText 3, under `LSP-rust-analyzer.sublime-settings` in `Goto Anything...` menu:

```json
// 此处的设置将覆盖 "LSP-rust-analyzer/LSP-rust-analyzer.sublime-settings" 中的设置
{
  "rust-analyzer.procMacro.ignored": {
    "leptos_macro": [
      // optional:
      // "component",
      "server"
    ],
  },
}
```

## 3) 使用 Rust Analyzer 设置 `leptosfmt`（可选）

`leptosfmt` 是 Leptos `view!` 宏的格式化程序（你通常会在其中编写 UI 代码）。 因为 `view!` 宏启用了一种 'RSX'（类似于 JSX）风格的 UI 编写方式，所以 cargo-fmt 很难自动格式化 `view!` 宏内的代码。 `leptosfmt` 是一个解决格式化问题的 crate，它可以使你的 RSX 风格的 UI 代码看起来整洁漂亮！

`leptosfmt` 可以通过命令行或在代码编辑器中安装和使用：

首先，使用 `cargo install leptosfmt` 安装该工具。

如果你只想从命令行使用默认选项，只需从项目的根目录运行 `leptosfmt ./**/*.rs` 即可使用 `leptosfmt` 格式化所有 Rust 文件。

如果你希望将编辑器设置为使用 `leptosfmt`，或者希望自定义 `leptosfmt` 体验，请参阅 [`leptosfmt` github 仓库的 README.md 页面](https://github.com/bram209/leptosfmt) 上的说明。

请注意，建议在每个工作区的基础上设置你的编辑器与 `leptosfmt`，以获得最佳结果。
