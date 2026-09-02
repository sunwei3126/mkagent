# WebUI

WebUI 是通过 `apps/webui` 加载的共享 React 渲染层（`packages/ui`）。只开浏览器不够：必须同时运行 headless server（`packages/server`），负责登录、`/api/*` 和 WebSocket RPC。

| 模式 | 适用场景 | 浏览器地址 |
| --- | --- | --- |
| 一次性 | 在同一端口测试打包后的 UI | `http://127.0.0.1:9100`（`server:dev:webui` 则为 `3100`） |
| 开发（HMR） | 改 `apps/webui` / `packages/ui` | `http://127.0.0.1:5175` |

## 前置要求

在仓库根目录：

```bash
bun install --frozen-lockfile
```

`MKAGENT_SERVER_TOKEN` 必填（至少 16 个字符，不能是单一字符重复）。本仓库本地开发用 `mkagent-local-dev-token`。登录表单优先使用 `MKAGENT_WEBUI_PASSWORD`，未设置则回退到同一 token。

`MKAGENT_WEBUI_DIR` 必须指向已构建的 `apps/webui/dist`。目录不存在时 server 仍会启动 RPC，但不会提供登录页。

## 一次性（RPC 端口上的打包资源）

先构建 Pi 子进程和 WebUI，再以 `MKAGENT_WEBUI_DIR=apps/webui/dist` 启动 server。未登录的 HTML 请求会 302 到 `/login`。`POST /api/auth` 成功后写入 session cookie，并加载 `/`。

### Windows PowerShell

`server:prod` 这类脚本用 POSIX 前缀设置环境变量（`VAR=value command`）。在 `bun run` 内部可以，直接粘到 PowerShell 会失败。先在 shell 里设变量，再启动入口：

```powershell
$env:MKAGENT_SERVER_TOKEN = "mkagent-local-dev-token"
$env:MKAGENT_WEBUI_DIR = "apps/webui/dist"
$env:MKAGENT_BUNDLED_ASSETS_ROOT = "$PWD\apps\electron"
$env:MKAGENT_RPC_HOST = "127.0.0.1"
$env:MKAGENT_RPC_PORT = "9100"
$env:MKAGENT_DEBUG = "true"

bun run server:build:subprocess
bun run webui:build
bun run packages/server/src/index.ts
```

若 `apps/webui/dist` 和 `packages/pi-agent-server/dist` 已经存在，可跳过两行 build。

### Unix / `bun run` 脚本

`server:prod` 会构建两份产物，但**不会**设置 `MKAGENT_SERVER_TOKEN`。先 export：

```bash
export MKAGENT_SERVER_TOKEN=mkagent-local-dev-token
bun run server:prod
```

本地 debug bundle（默认 token、`MKAGENT_DEBUG=true`、RPC 端口 **3100**）：

```bash
bun run server:dev:webui
```

打开 `http://127.0.0.1:3100`，不是 9100。

### 就绪标志

stdout 应包含：

```text
MKAGENT_SERVER_URL=ws://127.0.0.1:9100
MKAGENT_SERVER_TOKEN=mkagent-local-dev-token
MKAGENT_WEBUI_URL=http://0.0.0.0:9100
```

（`server:dev:webui` 时端口为 `3100`。）用该地址打开浏览器，把 token 填进登录表单。

## 开发模式（Vite HMR）

两个进程。Vite 在 5175 提供渲染层，并把 `/api`、`/login`、`/ws` 代理到 `127.0.0.1:${MKAGENT_RPC_PORT:-9100}`。

Windows 上不要用 `bun run webui:dev`：脚本以 `lsof -ti:5175 | xargs kill -9` 开头，系统没有 `lsof`。

### 终端 1 — headless server

这里不需要 WebUI 静态资源，由 Vite 提供渲染层。

```powershell
$env:MKAGENT_SERVER_TOKEN = "mkagent-local-dev-token"
$env:MKAGENT_BUNDLED_ASSETS_ROOT = "$PWD\apps\electron"
$env:MKAGENT_RPC_HOST = "127.0.0.1"
$env:MKAGENT_RPC_PORT = "9100"
$env:MKAGENT_DEBUG = "true"
bun run server:build:subprocess
bun run packages/server/src/index.ts
```

Unix 等价命令：

```bash
export MKAGENT_SERVER_TOKEN=mkagent-local-dev-token
export MKAGENT_BUNDLED_ASSETS_ROOT="$PWD/apps/electron"
export MKAGENT_DEBUG=true
bun run server:build:subprocess
bun run packages/server/src/index.ts
```

通过 Bun 的 script runner 调用时，`bun run server:dev` 做同样的事（构建子进程、默认 token、debug、bundled assets）。

### 终端 2 — Vite

```bash
bunx vite dev --config apps/webui/vite.config.ts
```

若 server 在 3100，Vite 必须用同一端口，否则登录会打到错误的代理：

```powershell
$env:MKAGENT_RPC_PORT = "3100"
bunx vite dev --config apps/webui/vite.config.ts
```

打开 `http://127.0.0.1:5175`。登录仍走 `/api` 和 `/login` 代理。

## 端口

| 来源 | 端口 | 浏览器地址 |
| --- | --- | --- |
| 默认 RPC / 一次性 | 9100 | `http://127.0.0.1:9100` |
| `server:dev:webui` | 3100 | `http://127.0.0.1:3100` |
| Vite HMR | 5175 | `http://127.0.0.1:5175` |

## 环境变量

| 变量 | 是否必需 | 说明 |
| --- | --- | --- |
| `MKAGENT_SERVER_TOKEN` | 是 | RPC bearer 与 JWT 签名密钥。最少 16 字符。 |
| `MKAGENT_WEBUI_DIR` | 一次性 | 指向 `apps/webui/dist`。纯 Vite 模式不需要。 |
| `MKAGENT_BUNDLED_ASSETS_ROOT` | 本地开发 | 指向 `apps/electron`，让 server 找到 bundled resources。 |
| `MKAGENT_RPC_HOST` | 否 | 默认 `127.0.0.1`。非 localhost 且无 TLS 时拒绝绑定，除非 `--allow-insecure-bind`。 |
| `MKAGENT_RPC_PORT` | 否 | 默认 `9100`。Vite 代理必须一致。 |
| `MKAGENT_WEBUI_PASSWORD` | 否 | 登录密码；未设置则回退到 server token。 |
| `MKAGENT_DEBUG` | 否 | 额外日志。 |
| `CONFIG_DIR` | 否 | 隔离 `~/.mkagent`。 |

完整环境变量表见 [development.md](./development.md)。

## 常见坑

- 未设置 `MKAGENT_SERVER_TOKEN` 就跑 `bun run server:prod`：进程退出，提示 `Server token is required`。
- 缺少 `apps/webui/dist` → RPC 起来了，但没有 `MKAGENT_WEBUI_URL`，一次性模式下浏览器没有页面。
- Windows 上 `bun run webui:dev` 失败（`lsof`）。
- `server:dev:webui`（3100）搭配 Vite 默认代理（9100），登录会表现为 server 没开。
- 残留的 `~/.mkagent/.server.lock` 会挡住第二次启动。先停掉另一个 server，或用 `CONFIG_DIR` 隔离。
- 登录有速率限制：每个客户端 IP 60 秒内 5 次。

## 相关文档

- [development.md](./development.md) — 环境变量与 workspace 命令
- [architecture.md](./architecture.md) — 鉴权握手
- [cli.md](./cli.md) — 同一 server 的 WebSocket CLI
