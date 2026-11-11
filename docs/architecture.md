# Architecture

## Overview

```
                          ┌──────────────────────────────────────────┐
                          │            tavily-proxy (:9100)          │
                          │                                          │
  MCP clients ──────────► │  /mcp   (Streamable HTTP, 2025-11-25)   │
  (x-api-key header)      │  /sse   (legacy SSE, 2024-11-05)       │
                          │          │                               │
  REST clients ─────────► │  /search /extract /crawl /map /research │
  (api_key in body)       │          │                               │
                          │     ┌────▼─────┐                        │
                          │     │ Auth     │                        │
                          │     │ check    │                        │
                          │     └────┬─────┘                        │
                          │          │                               │
                          │     ┌────▼─────┐    ┌──────────────┐   │
                          │     │ Key Pool │◄──►│ key-store.json│   │
                          │     │ (in-mem) │    │ (file-backed) │   │
                          │     └────┬─────┘    └──────────────┘   │
                          │          │                               │
                          │     pick best key                        │
                          │          │                               │
                          │     ┌────▼──────────────────────────┐   │
                          │     │  Tavily API (api.tavily.com)  │   │
                          │     │  Bearer <pool-key>             │   │
                          │     └───────────────────────────────┘   │
                          └──────────────────────────────────────────┘
```

The proxy has three layers:

1. **Auth layer** — verifies the caller's management key (different from the Tavily API keys in the pool)
2. **Key pool** — selects the best Tavily API key for each request, handles rotation and rate limiting
3. **Upstream client** — forwards the request to `api.tavily.com` using the selected pool key

## Authentication Model

There are **two distinct keys**:

| Key | Purpose | Where used |
|-----|---------|------------|
| **Management key** (`authKey`) | Authenticates *callers* (you, your agents) to the proxy | `x-api-key` header (MCP/management) or `api_key` body field (REST) |
| **Pool keys** (`tvly-*`) | Authenticates the *proxy* to Tavily's API | Injected automatically by the key pool — callers never see these |

The management key is compared with **timing-safe equality** to prevent timing attacks.

### Auth paths

```
MCP tools (/mcp, /sse)         → x-api-key header → timing-safe compare
REST endpoints (/search, etc.) → api_key in body OR Authorization: Bearer OR x-api-key header
Key management (/api/keys)     → x-api-key header only
Health check (GET /)           → no auth required
Discovery (/.well-known/mcp)   → no auth required
```

## Key Pool

The key pool (`LocalKVStore` in `key-pool.ts`) is a file-backed JSON store with an in-memory cache, protected by an async mutex for concurrency safety.

### Key Selection Algorithm

`pickBestKey()` runs under the mutex and follows this logic:

1. **Filter** — exclude keys where:
   - `remainingCredit <= 0`
   - `enabled === false`
   - `disabledUntil` is in the future (temporarily disabled)
2. **Sort** — descending by `remainingCredit` (highest first)
3. **Rate-limit check** — walk the sorted list looking for a key whose rolling 60 s request count is below `TAVILY_KEY_RPM` (default 100) and whose last request was at least 100 ms ago
4. **Atomic soft-lock** — stamp `lastRequestTime = Date.now()` on the chosen key *before* returning; the actual upstream request is added to the rolling window by `recordRequestStart()`
5. **Fallback wait** — if all keys are rate-limited, wait until the best key's burst interval ends or its oldest rolling-window request expires (capped at 10 s), then stamp and return it

The legacy stored `qps` field no longer controls request spacing.

If no key is available at all (empty pool, all expired, or wait exceeds 10 s), the request fails with a clear error message.

### Credit Tracking

The proxy calls Tavily's `GET /usage` endpoint for each enabled key in a periodic background task. The first sync begins 15 seconds after startup, followed by a sync every `TAVILY_SYNC_INTERVAL_MS` (default 5 minutes), with keys queried sequentially 1.5 seconds apart. Adding a key synchronises its balance immediately:

```
remainingCredit = max(0, key.limit - key.usage)
```

If `key.limit` is `null` (unlimited key), it falls back to `account.plan_limit`.

Successful business requests do not each trigger a separate `/usage` call. The next `pickBestKey()` uses the latest synced balance; if a key's cached balance reaches zero, it is filtered out.

### Error Handling

| Error type | Response | Key effect |
|------------|----------|------------|
| **401/403 from Tavily** | Immediate retry with next key | Credit set to `-1000` (effectively dead) |
| **Other errors** (429, 500, network) | Error returned to caller | `consecutiveErrors` incremented |
| **3 consecutive non-auth errors** | Key temporarily disabled | `disabledUntil` = now + 5 minutes |
| **Successful request** | Normal response | `consecutiveErrors` reset to 0, `disabledUntil` cleared |

### Key Fallback (MCP)

For MCP tool calls, `withKeyFallback()` automatically retries with the next available pool key on 401/403 errors. It tracks tried keys in a `Set` to avoid infinite loops:

```
key A → 401 → invalidate, try next
key B → 401 → invalidate, try next
key C → success → return result
```

If all keys are exhausted, it returns an MCP error response (`isError: true`).

### Key Fallback (REST)

REST endpoints use `withKeyFallbackRaw()` — same logic, but throws on non-auth errors instead of returning an error object (the caller gets an HTTP 500 with the error message).

### Runtime State File

The runtime state (`TAVILY_STORE_PATH`) is a JSON file with this structure:

```json
{
  "version": "1.1",
  "lastSync": "2025-04-29T10:30:00.000Z",
  "keys": {
    "tvly-xxxxxxxxxxxx": {
      "credit": 850,
      "lastUsed": "2025-04-29T10:30:00.000Z",
      "enabled": true,
      "qps": 1,
      "lastRequestTime": 1745920200000,
      "consecutiveErrors": 0,
      "disabledUntil": null
    }
  }
}
```

Persistence is atomic (a unique temporary file per write, then `rename`), with serialised writes. In-memory mutations take effect immediately; most writes are debounced to a 30-second flush. `flush()` writes dirty state immediately, and `close()` stops the timer and flushes pending state during graceful shutdown. The runtime state includes rolling `requestTimes` entries in addition to the fields shown above.

## MCP Transports

The proxy supports two MCP transport protocols simultaneously:

### Streamable HTTP (`/mcp`)

- **Protocol version**: `2025-11-25`
- **Mode**: Stateless — a new `McpServer` + `WebStandardStreamableHTTPServerTransport` is created per request (no session management)
- **Clients**: OpenCode, and any client using the Streamable HTTP transport
- **Accept header fix**: the proxy injects `Accept: application/json, text/event-stream` if the client omits it, working around a common MCP client issue

### Legacy SSE (`/sse`)

- **Protocol version**: `2024-11-05`
- **Endpoints**: `GET /sse` (connect) + `POST /sse/message?sessionId=xxx` (send)
- **Session management**: transports stored in an in-memory `Map<string, SSEServerTransport>`
- **Cleanup**: sessions are removed on `abort` signal (client disconnect)
- **External path**: when `TAVILY_EXTERNAL_PATH` is set, the SSE message endpoint is prefixed (e.g., `/tavily/sse/message` instead of `/sse/message`) for reverse proxy setups

### MCP Discovery

`GET /.well-known/mcp` returns a JSON document advertising both transports, capabilities, and auth requirements — following the MCP server discovery spec.

## Research (Async Flow)

The research endpoint is fundamentally different from the other four tools because Tavily's research API is **asynchronous**:

### MCP path (blocking)

When called as an MCP tool, the proxy handles the full lifecycle internally:

```
1. POST /research → get request_id
2. Poll GET /research/{request_id} with exponential backoff
   - Initial: poll_interval seconds (default 2)
   - Backoff: ×1.5 per poll, capped at 10 s
   - Timeout: 300 s (mini) or 900 s (pro/auto)
3. Return completed/failed result as MCP tool response
```

The key used to initiate the research is tracked internally. A second key cannot poll that research request; if no reservation is found, REST polling selects an available pool key.

### REST path (async)

When called via REST, the proxy returns the `request_id` immediately and the caller polls manually:

```
1. POST /research → { request_id, status: "pending", ... }
   (key is stored in researchKeyMap)
2. GET /research/{requestId} → poll until completed/failed
   (uses the stored key; falls back to best pool key if not found)
3. On completion or failure: cleanup from researchKeyMap; credit balances are refreshed by periodic sync
```

This matches Tavily's native async API pattern.

## Startup Sequence

```
1. Read TAVILY_AUTH_KEY from env (or authKey from key store JSON)
2. Create LocalKVStore at TAVILY_STORE_PATH
3. Load initial keys from TAVILY_KEY_STORE_PATH (if set):
   - Parse JSON, iterate keys array
   - Call kv.put(key, credit) for each entry
4. Create Hono app with auth middleware + routes
5. Start HTTP server on TAVILY_PORT
```

If `TAVILY_KEY_STORE_PATH` is set but the file is unreadable, the process exits with a fatal error. If it's unset, the pool starts empty (keys can be added via the management API or CLI).
