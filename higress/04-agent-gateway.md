# 04 — Agent 网关能力

> 对照第 9 章 §9.4 Agent Gateway 的治理语义逐项分析 Higress 的实现覆盖度。

## 1. Agent 编排 — `ai-agent` 插件

Higress 提供了内置的 Agent 编排插件 `ai-agent`，实现了基于 ReAct 模式的 API Agent：

```
用户请求 → LLM (推理) → 选择 API → 调用外部 API → LLM (整合) → 返回结果
                  ↑                                        │
                  └──────────── 多轮循环 ──────────────────┘
```

### 1.1 配置结构

| 配置项 | 说明 |
|--------|------|
| `llm` | AI 服务提供商（apiKey、serviceName、model、maxIterations、maxExecutionTime、maxTokens） |
| `apis` | 外部 API 服务（apiProvider + OpenAPI 文档描述） |
| `promptTemplate` | ReAct 模板定制 |
| `jsonResp` | JSON 格式化输出配置 |

### 1.2 能力特征

- 支持 GET 和 POST 方法的外部 API
- 支持多轮对话
- 支持流式与非流式模式
- 最大迭代次数可配置（默认 15 步）
- 单次请求超时可配置（默认 50000ms）
- 输出 token 限制可配置（默认 1000）

**源码位置**：`plugins/wasm-go/extensions/ai-agent/`

## 2. 任务关联对象

第 9 章定义了 Agent 网关应管理的任务关联对象：

| 对象 | 第 9 章定义 | Higress 实现 |
|------|------------|-------------|
| **Task** | 业务应用的任务模型 | ❌ 网关层不管理 Task |
| **Session** | 会话所属应用或执行系统 | ⚠️ `ai-statistics` 支持 `session_id_header` 追踪多轮对话 |
| **Turn** | Harness 定义并记录边界 | ❌ 网关层不区分 Turn |
| **Call/Attempt** | 调用系统与实际执行点 | ⚠️ `ai-statistics` 可记录每次 model/tool call |
| **State** | 业务 Task/State 模型 | ❌ 网关层不管理 State |
| **Checkpoint** | 业务模型定义内容与恢复契约 | ❌ 网关层不涉及 |
| **Outcome** | 业务应用或获授权 Verifier | ❌ 网关层不管理 Outcome |

**归因链路**（第 9 章）：`task_id → session_id → turn_id → model/tool call → attempt_id`

Higress 的归因能力：
- ✅ `session_id`：通过 `ai-statistics` 的 `session_id_header` 配置
- ✅ `model/tool call`：`ai-statistics` 记录每次调用的模型、token、耗时
- ❌ `task_id`、`turn_id`、`attempt_id`：需要应用层提供，网关不主动管理

## 3. 租户隔离维度

第 9 章定义了两个正交维度：**身份维度** 和 **财务维度**。

### 3.1 身份维度

| 身份类型 | Higress 支持 |
|----------|-------------|
| **人** | ✅ 通过认证插件（key-auth / jwt-auth / oidc / basic-auth / hmac-auth）识别 |
| **工作负载** | ⚠️ 可通过 mTLS（Istio 集成）识别，但非网关层主动管理 |
| **Agent 实例** | ⚠️ 可通过 consumer 概念区分，但无原生 Agent 实例标识 |
| **Task** | ❌ 不管理 |

### 3.2 财务维度

| 维度 | Higress 支持 |
|------|-------------|
| **组织** | ❌ 无原生组织概念 |
| **团队** | ❌ 无原生团队概念 |
| **项目** | ⚠️ 可通过 consumer 分组间接实现 |

### 3.3 隔离实现

Higress 的隔离机制主要通过 **consumer** 概念实现：
- 认证插件识别 consumer 身份
- `ai-token-ratelimit` 按 consumer 分组限流
- `ai-quota` 按 consumer 分配配额
- `ai-statistics` 按 consumer 维度统计

**局限**：consumer 是扁平的身份标识，不支持组织→团队→项目的层级结构。两个正交维度（身份 + 财务）在 Higress 中被简化为单一 consumer 维度。

## 4. 账本层级

第 9 章定义了四层账本：Attempt → Call/Turn → Session → Task。

| 层级 | 第 9 章观测内容 | Higress 实现 |
|------|---------------|-------------|
| **Attempt** | 实际端点、用量来源、状态、耗时 | ✅ `ai-statistics` 记录每次调用的模型、token、TTFT、总耗时 |
| **Call/Turn** | 多次尝试、工具交互、上下文变化 | ⚠️ 可通过 session_id 关联，但不主动记录 Turn 边界 |
| **Session** | 累计用量、等待与失败分布 | ⚠️ 可通过 session_id 聚合，但需外部系统计算 |
| **Task** | 跨会话归集、恢复阶段、Outcome 引用 | ❌ 不管理 Task 级别归集 |

**`ai-statistics` 提供的 Attempt 级数据**：
- input_token / output_token
- model / llm_service_duration
- llm_first_token_duration（TTFT）
- 按网关 / 路由 / 服务 / 模型 四维度

## 5. 会话历史管理 — `ai-history`

`ai-history` 插件管理多轮对话的上下文历史：
- 控制对话历史的存储和传递
- 与 `ai-proxy` 联动管理多轮交互

## 6. 上下文窗口管理 — `ai-context-limit`

`ai-context-limit` 在请求到达模型前进行 token 估算：
- 当估算 token 超过配置的 `max_context_tokens` 时拦截请求
- 支持安全缓冲系数 `buffer_ratio`（默认 1.10）
- 可按路由、服务、域名或 MCP Server 设置独立上限
- 使用内嵌 BPE 词表（o200k_base.tiktoken）进行 token 估算

**与第 9 章的关联**：这是防止上下文爆炸的"断路器"，保护后端模型免受超长请求。

## 7. 审批约束转发

第 9 章 §9.5 定义了审批状态与转发行为的对应关系：

| 审批状态 | 第 9 章要求 | Higress 实现 |
|---------|-----------|-------------|
| 待审批 | 不执行高风险操作 | ⚠️ `ext-auth` + `opa` 可实现外部审批，但非内置 |
| 批准 | 重新校验后才转发 | ⚠️ 通过 `ext-auth` 回调可实现 |
| 拒绝 | 终止，不隐式替换 | ✅ 默认行为 |
| 超时/撤销 | 终止，保持未执行状态 | ⚠️ 依赖 `ext-auth` 超时配置 |

**核心差距**：Higress 没有内置的"人-in-the-loop"审批流程。审批能力需要通过 `ext-auth`（外部授权）+ 外部审批系统组合实现。

## 8. 审计事件

第 9 章定义了标准审计事件结构。Higress 的审计能力：

| 字段 | Higress 可提供 |
|------|---------------|
| `event_type` | ⚠️ 可通过 `ai-statistics` 自定义 attributes |
| `subject_ref` | ✅ consumer 身份 |
| `task_id` | ❌ 需要应用层注入 |
| `call_id` | ⚠️ 可通过 trace ID 关联 |
| `attempt_id` | ⚠️ 可通过 span ID 关联 |
| `policy_version` | ❌ 无策略版本管理 |
| `approval_ref` | ❌ 无审批引用 |
| `decision` | ✅ allow/deny 可通过 `ext-auth` 记录 |
| `execution_status` | ✅ HTTP 状态码 |
| `usage_status` | ⚠️ token 用量可记录 |

## 9. 小结

| 第 9 章要求 | Higress 覆盖度 | 说明 |
|-------------|---------------|------|
| Agent 编排（ReAct） | ✅ 完整 | `ai-agent` 插件 |
| 任务关联对象 | ❌ 大部分缺失 | 仅 Session 和 Call 部分可追踪 |
| 归因链路 | ⚠️ 部分 | session_id + call 级统计 |
| 身份维度隔离 | ⚠️ 部分 | consumer 扁平模型，缺层级 |
| 财务维度隔离 | ❌ 基本缺失 | 无组织/团队/项目概念 |
| Attempt 级账本 | ✅ 完整 | `ai-statistics` |
| Session 级账本 | ⚠️ 部分 | 可聚合但需外部系统 |
| Task 级账本 | ❌ 缺失 | 不管理 Task |
| 审批约束转发 | ⚠️ 部分 | 需 ext-auth + 外部系统 |
| 审计事件 | ⚠️ 部分 | Attempt 级完整，Task/审批级缺失 |

**核心结论**：Higress 在 Agent 网关层面主要提供**基础设施**（编排、统计、会话管理），但不承担**业务状态权威**。这与第 9 章的指导原则一致——"网关不因此获得业务 Task、State、Checkpoint、恢复语义或 Outcome 的权威所有权"。
