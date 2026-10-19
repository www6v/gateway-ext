# 03 — MCP 网关能力

> 对照第 9 章 §9.3 MCP Gateway 的治理语义逐项分析 Higress 的实现覆盖度。

## 1. MCP 协议支持

### 1.1 协议版本基线

第 9 章要求以 MCP `2026-07-28` 版本为基线。Higress 的 `mcp-server` 插件完整支持：

| Profile | 精确版本 | 生命周期与传输 |
|---------|---------|---------------|
| **legacy** | `2024-11-05`、`2025-03-26`、`2025-06-18` | 保留 `initialize` / `notifications/initialized`、session 与 HTTP/SSE 兼容行为 |
| **modern** | `2026-07-28` | 每请求 `_meta`、无 initialize、无协议 session；不提供 GET/DELETE/Last-Event-ID 恢复 |

Modern profile 实现的方法：
- `server/discover` — 声明当前可用的 tools
- `tools/list` — 列出可用工具
- `tools/call` — 调用工具

**源码位置**：`plugins/wasm-go/extensions/mcp-server/` 及 `plugins/wasm-go/pkg/mcp/`

### 1.2 逐请求元数据

Modern RPC 请求必须携带的元数据，Higress 的验证行为：

| 元数据 | 位置 | Higress 验证 |
|--------|------|-------------|
| `protocolVersion` | `params._meta.io.modelcontextprotocol/protocolVersion` | ✅ 校验 |
| `clientCapabilities` | `params._meta.io.modelcontextprotocol/clientCapabilities` | ✅ 校验（可为空） |
| `clientInfo` | `params._meta.io.modelcontextprotocol/clientInfo` | ✅ 推荐携带 |

HTTP 头镜像：

| Header | Higress 行为 |
|--------|-------------|
| `MCP-Protocol-Version` | ✅ 校验与正文协议版本一致 |
| `Mcp-Method` | ✅ 校验与 JSON-RPC 方法一致 |
| `Mcp-Name` | ✅ 对具名请求必需 |

### 1.3 内容校验与安全约束

Modern profile 的 Higress 校验清单：
- ✅ Content-Type 校验
- ✅ Accept 校验（`application/json, text/event-stream`）
- ✅ 同源 Origin 校验
- ✅ JSON-RPC 单消息边界（批量请求被拒绝）
- ✅ 身份镜像头校验
- ✅ 资源上限检查
- ✅ Response envelope 和 trailing JSON 被拒绝

## 2. MCP 路由 — `mcp-router`

### 2.1 统一端点聚合

`mcp-router` 实现 MCP 工具调用的路由聚合：

```
Client → Higress (统一入口) → mcp-router 插件
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              MCP Server A    MCP Server B    MCP Server C
```

**路由机制**：
1. 解析 `tools/call` 请求中的 `params.name`
2. 检查是否匹配 `server-name/tool-name` 格式
3. 从 `servers` 配置中查找目标后端（domain + path）
4. 修改 `:authority` 和 `:path` 请求头
5. 从 tool name 中移除 `server-name/` 前缀
6. 重新进入网关路由引擎，转发到目标后端

**配置示例**：
```yaml
servers:
- name: random-user-server
  domain: mcp.example.com
  path: /mcp-servers/mcp-random-user-server
- name: rest-amap-server
  domain: mcp.example.com
  path: /mcp-servers/mpc-rest-amap-server
```

### 2.2 MCP Server 托管 — `mcp-server`

Higress 可以在网关侧直接托管 MCP 工具服务：
- 内置示例：夸克搜索（quark-search）、高德地图（amap-tools）
- 通过 `mcp.LoadMCPServer()` 框架注册自定义工具
- 客户端通过 MCP 协议经 Higress 统一入口调用

**价值**：
- 统一的认证授权
- 细粒度的限流防滥用
- 完整的审计日志
- 丰富的可观测能力
- 无中断动态更新

## 3. 安全边界

### 3.1 凭证管理

第 9 章要求"凭证不下发"。Higress 的处理：

| 场景 | Higress 行为 |
|------|-------------|
| `Authorization` 头 | 仅按显式 proxy auth policy 生成或转发 |
| `Cookie` | 默认不转发 |
| 下游 session | 默认不转发 |
| `Last-Event-ID` | 默认不转发 |
| 内部路由头 | 默认不转发 |
| `Mcp-Param-*` | 仅在 modern→modern 的当前 Tool RPC 中透传，不进入 discover/initialize/legacy RPC |

### 3.2 策略执行点条件

第 9 章指出网关成为不可绕过策略执行点的条件：

| 条件 | Higress 状态 |
|------|-------------|
| 凭证不下发 | ✅ proxy auth policy 控制 |
| 出口网络受控 | ⚠️ 依赖 K8s NetworkPolicy |
| 服务发现和域名解析收敛 | ✅ McpBridge CRD 统一管理 |
| 直连路径被禁止 | ⚠️ 需额外网络策略配合 |
| 旁路受到审计 | ✅ 网关侧审计日志 |

### 3.3 权限检查

第 9 章要求权限应取"调用者原有授权、工作负载授权、任务委托范围和资源侧限制的交集"。

Higress 的权限检查能力：
- **认证**：`key-auth`、`jwt-auth`、`basic-auth`、`hmac-auth`、`oidc` 插件
- **授权**：`ext-auth`（外部授权）、`opa`（策略引擎）
- **工具级鉴权**：`mcp-server` 插件在工具调用时可进行鉴权

**局限**：
- `tools/list` 过滤可减少误选，但 `tools/call` 仍必须再次鉴权 → Higress 的 `mcp-server` 支持此模式
- 任务委托范围的交集计算需要应用层配合

## 4. REST-to-MCP 桥接

第 9 章要求适配层定义工具描述、请求构造和结果裁剪。

Higress 的实现方式：

| 能力 | 实现 |
|------|------|
| OpenAPI → MCP 转换 | 独立工具 `openapi-to-mcpserver`（`github.com/higress-group/openapi-to-mcpserver`） |
| 工具描述和输入 Schema | 由转换工具生成 |
| 请求构造 | 转换工具处理 REST → MCP 的参数映射 |
| 结果裁剪 | 可在 `mcp-server` 插件中自定义 |
| Http2Rpc CRD | 内置 HTTP → Dubbo/gRPC 桥接（非 MCP 场景） |

## 5. Proxy Profile 矩阵

`mcp-server` 插件支持灵活的下游/上游协议组合：

| 下游 | 上游 | 状态 |
|------|------|------|
| modern | registered / REST / composed | ✅ 支持 |
| modern | `protocolStrategy: modern` | ✅ 每请求无状态转发 |
| modern | `protocolStrategy: legacy` | ✅ 单次下游交换内执行隔离的 legacy handshake |
| modern | `auto` + HTTP | ✅ 每次转发型 Tool 请求先 discover，再选择 modern 或完成一次 legacy 握手 |
| legacy | `auto` + HTTP | ✅ 原 legacy 路径，不探测 modern |
| 任意已支持下游 | `auto` + SSE | ✅ 原 legacy SSE 路径 |
| legacy | modern-only upstream | ❌ 不支持 |

## 6. MCP 相关的控制面支持

### 6.1 McpBridge CRD

`McpBridge`（`networking.higress.io/v1`）用于注册中心桥接：
- 控制器位于 `pkg/ingress/kube/mcpbridge/`
- 驱动 `registry/*` 中的服务发现实现
- 将外部注册中心的服务转化为 Istio ServiceEntry

### 6.2 McpServer CRD

`McpServer`（`networking.higress.io/v1`）用于声明 MCP Server：
- 控制器位于 `pkg/ingress/kube/mcpserver/`
- 生成 MCP 路由配置
- 与 WasmPlugin 联动加载 MCP 插件

### 6.3 MCP 路由生成

`pkg/ingress/mcp/generator.go` 负责将 McpServer 资源翻译为 Envoy 路由配置。

## 7. 小结

| 第 9 章要求 | Higress 覆盖度 | 说明 |
|-------------|---------------|------|
| MCP 2026-07-28 基线 | ✅ 完整 | modern profile 完整支持 |
| Legacy 兼容 | ✅ 完整 | 三个历史版本保留 |
| 逐请求元数据 | ✅ 完整 | `_meta` + HTTP 头校验 |
| 安全边界 | ✅ 完整 | 凭证控制、身份镜像、资源上限 |
| MCP 路由 | ✅ 完整 | `mcp-router` 统一端点聚合 |
| MCP Server 托管 | ✅ 完整 | `mcp-server` 插件内托管 |
| REST-to-MCP 桥接 | ✅ 完整 | `openapi-to-mcpserver` 工具 |
| 权限交集 | ⚠️ 部分 | 认证/授权基础设施完整，任务委托链需应用层配合 |
| 不可绕过策略点 | ⚠️ 部分 | 网关侧完整，需配合网络策略消除直连/旁路 |
