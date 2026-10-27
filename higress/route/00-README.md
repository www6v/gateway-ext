# Higress 路由策略全解析

> 分析对象：[higress-group/higress](https://github.com/higress-group/higress) CNCF Sandbox 项目
>
> 分析日期：2026-10-27

---

## 文档索引

| 文档 | 内容 |
|------|------|
| [01-路由配置面](./01-route-configuration-surfaces.md) | CRD、Ingress Annotation、Gateway API 三种配置入口 |
| [02-核心路由策略](./02-core-routing-strategies.md) | 路径、Header、权重、灰度、镜像、负载均衡、重试、回退 |
| [03-LLM/AI 路由](./03-llm-ai-routing.md) | 模型路由、语义路由、端点选择、协议转换、AI 代理 |
| [04-MCP 路由](./04-mcp-routing.md) | MCP 工具路由、REST-to-MCP 桥接、协议适配 |
| [05-流量治理策略](./05-traffic-management.md) | 限流、配额、认证、WAF、转换、染色 |
| [06-典型路由编排](./06-routing-pipelines.md) | 端到端路由流水线示例 |

## 一句话总结

Higress 提供**四层路由体系**：控制面翻译层（Ingress/Gateway API → Istio VirtualService）、WASM 插件链（AI/MCP/流量治理/安全）、服务发现层（McpBridge 统一 Nacos/Consul/Eureka/ZK/DNS）、推理调度层（GIE v1.4 InferencePool + ai-endpoint-picker），覆盖路径/Header/Query/权重/灰度/镜像/意图/语义/模型/MCP 全维度路由，并通过统一 WasmPlugin 绑定模型实现策略即代码。

## 整体架构图

```mermaid
graph TB
    subgraph "配置入口层"
        A1["Ingress Annotations<br/>higress.io/* + nginx.*"]
        A2["Gateway API<br/>HTTPRoute / Gateway"]
        A3["WasmPlugin CRD<br/>extensions.higress.io"]
        A4["McpBridge CRD<br/>服务发现注册"]
    end

    subgraph "控制面 (Istio pilot)"
        B1["Ingress Controller"]
        B2["Gateway API Controller"]
        B3["Annotation 解析器<br/>canary / mirror / LB / retry"]
        B4["VirtualService 翻译器"]
        B5["Envoy xDS<br/>配置下发"]
    end

    subgraph "数据面 (Envoy + WASM)"
        direction LR
        C1["Listener<br/>端口/协议"]
        C2["Route<br/>路径/Header/权重"]
        C3["Filter Chain<br/>WASM 插件链"]
        C4["Upstream<br/>负载均衡/连接池"]
    end

    subgraph "WASM 插件生态"
        D1["AI 路由<br/>model-router<br/>ai-proxy<br/>ai-intent<br/>ai-endpoint-picker"]
        D2["MCP 路由<br/>mcp-router<br/>mcp-server"]
        D3["流量治理<br/>traffic-tag<br/>rate-limit<br/>quota"]
        D4["安全策略<br/>ext-auth<br/>waf / opa<br/>ip-restriction"]
        D5["协议转换<br/>transformer<br/>de-graphql<br/>ai-transformer"]
    end

    subgraph "服务发现 (McpBridge)"
        E1["Nacos"]
        E2["Consul"]
        E3["Eureka"]
        E4["ZooKeeper"]
        E5["DNS"]
    end

    subgraph "后端服务"
        F1["LLM Providers<br/>OpenAI/Qwen/Claude/..."]
        F2["MCP Servers"]
        F3["微服务集群"]
        F4["vLLM 推理集群"]
    end

    A1 --> B1
    A2 --> B2
    A3 --> C3
    A4 --> B1
    B1 --> B3 --> B4 --> B5
    B2 --> B4
    B5 --> C1
    C1 --> C2 --> C3 --> C4

    C3 --> D1 & D2 & D3 & D4 & D5

    C4 --> E1 & E2 & E3 & E4 & E5
    C4 --> F1 & F2 & F3 & F4
```

## 路由决策流程图

```mermaid
flowchart TD
    Start(["客户端请求"]) --> Match{"路由匹配"}

    Match -->|"路径/Header/Query<br/>权重/灰度"| Core["核心路由决策<br/>VirtualService"]

    Core --> Plugin["WASM 插件链执行"]

    subgraph "插件处理流水线"
        P1["① 认证插件<br/>jwt-auth / key-auth"] --> P2["② 限流插件<br/>rate-limit / quota"]
        P2 --> P3["③ 路由增强插件<br/>model-router / traffic-tag"]
        P3 --> P4["④ 安全插件<br/>waf / opa / ip-restrict"]
        P4 --> P5["⑤ 转换插件<br/>transformer / ai-proxy"]
        P5 --> P6["⑥ 观测插件<br/>statistics / log"]
    end

    Plugin --> P1

    P6 --> LB{"负载均衡"}
    LB -->|"普通服务"| Upstream["后端服务"]
    LB -->|"LLM 推理"| EPP["ai-endpoint-picker<br/>KV cache / Prefix / LoRA"]
    EPP --> Upstream

    Upstream --> Resp(["响应客户端"])

    style Start fill:#e1f5fe
    style Resp fill:#e8f5e9
    style Match fill:#fff3e0
    style LB fill:#fff3e0
```

## 路由策略分类速查

| 策略类型 | 实现方式 | 关键插件/Annotation |
|---------|---------|-------------------|
| 路径路由 | Ingress path/pathType | 原生 Ingress spec |
| Header 路由 | Annotation + VirtualService | `canary-by-header`, `traffic-tag` |
| 权重路由 | Ingress annotation | `higress.io/destination`, `canary-weight` |
| 灰度发布 | Annotation + Plugin | `canary-*`, `frontend-gray` |
| 流量镜像 | Annotation | `mirror-target-service`, `mirror-percentage` |
| 负载均衡 | Annotation + Plugin | `load-balance`, `ai-load-balancer` |
| 模型路由 | Plugin | `model-router`, `model-mapper` |
| 语义路由 | Plugin | `ai-intent`, `model-router.autoRouting` |
| 端点选择 | Plugin | `ai-endpoint-picker` |
| MCP 路由 | Plugin | `mcp-router`, `mcp-server` |
| 限流配额 | Annotation + Plugin | `local-rate-limit`, `ai-token-ratelimit`, `ai-quota` |
| 认证鉴权 | Plugin | `key-auth`, `jwt-auth`, `oidc`, `ext-auth` |
| 协议转换 | Plugin | `ai-proxy`, `de-graphql`, `jsonrpc-converter` |
