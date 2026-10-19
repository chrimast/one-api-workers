<div align="center">
<h1>One API on Workers</h1>

A distributed, low-latency, high-performance AI gateway on Cloudflare Workers, with channel management, load balancing, and usage analytics.
</div>

<p align="right">
  <b>English</b> | <a href="./README.zhCN.md">中文</a>
</p>

<div align="right">
Based on <a href="https://github.com/dreamhunter2333/awsl-one-api" target="_blank">dreamhunter2333/awsl-one-api</a>, further developed with AI assistance.
</div>

## Overview

- Built on Cloudflare Workers: global edge network, low latency, high performance
- Unified proxy entry: `/v1/chat/completions`, `/v1/messages`, `/v1/responses`, `/v1/audio/speech`, `/v1/models`
- Weighted load balancing: multi-key per channel, retry, key rotation, and cross-channel fallback
- Quotas and billing: token-level quotas, global model pricing, channel-level model pricing
- Observability: writes to Cloudflare Analytics Engine; admin UI covers overview, trends, distributions, and usage log search
- Admin console: React + Vite UI for channels, tokens, pricing, API testing, and system settings
- Admin security: admin-token login by default; optional Telegram 2FA; rate limits and session cookies on the login path
- API docs: Chanfana-powered Swagger, ReDoc, and OpenAPI JSON, toggleable in system settings

## Supported Routes

### Proxy APIs

| Route | Description | Channel types |
| --- | --- | --- |
| `/v1/chat/completions` | OpenAI Chat Completions–compatible proxy | `openai`, `azure-openai`, `gemini` |
| `/v1/messages` | Anthropic Claude Messages–compatible proxy | `claude`, `claude-to-openai` |
| `/v1/responses` | OpenAI / Azure Responses proxy | `openai-responses`, `azure-openai-responses` |
| `/v1/audio/speech` | TTS speech generation proxy | `openai-audio`, `azure-openai-audio` |
| `/v1/models` | Model list filtered by token permissions and quota | Returned dynamically from channels the token can access |

## Deployment

### One-click deploy

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/chrimast/one-api-workers)

如果 Cloudflare 向导提示项目名称错误或返回 HTTP 400，请先在向导里把默认的 Worker/项目名改成你账号下唯一的名称，例如 `one-api-workers-keles`。Cloudflare 会读取 `wrangler.jsonc` 里的默认 `name` 和 D1 `database_name`，当前模板默认值都是 `one-api-workers`；如果你的账号里已经有同名 Worker、Pages 项目或 D1 数据库，创建阶段会失败。

### Manual deploy

```bash
创建 D1 数据库，获取数据库 Name、ID
将上述数据更新入 wrangler.jsonc，并把 name / d1_databases[].database_name 改成你账号下唯一的名称

# Publish the Worker
bun run deploy

# 首次部署成功后再设置管理员 Secret
wrangler secret put ADMIN_TOKEN
```

### Admin console

Current admin pages:

- `Dashboard`: total requests, success rate, cost, latency, and token/channel/model/provider distributions
- `Usage Logs`: filter detailed logs by time range, dimensions, keywords, and result
- `Channels`: channel config, weights, auto retry/rotation, model mapping, pull upstream model lists
- `Tokens`: API token management, channel access scope, quotas, usage reset
- `Pricing`: global model pricing; per-token and per-request billing
- `API Test`: test `/v1/chat/completions`, `/v1/messages`, `/v1/responses`, `/v1/audio/speech` from the console
- `System Settings`: Telegram admin verification, currency display precision, API docs toggle

## Project Structure

```text
one-api-workers/
├── src/
│   ├── admin/                    # Admin APIs: auth / channel / token / pricing / analytics / system
│   ├── analytics/                # Analytics Engine write and query
│   ├── db/                       # D1 init and migrations
│   ├── providers/                # Upstream provider proxies
│   ├── billing.ts                # Billing and amount precision
│   ├── channel-config.ts         # Channel config normalization
│   ├── system-config.ts          # System config and Telegram security settings
│   └── index.ts                  # Worker entry
├── frontend/
│   ├── src/pages/                # Dashboard / Channels / Tokens / Pricing / Usage Logs / Settings
│   ├── src/components/           # Layout, charts, UI components
│   └── package.json              # Frontend build and lint
├── public/                       # Frontend build output, served by the Worker
├── docs/                         # Usage and security docs
├── tests/                        # Mock upstream + local E2E scripts
├── wrangler.jsonc                # Production config
├── wrangler.local.jsonc          # Local Worker config
├── type.d.ts                     # Worker bindings and shared types
└── package.json                  # Root workspace and dev scripts
```

## Quick Start

### Requirements

- Bun 1.3+
- Cloudflare account with Workers + D1 + Analytics Engine (`usage_events_by_token`)

### Install dependencies

```bash
bun install
```

### Configure Cloudflare bindings

`wrangler.jsonc` / `wrangler.local.jsonc` already include the required binding shape; replace them with your own values:

- `d1_databases[].database_name` / `database_id`：替换为自己的 D1
- `analytics_engine_datasets[].dataset`：可选，默认一键部署配置未启用；需要用量分析时手动创建 Analytics Engine 数据集后再添加绑定
- `vars.FRONTEND_DEV_SERVER_URL`：仅本地联调时使用，默认 `http://127.0.0.1:5173`
- `assets`：保持 `public/` 与 `ASSETS` 绑定即可

如果需要启用用量分析，可以在 Cloudflare 创建 Analytics Engine 数据集后，把下面配置加回 `wrangler.jsonc`：

```jsonc
"vars": {
  "USAGE_ANALYTICS_DATASET": "usage_events_by_token"
},
"analytics_engine_datasets": [
  {
    "binding": "USAGE_ANALYTICS",
    "dataset": "usage_events_by_token"
  }
]
```

当前配置中的关键 secrets：

- `ADMIN_TOKEN`：管理员登录令牌，首次部署成功后设置
- `CF_API_TOKEN`：可选，用于查询 Analytics Engine SQL，启用后台分析看板和用量日志时再设置
- `CF_ACCOUNT_ID`：可选，与 `CF_API_TOKEN` 配套，用于 Cloudflare Analytics 查询

Example:

```bash
# 必需：后台登录
wrangler secret put ADMIN_TOKEN

# 可选：启用用量分析后再设置
wrangler secret put CF_API_TOKEN
wrangler secret put CF_ACCOUNT_ID
```

首次远程构建部署时不要在 `wrangler.jsonc` 中配置 `secrets.required`，否则 Cloudflare Builds 会在项目创建前检查远程 Secret 并失败。部署完成后可以在 Cloudflare 控制台的 Worker 设置里添加 Secret，或在本机登录 Wrangler 后执行上面的命令。

本地开发可以用 `.dev.vars` 提供这些值；`tests/` 下的脚本也会优先读取这个文件。

### Database init and migrations

D1 schema init and migrations run automatically on the first request. No manual SQL is required.

To trigger init after deploy (with admin auth):

```text
POST /api/admin/db_initialize
```

### Local development

Start frontend + Worker together:

```bash
bun run dev
```

Common URLs:

- Vite frontend: `http://127.0.0.1:5173`
- Local Worker: `http://127.0.0.1:8787`

Other useful commands:

```bash
bun run dev:worker
bun run build
bun run cf-typegen
cd frontend && bun run lint
```

## Documentation

- [Usage and configuration](docs/usage-and-configuration.md) ([中文](docs/usage-and-configuration.zhCN.md))
- [Admin auth and protection](docs/security/admin-auth-protection.md) ([中文](docs/security/admin-auth-protection.zhCN.md))
- [Testing and verification](docs/testing.md) ([中文](docs/testing.zhCN.md))

## Contributing

Issues and pull requests are welcome.

## License

MIT License

Credits: <a href="https://github.com/dreamhunter2333/awsl-one-api" target="_blank">dreamhunter2333/awsl-one-api</a>

## Support

如有问题或建议，请创建 Issue 或联系维护者。

## GitHub Actions 部署

本项目使用 GitHub Actions 构建并部署到 Cloudflare Workers，不依赖 Cloudflare Workers Builds 的失效 Build Token。

在 GitHub 仓库 `Settings → Secrets and variables → Actions` 中添加以下 Repository secrets：

- `CLOUDFLARE_API_TOKEN`：Cloudflare 用户 API Token，至少需要 Workers Scripts Edit；如果部署涉及 D1，请同时授予 D1 Edit。
- `CLOUDFLARE_ACCOUNT_ID`：Cloudflare Account ID。

配置完成后，推送到 `main` 会自动执行：

```text
安装 Bun 1.3.11 → 安装依赖 → 构建前端 → Wrangler 部署
```

也可以在仓库的 `Actions → Deploy to Cloudflare Workers → Run workflow` 中手动重新部署。

原有 Cloudflare Workers Builds 可以关闭或断开，避免与 GitHub Actions 重复部署。
