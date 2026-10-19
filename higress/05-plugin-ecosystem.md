# 05 — 插件生态

> Higress 的核心差异化在于其 **Wasm 插件生态**。本文档 catalog 所有内置插件。

## 1. 插件框架

### 1.1 SDK 矩阵

| SDK | 路径 | 插件数 | 语言 |
|-----|------|--------|------|
| **wasm-go** | `plugins/wasm-go/extensions/` | ~55 | Go |
| **wasm-cpp** | `plugins/wasm-cpp/extensions/` | ~12 | C++ |
| **wasm-rust** | `plugins/wasm-rust/` | examples | Rust |
| **wasm-assemblyscript** | `plugins/wasm-assemblyscript/` | examples | TypeScript/AS |
| **golang-filter** | `plugins/golang-filter/` | 2 | Go (原生 Envoy filter) |

### 1.2 分发方式

- Wasm 插件编译为 `.wasm` 文件
- 打包为 **OCI 镜像** 推送到 `higress-registry.cn-hangzhou.cr.aliyuncs.com/plugins/<name>:<version>`
- 通过 `WasmPlugin` CRD 的 `url: oci://...` 引用
- 支持**热更新**——无需重启 Envoy，无流量损失

### 1.3 Wasm-Go SDK 结构

每个插件是独立的 Go module：
```
plugins/wasm-go/extensions/<plugin-name>/
├── main.go          # 插件入口
├── go.mod / go.sum  # 独立 module
├── VERSION          # 版本号
├── README.md        # 中文文档
├── README_EN.md     # 英文文档（部分插件）
├── config/          # 配置结构定义（可选）
├── util/            # 工具函数（可选）
└── main_test.go     # 测试（可选）
```

### 1.4 MCP 插件框架

`plugins/wasm-go/pkg/mcp/` 提供 MCP Server 和 Filter 的通用框架：
```go
// 注册 MCP Server
mcp.LoadMCPServer(mcp.AddMCPServer(...))
mcp.InitMCPServer()

// 注册 MCP Filter
mcp.LoadMCPFilter(name, ...)
mcp.InitMCPFilter()
```

## 2. AI 插件 Catalog

### 2.1 核心代理

| 插件 | 优先级 | 功能 |
|------|--------|------|
| **ai-proxy** | 100 | ★ 旗舰插件，统一代理 66+ LLM 厂商，自动协议转换 |
| **ai-agent** | 200 | ReAct 模式 Agent 编排，支持多轮对话和工具调用 |
| **ai-statistics** | 200 | AI 可观测基础能力（token / 时延 / 模型 指标） |

### 2.2 路由与选择

| 插件 | 优先级 | 功能 |
|------|--------|------|
| **ai-intent** | 700 | LLM 意图识别，判断请求属于哪个领域 |
| **model-router** | 900 | 基于 model 参数 + 内容正则的路由 |
| **model-mapper** | - | 模型名称映射 |
| **ai-endpoint-picker** | - | 推理端点选择（KV cache / queue / prefix cache 感知） |
| **ai-load-balancer** | - | LLM 负载均衡（全局最小请求数 / prefix cache / metrics） |

### 2.3 提示词工程

| 插件 | 优先级 | 功能 |
|------|--------|------|
| **ai-prompt-decorator** | 450 | 在请求前后插入 prompt，支持字面量/正则替换 |
| **ai-prompt-template** | - | 提示词模板管理 |
| **ai-context-limit** | 1000 | 上下文窗口 token 估算与拦截 |
| **ai-history** | - | 多轮对话历史管理 |

### 2.4 缓存与检索

| 插件 | 优先级 | 功能 |
|------|--------|------|
| **ai-cache** | 认证阶段:10 | LLM 结果缓存（字符串匹配 + 语义向量） |
| **ai-rag** | 400 | RAG 检索增强（对接阿里云向量检索） |
| **ai-search** | - | 搜索增强 |

### 2.5 安全与合规

| 插件 | 优先级 | 功能 |
|------|--------|------|
| **ai-security-guard** | 300 | 内容安全检测（请求/响应合规检查） |
| **ai-json-resp** | - | 强制 JSON 格式输出 |
| **qwen3guard** | - | Qwen3 安全护栏 |

### 2.6 转换与处理

| 插件 | 优先级 | 功能 |
|------|--------|------|
| **ai-transformer** | 认证阶段:410 | 通过 LLM 转换请求/响应的 header 和 body |
| **ai-image-reader** | - | 多模态图片处理 |

### 2.7 治理

| 插件 | 优先级 | 功能 |
|------|--------|------|
| **ai-token-ratelimit** | 600 | 基于 token 的速率限制（Redis） |
| **ai-quota** | 750 | 基于 consumer 的配额管理（Redis） |

## 3. MCP 插件

| 插件 | 功能 |
|------|------|
| **mcp-server** | 在网关侧托管 MCP 工具服务（内置夸克搜索、高德地图） |
| **mcp-router** | MCP tools/call 请求路由聚合 |

## 4. 通用安全插件

| 插件 | 功能 |
|------|------|
| **key-auth** | API Key 认证 |
| **jwt-auth** | JWT 认证 |
| **basic-auth** | Basic 认证 |
| **hmac-auth-apisix** | HMAC 认证（APISIX 兼容） |
| **oidc** | OpenID Connect 认证 |
| **ext-auth** | 外部授权服务 |
| **opa** | Open Policy Agent 策略引擎 |
| **waf** | Web 应用防火墙 |
| **ip-restriction** | IP 黑白名单 |
| **replay-protection** | 重放攻击防护 |

## 5. 流量管理插件

| 插件 | 功能 |
|------|------|
| **cors** | 跨域资源共享 |
| **traffic-editor** | 流量编辑 |
| **traffic-tag** | 流量标记 |
| **frontend-gray** | 前端灰度发布 |
| **cluster-key-rate-limit** | 集群级 Key 限流 |
| **response-cache** | 响应缓存 |

## 6. 协议转换插件

| 插件 | 功能 |
|------|------|
| **de-graphql** | GraphQL 网关 |
| **jsonrpc-converter** | JSON-RPC 协议转换 |
| **transformer** | 请求/响应转换 |
| **nginx-rewrite-compatible** | Nginx rewrite 兼容 |
| **gw-error-format** | 网关错误格式统一 |

## 7. 可观测插件

| 插件 | 功能 |
|------|------|
| **log-request-response** | 请求/响应日志记录 |
| **geo-ip** | IP 地理位置 |

## 8. C++ 插件

| 插件 | 功能 |
|------|------|
| **bot-detect** | 机器人检测 |
| **key-auth** (C++) | API Key 认证 |
| **keycloak-authorization** | Keycloak 授权 |
| **model-mapper** (C++) | 模型映射 |
| **model-router** (C++) | 模型路由 |
| **oauth** | OAuth 认证 |
| **request-block** | 请求拦截 |
| **sni-misdirect** | SNI 误指向处理 |
| **waf** (C++) | WAF |

## 9. 插件执行链

多个插件可以挂载到同一路由，按**优先级**顺序执行：

```
请求进入
  ↓
[认证阶段]
  key-auth (识别 consumer)
  ai-cache (认证阶段:10, 查缓存)
  model-router (认证阶段:900, 路由决策)
  ↓
[默认阶段]
  ai-context-limit (1000, 上下文检查)
  ai-proxy (100, 模型代理)
  ai-statistics (200, 可观测)
  ai-prompt-decorator (450, 提示词注入)
  ai-rag (400, RAG 检索)
  ai-security-guard (300, 安全检查)
  ai-intent (700, 意图识别)
  ai-token-ratelimit (600, 限流检查)
  ai-quota (750, 配额检查)
  ↓
响应返回（响应阶段逆序执行）
```
