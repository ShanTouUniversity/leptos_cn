# 部署全栈 SSR 应用程序

可以将 Leptos 全栈 SSR 应用程序部署到任意数量的服务器或容器托管服务。将 Leptos SSR 应用程序投入生产的最简单方法可能是使用 VPS 服务，并在 VM 中本地运行 Leptos（[有关更多详细信息，请参阅此处](https://github.com/leptos-rs/start-axum?tab=readme-ov-file#executing-a-server-on-a-remote-machine-without-the-toolchain)）。或者，你可以将你的 Leptos 应用程序容器化，并在任何托管或云服务器上的 [Podman](https://podman.io/) 或 [Docker](https://www.docker.com/) 中运行它。

有许多不同的部署设置和托管服务，一般来说，Leptos 本身与你使用的部署设置无关。考虑到部署目标的多样性，在本页我们将介绍：

- [创建一个用于 Leptos SSR 应用程序的 `Containerfile`（或 `Dockerfile`）](#creating-a-containerfile)
- 使用 `Dockerfile` [部署到云服务](#cloud-deployments)——[例如，Fly.io](#deploy-to-flyio)
- 将 Leptos 部署到 [无服务器运行时](#deploy-to-serverless-runtimes)——例如，[AWS Lambda](#aws-lambda) 和 [JS 托管的 WASM 运行时，如 Deno 和 Cloudflare](#deno--cloudflare-workers)
- [尚未获得 Leptos SSR 支持的平台](#currently-unsupported-platforms)

_注意：Leptos 不支持使用任何特定的部署方法或托管服务。_

## 创建一个 Containerfile

人们部署使用 `cargo-leptos` 构建的全栈应用程序最流行的方式是使用支持通过 Podman 或 Docker 构建进行部署的云托管服务。这是一个示例 `Containerfile` / `Dockerfile`，它基于我们用于部署 Leptos 网站的示例。

### Debian

```dockerfile
# 从包含 Rust nightly 的构建环境开始
FROM rustlang/rust:nightly-bullseye as builder

# 如果你使用的是稳定版，请改用此版本
# FROM rust:1.74-bullseye as builder

# 安装 cargo-binstall，这使得安装其他
# cargo 扩展（如 cargo-leptos）变得更容易
RUN wget https://github.com/cargo-bins/cargo-binstall/releases/latest/download/cargo-binstall-x86_64-unknown-linux-musl.tgz
RUN tar -xvf cargo-binstall-x86_64-unknown-linux-musl.tgz
RUN cp cargo-binstall /usr/local/cargo/bin

# 安装 cargo-leptos
RUN cargo binstall cargo-leptos -y

# 添加 WASM 目标
RUN rustup target add wasm32-unknown-unknown

# 创建一个 /app 目录，所有内容最终都将位于其中
RUN mkdir -p /app
WORKDIR /app
COPY . .

# 构建应用程序
RUN cargo leptos build --release -vv

FROM debian:bookworm-slim as runtime
WORKDIR /app
RUN apt-get update -y \
  && apt-get install -y --no-install-recommends openssl ca-certificates \
  && apt-get autoremove -y \
  && apt-get clean -y \
  && rm -rf /var/lib/apt/lists/*

# -- 注意：将二进制文件名从“leptos_start”更新为与 Cargo.toml 中的应用程序名称匹配 --
# 将服务器二进制文件复制到 /app 目录
COPY --from=builder /app/target/release/leptos_start /app/

# /target/site 包含我们的 JS/WASM/CSS 等。
COPY --from=builder /app/target/site /app/site

# 如果在运行时需要 Cargo.toml，请复制它
COPY --from=builder /app/Cargo.toml /app/

# 设置任何所需的 env 变量并
ENV RUST_LOG="info"
ENV LEPTOS_SITE_ADDR="0.0.0.0:8080"
ENV LEPTOS_SITE_ROOT="site"
EXPOSE 8080

# -- 注意：将二进制文件名从“leptos_start”更新为与 Cargo.toml 中的应用程序名称匹配 --
# 运行服务器
CMD ["/app/leptos_start"]
```

### Alpine

```dockerfile
# 从包含 Rust nightly 的构建环境开始
FROM rustlang/rust:nightly-alpine as builder

RUN apk update && \
    apk add --no-cache bash curl npm libc-dev binaryen

RUN npm install -g sass

RUN curl --proto '=https' --tlsv1.2 -LsSf https://github.com/leptos-rs/cargo-leptos/releases/latest/download/cargo-leptos-installer.sh | sh

# 添加 WASM 目标
RUN rustup target add wasm32-unknown-unknown

WORKDIR /work
COPY . .

RUN cargo leptos build --release -vv

FROM rustlang/rust:nightly-alpine as runner

WORKDIR /app

COPY --from=builder /work/target/release/leptos_start /app/
COPY --from=builder /work/target/site /app/site
COPY --from=builder /work/Cargo.toml /app/

EXPOSE $PORT
ENV LEPTOS_SITE_ROOT=./site

CMD ["/app/leptos_start"]
```

> 阅读更多：[Leptos 应用程序的 `gnu` 和 `musl` 构建文件](https://github.com/leptos-rs/leptos/issues/1152#issuecomment-1634916088)。

## 云部署

### 部署到 Fly.io

部署 Leptos SSR 应用程序的一种选择是使用 [Fly.io](https://fly.io/) 之类的服务，该服务采用 Leptos 应用程序的 Dockerfile 定义，并在快速启动的微型虚拟机中运行它；Fly 还提供各种存储选项和托管数据库以用于你的项目。以下示例将展示如何部署一个简单的 Leptos 入门应用程序，只是为了让你入门；如有需要，[请参阅此处以了解有关在 Fly.io 上使用存储选项的更多信息](https://fly.io/docs/database-storage-guides/)。

首先，在你的应用程序的根目录中创建一个 `Dockerfile`，并使用建议的内容（如上）填充它；确保将 Dockerfile 示例中的二进制文件名更新为你的应用程序的名称，并根据需要进行其他调整。

此外，确保你已安装 `flyctl` CLI 工具，并在 [Fly.io](https://fly.io/) 上设置了一个帐户。要在 MacOS、Linux 或 Windows WSL 上安装 `flyctl`，请运行：

```sh
curl -L https://fly.io/install.sh | sh
```

如果你遇到问题，或者要安装到其他平台，[请参阅此处的完整说明](https://fly.io/docs/hands-on/install-flyctl/)

然后登录 Fly.io

```sh
fly auth login
```

并使用以下命令手动启动你的应用程序

```sh
fly launch
```

`flyctl` CLI 工具将引导你完成将你的应用程序部署到 Fly.io 的过程。

```admonish note
默认情况下，Fly.io 会在一段时间后自动停止没有流量进入的机器。虽然 Fly.io 的轻量级虚拟机启动速度很快，但如果你想最大限度地减少 Leptos 应用程序的延迟并确保它始终能够快速响应，请进入生成的 `fly.toml` 文件并将 `min_machines_running` 从默认值 0 更改为 1。

[有关更多详细信息，请参阅 Fly.io 文档中的此页面](https://fly.io/docs/apps/autostart-stop/)。
```

如果你希望使用 Github Actions 来管理你的部署，你将需要通过 [Fly.io](https://fly.io/) Web UI 创建一个新的访问令牌。

转到“帐户”>“访问令牌”并创建一个名为“github_actions”之类的令牌，然后通过进入你的项目的 Github 仓库，然后单击“设置”>“秘密和变量”>“操作”，并将 Fermyon 云令牌添加到你的 Github 仓库的秘密中，并创建一个名为“FLY_API_TOKEN”的“新存储库秘密”。

要生成一个用于部署到 Fly.io 的 `fly.toml` 配置文件，你必须首先从项目源目录中运行以下命令

```sh
fly launch --no-deploy
```

以创建一个新的 Fly 应用程序并将其注册到服务中。Git 提交你的新 `fly.toml` 文件。

要设置 Github Actions 部署工作流，请将以下内容复制到 `.github/workflows/fly_deploy.yml` 文件中：

```admonish example collapsible=true

	# 有关更多详细信息，请参阅：https://fly.io/docs/app-guides/continuous-deployment-with-github-actions/

	name: 部署到 Fly.io
	on:
	push:
		branches:
		- main
	jobs:
	deploy:
		name: 部署应用程序
		runs-on: ubuntu-latest
		steps:
		- uses: actions/checkout@v4
		- uses: superfly/flyctl-actions/setup-flyctl@master
		- name: 部署到 fly
			id: deployment
			run: |
			  flyctl deploy --remote-only | tail -n 1 >> $GITHUB_STEP_SUMMARY
			env:
			  FLY_API_TOKEN: ${{ secrets.FLY_API_TOKEN }}

```

下次提交到你的 Github `main` 分支时，你的项目将自动部署到 Fly.io。

请参阅 [此处的示例仓库](https://github.com/diversable/fly-io-leptos-ssr-test-deploy)。

### Railway

另一个云部署提供商是 [Railway](https://railway.app/)。
Railway 与 GitHub 集成以自动部署你的代码。

有一个自以为是的社区模板可以让你快速入门：

[![在 Railway 上部署](https://railway.app/button.svg)](https://railway.app/template/pduaM5?referralCode=fZ-SY1)

该模板已设置 renovate 以保持依赖项最新，并支持 GitHub Actions 在部署之前测试你的代码。

Railway 有一个免费套餐，不需要信用卡，而且由于 Leptos 需要的资源很少，该免费套餐应该可以使用很长时间。

请参阅 [此处的示例仓库](https://github.com/marvin-bitterlich/leptos-railway)。

## 部署到无服务器运行时

Leptos 支持部署到 FaaS（函数即服务）或“无服务器”运行时（如 AWS Lambda），以及 [WinterCG](https://wintercg.org/) 兼容的 JS 运行时（如 [Deno](https://deno.com/deploy) 和 Cloudflare）。请注意，与虚拟机或容器类型的部署相比，无服务器环境确实对 SSR 应用程序可用的功能有一些限制（请参阅下面的注释）。

### AWS Lambda

借助 [Cargo Lambda](https://www.cargo-lambda.info/) 工具，Leptos SSR 应用程序可以部署到 AWS Lambda。[leptos-rs/start-aws](https://github.com/leptos-rs/start-aws) 提供了一个使用 Axum 作为服务器的入门模板仓库；那里的说明可以适用于你使用 Leptos+Actix-web 服务器。入门仓库包括一个用于 CI/CD 的 Github Actions 脚本，以及有关设置你的 Lambda 函数和获取云部署所需凭据的说明。

但是，请记住，某些本机服务器功能不适用于 Lambda 等 FaaS 服务，因为环境在不同请求之间不一定一致。特别是，['start-aws' 文档](https://github.com/leptos-rs/start-aws#state) 指出，“由于 AWS Lambda 是一个无服务器平台，因此你需要更加小心地管理长期存在的州。写入磁盘或使用状态提取器在不同请求之间无法可靠地工作。相反，你需要一个数据库或其他微服务，你可以从 Lambda 函数中查询它们。”

要记住的另一个因素是函数即服务的“冷启动”时间——根据你的用例和你使用的 FaaS 平台，这可能满足也可能不满足你的延迟要求；你可能需要始终保持一个函数运行以优化你的请求的速度。

### Deno 和 Cloudflare Workers

目前，Leptos-Axum 支持在 Javascript 托管的 WebAssembly 运行时（如 Deno、Cloudflare Workers 等）中运行。此选项需要对你的源代码设置进行一些更改（例如，在 `Cargo.toml` 中，你必须使用 `crate-type = ["cdylib"]` 定义你的应用程序，并且必须为 `leptos_axum` 启用“wasm”功能）。[Leptos HackerNews JS-fetch 示例](https://github.com/leptos-rs/leptos/tree/leptos_0.6/examples/hackernews_js_fetch) 演示了所需的修改，并展示了如何在 Deno 运行时中运行应用程序。此外，[`leptos_axum` crate 文档](https://docs.rs/leptos_axum/latest/leptos_axum/#js-fetch-integration) 是为 JS 托管的 WASM 运行时设置你自己的 `Cargo.toml` 文件时的有用参考。

虽然 JS 托管的 WASM 运行时的初始设置并不繁琐，但要记住更重要的限制是，由于你的应用程序将在服务器和客户端上编译为 WebAssembly (`wasm32-unknown-unknown`)，因此你必须确保你应用程序中使用的 crate 都与 WASM 兼容；根据你的应用程序的要求，这可能是也可能不是一个障碍，因为并非 Rust 生态系统中的所有 crate 都支持 WASM。

如果你愿意接受 WASM 服务器端的限制，那么现在开始的最佳方式是查看官方 Leptos Github 仓库中 [使用 Deno 运行 Leptos 的示例](https://github.com/leptos-rs/leptos/tree/leptos_0.6/examples/hackernews_js_fetch)。

## 正在进行 Leptos 支持的平台

### 部署到 Spin 无服务器 WASI（使用 Leptos SSR）

服务器端的 WebAssembly 最近一直在蓬勃发展，开源无服务器 WebAssembly 框架 Spin 的开发人员正在努力原生支持 Leptos。虽然 Leptos-Spin SSR 集成仍处于早期阶段，但有一个你可以尝试的有效示例。

让 Leptos SSR 和 Spin 协同工作的完整说明可在 [Fermyon 博客上的一篇文章](https://www.fermyon.com/blog/leptos-spin-get-started) 中找到，或者，如果你想跳过文章并直接开始玩一个有效的入门仓库，[请参阅此处](https://github.com/diversable/leptos-spin-ssr-test)。

### 部署到 Shuttle.rs

一些 Leptos 用户询问了使用对 Rust 友好的 [Shuttle.rs](https://www.shuttle.rs/) 服务部署 Leptos 应用程序的可能性。不幸的是，Leptos 目前尚未得到 Shuttle.rs 服务的官方支持。

但是，Shuttle.rs 的工作人员致力于在未来获得 Leptos 支持；如果你想了解这项工作的最新状态，请关注 [此 Github 问题](https://github.com/shuttle-hq/shuttle/issues/1002#issuecomment-1853661643)。

此外，已经做出了一些努力来让 Shuttle 与 Leptos 协同工作，但到目前为止，部署到 Shuttle 云仍然无法按预期工作。如果你想自己调查或贡献修复程序，这项工作可在此处获得：[用于 Shuttle.rs 的 Leptos Axum 入门模板](https://github.com/Rust-WASI-WASM/shuttle-leptos-axum)。
