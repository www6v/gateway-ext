# 06 — 扩展点

> 本文档梳理用户扩展 Higress 的所有方式。

## 1. Wasm 插件（主要扩展方式）

### 1.1 支持的语言

| 语言 | SDK 路径 | 成熟度 |
|------|---------|--------|
| **Go** | `plugins/wasm-go/` | ★ 生产级，55+ 内置插件 |
| **C++** | `plugins/wasm-cpp/` | 生产级，12 内置插件 |
| **Rust** | `plugins/wasm-rust/` | 可用，示例级 |
| **AssemblyScript** | `plugins/wasm-assemblyscript/` | 可用，示例级 |

### 1.2 开发流程

```bash
# 1. 创建插件目录（以 Go 为例）
mkdir plugins/wasm-go/extensions/my-plugin/
cd plugins/wasm-go/extensions/my-plugin/

# 2. 初始化 Go module
go mod init github.com/alibaba/higress/v2/plugins/wasm-go/extensions/my-plugin

# 3. 编写 main.go
# - import "github.com/higress-group/proxy-wasm-go-sdk/proxywasm"
# - 实现 proxywasm 接口（onRequest / onResponse / ...）

# 4. 编译为 Wasm
tinygo build -o plugin.wasm -scheduler=none -target=wasi ./

# 5. 打包为 OCI 镜像
docker build -t my-registry/my-plugin:v1 -f Dockerfile .
docker push my-registry/my-plugin:v1
```

### 1.3 部署与挂载

通过 `WasmPlugin` CRD 声明：

```yaml
apiVersion: extensions.higress.io/v1alpha1
kind: WasmPlugin
metadata:
  name: my-plugin
spec:
  selector:
    ingress: higress  # 匹配 IngressClass
  url: oci://my-registry/my-plugin:v1
  phase: DEFAULT       # AUTHN | AUTHZ | STATS | DEFAULT
  priority: 500        # 执行优先级
  pluginConfig:        # 插件自定义配置
    key: value
  matchRules:          # 匹配规则（可选）
  - ingress:
    - default/my-ingress
    config:
      key: override-value
```

### 1.4 热更新

- Wasm 插件版本更新**不需要重启 Envoy**
- 通过修改 `WasmPlugin` CRD 的 `url` 触发热加载
- 加载过程中无流量损失

## 2. Go 原生 Filter

### 2.1 适用场景

当 Wasm 沙箱的能力不够时（如需要完整的 Go 标准库、高吞吐、复杂状态管理），可以使用原生 Go Envoy filter。

### 2.2 当前使用

- `mcp-server` — MCP Server 托管（高吞吐路径）
- `mcp-session` — MCP Session 管理

### 2.3 开发路径

`plugins/golang-filter/` 目录下的框架。

## 3. MCP Server 插件

### 3.1 内置工具注册

使用 `plugins/wasm-go/pkg/mcp/` 框架：

```go
import "github.com/alibaba/higress/v2/plugins/wasm-go/pkg/mcp"

func main() {
    mcp.LoadMCPServer(
        mcp.AddMCPServer("my-server",
            mcp.WithTools(
                mcp.Tool{
                    Name:        "my_tool",
                    Description: "My custom tool",
                    InputSchema: schema,
                    Handler:     myToolHandler,
                },
            ),
        ),
    )
    mcp.InitMCPServer()
}
```

### 3.2 外部 MCP Server 托管

通过 `openapi-to-mcpserver` 工具将 OpenAPI 规范转换为远程 MCP Server：

```bash
# 安装工具
go install github.com/higress-group/openapi-to-mcpserver@latest

# 转换 OpenAPI → MCP Server
openapi-to-mcpserver --input openapi.yaml --output mcp-server/
```

## 4. 服务发现后端

### 4.1 内置后端

| 后端 | 路径 | 状态 |
|------|------|------|
| Nacos | `registry/nacos/` | ✅ 完整支持（v2） |
| ZooKeeper | `registry/zookeeper/` | ✅ 完整支持 |
| Consul | `registry/consul/` | ✅ 完整支持 |
| Eureka | `registry/eureka/` | ✅ 完整支持 |
| 静态直连 | `registry/direct/` | ✅ 完整支持 |

### 4.2 自定义后端

实现 `registry.Watcher` 接口：

```go
// registry/watcher.go
type Watcher interface {
    // Start 开始监听服务变化
    Start(stop <-chan struct{}) error
    // GetServices 返回当前已知的服务列表
    GetServices() []*ServiceEntry
    // AddEventHandler 注册变更回调
    AddEventHandler(handler EventHandler)
}
```

通过 `McpBridge` CRD 注册自定义后端。

## 5. CRD 扩展

### 5.1 现有 CRD

| CRD | 用途 |
|-----|------|
| `WasmPlugin` | 插件加载与匹配 |
| `McpBridge` | 服务发现桥接 |
| `McpServer` | MCP Server 声明 |
| `Http2Rpc` | HTTP → Dubbo/gRPC |

### 5.2 添加新 CRD

1. 在 `api/` 下定义 protobuf：
   ```
   api/networking/v1/my_crd.proto
   ```

2. 生成 Go 代码：
   ```bash
   cd api && make gen-api
   ```

3. 生成客户端：
   ```bash
   cd api && make gen-client
   ```

4. 在 `pkg/ingress/kube/` 下实现控制器

5. 在 `pkg/ingress/translation/` 中添加翻译逻辑

## 6. Ingress 注解

### 6.1 nginx 兼容注解

`pkg/ingress/kube/annotations/` 实现了大量 nginx-ingress 兼容注解：
- `nginx.ingress.kubernetes.io/rewrite-target`
- `nginx.ingress.kubernetes.io/ssl-redirect`
- `nginx.ingress.kubernetes.io/proxy-body-size`
- 等等

### 6.2 自定义注解

可以在 `pkg/ingress/kube/annotations/` 中添加新的注解处理器。

## 7. Gateway API

Higress 是 **Gateway API Conformant** 实现：
- 支持 `GatewayClass`、`Gateway`、`HTTPRoute`、`TLSRoute`、`GRPCRoute`
- 支持 Gateway API Inference Extension（推理感知路由）
- 通过 `global.enableGatewayAPI` 和 `global.enableInferenceExtension` 控制

## 8. 外部授权 — `ext-auth`

通过 `ext-auth` 插件接入外部授权服务：
- HTTP/gRPC 外部授权协议
- 支持请求/响应阶段的鉴权
- 可返回自定义 header、状态码
- 支持 OPA 策略引擎

## 9. 控制台扩展

Higress Console 是独立项目（`higress-group/higress-console`）：
- 提供 Web UI 管理路由、插件、证书、服务来源
- 在线体验：`https://demo.higress.io/`

## 10. 扩展方式选择指南

| 场景 | 推荐方式 |
|------|---------|
| 自定义请求/响应处理逻辑 | Wasm 插件 |
| 接入新的 LLM 厂商 | `ai-proxy` 添加 provider |
| 托管 MCP 工具服务 | `mcp-server` 插件 |
| 接入新的服务注册中心 | 实现 `registry.Watcher` |
| 自定义路由规则 | Ingress 注解 / Gateway API |
| 自定义鉴权逻辑 | `ext-auth` + 外部服务 |
| 高性能数据处理 | Go 原生 filter |
| 自定义控制面行为 | 添加 CRD + 控制器 |
