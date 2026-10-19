# Nacos 3.x 作为 Agentic Resource Registry：源码级分析

> **版本基线**：`nacos-develop`（Nacos 3.3.0-RC），源码路径 `nacos-develop/`
> **参考框架**：[《第 15 章 AI 资产的发现与管理》](https://github.com/aliyun/ai-agent-handbook/blob/main/04-governance/%E7%AC%AC%2015%20%E7%AB%A0%E3%80%80AI%20%E8%B5%84%E4%BA%A7%E7%9A%84%E5%8F%91%E7%8E%B0%E4%B8%8E%E7%AE%A1%E7%90%86.md)
> **写作目标**：把 Nacos 源码实现映射到参考文章提出的 "Agentic Resource Registry" 概念，作为网关项目接入或自研 AI Registry 的设计蓝图。

---

## 目录

- [第 0 章 TL;DR](#第-0-章--tldr)
- [第 1 章 Nacos 3.x 的总体定位](#第-1-章--nacos-3x-的总体定位)
  - [1.1 从"微服务双引擎"到"微服务 + AI 资产"双栈](#11-从微服务双引擎到微服务--ai-资产双栈)
  - [1.2 模块全景（30 个 Maven 子模块）](#12-模块全景30-个-maven-子模块)
- [第 2 章 统一资源模型](#第-2-章--统一资源模型)
  - [2.1 三层资源身份](#21-三层资源身份)
  - [2.2 双表模型](#22-双表模型)
  - [2.3 生命周期状态机](#23-生命周期状态机)
  - [2.4 接口面分层](#24-接口面分层)
- [第 3 章 五类 AI 资产对照](#第-3-章--五类-ai-资产对照)
  - [3.1 Prompt：结构化模板与版本化](#31-prompt结构化模板与版本化)
  - [3.2 Skill：能力包模型与可信准入](#32-skill能力包模型与可信准入)
  - [3.3 MCP Server：能力定义与端点的分离](#33-mcp-server能力定义与端点的分离)
  - [3.4 Agent：A2A AgentCard 与 RAD 协议](#34-agent-a2a-agentcard-与-rad-协议)
  - [3.5 AgentSpec：声明式依赖与稳态 Agent 发现](#35-agentspec声明式依赖与稳态-agent-发现)
- [第 4 章 治理流水线](#第-4-章--治理流水线)
  - [4.1 Nacos 的发布流水线](#41-nacos-的发布流水线)
  - [4.2 流水线与生命周期的协作](#42-流水线与生命周期的协作)
  - [4.3 与参考文章的对齐](#43-与参考文章的对齐)
- [第 5 章 双模发现机制](#第-5-章--双模发现机制)
  - [5.1 稳态路径：AgentSpec 声明式依赖](#51-稳态路径agentspec-声明式依赖)
  - [5.2 动态路径：ARD + RAD 组合](#52-动态路径ard--rad-组合)
  - [5.3 检索 vs 协议适配的职责划分](#53-检索-vs-协议适配的职责划分)
- [第 6 章 端点发现与健康检查](#第-6-章--端点发现与健康检查)
  - [6.1 Naming v2 核心类](#61-naming-v2-核心类)
  - [6.2 MCP / Agent 端点的 Naming 复用](#62-mcp--agent-端点的-naming-复用)
  - [6.3 推送与订阅](#63-推送与订阅)
  - [6.4 双模一致性](#64-双模一致性)
- [第 7 章 统一变更评审与安全](#第-7-章--统一变更评审与安全)
  - [7.1 鉴权框架](#71-鉴权框架)
  - [7.2 可见性](#72-可见性)
  - [7.3 Trace 与审计](#73-trace-与审计)
  - [7.4 安全边界](#74-安全边界)
- [第 8 章 Context 装配：发现与执行的分离](#第-8-章--context-装配发现与执行的分离)
  - [8.1 客户端 SDK 层的实现](#81-客户端-sdk-层的实现)
  - [8.2 分离的体现](#82-分离的体现)
- [第 9 章 插件与扩展性边界](#第-9-章--插件与扩展性边界)
  - [9.1 Nacos 的 10 类插件 SPI](#91-nacos-的-10-类插件-spi)
  - [9.2 四类 AI 插件 SPI](#92-四类-ai-插件-spi)
  - [9.3 加载机制](#93-加载机制)
  - [9.4 插入新资源类型的套路](#94-插入新资源类型的套路)
- [第 10 章 社区协议兼容](#第-10-章--社区协议兼容)
  - [10.1 ai-registry-adaptor 模块](#101-ai-registry-adaptor-模块)
  - [10.2 协议契约的严格遵循](#102-协议契约的严格遵循)
  - [10.3 让外部 MCP 客户端直接使用 Nacos](#103-让外部-mcp-客户端直接使用-nacos)
- [第 11 章 成熟度模型对照](#第-11-章--成熟度模型对照)
  - [11.1 Nacos 已经具备的能力 ✅](#111-nacos-已经具备的能力-)
  - [11.2 Nacos 还需要补充的能力 ⚠️](#112-nacos-还需要补充的能力-)
- [第 12 章 对网关项目的启示](#第-12-章--对网关项目的启示)
  - [12.1 三种接入方案](#121-三种接入方案)
  - [12.2 关键设计蓝图](#122-关键设计蓝图)
  - [12.3 关键文件索引](#123-关键文件索引)
- [结语](#结语)

---

## 第 0 章  TL;DR

参考文章提出把 **Prompt / Skill / MCP Server / Agent / AgentSpec** 五类 AI 资产纳入统一的 "Agentic Resource Registry"，通过 ARD/RAD 协议进行发现，通过流水线进行治理。

**结论先行**：Nacos 3.x（3.2 引入 AI、3.3 系统性补齐）已经把这件事**完整做完了**——不是零散的功能点，而是一套完整的双栈注册中心：

| 参考文章概念 | Nacos 3.x 对应实现 |
|---|---|
| 统一资产注册中心 | `ai` 模块 + `ai_resource` / `ai_resource_version` 双表 |
| 五类资产 | `mcp` / `agent` / `prompt` / `skill` / `agentspec`，均有独立 spec |
| 版本化 + 标签（latest/stable） | `AiResource.versionInfo.labels` + `editingVersion` / `reviewingVersion` |
| 发布流水线（草稿→审核→上线） | `PublishPipelineManager` + `PublishPipelineExecutor` + `PublishPipelineService` SPI |
| 端点发现 + 健康检查 | 复用 `naming` 模块（Distro AP + JRaft CP 双模一致性） |
| ARD 动态发现 | `ai-registry-adaptor` 暴露 ARD 协议（`ArdSearchController` / `ArdWellKnownController`） |
| RAD 远程发现 | `A2aRadClientAdapter`（客户端 SDK），协议版本 0.5.0 |
| Context 装配（发现/执行分离） | `McpTransportRouter` / `AgentTransportRouter` / `AgentWatchManager` |
| 社区协议兼容 | MCP Registry / Skills.sh / `.well-known` / ARD 全兼容 |
| 向量检索 | `AiResourceSearchRepository` + `AiResourceEmbeddingService` + PostgreSQL pgvector 插件 |
| 外部资产导入 | `AiResourceImportService` SPI + 默认 `SkillsShImportService` / `McpRegistryImportService` / `McpOfficialImportService` / `SkillWellKnownImportService` |

换句话说：**可以直接使用 Nacos 3.x 作为 AI Registry**，不必从零自研；如果要自研，Nacos 的领域模型、插件 SPI、协议适配层都是高质量的设计蓝图。

---

## 第 1 章  Nacos 3.x 的总体定位

### 1.1 从"微服务双引擎"到"微服务 + AI 资产"双栈

Nacos 原本是 Naming（服务注册与发现）+ Config（动态配置中心）的双引擎。3.x 系列把 **AI Registry** 提升为与 Naming、Config 并列的第三大领域。

官方规范明确说明（`specs/zh-cn/ai/ai-registry-spec.md`）：

> AI Registry 是 Nacos 中负责 AI 资源注册、治理、发现和分发的领域。它是 Nacos 3.x 中与 Config、Naming 并列的一等能力。

`AGENTS.md` 第 39-41 行也强调 Nacos 作为 "Agentic Resource Registry" 的定位。

### 1.2 模块全景（30 个 Maven 子模块）

根 `pom.xml` 列出了 30 个子模块。按职责分组如下（每行给出模块→一句话职责→关键包路径）：

**客户端与 API 层**
- `api`：公共 API 与 POJO（`NamingService` / `ConfigService` / `AiService` / Protobuf 帧）——`api/src/main/java/com/alibaba/nacos/api/`
- `client`：Java SDK 实现（`NacosConfigService` / `NacosNamingService` / `NacosAiService`）——`client/src/main/java/com/alibaba/nacos/client/`
- `client-basic`：极简 SDK，面向轻量接入
- `maintainer-client`：运维/管理工具客户端

**服务端核心领域**
- `config`：配置中心（发布/CAS/监听/长轮询/灰度）——`config/src/main/java/com/alibaba/nacos/config/server/`
- `naming`：服务注册中心（注册/健康检查/订阅/推送）——`naming/src/main/java/com/alibaba/nacos/naming/`
- `ai`：**AI Registry 核心**——五类资产的全生命周期、端点注册、订阅、向量检索、发布流水线、AI storage 插件路由——`ai/src/main/java/com/alibaba/nacos/ai/`
- `copilot`：与 Nacos Copilot（AI 助手）集成
- `ai-registry-adaptor`：**社区注册协议适配层**（MCP Registry / Skills.sh / ARD / `.well-known`）——`ai-registry-adaptor/src/main/java/com/alibaba/nacos/airegistry/`

**一致性与集群**
- `consistency`：一致性协议抽象（`ConsistencyProtocol` + `APProtocol` / `CPProtocol`）——`consistency/src/main/java/com/alibaba/nacos/consistency/`
- `core`：服务端内核，包含 `distributed/distro`（AP）与 `distributed/raft`（CP，JRaft 实现）、集群、gRPC 连接管理——`core/src/main/java/com/alibaba/nacos/core/`

**持久化**
- `persistence`：Derby / MySQL / PostgreSQL 多方言

**安全与扩展**
- `auth`：`@Secured` 注解 + `ProtocolAuthService` + HTTP/gRPC 资源解析器
- `plugin`：10 类插件 SPI 契约（auth、visibility、datasource、config、encryption、trace、environment、control、**ai**、lock）
- `plugin-default-impl`：官方默认实现，含 `nacos-default-ai-pipeline-plugin` / `nacos-default-ai-trace-plugin` / `nacos-default-ai-vector-plugin`（PostgreSQL pgvector）/ `nacos-default-ai-importer-plugin`（含 MCP Registry / Skills.sh / MCP Official / SkillWellKnown 四种导入器）/ `nacos-oidc-auth-plugin` / `nacos-ldap-auth-plugin`

**基础设施**
- `common`：HTTP 客户端、`NotifyCenter` 事件总线、`RpcClient`
- `sys`、`logger-adapter-impl`：系统环境与日志适配

**接入与生态**
- `console` / `console-ui` / `console-ui-next`：Web 控制台
- `address`、`dns`：寻址与动态 DNS
- `istio`：Istio/Envoy xDS 协议适配
- `k8s-sync`：与 Kubernetes Service 双向同步
- `prometheus`：指标导出
- `lock`：基于 JRaft 的分布式锁

**打包与启动**
- `bootstrap`（启动入口）、`server`（聚合打包）、`distribution`（发布包）
- `resources` / `test` / `example` / `doc` / `specs` / `skills` / `Codex`：文档、测试、示例、规范、AI Skill 样例、Codex 辅助

**对网关项目的启示**：Nacos 不是单点的"注册中心"，而是一个完整的基础设施栈。网关接入 AI Registry 时，可以直接复用其 AI 模块，不必触碰 Naming/Config 的旧语义。

---

## 第 2 章  统一资源模型

参考文章强调："AI 资产的注册中心需要统一的资源模型与版本化能力。" Nacos 的设计完全对齐这一点，并通过 `ai_resource` + `ai_resource_version` 双表实现。

### 2.1 三层资源身份

参见 `specs/en/design/resource-model-spec.md` 与 `specs/zh-cn/ai/ai-resource-model-spec.md`：

```
namespaceId  →  resourceType  →  resourceName
     ↑               ↑                ↑
  租户/环境       五大 AI 类型之一    稳定资源名
```

| 资源类型 | resourceType | resourceName 别名 | 规范 |
|---|---|---|---|
| MCP Server | `mcp` | `mcpName` | `specs/zh-cn/ai/mcp-server-spec.md` |
| Agent | `agent` | `agentName` | `specs/zh-cn/ai/agent-management-spec.md` |
| Prompt | `prompt` | `promptKey` | `specs/zh-cn/ai/prompt-spec.md` |
| Skill | `skill` | `name`（从 SKILL.md 解析） | `specs/zh-cn/ai/skill-spec.md` |
| AgentSpec | `agentspec` | `name`（从 manifest.json 解析） | `specs/zh-cn/ai/agentspec-spec.md` |

> **与 Config/Naming 的对比**：Config 用 `namespaceId+group+dataId`，Naming 用 `namespaceId+group+serviceName`，AI 资源用 `namespaceId+type+name`——**AI 资源不使用 Config 风格的 groupName 作为身份**（`ai-registry-spec.md` 第 2 节"资源身份稳定"原则）。

### 2.2 双表模型

源码位置：
- `ai/src/main/java/com/alibaba/nacos/ai/model/AiResource.java`
- `ai/src/main/java/com/alibaba/nacos/ai/model/AiResourceVersion.java`

**`AiResource` 元数据行**（一个资源名对应一行）：

| 字段 | 含义 |
|---|---|
| `namespaceId`, `type`, `name` | 身份三元组（不可变） |
| `desc` | 资源描述 |
| `status` | `enable` / `disable` |
| `owner` | 创建者或所属 identity |
| `scope` | 可见性 scope（`PUBLIC` / `PRIVATE`），由 `VisibilityService` 解析 |
| `bizTags` | 业务标签（JSON 数组），用于筛选或 UI 分组 |
| `ext` | 资源类型自有的扩展 JSON（例如 Agent 的目录扩展、MCP 的 Schema Version） |
| `from` | `bootstrap` / `import` / `sync` 来源标记 |
| `versionInfo` | **版本治理摘要 JSON**：`{editingVersion, reviewingVersion, onlineCnt, labels}` |
| `metaVersion` | **元数据 CAS 乐观锁**——用于所有元数据更新的并发控制 |
| `downloadCount` | 聚合下载/使用次数 |

**`AiResourceVersion` 版本行**（一个资源名下可有 N 个版本）：

| 字段 | 含义 |
|---|---|
| `namespaceId`, `type`, `name`, `version` | 版本身份四元组 |
| `author` | 版本作者 |
| `desc` | 版本描述或 commit message |
| `status` | 生命周期状态：`draft` / `reviewing` / `reviewed` / `online` / `offline` |
| `storage` | **AI Storage Plugin 指针 JSON**——内容存在哪里（默认指向 Config 存储） |
| `publishPipelineInfo` | 与发布流水线执行关联的审核状态 JSON |
| `downloadCount` | 该版本的下载/使用次数 |

**关键设计要点**：
1. **元数据与版本分离**——参考文章强调的"版本管理"在这里直接体现为两张表。
2. **内容存储插件化**——`storage` 字段指向 `AiResourceStorage` 插件实现，默认是 `NacosConfigAiResourceStorage`（借道 Config 存储，但**语义上不是用户 Config**）。
3. **Labels 而非 Config 风格的灰度**——`versionInfo.labels` 包含 `latest`（服务端维护）以及自定义标签（如 `stable`、`canary`），但 **labels 不得指向 `draft` 或 `reviewing` 版本**。
4. **CAS 乐观锁**——`metaVersion` 用于所有元数据更新，避免 ABA 问题。

### 2.3 生命周期状态机

参见 `specs/zh-cn/ai/ai-resource-lifecycle-spec.md`：

```
create/upload draft  →  update draft  →  submit
                                            ↓
                                      reviewing
                                            ↓
                                       reviewed  ←──  reject
                                            ↓
                         publish ──────── online  ⇄  offline
                                            ↓
                                          delete
```

**关键规则**：
- `submit` 解析 `editingVersion`（若无则解析 `reviewingVersion`），送进发布流水线。
- 流水线通过/拒绝都把版本改为 `reviewed`；拒绝后需要显式 redraft。
- `publish` 把版本改为 `online`，清理 working 指针，由服务端维护 `latest` label。
- `force-publish` 绕过流水线校验，但仅接受 `draft` / `reviewing` / `reviewed`，不得对 `online` / `offline` 使用。
- `latest` label 由**服务端维护**——客户端可以传 `latest`，但服务端会忽略其值并合并回自己维护的指针。
- 删除 `latest` 指向的版本时，自动选择剩余 online 版本中 SemVer 最大的一个。

### 2.4 接口面分层

| 接口面 | 受众 | 路径前缀 |
|---|---|---|
| Client API | 运行时客户端、Agent framework | `/v3/client/ai/...` |
| Admin API | 管理工具、Maintainer SDK | `/v3/admin/ai/...` |
| Console API | Nacos 控制台 | `/v3/console/ai/...` |
| gRPC | Java SDK 运行时流量 | `nacos_grpc_service.proto` |
| AI Registry 适配器 | 外部社区 registry 客户端 | 独立端口（默认 9080） |

**关键源码锚点**：
- `AiService.java`（`api/src/main/java/com/alibaba/nacos/api/ai/AiService.java`）：组合接口 `extends McpService, A2aService, SkillService, AgentSpecService, PromptService`，并提供 `mcp()` / `skill()` / `agentSpec()` / `prompt()` / `agent()` 子访问器
- `AiFactory.createAiService(Properties)`（`api/src/main/java/com/alibaba/nacos/api/ai/AiFactory.java`）：反射创建 `com.alibaba.nacos.client.ai.NacosAiService`
- `NacosAiService.java`（`client/src/main/java/com/alibaba/nacos/client/ai/NacosAiService.java`）：客户端实现
- 15 个 REST Controller 在 `ai/src/main/java/com/alibaba/nacos/ai/controller/`，按 `{Resource}{Admin,Client}Controller` 命名

**对网关项目的启示**：
- **资源身份三元组 `namespace/type/name`** 是设计蓝图的基石——网关如果要自研 Registry，可以照搬这套模型。
- **元数据/版本双表 + CAS 乐观锁** 是处理 AI 资产高频变更的可靠模式。
- **Labels 与 `latest` 服务端维护** 是把灰度发布与版本路由解耦的优雅方案。

---

## 第 3 章  五类 AI 资产对照

参考文章按 Prompt / Skill / MCP / Agent / AgentSpec 五类资产分别阐述。下面逐类对照 Nacos 实现。

### 3.1 Prompt：结构化模板与版本化

**参考文章要点**：Prompt 应从纯文本演进为包含"指令层、变量 Schema、运行依赖"的结构化对象；支持版本、评测与灰度。

**Nacos 实现**：
- 身份：`namespaceId -> prompt -> promptKey`
- 内容模型：每个版本包含 prompt template、可选变量定义、用于运行时条件获取的 md5、作者、commit message、状态、存储指针等版本元数据
- 关键类：`PromptAdminController` / `PromptClientController`（`ai/src/main/java/com/alibaba/nacos/ai/controller/`）、`api/ai/model/prompt/Prompt.java`
- 存储：默认走 `NacosConfigAiResourceStorage`；每个版本的 `storage` 字段指向选定的 provider
- 运行时行为：客户端可按 `promptKey` + `version` + `label` + `md5` 查询；md5 命中返回 304 not-modified
- 检索：参与通用 AI Resource Search，并提供 `GET /v3/client/ai/prompt/search` 专用 Facade

**关键设计亮点**：
- **Nacos 把 Prompt 视为 AI artifact，不解析也不执行 template**——这是一个重要的边界划分（避免注册中心变成执行引擎）。
- Prompt handler 在构建 Search chunk 时**显式过滤 credential、secret 默认值与运行时参数值**，避免泄露到检索索引。

**对网关项目的启示**：网关可以把 Prompt 作为静态资产拉取并本地缓存，通过 md5 做增量更新；网关不必解析 Prompt 内容。

### 3.2 Skill：能力包模型与可信准入

**参考文章要点**：Skill 不是单一文件，而是包含 `SKILL.md`、脚本、模板、依赖的完整能力包；外部 Skill 需要经过来源核实、静态检查、隔离验证和人工评审才能进入内部 Registry。

**Nacos 实现**：
- 身份：`namespaceId -> skill -> name`（name 在上传时从 `SKILL.md` frontmatter 解析）
- 包模型：ZIP 包，必须包含 `SKILL.md`（YAML frontmatter + Markdown 指令正文），可选 `scripts/`、`references/`、`assets/` 目录
- 与上游标准对齐：[Agent Skills Specification](https://agentskills.io/specification) 定义的"一个目录至少包含一个 `SKILL.md` 文件"
- 上传流程：
  - 预检（precheck）：接收 ZIP，解析单个或批量 Skill，返回 `READY` / `VERSION_ADJUSTED` / `DRAFT_EXISTS` / `REVIEWING_EXISTS` / `NO_PERMISSION` / `NOT_A_SKILL` / `INVALID_SKILL`
  - 上传：接收 ZIP + 可选 `targetVersion`，版本来源优先级为 SKILL.md frontmatter `version` → frontmatter `metadata.version` → `_meta.json` → 请求参数 → 服务端默认
- 关键类：`SkillAdminController` / `SkillClientController` / `SkillDownloadCountManager`
- frontmatter 管理：`ai_resource.ext` 保存受限的展示快照（最多 64 项，值最多 1024 字符，序列化后最多 16 KiB），超长截断时 `frontMatterTruncated=true`
- 客户端监听契约：**Skill 不提供推送通道**，客户端 SDK 通过周期性条件查询 `GET /v3/client/ai/skills` 实现监听，使用单线程 `schedule + 任务尾端自调度`（避免 `scheduleAtFixedRate` 在慢响应时堆积）；响应头包含 `ETag`、`X-Nacos-Skill-Md5`、`X-Nacos-Skill-Resolved-Version`；304 响应不携带 `Content-Type` 与 resolved version
- 导入：通过 `AiResourceImportService` SPI，默认实现有 `SkillsShImportService` 与 `SkillWellKnownImportService`（`plugin-default-impl/nacos-default-ai-importer-plugin/`）

**关键设计亮点**：
- **预检（precheck）机制**是参考文章"可信准入"的精确实现——客户端在真上传之前就能知道 ZIP 是否合法、版本是否会冲突、权限是否足够。
- **Nacos 注册中心路径在执行过程中不会执行包内脚本**——脚本执行属于发布流水线插件或客户端激活行为。这条边界与 Prompt 的"不解析不执行"相呼应。
- **md5 作为版本级字段在 upload 或 publish 时一次性计算**，运行时查询不重算——避免热点路径上的 hash 计算开销。

**对网关项目的启示**：网关若作为 Skill 消费方，需要实现：
1. ZIP 包下载与本地缓存
2. 基于 md5 的增量更新（304 协议）
3. 任务尾端自调度的轮询模式
4. frontmatter 解析（`name`、`description` 是必填）

### 3.3 MCP Server：能力定义与端点的分离

**参考文章要点**：MCP 管理需要把"能力定义（Tools/Schema）"与"运行端点"分开管理；支持三种接入方式（自动注册、API 转换、外部导入）；MCP Router 作为统一入口按需代理工具调用。

**Nacos 实现**：
- 身份：`namespaceId -> mcp -> mcpName`
- **首期迁移采用双管理路由状态**（`mcp-server-spec.md` 第 1 节）：
  - `SYNCING`：后台对账 Resource/Version row 期间，历史 MCP 管理路径仍是权威
  - `LIFECYCLE_MANAGED`：完整兼容管理操作集的读写统一使用通用 AI Resource 生命周期
- 事实所有权边界：
  - MCP 管理身份、enable 状态、owner、scope、label、working Version 指针 → `ai_resource`
  - Version 状态、author、Pipeline 状态、内容指针 → `ai_resource_version`
  - Server / Tools / Resources payload → 现有 MCP Config 对象（保持字节不变）
  - Runtime Endpoint（Naming 实例）→ Naming 运行时状态
- **LIFECYCLE_MANAGED 不是数据面切换**——历史 Manifest、Config 对象、Direct Service、普通 Service Ref、Client 自有 Runtime Service **都不会被降级为可随时删除的投影**
- 端点管理：`McpEndpointOperationService`（`ai/src/main/java/com/alibaba/nacos/ai/service/McpEndpointOperationService.java`）负责 MCP 端点的注册、健康检查、订阅，复用 Naming 引擎
- 客户端路由：`McpTransportRouter`（`client/.../ai/remote/McpTransportRouter.java`）在 HTTP 与 gRPC 之间自动选择
- 关键类：`McpAdminController` / `McpClientController` / `McpServerValidationService`

**关键设计亮点**：
- **管理身份的平滑迁移**：通过 `SYNCING → LIFECYCLE_MANAGED` 的单向状态机，把存量 MCP 资源无缝迁移到新生命周期模型，而不是搞一个大爆炸式的切换。这是处理历史包袱的成熟模式。
- **端点复用 Naming**：MCP endpoint 就是一种 Naming 实例——ref 模式复用用户 Service，direct 模式由 Nacos 自建 Service。健康检查、负载均衡、订阅推送全部复用，无需重复建设。
- **Descriptor 指向不变的 Config 内容**：Version Storage Descriptor 通过 Allowlist 限定的 `mcp-config-v1` Key 格式指向现有 MCP Server 与可选 Tools/Resources Config 对象，**不会复制、重写或扩展这些 Payload**。

**对网关项目的启示**：
- 网关作为 MCP Router 时，可以参考 Nacos 的"**capability 定义与 runtime endpoint 分离**"模式：
  - 定义（Tools/Schema）从 MCP Registry 拉取并缓存
  - Endpoint 通过 Naming/健康检查机制维护
  - Router 层按需从 definition 与 endpoint 组合出完整的 MCP Server 视图
- `McpTransportRouter` 的 HTTP/gRPC 自动切换是网关可以直接复用的客户端模式。

### 3.4 Agent：A2A AgentCard 与 RAD 协议

**参考文章要点**：Agent Registry 管理带版本的 Agent 调用描述（如 A2A AgentCard）与运行端点；RAD 协议提供协议无关的远程发现能力，涵盖 Search（候选筛选）、Discover（解析端点）与 Watch（状态感知）。

**Nacos 实现**：

**三类事实的清晰分离**（`agent-management-spec.md` 第 1 节）：

```
Agent
  1 --- N AgentVersion
             1 --- N AgentCallInterface
                         1 --- N DECLARED Endpoint

Runtime publisher --- N RUNTIME Endpoint
RUNTIME Endpoint --- versionRange match ---> AgentVersion + AgentCallInterface
```

| 事实 | 职责 | 事实源 |
|---|---|---|
| `Agent` | 稳定身份、目录元数据、owner、可见性、版本治理 | `ai_resource` |
| `AgentVersion` | 一个版本化调用定义，只在 draft 阶段可修改 | `ai_resource_version` + AI Storage |
| `AgentCallInterface` | 一个协议绑定及其声明地址 | `AgentVersionContent` |
| `RuntimeEndpoint` | 活跃 publisher 的可调用地址和兼容版本范围 | Naming 运行时状态 |

**关键类**：
- `AgentAdminController` / `AgentClientController`
- `A2aAdminController`（A2A AgentCard 兼容 facade）
- `AgentTransportRouter` / `McpTransportRouter`（客户端 SDK）
- `AgentWatchManager`（`client/.../ai/watch/AgentWatchManager.java`）
- `A2aRadClientAdapter`（`client/.../ai/remote/A2aRadClientAdapter.java`）：RAD 协议客户端适配

**RAD 协议（0.5.0）五大操作**（`specs/zh-cn/ai/rad-protocol-spec.md`）：

| 操作 | 输入 | 输出 | 语义 |
|---|---|---|---|
| `Search` | `AgentSearchRequest` | `AgentCatalogPage` | 分页搜索候选 Agent |
| `Discover` | `AgentDiscoveryRequest` | `AgentDiscoveryResult` | 返回一个 Agent 版本的完整调用快照 |
| `Watch` | `AgentDiscoveryRequest` | 逻辑 `AgentDiscoveryResult` 流 | 通过失效提示与重新查询流程观察完整替换快照 |
| `Register` | `AgentEndpointRegistrationBatch` | 成功或错误 | 完整替换当前 publisher 的运行时 Endpoint Batch |
| `Deregister` | `AgentEndpointDeregistrationBatch` | 成功或错误 | 从 publisher 期望 Batch 中移除 Endpoint 自然键 |

**Endpoint 自然键**：
```
(namespaceId, agentName, protocol,
 normalizedHost(uri), effectivePort(uri), normalizedTransport)
```
Path、Query、Metadata、Priority、Weight 不参与身份——同一 publisher 可以更新这些字段而不改变 Endpoint 身份。

**EndpointSet 与 sourceRevision**：
- `EndpointSet { source = DECLARED | RUNTIME, sourceRevision, endpoints[] }`
- DECLARED 使用 Version `contentDigest` 作为 sourceRevision
- RUNTIME 使用 `murmur3-x64-128-v1:<32 hex>` 作为 sourceRevision
- 消费者把两种形式都视为 opaque，不自行计算

**关键设计亮点**：
- **Agent 元数据不内嵌协议 descriptor、Endpoint、健康状态或完整版本历史**——元数据只做目录，调用详情按需 Discover。这是参考文章"发现摘要 vs 执行内容分离"原则的精确实现。
- **A2A 是首个协议 Adapter，但模型协议无关**——通用 Agent 模型不包含 A2A 专有字段；A2A AgentCard 只是 `agent` Version 内的一种协议 binding。这让 Nacos 可以支持未来任意 Agent 协议。
- **Endpoint 与 Agent 定义不互相拥有生命周期**——删除或 disable Agent 定义不会删除仍然活跃的运行时 publisher 状态。这是生产级容错的关键。
- **publisher 提交完整期望 Batch**——每个 publisher 对 `(namespaceId, agentName, protocol)` 提交完整期望快照，原子替换。这种"整批替换"模型避免了复杂的增量同步问题。

**对网关项目的启示**：
- 网关作为 Agent 消费方时，应该：
  1. 通过 RAD `Search` 找到候选 Agent
  2. 通过 RAD `Discover` 拿到完整调用快照（包含 `callInterfaces[]` 与 `endpointSets[]`）
  3. 通过 RAD `Watch` 观察变化（失效提示 → 重新 Discover）
  4. 客户端本地做 Endpoint 选择（按 `priority` 与 `weight`）与重试
- 网关作为 Agent 暴露方时，应该通过 RAD `Register` / `Deregister` 上报端点，整批替换模型。

### 3.5 AgentSpec：声明式依赖与稳态 Agent 发现

**参考文章要点**：稳态 Agent 通过声明式依赖（AgentSpec）在构建、部署或启动期解析精确版本。

**Nacos 实现**：
- 身份：`namespaceId -> agentspec -> name`
- 包模型：ZIP 包，包含 `manifest.json` 作为主描述文件 + 可选资源文件
- 与 Skill 的差异：不维护独立 manifest index；版本元数据和存储指针是事实来源
- 客户端监听协议：HTTP 轮询 + 条件查询（MD5 ETag）
  - 默认轮询间隔 10000 ms，可通过 `nacosAiAgentSpecCacheUpdateInterval` 配置
  - 304 响应携带 `ETag` + `X-Nacos-AgentSpec-Md5`
  - 200 响应携带 `X-Nacos-AgentSpec-Resolved-Version`
- 检索：`GET /v3/client/ai/agentspecs/search`

**对网关项目的启示**：AgentSpec 是"Agent 之间的依赖契约"——网关可以把它作为 Agent 组合拓扑的声明式规范，在启动期解析所有精确版本依赖，避免运行时的动态发现不确定性。

---

## 第 4 章  治理流水线

参考文章强调："所有资产变更需经过从草稿到在线的 Pipeline，包含格式检查、安全扫描、任务评测和人工判断。"

### 4.1 Nacos 的发布流水线

**关键类**（`ai/src/main/java/com/alibaba/nacos/ai/pipeline/`）：
- `PublishPipelineManager`：路由到匹配的流水线
- `PublishPipelineExecutor`：执行流水线节点
- 流水线状态写入 `AiResourceVersion.publishPipelineInfo` 与 `pipeline_execution`

**流水线插件 SPI**（`plugin/ai/src/main/java/com/alibaba/nacos/plugin/ai/pipeline/spi/PublishPipelineService.java`）：
- 可接入外部 CI、审查、漏洞扫描
- 默认实现：`plugin-default-impl/nacos-default-ai-pipeline-plugin`

**流水线上下文模型**（`plugin/ai/src/main/java/com/alibaba/nacos/plugin/ai/pipeline/model/`）：
- `PublishPipelineContext`：通用上下文
- `SkillPipelineContext` / `AgentSpecPipelineContext` / `ResourceFilesPipelineContext`：类型化上下文
- `PublishPipelineResult`：执行结果
- `Checkpoint`：流水线检查点
- `ResourceFileContent`：资源文件内容

### 4.2 流水线与生命周期的协作

参见 `ai-resource-lifecycle-spec.md` 第 4 节：
- `submit` 触发流水线执行
- 流水线通过 → 版本变为 `reviewed`，等待显式 publish
- 流水线拒绝 → 版本变为 `reviewed`，用户必须显式 redraft
- `force-publish` 绕过流水线，但必须保持为管理操作（审计日志记录）

### 4.3 与参考文章的对齐

| 参考文章 | Nacos 实现 |
|---|---|
| 格式检查 | `PublishPipelineService` 节点 |
| 安全扫描 | `PublishPipelineService` 节点 |
| 任务评测 | `PublishPipelineService` 节点（可接入外部评测系统） |
| 人工判断 | `reviewed` 状态等待显式 publish（Admin 决策点） |
| 灰度发布 | `versionInfo.labels`（`stable` / `canary` 等自定义标签） |

**对网关项目的启示**：
- 网关的 AI 资产发布流程可以照搬这套 draft → reviewing → reviewed → online 状态机
- 流水线 SPI 化，让网关可以根据企业合规要求插入自定义检查节点
- 把 `force-publish` 作为管理操作保留（带审计日志），是处理紧急情况的必要出口

---

## 第 5 章  双模发现机制

参考文章提出双模发现路径：稳态 Agent 通过 AgentSpec 声明式依赖；动态 Agent 通过 ARD/RAD 协议按需发现。

### 5.1 稳态路径：AgentSpec 声明式依赖

参见 3.5 节。AgentSpec 在构建、部署或启动期解析精确版本，避免运行时不确定性。

### 5.2 动态路径：ARD + RAD 组合

**ARD（Agentic Resource Discovery）**：外部协议，由 `ai-registry-adaptor` 暴露
- `ArdSearchController`：`POST /v3/ai/ard/search`——跨 namespace 搜索最新 online 的 Agent / Skill / Prompt / MCP
- `ArdWellKnownController`：`GET /.well-known/ai-catalog.json`——host 维度的 ARD catalog
- 还有 `/v3/ai/ard/explore`（facets）、`/v3/ai/ard/agents`（过滤列出）、`/v3/ai/ard/ai-catalog.json`（namespace catalog）、`/v3/ai/ard/artifacts`（artifact 读取）

**RAD（Remote Agent Discovery）**：Nacos 内部协议，协议版本 0.5.0
- 客户端 SDK：`A2aRadClientAdapter`（`client/.../ai/remote/A2aRadClientAdapter.java`）
- 五大操作：Search / Discover / Watch / Register / Deregister
- 传输绑定：HTTP、gRPC 与 Java SDK

**AI Resource Search（共享检索核心）**：
- `AiResourceSearchRepository`（接口）+ `JdbcAiResourceSearchRepository`（JDBC 实现）
- `AiResourceEmbeddingService`（接口）+ `HashingAiResourceEmbeddingService`（默认哈希实现）
- 关系索引表：`ai_resource_search_document` / `ai_resource_search_chunk` / `ai_resource_task`
- 向量索引：可选 PostgreSQL pgvector（`ai_resource_search_embedding_pg` 表），插件实现 `PostgresqlAiResourceVectorIndex`
- 索引一致性：资源写入是事实源；document + chunk 在同一数据源事务中原子替换；关系索引与向量索引之间不要求分布式事务，通过幂等的 `ai_resource_task` 任务 + 周期性 reconciliation 收敛

### 5.3 检索 vs 协议适配的职责划分

参见 `specs/zh-cn/ai/ai-resource-search-spec.md` 第 2 节：

| 职责 | 归属 |
|---|---|
| 标准 search document/chunk/facet 投影 | `ai` 模块（AI Resource Search） |
| 资源类型处理器注册、Query Planner、多召回通道融合 | `ai` 模块 |
| 持久化索引任务与 reconciliation | `ai` 模块 |
| 关键词召回、可选向量召回 | `ai` 模块 |
| 请求解析、协议校验、类型专用 filter 转换 | 协议适配器（ARD / RAD / 资源专用 Facade） |
| 响应 DTO、错误映射、artifact URL | 协议适配器 |

**关键设计亮点**：
- **一个检索核心，多个协议 facade**——RAD、ARD、通用 Search、资源专用 Search 复用同一套索引与 Query Planner，避免多套搜索引擎。
- **向量索引是可选召回通道，不是必选**——关键词 + 结构化过滤是基础 Search 的必选组成；向量索引通过插件按需启用。这让 Nacos 在没有 pgvector 的环境下也能跑 ARD。
- **readiness 是类型共享信号**——某个资源类型的 projection generation 未 READY 时，所有 Search Facade 仍然可用，返回当前快照，不回退到全量扫描。

**对网关项目的启示**：
- 网关接入 Nacos AI Registry 时，优先使用 SDK 的 RAD 能力（`A2aRadClientAdapter`）；如果网关需要暴露给外部 MCP 客户端，可以启用 `ai-registry-adaptor` 的 ARD 端口
- 网关自研检索时，可以参考"共享检索核心 + 协议 facade"的分层——避免为每种协议维护独立索引

---

## 第 6 章  端点发现与健康检查

参考文章强调："注册中心需要提供端点发现、健康检查、订阅推送与负载均衡能力。" Nacos 的 AI Registry **直接复用 Naming 引擎**——这是 Nacos 作为"微服务 + AI 资产"双栈注册中心的最大红利。

### 6.1 Naming v2 核心类

`naming/src/main/java/com/alibaba/nacos/naming/core/v2/`：

| 包/类 | 角色 |
|---|---|
| `pojo/Service.java` | v2 服务模型（namespace+group+name 三元组） |
| `service/ClientOperationService.java` | 注册/注销入口 |
| `service/impl/` | ServiceStorage、Service 列表维护 |
| `index/` | 服务索引 |
| `metadata/` | 实例元数据 |
| `client/` + `client/impl/` + `client/managers/` | 客户端连接管理 |
| `event/` | Service/Client/Metadata 事件 |

### 6.2 MCP / Agent 端点的 Naming 复用

- **MCP 端点**：`McpEndpointOperationService`（`ai/.../service/`）把 MCP endpoint 建模为 Naming 实例——ref 模式复用用户 Service，direct 模式由 Nacos 自建 Service
- **Agent 端点**：通过 `AgentEndpointPublicationManager`（`client/.../ai/`）与 RAD Register/Deregister 操作上报
- 健康检查：复用 Naming 的 TCP/HTTP/MySQL 探针；ephemeral 实例由客户端心跳保活（gRPC 连接感知或显式 beat），持久实例由服务端主动探测

### 6.3 推送与订阅

Nacos 采用"客户端长轮询 + gRPC 服务端推送"双通道：
- **HTTP 长轮询**（`config/.../service/LongPollingService.java`）：客户端发 HTTP 请求带 MD5 列表，服务端 hold 住请求（默认 30s），期间若有变更立即返回
- **gRPC 推送**（v2 客户端）：客户端通过 gRPC 长连接注册 listener，服务端 `NotifyCenter`（`common/.../notify/NotifyCenter.java`）监听变更事件直接下推
- **Skill / AgentSpec 的特殊处理**：这两类资源**不提供推送通道**，客户端通过周期性条件查询（md5 ETag + 任务尾端自调度）实现监听——参见 3.2、3.5 节

### 6.4 双模一致性

参见 `consistency/src/main/java/com/alibaba/nacos/consistency/ConsistencyProtocol.java`：

| 协议 | 实现 | 用途 | 一致性模型 |
|---|---|---|---|
| Distro | `core/distributed/distro/DistroProtocol.java` | 临时（ephemeral）实例数据、客户端元数据 | **AP**，按 responsibility 分片，对等节点互同步 |
| JRaft | `core/distributed/raft/JRaftProtocol.java` | 持久实例、命名空间、配置、AI 资源元数据、分布式锁 | **CP**，Raft 选主 + 日志复制 |

`ProtocolManager`（`core/distributed/ProtocolManager.java`）持有两个协议实例，按数据类型路由。

**对 AI Registry 的意义**：
- AI 资源的**元数据**（`ai_resource` / `ai_resource_version`）走**持久化 + CP**（JRaft/DB），保证一致性
- AI 资源的**运行时端点**（MCP endpoint / Agent endpoint）走**Naming 的 Distro AP**，保证高可用低延迟
- 这种 AP/CP 双模是 Nacos 架构的核心亮点，参考文章没有明确讨论这一点，但实际生产部署中这是必须权衡的

**对网关项目的启示**：
- 网关的 AI 端点发现可以直接复用 Nacos SDK 的 `selectInstances` / `selectOneHealthyInstance`，自动获得健康检查、负载均衡、订阅推送能力
- 如果网关自研 Registry，可以参考 Distro+JRaft 双模：端点走 AP（高可用），元数据走 CP（强一致）

---

## 第 7 章  统一变更评审与安全

参考文章强调："AI 资产的变更需要统一的评审机制与细粒度的权限控制。"

### 7.1 鉴权框架

`auth/` 模块（`auth/src/main/java/com/alibaba/nacos/auth/`）：

- **注解**：`@Secured(action=READ|WRITE, signType=CONFIG|NAMING|AI|CONSOLE, apiType=OPEN_API|ADMIN_API|CONSOLE_API)`
- **服务**：`ProtocolAuthService` → `HttpProtocolAuthService` / `GrpcProtocolAuthService`
- **资源解析器**：
  - `parser/http/AiCapabilityHttpResourceParser.java`：解析 AI HTTP 路径
  - `parser/grpc/AiGrpcResourceParser.java`：解析 AI gRPC 请求
- **身份上下文**：`context/HttpIdentityContextBuilder.java` / `context/GrpcIdentityContextBuilder.java`
- **默认实现**：`plugin-default-impl/nacos-default-auth-plugin`（JWT + Bcrypt）
- **可扩展**：`nacos-oidc-auth-plugin` / `nacos-ldap-auth-plugin`，或自定义 RAM 鉴权

### 7.2 可见性

`plugin/visibility/` 模块：
- `VisibilityService` SPI：决定 AI 资源是 PUBLIC 还是 PRIVATE
- `VisibilityHelper`（`ai/.../service/VisibilityHelper.java`）：AI 模块内的可见性辅助
- 跨 namespace 可见性：通过 scope 字段 + visibility query advice 控制

参见 `ai-resource-model-spec.md` 第 6 节：
- 创建操作通过可见性服务解析默认 scope
- 读操作中，资源存在但不可见 → 返回 not found（避免信息泄露）
- 写操作修改前必须检查写权限
- 查询操作使用 visibility query advice（数据库层过滤），而不是先读大结果集再过滤

### 7.3 Trace 与审计

`plugin-default-impl/nacos-default-ai-trace-plugin/`：
- `AiResourceTraceLogSubscriber`：订阅 `AiResourceTraceEvent`，写入 `ai-resource-trace.log`（JSON 行格式）
- 覆盖事件：create draft / update draft / submit / review approved/rejected / publish / force publish / online/offline / delete / label update / description update / scope update / download

### 7.4 安全边界

- Descriptor、extension、Endpoint metadata **不得包含明文 credential**
- 审计记录**不得记录完整 native descriptor、security scheme 或敏感 Endpoint metadata**
- Prompt / Skill 的 Search chunk **过滤 credential、secret 默认值与运行时参数值**

**对网关项目的启示**：
- 网关暴露 AI Registry 端点时，应该使用 `@Secured` + 资源解析器模式，而不是硬编码权限判断
- 跨 namespace 的可见性必须下沉到数据库层（visibility query advice），避免在应用层过滤大结果集
- 网关的审计日志可以参考 `AiResourceTraceEvent` 的设计——把 AI 资产的所有变更事件化

---

## 第 8 章  Context 装配：发现与执行的分离

参考文章提出："发现结果需经 Context Compiler 处理，分离'发现摘要'与'执行内容'，依据优先级和信任边界装配 Token。"

### 8.1 客户端 SDK 层的实现

`client/src/main/java/com/alibaba/nacos/client/ai/`：

**Transport 路由**：
- `McpTransportRouter` + `McpGrpcTransport` / `McpHttpTransport`：MCP 端点的 HTTP/gRPC 自动路由
- `AgentTransportRouter` + `AgentGrpcTransport` / `AgentHttpTransport`：Agent 端点的 HTTP/gRPC 自动路由
- `PromptTransportRouter` / `SkillTransportRouter`：Prompt 与 Skill 的传输路由
- `AgentTransport`：通用 Agent 传输抽象

**Watch 管理**：
- `AgentWatchManager`（`client/.../ai/watch/AgentWatchManager.java`）：Agent 监听管理
- `AgentWatchTransportRouter` + `GrpcAgentWatchTransport` / `HttpAgentWatchTransport` / `PollingAgentWatchTransport`：多传输监听
- `A2aRadWatchAdapter`：RAD Watch 协议适配
- `AgentWatchRetryPolicy` / `AgentWatchClientMetrics`：重试策略与客户端指标

**RAD 适配**：
- `A2aRadClientAdapter`（`client/.../ai/remote/A2aRadClientAdapter.java`）：RAD 协议客户端
- `AgentCapabilityResolver`（`client/.../ai/remote/AgentCapabilityResolver.java`）：Agent 能力解析与协商

**事件模型**（`client/.../ai/event/`）：
- `PromptChangedEvent` / `McpServerChangedEvent` / `SkillChangedEvent` / `AgentCardChangedEvent` / `AgentSpecChangedEvent`：五类资产变更事件
- `AiChangeNotifier`：变更通知聚合
- 各 `XxxListenerInvoker`：把 SDK 内部事件分发给业务 listener

### 8.2 分离的体现

参考文章强调的"发现摘要 vs 执行内容分离"在 Nacos 中的体现：

1. **RAD Discover 返回调用元数据，但不代理调用**：
   - `AgentDiscoveryResult` 包含 `callInterfaces[]` 与 `endpointSets[]`，但不包含 Agent message/task/session/stream
   - 客户端拿到 Discover 结果后，自己选择 transport、发起调用、处理重试

2. **Search 返回目录信息，不返回完整内容**：
   - `AgentCatalogPage` / `McpServerBasicInfo` / `SkillBasicInfo` / `PromptMetaSummary` / `AgentSpecBasicInfo`：只包含目录字段（name、description、tags、protocols、labels）
   - 完整内容需要按 version/label 拉取

3. **Labels 与 version 的路由在客户端解析**：
   - 客户端可以按 `latest`、明确 version、自定义 label 查询
   - `X-Nacos-Skill-Resolved-Version` / `X-Nacos-AgentSpec-Resolved-Version` 响应头告诉客户端解析出的真实版本

**对网关项目的启示**：
- 网关作为 AI 资产的消费方时，可以参考 Nacos SDK 的"Router + Transport"模式：
  - Router 决定用哪个传输（HTTP/gRPC）
  - Transport 处理具体协议细节
  - Watch Manager 处理订阅与重连
- 网关作为 AI 资产的聚合方时，可以：
  - 用 Search 拉目录（小 Token 成本）
  - 用 Discover / 精确版本查询拉完整内容（大 Token 成本）
  - 本地缓存 + md5/ETag 增量更新

---

## 第 9 章  插件与扩展性边界

参考文章强调："Registry 应该提供清晰的扩展点，让企业可以插入自定义的检查、存储、索引逻辑。"

### 9.1 Nacos 的 10 类插件 SPI

`plugin/` 模块定义的插件契约：

| 类型 | SPI 接口 | 默认实现 |
|---|---|---|
| auth | `AuthPluginService` | `nacos-default-auth-plugin` / `nacos-oidc-auth-plugin` / `nacos-ldap-auth-plugin` |
| visibility | `VisibilityService` | （内置默认实现） |
| datasource | 数据库方言 SPI | `nacos-default-datasource-plugin`（MySQL/PostgreSQL/Derby/Oracle） |
| config | 配置变更 SPI | （内置默认实现） |
| encryption | 配置加解密 SPI | （内置默认实现） |
| trace | 可观测性 SPI | `nacos-default-ai-trace-plugin`（AI 资源审计） |
| environment | 自定义环境 SPI | （内置默认实现） |
| control | 流控/连接控制 SPI | `nacos-default-control-plugin` |
| **ai** | 4 个子 SPI（见下） | `nacos-default-ai-pipeline-plugin` / `nacos-default-ai-vector-plugin` / `nacos-default-ai-importer-plugin` |
| lock | 分布式锁 | （基于 JRaft CP） |

### 9.2 四类 AI 插件 SPI

`plugin/ai/src/main/java/com/alibaba/nacos/plugin/ai/`：

| SPI | 路径 | 作用 | 默认实现 |
|---|---|---|---|
| `AiResourceStorage` / `AiResourceStorageBuilder` | `storage/spi/` | AI 资源内容存储（可替换默认 Config 存储） | `NacosConfigAiResourceStorage`（`ai/.../storage/`） |
| `PublishPipelineService` | `pipeline/spi/` | 发布流水线（接入 CI、审查、漏洞扫描等） | `nacos-default-ai-pipeline-plugin` |
| `AiResourceVectorIndex` / `AiResourceVectorIndexBuilder` | `vector/spi/` | 向量索引实现 | `PostgresqlAiResourceVectorIndex`（`nacos-default-ai-vector-plugin`） |
| `AiResourceImportService` / `AiResourceImportServiceBuilder` | `importer/spi/` | 外部注册中心导入 | `SkillsShImportService` / `SkillWellKnownImportService` / `McpRegistryImportService` / `McpOfficialImportService` |

### 9.3 加载机制

- 标准 **Java SPI**（`META-INF/services/`）
- 通过 `PluginProvider`（`api/.../plugin/PluginProvider.java`）发现
- `PluginStartupLifecycle` 管理插件启动顺序
- `PluginConfigSpec` 定义插件配置 Schema

### 9.4 插入新资源类型的套路

按以下顺序：
1. 在 `ai/.../model/` 定义新资源类型的数据结构
2. 继承 `AiResource` 扩展元数据（如果需要）
3. 在 `ai/.../controller/` 增加 `{Resource}AdminController` + `{Resource}ClientController`
4. 复用 `AiResource` / `AiResourceVersion` 表与 `NacosConfigAiResourceStorage`（或自定义 `AiResourceStorage` 插件）
5. 新增客户端 SDK 子能力（类似 `PromptService` / `SkillService`）
6. 在 `AiService` 增加 default 访问器
7. 注册 Search 处理器（`AiResourceSearchRepository` 的类型处理器）

**对网关项目的启示**：
- 网关自研 Registry 时，可以照搬 Nacos 的 4 类 AI SPI：
  - **Storage SPI**：让企业把 AI 资产内容存到自建的对象存储 / Git / 数据库
  - **Pipeline SPI**：让企业插入合规检查节点
  - **Vector SPI**：让企业接入自建向量库（Milvus / Pinecone / Weaviate）
  - **Import SPI**：让企业从外部市场（HuggingFace / GitHub / 内部 GitLab）导入 AI 资产
- 插件加载使用 Java SPI + `META-INF/services/` 是最简单的扩展机制，但生产部署时应该考虑插件的版本管理与灰度发布

---

## 第 10 章  社区协议兼容

参考文章强调："Registry 应该支持社区标准协议，让现有工具可以无缝接入。"

### 10.1 `ai-registry-adaptor` 模块

`ai-registry-adaptor/src/main/java/com/alibaba/nacos/airegistry/`：

**启动与开关**：
- `NacosAiRegistry` / `NacosAiRegistryStartUp`：独立 Spring Boot 应用
- `@ConditionalOnArdEnabled` 等条件注解
- 默认端口 9080，可通过 `nacos.ai.registry.port` 配置
- 三个独立开关：
  - `nacos.ai.mcp.registry.enabled`（默认 false）
  - `nacos.ai.skill.registry.enabled`（默认 false）
  - `nacos.ai.ard.enabled`（默认 false）

**MCP Registry 兼容**（`McpRegistryController` + `NacosMcpRegistryService`）：

| 方法 | 路径 | 行为 |
|---|---|---|
| GET | `/v0/servers` | 按 cursor/limit/search 列出 MCP servers |
| GET | `/v0/servers/{name}/versions` | 列出指定 server 的版本 |
| GET | `/v0/servers/{serverName}/versions/{version}` | 返回指定 server 版本（支持 `latest`） |

**Skill Registry 兼容**（`SkillsRegistryController` + `NacosSkillsRegistryService`）：

| 方法 | 路径 | 行为 |
|---|---|---|
| GET | `/registry/{namespaceId}/.well-known/agent-skills/index.json` | Agent Skills discovery v0.2.0 index |
| GET | `/registry/{namespaceId}/.well-known/skills/index.json` | legacy v0.1 兼容 index |
| GET | `/registry/{namespaceId}/api/search` | CLI 兼容搜索 |
| GET | `/registry/{namespaceId}/.well-known/agent-skills/{skillName}/SKILL.md` | 导出 SKILL.md |
| GET | `/registry/{namespaceId}/.well-known/agent-skills/{skillName}.zip` | 导出 Skill archive |

**ARD 兼容**（`ArdSearchController` + `ArdWellKnownController`）：

| 方法 | 路径 | 行为 |
|---|---|---|
| POST | `/v3/ai/ard/search` | 跨 namespace 搜索 Agent/Skill/Prompt/MCP |
| POST | `/v3/ai/ard/explore` | 返回 facets |
| GET | `/v3/ai/ard/agents` | 按过滤条件列出 |
| GET | `/v3/ai/ard/ai-catalog.json` | namespace 维度 catalog |
| GET | `/v3/ai/ard/artifacts` | artifact 读取 |
| GET | `/.well-known/ai-catalog.json` | host 维度 catalog |

### 10.2 协议契约的严格遵循

参见 `specs/zh-cn/ai/ai-registry-adaptor-spec.md` 第 6.1 节：
- 列表响应使用 `items`，不得重命名为 `results`
- 搜索结果的 `score` 为 0 到 100 的整数
- 搜索结果的 `source` 为标识当前 registry 的绝对 URI
- ARD 错误严格使用 `{ "errorCode": "...", "message": "..." }`，不得包裹为 Nacos `Result<T>`
- ARD 模块保存固定版本的上游 OpenAPI 与 Schema 测试 fixture，自动化测试必须使用这些 fixture 校验

### 10.3 让外部 MCP 客户端直接使用 Nacos

启用适配器后，以下工具可以直接把 Nacos 当作官方 Registry：
- **Claude Code**：通过 MCP Registry 协议
- **Cursor**：通过 MCP Registry 协议
- **MCP Inspector**：通过 MCP Registry 协议
- **skills CLI**：通过 skills.sh 协议
- **ARD 客户端**：通过 ARD 协议

**关键设计亮点**：
- **适配器是可选兼容面**——默认关闭，运维主动开启
- **适配器只负责协议转换，不重新实现资源模型**——调用内部 `McpOperationService` / `SkillOperationService` 等
- **适配器被视为公共协议端点**——运维应通过可信网络、网关鉴权、TLS、限流等外部保护措施暴露

**对网关项目的启示**：
- 网关如果想让外部 MCP 客户端直接访问内部 AI Registry，可以启用 `ai-registry-adaptor` 模块
- 网关也可以自己实现一套 ARD/MCP Registry 协议适配层——Nacos 的 `airegistry` 包是现成的参考实现
- 协议契约必须严格遵循上游 Schema——Nacos 在 `ai-registry-adaptor` 模块保存了固定版本的上游 OpenAPI 与测试 fixture，作为一致性测试基线

---

## 第 11 章  成熟度模型对照

参考文章提出 L0（分散维护）到 L5（持续改进）的成熟度模型。下面是 Nacos 3.x 的能力对照：

| 成熟度级别 | 参考文章描述 | Nacos 3.x 状态 |
|---|---|---|
| **L0 分散维护** | 各团队各自维护 Prompt/Skill/MCP | ✅ 已解决——统一 Registry |
| **L1 权威 Registry** | 建立权威 Registry 与版本控制 | ✅ 完整实现——`ai_resource`/`ai_resource_version` 双表 + labels |
| **L2 治理流水线** | 草稿→审核→上线的 Pipeline | ✅ 完整实现——`PublishPipelineManager` + SPI |
| **L3 动态发现** | ARD/RAD 协议支持 | ✅ 完整实现——`ai-registry-adaptor` + RAD 0.5.0 |
| **L4 自动化评测** | 流水线集成评测系统 | ⚠️ 部分实现——Pipeline SPI 可接入外部评测，但无内置评测系统 |
| **L5 持续改进** | 血缘、联邦集群、细粒度计费 | ⚠️ 待补充——血缘暂未原生支持；联邦集群有 sync 但非 AI 专用；计费通过 control 插件扩展 |

### 11.1 Nacos 已经具备的能力 ✅

- 五类 AI 资产的统一资源模型与版本化
- 完整的生命周期治理（draft → reviewing → reviewed → online/offline）
- 标签化（latest / stable / 自定义）→ 灰度发布
- 端点注册与健康检查（复用 Naming）
- 订阅/推送/Watch（gRPC 推送 + HTTP 长轮询 + fuzzyWatch 通配）
- 多租户隔离（Namespace 维度）
- 插件化（4 类 AI 插件 SPI：storage / pipeline / vector / importer）
- 社区协议兼容（MCP Registry / Skills.sh / ARD / `.well-known`）
- 向量检索 / RAG（PostgreSQL pgvector 插件）
- 外部导入（4 种默认导入器）
- 认证鉴权（`@Secured` + 资源解析器 + OIDC/LDAP 插件）
- AP/CP 双模一致性
- 双模发现（稳态 AgentSpec + 动态 ARD/RAD）

### 11.2 Nacos 还需要补充的能力 ⚠️

- **资产血缘（Lineage）**：暂未发现原生支持，需要通过 trace 插件或自定义插件扩展
- **联邦集群 AI Registry 同步**：Nacos 3.x 有 sync 机制，但跨集群 AI Registry 尚未看到专门 spec
- **细粒度计费/配额**：通过 control 插件扩展，但没有 AI 资源专属的计费模型
- **内置评测系统**：Pipeline SPI 可接入外部评测，但没有内置的评测数据集与评分机制
- **实例级细粒度 RBAC**：部分具备，依赖 visibility 插件扩展

**对网关项目的启示**：
- 网关项目可以直接使用 Nacos 3.x 达到 L1-L3 成熟度
- L4 需要自建评测系统并通过 Pipeline SPI 接入
- L5 的血缘、联邦集群、计费需要在 Nacos 之上做二次开发

---

## 第 12 章  对网关项目的启示

### 12.1 三种接入方案

**方案 A：直接使用 Nacos 3.x**
- 部署 Nacos 3.x 集群作为 AI Registry
- 网关通过 `NacosAiService` SDK 接入
- 启用 `ai-registry-adaptor` 暴露 ARD/MCP Registry 给外部客户端
- 优点：零自研，立刻达到 L3 成熟度
- 缺点：引入 Nacos 集群运维成本

**方案 B：自研轻量级 AI Registry，复用 Nacos 的设计蓝图**
- 照搬 `AiResource` / `AiResourceVersion` 双表模型
- 照搬五类资产的 REST 路径设计（`/v3/{client,admin}/ai/{resource}`）
- 照搬 4 类 AI 插件 SPI
- 自研 AP/CP 双模一致性（或直接使用 etcd / ZooKeeper / PostgreSQL）
- 优点：完全自主，按需裁剪
- 缺点：自研工作量大，容易踩 Nacos 已经踩过的坑

**方案 C：混合方案**
- 元数据与生命周期治理使用 Nacos（或复用其设计）
- 端点发现与健康管理使用 Kubernetes Service / Istio（如果网关已经跑在 K8s 上）
- 向量检索使用自建向量库 + Nacos 的 Vector SPI 对接
- 优点：各取所长
- 缺点：集成复杂度高

### 12.2 关键设计蓝图

无论选哪种方案，以下几点都值得直接参考 Nacos 的实现：

1. **资源身份三元组**：`namespaceId / resourceType / resourceName`
2. **元数据/版本双表 + CAS 乐观锁**：处理高频变更
3. **Labels 与服务端维护 `latest`**：灰度发布与版本路由解耦
4. **状态机**：draft → reviewing → reviewed → online/offline
5. **4 类 AI 插件 SPI**：storage / pipeline / vector / importer
6. **AP/CP 双模一致性**：端点走 AP（高可用），元数据走 CP（强一致）
7. **共享检索核心 + 多协议 facade**：避免为每种协议维护独立索引
8. **RAD 协议的 5 大操作**：Search / Discover / Watch / Register / Deregister
9. **Endpoint 自然键与 sourceRevision**：端点身份与内容变化分离
10. **整批替换的 Register 模型**：避免复杂的增量同步

### 12.3 关键文件索引

**规范文档**（权威设计蓝图，位于 `nacos-develop/specs/`）：
- `zh-cn/ai/ai-registry-spec.md`：AI Registry 总览
- `zh-cn/ai/ai-resource-model-spec.md`：资源模型
- `zh-cn/ai/ai-resource-lifecycle-spec.md`：生命周期
- `zh-cn/ai/ai-resource-search-spec.md`：检索能力
- `zh-cn/ai/ai-registry-adaptor-spec.md`：协议适配器
- `zh-cn/ai/mcp-server-spec.md` / `prompt-spec.md` / `skill-spec.md` / `agent-management-spec.md` / `agentspec-spec.md`：五类资产
- `zh-cn/ai/rad-protocol-spec.md`：RAD 协议
- `en/design/resource-model-spec.md`：通用资源模型

**API 契约**（位于 `nacos-develop/api/`）：
- `src/main/java/com/alibaba/nacos/api/ai/AiService.java`
- `src/main/java/com/alibaba/nacos/api/ai/AiFactory.java`
- `src/main/java/com/alibaba/nacos/api/ai/{McpService,A2aService,SkillService,AgentSpecService,PromptService}.java`
- `src/main/java/com/alibaba/nacos/api/ai/model/`：五类资产的 DTO

**AI 核心实现**（位于 `nacos-develop/ai/src/main/java/com/alibaba/nacos/ai/`）：
- `controller/`：15 个 REST Controller
- `service/`：`McpEndpointOperationService` / `McpServerValidationService` / `SkillDownloadCountManager` / `VisibilityHelper` / `SyncEffectService`
- `service/search/`：`AiResourceSearchRepository` / `JdbcAiResourceSearchRepository` / `AiResourceEmbeddingService` / `HashingAiResourceEmbeddingService`
- `storage/`：`NacosConfigAiResourceStorage` / `NacosConfigAiResourceStorageBuilder`
- `pipeline/`：`PublishPipelineManager` / `PublishPipelineExecutor`
- `importer/`：`AiResourceImportManager`
- `remote/handler/`：gRPC 请求处理器
- `model/`：`AiResource` / `AiResourceVersion`

**AI 适配器**（位于 `nacos-develop/ai-registry-adaptor/`）：
- `NacosAiRegistry.java` / `NacosAiRegistryStartUp.java`
- `controller/McpRegistryController.java` / `SkillsRegistryController.java` / `ArdSearchController.java` / `ArdWellKnownController.java`
- `service/NacosMcpRegistryService.java` / `NacosSkillsRegistryService.java`
- `model/ard/` / `model/skills/`

**插件 SPI**（位于 `nacos-develop/plugin/ai/`）：
- `storage/spi/AiResourceStorage.java` + `AiResourceStorageBuilder.java`
- `pipeline/spi/PublishPipelineService.java`
- `vector/spi/AiResourceVectorIndex.java` + `AiResourceVectorIndexBuilder.java`
- `importer/spi/AiResourceImportService.java` + `AiResourceImportServiceBuilder.java`

**默认插件实现**（位于 `nacos-develop/plugin-default-impl/`）：
- `nacos-default-ai-pipeline-plugin/`
- `nacos-default-ai-trace-plugin/`：`AiResourceTraceLogSubscriber`
- `nacos-default-ai-vector-plugin/`：`PostgresqlAiResourceVectorIndex`
- `nacos-default-ai-importer-plugin/`：`SkillsShImportService` / `SkillWellKnownImportService` / `McpRegistryImportService` / `McpOfficialImportService`

**客户端 SDK**（位于 `nacos-develop/client/src/main/java/com/alibaba/nacos/client/ai/`）：
- `NacosAiService.java`
- `remote/McpTransportRouter.java` / `AgentTransportRouter.java` / `PromptTransportRouter.java` / `SkillTransportRouter.java`
- `remote/A2aRadClientAdapter.java`
- `remote/AgentCapabilityResolver.java`
- `watch/AgentWatchManager.java`
- `event/`：五类变更事件

**基座能力**：
- `naming/.../core/v2/`：端点注册与健康检查
- `config/.../service/`：版本化存储、灰度规则、长轮询
- `consistency/.../ConsistencyProtocol.java` + `ap/APProtocol.java` + `cp/CPProtocol.java`
- `core/.../distributed/distro/DistroProtocol.java` + `core/.../distributed/raft/JRaftProtocol.java`
- `auth/`：`@Secured` + 资源解析器
- `common/.../notify/NotifyCenter.java`：事件总线

---

## 结语

Nacos 3.x 把自身从"微服务双引擎"演进为"微服务 + AI 资产双栈注册中心"，完整实现了参考文章提出的 Agentic Resource Registry 概念。它的设计有几个值得学习的特征：

1. **演进式架构**：通过 `SYNCING → LIFECYCLE_MANAGED` 状态机平滑迁移存量 MCP 资源，而不是大爆炸式切换
2. **职责分离**：元数据 vs 版本、capability 定义 vs runtime endpoint、管理面 vs 运行时、检索核心 vs 协议 facade
3. **插件化边界**：4 类 AI 插件 SPI 让企业可以按需替换存储、流水线、向量索引、导入源
4. **协议兼容**：把 Nacos 暴露为 MCP Registry / Skills.sh / ARD 公共端点，让现有工具无缝接入
5. **AP/CP 双模一致性**：端点走 AP 高可用、元数据走 CP 强一致——这是生产部署的关键权衡

对网关项目而言，**优先推荐方案 A（直接使用 Nacos 3.x）**——它已经经过了阿里巴巴大规模生产验证，规范文档（中英双语共 17 份 AI spec）详尽，插件生态成熟。只有当运维成本不可接受或有特殊合规要求时，才考虑方案 B 或 C。
