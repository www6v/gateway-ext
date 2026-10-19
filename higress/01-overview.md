# 01 — 架构总览

## 1. 项目定位

Higress 是阿里巴巴内部孵化、现归属 CNCF Sandbox 的云原生 AI API 网关。项目名取自"Hi"（AI 谐音）+ "gress"（Ingress 的词根），直观表明其"以 AI 为核心的入口网关"定位。

| 维度 | 说明 |
|------|------|
| 代码仓 | `github.com/higress-group/higress` |
| 开源协议 | Apache 2.0 |
| 主语言 | Go（控制面）+ C++/Rust/Go/AssemblyScript（Wasm 插件） |
| Go Module | `github.com/alibaba/higress/v2` |
| 成熟度 | CNCF Sandbox；已在阿里巴巴内部支撑数十万 RPS 生产负载 |
| 标准合规 | Kubernetes Ingress Controller、Gateway API Conformant、Gateway API Inference Extension Conformant |

## 2. 核心架构

```
                        ┌──────────────── Kubernetes Cluster ─────────────────┐
                        │  Ingress · Gateway API · Knative Ingress            │
                        │  WasmPlugin · McpBridge · McpServer · Http2Rpc CRDs │
                        └────────────────────────┬────────────────────────────┘
                                                 │ informers / watchers
                                                 ▼
            ┌──────────────── Higress 控制面 (cmd/higress) ────────────────────┐
            │                                                                   │
            │  pkg/ingress/kube/* 控制器                                         │
            │       ↓                                                           │
            │  pkg/ingress/translation/  ← 聚合 Ingress/Gateway/MCP 资源         │
            │       ↓                                                           │
            │  registry/* (Nacos/ZK/Consul/Eureka/Direct)                       │
            │       ↓                                                           │
            │  Istio aggregate config store → Istio xDS DiscoveryServer         │
            │       │  gRPC (delta-xDS / SotW)                                  │
            └───────┼───────────────────────────────────────────────────────────┘
                    ▼
            ┌────────────────── Envoy 数据面 ───────────────────────┐
            │  HTTP filters → Wasm plugins (Proxy-Wasm ABI)         │
            │  Go native filters (golang-filter)                    │
            │  Routes / Clusters / Listeners 由 xDS 动态下发         │
            └───────────────────────────────────────────────────────┘
```

### 关键设计要点

1. **控制面 = Istio 控制面的嵌入与扩展**
   - `cmd/higress/main.go` 仅 32 行，调用 `pkg/cmd.GetRootCommand()`
   - `pkg/bootstrap/server.go` 构造 `Server`，直接 import Istio 的 `pilot/pkg/xds.DiscoveryServer`、`pilot/pkg/serviceregistry/aggregate`、`pilot/pkg/model`
   - 本质是把 Istio 控制面作为库嵌入，用自己的 Ingress/Gateway/MCP 控制器替换标准控制器栈
   - `istio/` 目录是 git submodule 占位，通过 `go.mod` 的 `replace` 指令指向 `istio.io/istio`

2. **数据面 = 原版 Envoy + Wasm 插件**
   - `envoy/` 目录同样是 submodule 占位，通过 `go.mod replace` 指向 `go-control-plane` 和 Envoy proto
   - 插件通过 Proxy-Wasm ABI 在 Envoy 沙箱中运行，支持 Go / Rust / C++ / AssemblyScript
   - 特殊场景（如 MCP Server 高吞吐路径）使用 `golang-filter` 原生 Go Envoy filter

3. **配置翻译链 = K8s 资源 → Istio 内部配置 → xDS**
   - `pkg/ingress/translation/` 实现 `istiomodel.ConfigStoreController` 和 `istiomodel.IngressStore`
   - 把聚合的 Ingress / Gateway API / Knative / MCP 资源喂入 Istio 的 xDS 管线

## 3. 源码目录结构

```
higress/
├── cmd/higress/main.go          # 二进制入口
├── pkg/                         # 控制面核心 Go 包
│   ├── bootstrap/               #   Server 启动（xDS + gRPC + HTTP + 证书）
│   ├── cmd/                     #   Cobra 命令（server / version）
│   ├── cert/                    #   Let's Encrypt 证书管理
│   ├── config/                  #   常量与环境变量
│   ├── common/                  #   公共工具
│   ├── kube/                    #   K8s 客户端封装
│   └── ingress/                 #   ★ 核心翻译层
│       ├── config/              #     IngressConfig、Knative Ingress、Http2Rpc
│       ├── kube/                #     各资源控制器
│       │   ├── annotations/     #       nginx 兼容注解解析
│       │   ├── controller/      #       顶层控制器模型
│       │   ├── gateway/         #       Gateway API
│       │   ├── ingress/         #       networking.k8s.io/v1 Ingress
│       │   ├── kingress/        #       Knative Ingress
│       │   ├── mcpbridge/       #       McpBridge CRD
│       │   ├── mcpserver/       #       McpServer CRD
│       │   ├── wasmplugin/      #       WasmPlugin CRD
│       │   └── http2rpc/        #       Http2Rpc CRD
│       ├── mcp/                 #     MCP 路由生成器
│       └── translation/         #     IngressTranslation（聚合 → xDS）
├── api/                         # Protobuf + 生成的 CRD Go 代码
│   ├── networking/v1/           #   Http2Rpc, McpBridge
│   └── extensions/v1alpha1/     #   WasmPlugin
├── client/                      # 生成的 CRD Go 客户端
├── plugins/                     # ★ 数据面插件
│   ├── wasm-go/extensions/      #   Go Wasm 插件（~55 个，含全部 AI 插件）
│   ├── wasm-cpp/extensions/     #   C++ Wasm 插件（~12 个）
│   ├── wasm-rust/               #   Rust Wasm SDK
│   ├── wasm-assemblyscript/     #   AssemblyScript SDK
│   └── golang-filter/           #   原生 Go Envoy filter
├── registry/                    # ★ 服务发现注册中心
│   ├── nacos/                   #   Nacos v2
│   ├── consul/                  #   Consul
│   ├── eureka/                  #   Eureka
│   ├── zookeeper/               #   ZooKeeper
│   ├── direct/                  #   静态直连
│   └── memory/                  #   内存缓存
├── istio/                       # Istio submodule 占位（go.mod replace 填充）
├── envoy/                       # Envoy submodule 占位
├── helm/                        # Helm Charts
│   ├── core/                    #   实际 Chart（values.yaml + templates + CRDs）
│   └── higress/                 #   Umbrella Chart
├── docker/                      # Dockerfile
├── hgctl/                       # CLI 工具（独立 Go module）
├── test/                        # 测试（e2e + gateway 一致性）
├── samples/                     # 示例 manifest
└── tools/                       # 构建与 CI 脚本
```

## 4. 自定义资源 (CRD)

| CRD | API Group | 用途 |
|-----|-----------|------|
| `WasmPlugin` | `extensions.higress.io/v1alpha1` | 声明 Wasm 插件的加载、匹配规则、配置 |
| `McpBridge` | `networking.higress.io/v1` | 注册中心桥接（Nacos/ZK/Consul/Eureka） |
| `McpServer` | `networking.higress.io/v1` | MCP Server 声明 |
| `Http2Rpc` | `networking.higress.io/v1` | HTTP → Dubbo/gRPC 桥接 |

## 5. 部署模式

| 模式 | 说明 |
|------|------|
| **Docker 单机** | `all-in-one` 镜像，单容器暴露 8001(UI) / 8080(HTTP) / 8443(HTTPS) |
| **Helm + K8s** | 完整 K8s 部署，支持 HPA、PDB、多副本 |
| **区域镜像** | 杭州(默认) / 北美(us-west-1) / 东南亚(ap-southeast-7) |

Helm 关键 values：
- `global.ingressClass` — IngressClass（默认 `higress`）
- `global.gatewayClass` — GatewayClass（默认 `higress`）
- `global.enableIstioAPI` — 是否同时监听 Istio 资源
- `global.enableGatewayAPI` — 是否启用 Gateway API
- `global.enableDeltaXDS` — 增量 xDS
- `global.enableRedis` — 是否启用 Redis（限流/配额依赖）
- `global.enableInferenceExtension` — Gateway API Inference Extension

## 6. hgctl CLI

独立 Go module，提供：
- `install / uninstall / upgrade` — K8s 或 Docker 安装管理
- `config` — 查看 Envoy 配置（bootstrap / cluster / endpoint / listener / route）
- `plugin` — 管理 Wasm 插件（开发、构建、推送）
- `agent` — AI Agent 子命令
- `profile` — 安装 profile（预设配置集）
- `dashboard` — 控制台入口

## 7. 构建系统

- `Makefile` — 薄包装，支持 `BUILD_WITH_CONTAINER=1` 容器化构建
- `Makefile.core.mk` — 实际目标定义
- 关键依赖：Gateway API `v1.6.0`、Inference Extension `v1.4.0`
- 目标架构：`amd64` / `arm64`
- 链接时版本注入：`-X .../pkg/cmd/lversion.higressVersion=...`
