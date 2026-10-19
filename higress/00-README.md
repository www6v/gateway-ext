# Higress AI Gateway 源码分析

> 参考文档：[第 9 章 AI 网关与统一流量治理](https://github.com/aliyun/ai-agent-handbook/blob/main/03-run/第%209%20章%20%20AI%20网关与统一流量治理.md)
>
> 分析对象：[higress-group/higress](https://github.com/higress-group/higress) CNCF Sandbox 项目
>
> 分析日期：2026-10-19

---

## 文档索引

| 文档 | 内容 |
|------|------|
| [01-架构总览](./01-overview.md) | 项目定位、核心架构、控制面/数据面设计、部署模式 |
| [02-LLM 网关能力](./02-llm-gateway.md) | 模型路由、可靠性、预算治理、超时与流式处理、可观测性 |
| [03-MCP 网关能力](./03-mcp-gateway.md) | MCP 协议支持、工具路由、安全边界、REST-to-MCP 桥接 |
| [04-Agent 网关能力](./04-agent-gateway.md) | Agent 编排、任务关联、租户隔离、账本与归因 |
| [05-插件生态](./05-plugin-ecosystem.md) | Wasm 插件框架、AI 插件全 catalog、非 AI 插件概览 |
| [06-扩展点](./06-extension-points.md) | 用户扩展 Higress 的所有方式 |
| [07-差距分析](./07-gap-analysis.md) | 对照第 9 章治理框架的能力覆盖与缺口 |

## 一句话总结

Higress 是一个以 **Istio 控制面 + Envoy 数据面 + Wasm 插件生态** 为底座的 AI Native API Gateway，在 LLM 代理与 MCP 托管两个方向上提供了业界最完整的开源实现；对照第 9 章定义的三层治理语义（LLM / MCP / Agent Gateway），它在 LLM 和 MCP 层面覆盖度高，在 Agent 层面的任务归因、严格预算和审批链路上存在结构性缺口。
