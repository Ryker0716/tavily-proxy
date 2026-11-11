# Tavily Proxy

**English** · **[简体中文](README.zh-CN.md)**

Self-hosted proxy between your clients and the [Tavily API](https://tavily.com/), with a shared API key pool, automatic rotation, per-key rate limiting, and MCP/REST access.

## Acknowledgements

This project started as a fork of **[tavily-proxy by StarsInDmajor](https://github.com/StarsInDmajor/tavily-proxy)**.

**Thanks to the original author for designing and open-sourcing the project.** The upstream implementation provided the MCP/REST proxy and key-pool foundation on which this fork builds.

**[Ryker0716](https://github.com/Ryker0716)** now maintains this fork independently. It retains the original author's copyright notice and the **[MIT License](LICENSE)**. Development here is independent of upstream rather than an official upstream release.

Please submit issues concerning the changes in this fork to [Ryker0716/tavily-proxy/issues](https://github.com/Ryker0716/tavily-proxy/issues). Upstream issues belong in the [original repository](https://github.com/StarsInDmajor/tavily-proxy).

## Features

- **Multi-key pool** — register multiple Tavily API keys; eligible keys are selected in descending order of remaining credits
- **Transparent rotation** — 401/403 keys are invalidated immediately; 3 consecutive errors trigger a 5-minute cooldown
- **Per-key rate limiting** — a rolling 60-second window (default 100 requests/key) plus a 100 ms burst interval; automatic waits are capped at 10 seconds
- **Dual MCP transports** — Streamable HTTP (`/mcp`) for modern clients and legacy SSE (`/sse`) for older clients
- **Tavily-compatible REST API** — search, extract, crawl, map, and research under compatible paths
- **Periodic credit tracking** — synchronises each key's balance from `/usage` after startup and every 5 minutes by default
- **Reduced disk writes** — in-memory state is updated immediately and persisted periodically (30 seconds), with atomic writes and graceful-shutdown flushing

### Changes in this fork

- Accept REST credentials via `api_key`, `Authorization: Bearer`, or `x-api-key`, including research polling
- Move credit synchronisation from each successful request to a periodic background task
- Replace fixed-interval QPS spacing with rolling per-key RPM accounting
- Debounce serialised state-file writes and preserve request counters during credit updates

## Quick Start

### 1. Prepare a key store

Save the following file as `keys.json`. Replace both example keys with real values and set your own management key.

```json
{
  "authKey": "your-secret-management-key",
  "keys": [
    { "key": "tvly-xxxxxxxxxxxx", "credit": 1000 },
    { "key": "tvly-yyyyyyyyyyyy", "credit": 1000 }
  ]
}
```

There are two different kinds of credential:

| Credential | Purpose | Exposure |
|---|---|---|
| **Management key (`authKey`)** | Authenticates your clients to the proxy | Sent by clients; choose a dedicated secret |
| **Tavily pool key (`tvly-*`)** | Authenticates the proxy to Tavily | Selected and injected by the proxy; never expose it to clients |

Do not put your real management key, Tavily keys, or key-store file in a committed README or a public repository.

### 2. Start the service

Requires **Node.js 18 or newer**.

```bash
git clone https://github.com/Ryker0716/tavily-proxy.git
cd tavily-proxy/service
npm install
npm run build

TAVILY_KEY_STORE_PATH=../keys.json \
TAVILY_STORE_PATH=/absolute/path/to/runtime-keys.json \
TAVILY_AUTH_KEY=your-secret-management-key \
TAVILY_PORT=9100 \
npm start
```

Set `TAVILY_STORE_PATH` explicitly. The compiled service's legacy built-in default points to the original author's home directory. Runtime state is held separately from the initial key file, and keys added through the management API are not written back to the initial key file.

The service listens on `http://localhost:9100` by default. `GET /` provides an unauthenticated health check.

For the Nix/Home Manager deployment on **x86_64 Linux**, see [Deployment](docs/deployment.md).

## Usage

### MCP (OpenCode, Claude Code, etc.)

```json
{
  "mcpServers": {
    "tavily-proxy": {
      "url": "http://localhost:9100/mcp",
      "headers": { "x-api-key": "your-secret-management-key" }
    }
  }
}
```

Five MCP tools are exposed: `tavily-search`, `tavily-extract`, `tavily-crawl`, `tavily-map`, `tavily-research`. Parameters follow the [official Tavily API](https://docs.tavily.com/documentation/api-reference).

- **`/mcp`** — Streamable HTTP, protocol `2025-11-25`
- **`/sse`** and **`/sse/message`** — legacy SSE, protocol `2024-11-05`

### REST API

Use your **management key**, not a pool key. All three credential forms below are accepted by REST endpoints, including `GET /research/:requestId`.

```bash
# 1. api_key in the request body
curl -X POST http://localhost:9100/search \
  -H "Content-Type: application/json" \
  -d '{"api_key": "your-secret-management-key", "query": "test"}'

# 2. Bearer authentication (tavily-cli compatible)
curl -X POST http://localhost:9100/extract \
  -H "Authorization: Bearer your-secret-management-key" \
  -H "Content-Type: application/json" \
  -d '{"urls": ["https://example.com"]}'
```

`x-api-key: your-secret-management-key` is the third accepted credential form. The proxy removes the caller credential from the request body before selecting a pool key and forwarding to Tavily.

### Official `tvly` CLI

[tavily-cli](https://github.com/tavily-ai/tavily-cli) **0.1.8 or newer** supports a custom base URL:

```bash
uv tool install tavily-cli==0.1.8

cat > ~/.tavily/config.json << 'EOF'
{
  "api_key": "<your authKey>",
  "api_base_url": "http://localhost:9100"
}
EOF
chmod 600 ~/.tavily/config.json

tvly auth
tvly search "..." --json
tvly research "..." --model mini
```

The management key must be in `api_key` (not a `tvly-*` pool key), and `api_base_url` must point to this proxy. Without the latter, the CLI calls Tavily directly instead of using this proxy's key pool. `TAVILY_API_BASE_URL` overrides the config-file value.

### Key management

Via the Nix/Home Manager installation:

```bash
tavily-cli status          # Service status + key list
tavily-cli add tvly-xxx    # Add a key and synchronise its credits immediately
tavily-cli remove tvly-xxx # Remove a key
tavily-cli list            # List keys with remaining credits
tavily-cli health          # Health check
```

Or call the management API directly:

```bash
curl http://localhost:9100/api/keys \
  -H "x-api-key: your-secret-management-key"
```

See [API Reference](docs/api.md) for add/delete requests and research polling.

## Configuration

| Variable | Default | Description |
|---|---|---|
| `TAVILY_PORT` | `9100` | HTTP listen port |
| `TAVILY_STORE_PATH` | legacy source path | **Set explicitly**; runtime key-pool state file |
| `TAVILY_KEY_STORE_PATH` | unset | Initial key-store JSON, read at startup |
| `TAVILY_AUTH_KEY` | falls back to `authKey` in the initial key store | Management key; one of these is required |
| `TAVILY_KEY_RPM` | `100` | Requests per key in a rolling 60-second window |
| `TAVILY_SYNC_INTERVAL_MS` | `300000` | Credit-synchronisation interval in milliseconds |
| `TAVILY_EXTERNAL_PATH` | unset | Reverse-proxy prefix for the SSE message endpoint |

Credit sync begins 15 seconds after startup, then runs at the configured interval with keys queried sequentially. Request state is persisted every 30 seconds when dirty; new keys and graceful shutdown are flushed immediately.

## Documentation

| Language | Document | Contents |
|---|---|---|
| 中文 | [中文说明](README.zh-CN.md) | 致谢、本分支增强、部署、调用与配置 |
| English | [Architecture](docs/architecture.md) | Key-pool selection, authentication, persistence, MCP transports |
| English | [API Reference](docs/api.md) | REST endpoints, MCP tools, management API |
| English | [Deployment](docs/deployment.md) | Home Manager, environment variables, reverse proxy |
| English | [Development](docs/development.md) | Project structure, testing, build and Nix integration |

The detailed documents are currently in English. The complete Chinese guide stays in [README.zh-CN.md](README.zh-CN.md) rather than duplicating four translations that can drift.

## Credit Costs (upstream Tavily)

| Operation | Cost |
|-----------|------|
| Search `basic` | 1 credit |
| Search `advanced` | 2 credits |
| Extract `basic` | 1 credit / 5 URLs |
| Extract `advanced` | 2 credits / 5 URLs |
| Crawl | 2 credits / call |
| Map | 1 credit / 10 pages (2 with `instructions`) |
| Research `mini` | 4–20 credits |
| Research `pro` | 20–250 credits |

Costs and Tavily limits may change; consult the official API documentation for current values.

## License and attribution

Copyright for the original work remains with **StarsInDmajor**. Modifications in this fork retain that notice and are distributed under the same **[MIT License](LICENSE)**. See [LICENSE](LICENSE) for the complete copyright and permission notice.

Tavily is a third-party service/API; this fork is not an official Tavily product.
