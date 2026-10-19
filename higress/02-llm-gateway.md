# 02 — LLM 网关能力

> 对照第 9 章 §9.2 LLM Gateway 的治理语义逐项分析 Higress 的实现覆盖度。

## 1. 模型路由策略

第 9 章定义了四种路由方式：**静态映射、成本感知、语义路由、端点选择**。Higress 通过多个插件的组合完整覆盖。

### 1.1 静态映射 — `ai-proxy` + `model-mapper`

| 实现要素 | Higress 做法 |
|----------|-------------|
| 模型别名/版本映射 | `ai-proxy` 的 `modelMapping` 字段，支持精确匹配、前缀匹配(`gpt-3-*`)、通配兜底(`*`)、正则匹配(`~gpt(.*)`) |
| 协议自动转换 | 自动检测 OpenAI `/v1/chat/completions` 与 Claude `/v1/messages` 协议，透明转换为目标厂商的原生协议 |
| 多路径覆盖 | 同一插件支持 text completion、embeddings、image generation、audio speech、rerank |

**源码位置**：`plugins/wasm-go/extensions/ai-proxy/provider/`

### 1.2 语义路由 — `ai-intent` + `model-router`

| 实现要素 | Higress 做法 |
|----------|-------------|
| 意图识别 | `ai-intent` 插件调用 LLM 智能判断用户请求属于哪个领域（金融/电商/法律等），将分类结果写入 `intent_category` 属性供下游插件消费 |
| 内容路由 | `model-router` 插件根据 `messages` 内容正则匹配，将模型名写入 `x-higress-llm-model` 请求头，驱动后续路由 |
| 自动路由 | `model-router` 支持 `autoRouting` 配置，按规则顺序匹配用户消息内容并自动选择模型 |

### 1.3 端点选择 — `ai-endpoint-picker` + `ai-load-balancer`

| 实现要素 | Higress 做法 |
|----------|-------------|
| Endpoint 级调度 | `ai-endpoint-picker` 在单个 upstream cluster 内选择健康 endpoint，支持 queue depth、KV cache 利用率、prefix cache 近似匹配、LoRA 亲和 |
| Cluster 级调度 | `ai-load-balancer` 支持 `global_least_request`（Redis 全局最小请求数）、`prefix_cache`（prompt 前缀匹配）、`endpoint_metrics`（基于 Prometheus 指标） |
| Prefix cache 感知 | `ai-endpoint-picker` 实现了近似前缀缓存评分：基于 canonical JSON 的语义 segment 构建 LRU，使用 `0.75 × matched/total + 0.25 × min(matched/8192,1)²` 公式 |

### 1.4 成本感知 — `ai-proxy` + 外部编排

Higress 本身不内置成本感知路由器，但提供了基础设施：
- `ai-statistics` 插件输出 token 用量指标
- `model-router` 可结合外部系统实现成本决策
- 成本决策通常由应用层或控制面完成，不在数据面网关的职责范围

## 2. 可靠性与容错

第 9 章要求"先判断能否重试，再选择重试目标"，并按错误类型分级处理。

### 2.1 错误分类与重试 — `ai-proxy` 内置

| 错误类型 | Higress 处理 |
|----------|-------------|
| 限流响应 (429) | `ai-proxy` 的 `failover` 配置：当 apiToken 不可用时移出列表，健康检测通过后重新加入 |
| 服务端错误 (5xx) | `retryOnFailure` 配置：请求失败时立即重试 |
| 认证错误 (401/403) | 由 `failover` 机制处理，apiToken 失败后摘除 |
| 上下文超限 | `ai-context-limit` 插件在请求到达模型前拦截超长请求，返回 400 |

**源码位置**：
- `plugins/wasm-go/extensions/ai-proxy/provider/failover.go`
- `plugins/wasm-go/extensions/ai-proxy/provider/retry.go`

### 2.2 API Token 高可用

`ai-proxy` 支持配置多个 `apiTokens`，请求时随机选择。配合 `failover` 策略：
- 某个 token 连续失败 → 移出可用列表
- 健康检测通过或冷却时间到期 → 重新加入
- 实现了 token 级别的故障转移

## 3. 预算治理

第 9 章区分了三种预算语义：**速率限制、配额、严格预算**。

### 3.1 速率限制 — `ai-token-ratelimit`

| 能力 | 实现 |
|------|------|
| 基于 token 的限流 | 请求阶段检查窗口，响应后累计 token 用量 |
| 多维度分组 | 支持按 header / param / consumer / cookie / per_header / per_param / per_consumer 分组 |
| 多时间窗口 | 支持 per_second / per_minute / per_hour / per_day |
| 全局 + 规则混合 | `global_threshold`（全局兜底）+ `rule_items`（按维度细分），**all-match OR 叠加**语义 |
| Redis 后端 | 基于 Redis 实现分布式计数器，key 格式包含 `{rule_name}` hash tag 兼容 Redis Cluster |

**行为语义**：所有匹配规则叠加生效，任一触发即拒绝（非 first-match-wins）。

**局限**：请求阶段检查窗口、响应后累计 → 存在并发超额窗口，**不能宣称严格预算**。

### 3.2 配额 — `ai-quota`

| 能力 | 实现 |
|------|------|
| 基于 consumer 的配额 | 按认证身份（consumer）分配固定 quota |
| 配额管理 | 支持查询、刷新、增减配额（通过 admin_path + admin_consumer） |
| Redis 后端 | 配额存储在 Redis，key 前缀可配置 |
| 与认证联动 | 需配合 `key-auth` / `jwt-auth` 等认证插件获取 consumer 身份 |
| 与统计联动 | 需配合 `ai-statistics` 插件获取 token 统计信息 |

**局限**：同样是"请求检查 → 响应扣减"模式，**不能仅凭现有检查与后结算机制宣称严格并发预算**。

### 3.3 严格预算 — 未实现

第 9 章要求严格预算需要"原子预留、可证明的最大消耗、结算与退款"。Higress 当前的 `ai-token-ratelimit` 和 `ai-quota` 都做不到：
- 没有请求前的原子预留
- 没有可证明的最大消耗上界
- 没有结算与退款机制
- Redis 故障时行为不一致（读失败 vs 写失败 vs 超时的对账未统一）

## 4. 超时与流式处理

第 9 章要求区分五种超时。Higress 的处理：

| 超时类型 | Higress 支持 |
|----------|-------------|
| 连接超时 | Envoy 原生 `connect_timeout` |
| 首字节超时 | Envoy 原生 `stream_idle_timeout` |
| TTFT（首 token 时延） | `ai-statistics` 插件统计 `llm_first_token_duration` 指标 |
| 流式空闲超时 | Envoy 原生 `stream_idle_timeout` |
| 总截止时间 | `ai-proxy` 的 `timeout` 字段（当前仅用于获取上下文信息，不影响实际转发） |

**流式处理能力**：
- Higress 强调"真正的完整流式处理"——请求/响应 body 的流式处理
- Wasm 插件可以自定义 SSE 协议处理
- `ai-cache` 同时支持流式和非流式响应缓存
- `ai-security-guard` 在检查响应时会使流式变为非流式（需要缓冲完整响应）

## 5. 可观测性 — `ai-statistics`

| 维度 | 指标 |
|------|------|
| **用量** | input_token、output_token（按网关/路由/服务/模型四维度） |
| **时延** | `llm_first_token_duration`（TTFT）、`llm_service_duration`（总耗时） |
| **可靠性** | HTTP 状态码（Envoy 原生） |
| **关联** | `session_id_header` 支持追踪多轮 Agent 对话 |
| **日志** | input_token、output_token、model、question、answer（可配置） |
| **链路追踪** | 支持将 attributes 写入 span，RAG 检索文档 ID 也会记录 |

**轻量模式**：`use_default_response_attributes: true` 使用轻量级默认属性（model + token 统计），不缓冲流式响应体，适合高并发生产环境。

**调试模式**：`use_default_attributes: true` 启用完整属性（包括 messages、answer、question），适用于调试和审计场景。

## 6. 语义缓存 — `ai-cache`

| 能力 | 实现 |
|------|------|
| 字符串匹配缓存 | 精确匹配请求内容，命中则直接返回缓存响应 |
| 语义缓存 | 基于向量数据库的 embedding 相似度匹配 |
| 缓存后端 | 支持 Redis（字符串匹配）+ 向量数据库（语义匹配） |
| 流式支持 | 同时支持流式和非流式响应缓存 |
| 缓存键策略 | `lastQuestion`（最后一个问题）/ `allQuestions`（拼接所有问题）/ `disabled` |
| 缓存跳过 | `x-higress-skip-ai-cache: on` 请求头跳过缓存 |

## 7. 内容安全 — `ai-security-guard`

| 能力 | 实现 |
|------|------|
| 请求检查 | 检测用户输入是否合规（`checkRequest`） |
| 响应检查 | 检测模型输出是否合规（`checkResponse`，会使流式变为非流式） |
| 集成 | 对接阿里云内容安全检测服务 |
| 风险等级 | 内容合规 / 提示词攻击 / 敏感内容 三级可调 |
| 结构化拦截 | 支持 OpenAI 格式的结构化拒答响应（`x_higress_guardrail`） |

## 8. 小结

| 第 9 章要求 | Higress 覆盖度 | 说明 |
|-------------|---------------|------|
| 静态映射 | ✅ 完整 | `ai-proxy` modelMapping + 自动协议转换 |
| 语义路由 | ✅ 完整 | `ai-intent` + `model-router` |
| 端点选择 | ✅ 完整 | `ai-endpoint-picker` + `ai-load-balancer` |
| 成本感知路由 | ⚠️ 部分 | 有基础设施（统计 + 路由），缺内置成本决策 |
| 错误分类重试 | ✅ 完整 | failover + retryOnFailure |
| 速率限制 | ✅ 完整 | `ai-token-ratelimit`（基于 token） |
| 配额 | ✅ 完整 | `ai-quota`（基于 consumer） |
| 严格预算 | ❌ 未实现 | 缺原子预留、最大消耗证明、结算退款 |
| 五种超时 | ⚠️ 部分 | TTFT 可统计，总截止时间未强制 |
| 流式处理 | ✅ 完整 | 真正的流式处理，SSE 友好 |
