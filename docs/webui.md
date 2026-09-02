# WebUI

The WebUI is the shared React renderer (`packages/ui`) loaded through `apps/webui`. The browser is not enough: a headless server (`packages/server`) must be running for login, `/api/*`, and the WebSocket RPC.

| Mode | When to use | Browser URL |
| --- | --- | --- |
| One-shot | Test the bundled UI on one port | `http://127.0.0.1:9100` (or `3100` for `server:dev:webui`) |
| Dev (HMR) | Iterate on `apps/webui` / `packages/ui` | `http://127.0.0.1:5175` |

## Prerequisites

From the repo root:

```bash
bun install --frozen-lockfile
```

`MKAGENT_SERVER_TOKEN` is required (minimum 16 characters, not a single repeated character). The local-dev value used in this repo is `mkagent-local-dev-token`. The login form accepts `MKAGENT_WEBUI_PASSWORD` if set, otherwise the same token.

`MKAGENT_WEBUI_DIR` must point at a built `apps/webui/dist`. If that directory is missing, the server still starts RPC but does not serve the login page.

## One-shot (bundled assets on the RPC port)

Build the Pi subprocess and the WebUI, then start the server with `MKAGENT_WEBUI_DIR=apps/webui/dist`. Unauthenticated HTML requests redirect to `/login`. A successful `POST /api/auth` sets a session cookie and loads `/`.

### Windows PowerShell

Repo scripts such as `server:prod` prefix environment variables in POSIX form (`VAR=value command`). That is fine inside `bun run`, but pasting the same line into PowerShell fails. Set the variables in the shell, then start the entrypoint:

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

Skip the two `bun run …build` lines when `apps/webui/dist` and `packages/pi-agent-server/dist` already exist.

### Unix / `bun run` scripts

`server:prod` builds both artifacts but does **not** set `MKAGENT_SERVER_TOKEN`. Export it first:

```bash
export MKAGENT_SERVER_TOKEN=mkagent-local-dev-token
bun run server:prod
```

For a local debug bundle (default token, `MKAGENT_DEBUG=true`, RPC port **3100**):

```bash
bun run server:dev:webui
```

Open `http://127.0.0.1:3100`, not 9100.

### Ready

Stdout should contain:

```text
MKAGENT_SERVER_URL=ws://127.0.0.1:9100
MKAGENT_SERVER_TOKEN=mkagent-local-dev-token
MKAGENT_WEBUI_URL=http://0.0.0.0:9100
```

(`3100` instead of `9100` when using `server:dev:webui`.) Open that host/port and paste the token into the login form.

## Dev mode (Vite HMR)

Two processes. Vite serves the renderer on 5175 and proxies `/api`, `/login`, and `/ws` to `127.0.0.1:${MKAGENT_RPC_PORT:-9100}`.

Do not use `bun run webui:dev` on Windows: the script starts with `lsof -ti:5175 | xargs kill -9`, which is not available.

### Terminal 1 — headless server

WebUI static assets are not required here; Vite serves the renderer.

```powershell
$env:MKAGENT_SERVER_TOKEN = "mkagent-local-dev-token"
$env:MKAGENT_BUNDLED_ASSETS_ROOT = "$PWD\apps\electron"
$env:MKAGENT_RPC_HOST = "127.0.0.1"
$env:MKAGENT_RPC_PORT = "9100"
$env:MKAGENT_DEBUG = "true"
bun run server:build:subprocess
bun run packages/server/src/index.ts
```

Unix equivalent:

```bash
export MKAGENT_SERVER_TOKEN=mkagent-local-dev-token
export MKAGENT_BUNDLED_ASSETS_ROOT="$PWD/apps/electron"
export MKAGENT_DEBUG=true
bun run server:build:subprocess
bun run packages/server/src/index.ts
```

`bun run server:dev` does the same (subprocess build, default token, debug, bundled assets) when invoked through Bun's script runner.

### Terminal 2 — Vite

```bash
bunx vite dev --config apps/webui/vite.config.ts
```

If the server is on 3100, the Vite process must use the same port or login will miss the proxy:

```powershell
$env:MKAGENT_RPC_PORT = "3100"
bunx vite dev --config apps/webui/vite.config.ts
```

Open `http://127.0.0.1:5175`. Login still goes through the `/api` and `/login` proxies.

## Ports

| Source | Port | Browser URL |
| --- | --- | --- |
| Default RPC / one-shot | 9100 | `http://127.0.0.1:9100` |
| `server:dev:webui` | 3100 | `http://127.0.0.1:3100` |
| Vite HMR | 5175 | `http://127.0.0.1:5175` |

## Environment

| Variable | Required | Notes |
| --- | --- | --- |
| `MKAGENT_SERVER_TOKEN` | yes | RPC bearer and JWT signing secret. Min 16 chars. |
| `MKAGENT_WEBUI_DIR` | one-shot | Path to `apps/webui/dist`. Not needed for Vite-only mode. |
| `MKAGENT_BUNDLED_ASSETS_ROOT` | local dev | Point at `apps/electron` so the server finds bundled resources. |
| `MKAGENT_RPC_HOST` | no | Default `127.0.0.1`. Non-localhost bind without TLS is refused unless `--allow-insecure-bind`. |
| `MKAGENT_RPC_PORT` | no | Default `9100`. The Vite proxy must match. |
| `MKAGENT_WEBUI_PASSWORD` | no | Login password; falls back to the server token. |
| `MKAGENT_DEBUG` | no | Extra logs. |
| `CONFIG_DIR` | no | Isolate `~/.mkagent`. |

The full environment table is in [development.md](./development.md).

## Pitfalls

- `bun run server:prod` without `MKAGENT_SERVER_TOKEN` exits: `Server token is required`.
- Missing `apps/webui/dist` → RPC comes up, no `MKAGENT_WEBUI_URL`, one-shot browser has nothing to load.
- `bun run webui:dev` fails on Windows (`lsof`).
- Mixing `server:dev:webui` (3100) with Vite's default proxy (9100) makes login look like the server is down.
- A stale `~/.mkagent/.server.lock` can block a second start. Stop the other server, or isolate with `CONFIG_DIR`.
- Login is rate-limited to 5 attempts / 60s per client IP.

## Related

- [development.md](./development.md) — environment variables and workspace commands
- [architecture.md](./architecture.md) — auth handshake
- [cli.md](./cli.md) — the same server over WebSocket
