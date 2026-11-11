# Tavily Proxy 中文说明

**[English](README.md)** · **简体中文**

自托管 Tavily 代理：在客户端与 [Tavily API](https://tavily.com/) 之间提供共享密钥池、自动轮换、单密钥限流，以及 MCP/REST 双协议接入。

## 致敬原作者

本项目最初基于 **[StarsInDmajor/tavily-proxy](https://github.com/StarsInDmajor/tavily-proxy)** 创建。

**感谢原作者 StarsInDmajor 设计并开源这一项目。** 上游实现的 MCP/REST 代理和密钥池，是本分支后续开发的基础。

本分支现由 **[Ryker0716](https://github.com/Ryker0716)** 独立维护，保留原作者的版权声明，并继续采用 **[MIT License](LICENSE)**。当前维护方向独立于上游，不代表上游官方版本。

本分支改动相关的问题请提交至 [Ryker0716/tavily-proxy/issues](https://github.com/Ryker0716/tavily-proxy/issues)；上游项目的问题请回到[原作者仓库](https://github.com/StarsInDmajor/tavily-proxy)反馈。

## 主要功能

- **多密钥池**：登记多枚 Tavily API 密钥，在可用密钥中优先选择剩余额度最高者
- **自动轮换**：401/403 立即废弃对应密钥；连续 3 次失败后暂停使用该密钥 5 分钟
- **单密钥限流**：默认滚动 60 秒最多 100 次请求，并保留 100 ms 突发请求间隔；最长自动等待 10 秒
- **双 MCP 传输**：现代客户端走 Streamable HTTP（`/mcp`），旧客户端走 SSE（`/sse`）
- **Tavily 兼容 REST 接口**：搜索、网页提取、爬取、站点地图、深度研究
- **周期性额度同步**：启动后首次同步，随后默认每 5 分钟调用 `/usage` 更新密钥余额
- **减少磁盘写入**：请求状态先在内存中实时生效，默认每 30 秒批量落盘；采用原子写入，正常退出时刷新待写状态

### 本分支主要增强

- REST 接口支持 `api_key`、`Authorization: Bearer`、`x-api-key` 三种认证方式，包括研究任务轮询接口
- 将每次请求后的额度同步改为后台周期同步，避免 `/usage` 高频触发 429
- 用滚动 60 秒 RPM 预算代替固定 QPS 间距
- 串行、批量刷新密钥状态文件，减少磁盘读写，并在额度更新时保留请求计数

> [!NOTE]
> 本仓库不是 Tavily 官方产品。调用仍消耗各上游 Tavily 密钥的额度；增加密钥不会获得 Tavily 的免费额度，也不应被用来绕过平台限制。

## 快速开始

### 1. 准备密钥文件

以下配置保存为 `keys.json`，**替换其中所有示例管理密钥和 Tavily 密钥**。

```json
{
  "authKey": "your-secret-management-key",
  "keys": [
    { "key": "tvly-xxxxxxxxxxxx", "credit": 1000 },
    { "key": "tvly-yyyyyyyyyyyy", "credit": 1000 }
  ]
}
```

两个「key」用途完全不同，不要混用：

| 密钥 | 用途 | 谁能看到 |
|---|---|---|
| **管理密钥 `authKey`** | 验证 OpenCode、CLI 等客户端是否有权使用本代理 | 由客户端发送；请使用独立生成的随机密钥 |
| **池内 Tavily 密钥 `tvly-*`** | 代理向 Tavily 发起真实请求 | 由代理自动选择并注入；不要下发给客户端 |

**不要把真实密钥、密钥文件或含密钥的截图提交到 GitHub。** 建议将实际保存位置放在仓库之外，并设置仅当前用户可读。

### 2. 从源码启动

要求 **Node.js 18 或更高版本**。

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

默认服务地址：`http://localhost:9100`。

- `GET /` 是无需认证的健康检查。
- **请显式设置 `TAVILY_STORE_PATH`**。源码编译版遗留的内置默认值指向原作者主目录，不应直接照搬。
- `TAVILY_STORE_PATH` 是运行时状态文件，`TAVILY_KEY_STORE_PATH` 是启动时读取的初始密钥文件，两者用途不同。管理 API 新增的密钥只写入运行时状态文件，不会自动回写初始 JSON。

Nix/Home Manager（**x86_64 Linux**）部署见[部署文档](docs/deployment.md)。

## 调用方式

### MCP：OpenCode、Claude Code 等

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

对外提供五个工具：`tavily-search`、`tavily-extract`、`tavily-crawl`、`tavily-map`、`tavily-research`。参数参照[官方 Tavily API](https://docs.tavily.com/documentation/api-reference)。

- **`/mcp`**：Streamable HTTP，协议版本 `2025-11-25`
- **`/sse` + `/sse/message`**：旧版 SSE，协议版本 `2024-11-05`

填在 `x-api-key` 里的是**管理密钥**，不是池内的 `tvly-*` 密钥。

### REST 接口

REST 请求支持以下三种认证方式，包括 `GET /research/:requestId` 轮询。

```bash
# 方式一：请求体中的 api_key
curl -X POST http://localhost:9100/search \
  -H "Content-Type: application/json" \
  -d '{"api_key": "your-secret-management-key", "query": "test"}'

# 方式二：Bearer 认证（tvly CLI 使用此形式）
curl -X POST http://localhost:9100/extract \
  -H "Authorization: Bearer your-secret-management-key" \
  -H "Content-Type: application/json" \
  -d '{"urls": ["https://example.com"]}'
```

方式三：请求头 `x-api-key: your-secret-management-key`。

代理会先验证调用方的管理密钥，再移除请求体里的 `api_key`，自动选择池内密钥转发给 Tavily。

### 官方 `tvly` CLI

[官方 tavily-cli](https://github.com/tavily-ai/tavily-cli) **0.1.8 及以上版本**支持自定义 API 地址：

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

- `api_key` 必须填本代理的**管理密钥**，不是池内的 Tavily 密钥。
- `api_base_url` 必须指向本代理；不设这一项，CLI 会直接调用 Tavily，无法使用本地密钥池。
- `TAVILY_API_BASE_URL` 环境变量会覆盖配置文件中的地址。

### 密钥管理

Nix/Home Manager 安装提供的 `tavily-cli`：

```bash
tavily-cli status          # 服务状态与密钥列表
tavily-cli add tvly-xxx    # 添加密钥并立即同步余额
tavily-cli remove tvly-xxx # 删除密钥
tavily-cli list            # 查询剩余额度
tavily-cli health          # 健康检查
```

或直接调用管理 API（使用管理密钥）：

```bash
# 查看密钥池；返回内容会遮蔽完整 API 密钥
curl http://localhost:9100/api/keys \
  -H "x-api-key: your-secret-management-key"

# 添加密钥，立即向 Tavily 同步真实额度
curl -X POST http://localhost:9100/api/keys \
  -H "x-api-key: your-secret-management-key" \
  -H "Content-Type: application/json" \
  -d '{"apiKey": "tvly-xxxxxxxxxxxx"}'
```

更多删除与研究轮询请求见 [API Reference](docs/api.md)。

## 配置项

| 环境变量 | 默认值 | 用途 |
|---|---|---|
| `TAVILY_PORT` | `9100` | HTTP 监听端口 |
| `TAVILY_STORE_PATH` | 遗留源码路径 | **请显式设置**；运行时密钥池状态文件 |
| `TAVILY_KEY_STORE_PATH` | 不设置 | 启动时读取的初始密钥 JSON |
| `TAVILY_AUTH_KEY` | 回退至初始 JSON 的 `authKey` | 管理密钥；两处至少配置一处 |
| `TAVILY_KEY_RPM` | `100` | 每枚密钥在滚动 60 秒内的请求上限 |
| `TAVILY_SYNC_INTERVAL_MS` | `300000` | 额度同步间隔，单位毫秒 |
| `TAVILY_EXTERNAL_PATH` | 不设置 | 路径反代时 SSE 消息端点的前缀，如 `/tavily` |

额度同步在服务启动 15 秒后执行第一次，此后按上述间隔顺序查询各密钥。请求状态内存在内存中立即生效，待写状态默认每 30 秒落盘；新增密钥和收到 `SIGTERM`/`SIGINT` 时会立即刷新。

## 深度文档

| 语言 | 文档 | 内容 |
|---|---|---|
| 中文 | [本说明](README.zh-CN.md) | 致谢、功能、部署、调用与配置 |
| English | [架构说明](docs/architecture.md) | 密钥选择、认证、持久化、MCP 传输与研究轮询 |
| English | [API 参考](docs/api.md) | REST、MCP 工具、密钥管理接口 |
| English | [部署指南](docs/deployment.md) | Home Manager、环境变量、反向代理 |
| English | [开发指南](docs/development.md) | 项目结构、测试、构建与 Nix 集成 |

目前完整中文指南集中在本文，细分文档以英文维护，避免四份译文与实现脱节。

## 额度消耗（Tavily 上游）

| 操作 | 消耗额度 |
|---|---|
| 搜索 `basic` | 1 点 |
| 搜索 `advanced` | 2 点 |
| 提取 `basic` | 每 5 个 URL 1 点 |
| 提取 `advanced` | 每 5 个 URL 2 点 |
| Crawl | 每次 2 点 |
| Map | 每 10 页 1 点（带 `instructions` 为 2 点） |
| Research `mini` | 4–20 点 |
| Research `pro` | 20–250 点 |

上游价格和限额可能调整，以 Tavily 官方最新文档为准。

## 许可与署名

原始作品的版权归 **StarsInDmajor** 所有。本分支在修改文件中保留该声明，并继续遵循 **[MIT License](LICENSE)**。完整版权与许可声明见 [LICENSE](LICENSE)。

**再次感谢原作者开放源代码，让本分支能够继续演进。**
