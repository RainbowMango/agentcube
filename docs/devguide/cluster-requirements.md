# AgentCube 对 Kubernetes 集群的依赖说明

> 适用版本：基于当前 `main` 分支与 `manifests/charts/base` Helm Chart。
> 本文仅描述 AgentCube 运行所**依赖的 Kubernetes 能力与外部组件**，便于在新集群（如托管 K8s / CCE Turbo 等）上评估可行性。
> 证据均可在仓库中追溯，文末附「证据索引」。

## 1. 概览

AgentCube 由以下进程组成（详见架构提案 `docs/design/agentcube-proposal.md`）：

| 组件 | 部署形态 | 作用 |
|------|---------|------|
| **Workload Manager** | Deployment（1 副本） | 控制面：watch CRD、创建/回收 Sandbox、提供内部 HTTP API |
| **Router** | Deployment（1 副本） | 数据面：会话管理、JWT 签名、把请求反向代理进 Sandbox |
| **Sandbox Pod** | 由 Workload Manager 动态创建 | 运行不可信 / LLM 生成代码的隔离沙箱 |
| **Volcano Agent Scheduler**（可选） | Deployment | 批 / gang 调度，默认关闭 |
| **SPIRE 套件**（可选） | Server/Agent/Controller-Manager | 组件间 mTLS 身份，默认关闭 |

> 镜像默认来自 `ghcr.io/volcano-sh/*`，详见 `manifests/charts/base/values.yaml`。

---

## 2. 强制依赖（不满足则无法运行）

### 2.1 Kubernetes 版本
- **要求 ≥ 1.34**。依据：`go.mod` 中 `k8s.io/api`、`k8s.io/apimachinery`、`k8s.io/client-go` 均为 `v0.34.1`。
- 仅使用稳定 API（`apiextensions.k8s.io/v1`、`authentication.k8s.io/v1`），无特殊 feature gate 要求。

### 2.2 需要安装的 CRD（需要 cluster-admin 安装一次）
| CRD（Kind） | API Group / Version | 来源 |
|-------------|--------------------|------|
| `CodeInterpreter`、`AgentRuntime` | `runtime.agentcube.volcano.sh/v1alpha1` | 本项目，`manifests/charts/base/crds/` |
| `Sandbox` | `agents.x-k8s.io/v1alpha1` | 上游 `sigs.k8s.io/agent-sandbox` |
| `SandboxClaim`、`SandboxTemplate`、`SandboxWarmPool` | `extensions.agents.x-k8s.io/v1alpha1` | 上游 agent-sandbox extensions |

> agent-sandbox 的 CRD 由上游项目提供，需要随 AgentCube 一并安装。

### 2.3 RBAC（集群级权限）
Workload Manager 的 ServiceAccount 需要一个 **ClusterRole**，对以下资源具备相应权限：

| API Group | 资源 | Verbs |
|-----------|------|-------|
| `agents.x-k8s.io` | `sandboxes` | get, list, watch, create, update, patch, delete |
| `extensions.agents.x-k8s.io` | `sandboxclaims`, `sandboxtemplates`, `sandboxwarmpools` | get, list, watch, create, update, patch, delete |
| `runtime.agentcube.volcano.sh` | `codeinterpreters`, `agentruntimes` | get, list, watch, create, update, patch, delete |
| `""`（core） | `pods`, `secrets` | get, list, watch, create, update, patch, delete |
| `authentication.k8s.io` | `tokenreviews` | create |

Router 仅需 **namespace 级 Role**：对 `secrets` 的 `get`、`create`。

> 依据：`manifests/charts/base/templates/rbac/workloadmanager.yaml`、`rbac-router.yaml`。

### 2.4 外部存储：Redis / ValKey（必需）
- 会话状态、Sandbox 元数据存储在 Redis/ValKey 中，**必须提供一个可达的 Redis 实例**（集群内或外部托管均可，如华为云 DCS）。
- 配置项（`values.yaml`）：`redis.addr`、`redis.password` 或 `redis.secretName` / `redis.secretKey`。
- 进程通过环境变量 `REDIS_ADDR`、`REDIS_PASSWORD` 读取。
- 依据：`go.mod`（`redis/go-redis`、`valkey-io/valkey-go`）、`pkg/store/`。

### 2.5 网络：Pod 间直连可达（重要）
- Router 需要从 Kubernetes API 获取 Sandbox 的 **Pod IP，并直接向该 IP 发起连接**转发流量（`pkg/router` 的 `forwardToSandbox`）。
- **要求 Router Pod 能够直接路由到 Sandbox Pod 的 IP**（标准 K8s 扁平 Pod 网络即满足；CCE Turbo 的 Cloud Native 2.0 / ENI 网络天然满足）。
- 组件间通过集群内 DNS 通信（如 `http://workloadmanager.<ns>.svc.cluster.local:8080`）。
- 默认 Service 为 `ClusterIP`，端口 `8080`；如需集群外访问 Router，可改为 `NodePort` / `LoadBalancer`。

---

## 3. 沙箱运行时（RuntimeClass）——需重点确认

AgentCube 的安全定位是「在隔离沙箱中运行不可信 / LLM 生成代码」，因此 Sandbox Pod 的运行时（RuntimeClass）非常关键。当前实现情况如下，请注意 **二进制默认值与 Helm 部署值不一致**：

| 层级 | RuntimeClassName 取值 | 说明 |
|------|----------------------|------|
| 二进制 flag 默认值 | `kuasar-vmm` | `cmd/workload-manager/main.go` 中 `--runtime-class-name` 默认值 |
| **Helm Chart 实际下发** | **空字符串**（`--runtime-class-name=`） | `templates/workloadmanager.yaml` 写死为空，会覆盖二进制默认值 |
| 效果 | 使用**集群默认运行时**（通常为 runc） | 当前 Chart 默认**不**强制 microVM 隔离 |

含义：
- **当前 Helm 默认部署不要求集群提供任何特殊 RuntimeClass**，开箱即用。
- 若要启用 microVM / 安全容器隔离（推荐用于生产、面向不可信代码），需要：
  1. 集群节点上注册可用的 **RuntimeClass**（如 kuasar、Kata 系安全容器运行时）；
  2. 把 Workload Manager 启动参数 `--runtime-class-name` 设为该 RuntimeClass 名称（目前需修改 `templates/workloadmanager.yaml` 的 `args`，该值尚未在 `values.yaml` 暴露）。
- 不同平台提供的安全容器 RuntimeClass 名称可能不同，需与集群方确认其**是否提供安全容器运行时及其确切名称 / 节点规格要求**。

---

## 4. 可选依赖（默认关闭，启用才需要）

### 4.1 SPIRE / SPIFFE（组件间 mTLS）
- 默认 `spire.enabled=false`。启用后用于 Router 与 Workload Manager 间的 mTLS。
- 启用后对集群的额外要求：
  - SPIRE 的 CRD（`spire.spiffe.io`：`ClusterSPIFFEID` 等）；
  - `hostPath` 挂载 `/run/spire/sockets`（`DirectoryOrCreate`）；
  - SPIRE Server/Agent 对 `pods`、`nodes`、`namespaces`、`endpoints` 的只读权限，以及 `tokenreviews: create`；
  - `securityContext.fsGroup: 1000`。
- 依据：`manifests/charts/base/templates/spire/`、`pkg/mtls/`。

### 4.2 Volcano Agent Scheduler（批 / gang 调度）
- 默认 `volcano.scheduler.enabled=false`。启用后通过 `schedulerName: agent-scheduler` 调度。
- 额外 RBAC：`scheduling.volcano.sh/podgroups`、`resource.k8s.io/resourceclaims` 等。
- 依据：`manifests/charts/base/templates/volcano-agent-scheduler-development.yaml`。

---

## 5. 主机 / 节点级要求

- **默认无需特权、无需 hostPath、无需特定节点标签 / 污点。**
- 仅当启用 **SPIRE** 时需要 `hostPath: /run/spire/sockets`。
- 仅当启用 **安全容器运行时** 时，对应节点需满足该运行时的形态要求（如嵌套虚拟化 / 裸金属 / 特定节点规格）。
- 镜像默认在 `ghcr.io`，若集群无外网访问，需将镜像同步到内网 / 云厂商镜像仓库并改写 `values.yaml` 中镜像地址。

---

## 6. 最小依赖清单（TL;DR）

满足以下条件即可跑起 **默认配置**（不含 mTLS / Volcano / microVM）：

1. Kubernetes ≥ 1.34；
2. 有权限安装 AgentCube + agent-sandbox 的 CRD，并创建对应 ClusterRole；
3. 一个可达的 Redis / ValKey 实例；
4. 标准扁平 Pod 网络（Router 可直连 Sandbox Pod IP）；
5. 集群可拉取（或已镜像同步）所需容器镜像。

如需 **生产级隔离**，额外满足：集群提供安全容器 RuntimeClass，并将 `--runtime-class-name` 指向它。

---

## 7. 证据索引

| 主题 | 文件 |
|------|------|
| Scheme / CRD 注册 | `cmd/workload-manager/main.go` |
| RuntimeClass flag | `cmd/workload-manager/main.go`（`--runtime-class-name`），`templates/workloadmanager.yaml`（args） |
| CRD 定义 | `manifests/charts/base/crds/runtime.agentcube.volcano.sh_*.yaml` |
| RBAC | `manifests/charts/base/templates/rbac/workloadmanager.yaml`、`rbac-router.yaml`、`templates/spire/rbac.yaml` |
| Helm 配置项 | `manifests/charts/base/values.yaml` |
| Redis / Store | `go.mod`、`pkg/store/` |
| Router 转发到 Sandbox | `pkg/router/`（`forwardToSandbox`） |
| mTLS / SPIRE | `pkg/mtls/`、`manifests/charts/base/templates/spire/` |
| K8s 版本 | `go.mod`（`k8s.io/*` v0.34.1） |

