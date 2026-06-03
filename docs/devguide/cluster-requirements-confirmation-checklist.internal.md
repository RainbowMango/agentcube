<!--
内部文档 / INTERNAL — 暂不外发。
用途：作为 AgentCube 在新集群（CCE Turbo）落地时，向对接团队 / 集群负责人逐项确认的清单。
正式对外只先发《cluster-requirements.md》。
-->

# 【内部】AgentCube 集群对接确认 Checklist（CCE Turbo）

> 状态：内部使用，**暂不发给对接团队**。
> 配套文档：对外只先给 `cluster-requirements.md`（仅说明 K8s 依赖）。本清单待时机成熟再转交。
> 用法：每一项请对接团队向 CCE Turbo 集群负责人确认，并回填「确认结果」。

## 0. 优先级最高的阻塞项 ⛔

- [ ] **安全容器运行时（RuntimeClass）** —— 这是唯一需要先达成共识、否则其余都白谈的项。
  - 背景：AgentCube 的核心价值是「在隔离沙箱中跑不可信 / LLM 生成代码」。当前 Helm Chart **默认 `--runtime-class-name=` 为空 → 用集群默认 runc**，即默认**未启用** microVM 隔离；二进制默认值是 `kuasar-vmm`，但被 Chart 覆盖。
  - CCE Turbo 的安全容器是 **Kata 系**（华为自有），通常要求特定节点形态（如裸金属 / 特定规格），名称也与 `kuasar-vmm` 不同。
  - **需要确认 / 决策（三选一）**：
    - [ ] 方案 A：CCE 侧提供并注册 `kuasar-vmm` RuntimeClass（多数托管集群较难）；
    - [ ] 方案 B：使用 CCE 提供的 Kata RuntimeClass，我们把 `--runtime-class-name` 改成该名称（代码已支持参数化，但 `values.yaml` 尚未暴露，需改模板 + 联调验证）；
    - [ ] 方案 C：暂用 runc，不启用 microVM 隔离（**牺牲安全隔离，不推荐用于不可信代码**，仅可作 PoC）。
  - 待 CCE 负责人回答：
    - [ ] 是否提供安全容器运行时？ RuntimeClass 名称：____________
    - [ ] 该运行时要求的节点规格 / 池子：____________
    - [ ] 是否支持嵌套虚拟化 / 裸金属节点？____________

## 1. 控制面 / API 能力

- [ ] Kubernetes 控制面版本 ≥ 1.34？实际版本：____________
- [ ] 允许安装自定义 CRD（需要一次 cluster-admin）？
  - [ ] `runtime.agentcube.volcano.sh`（CodeInterpreter、AgentRuntime）
  - [ ] agent-sandbox：`agents.x-k8s.io`、`extensions.agents.x-k8s.io`
- [ ] 允许创建 ClusterRole / ClusterRoleBinding（workload-manager 需集群级权限）？
- [ ] `authentication.k8s.io/v1` TokenReview 可用？（托管集群一般默认有）
- [ ] 安装 CRD / ClusterRole 是否需要走审批流程？流程：____________

## 2. 网络

- [ ] Pod 网络为扁平可直连模型（Router 可直接访问 Sandbox Pod IP）？
  - 备注：CCE Turbo 的 Cloud Native 2.0 / ENI 网络应天然满足，需确认所选网络模型确为此模式。
- [ ] 是否需要 Router 对集群外暴露？如需，使用 LoadBalancer（对接华为云 ELB）还是 NodePort？____________
- [ ] 是否有 NetworkPolicy / 安全组会拦截 Pod 间 8080 端口通信？____________

## 3. 外部依赖

- [ ] 提供 Redis / ValKey 实例（可用华为云 DCS）？地址 / 规格：____________
  - [ ] 是否需要持久化 / 高可用？
  - [ ] 凭据如何下发（Chart 创建 Secret，还是提供已有 `secretName`）？
- [ ] 容器镜像可达？默认在 `ghcr.io`：
  - [ ] 是否允许外网拉取，或需同步到华为云 SWR？目标仓库：____________

## 4. 可选能力（按需）

- [ ] 是否需要组件间 mTLS（SPIRE）？如需：
  - [ ] 允许 `hostPath: /run/spire/sockets`？
  - [ ] 允许安装 `spire.spiffe.io` CRD 与对应 RBAC？
  - [ ] 允许 `fsGroup` / 相关 securityContext？
- [ ] 是否需要 Volcano 批 / gang 调度？如需：
  - [ ] 允许安装 `scheduling.volcano.sh`、`resource.k8s.io` 相关权限？

## 5. 节点 / 资源

- [ ] 安全容器节点池规格与数量是否满足 WarmPool 预热 + 并发会话？预估容量：____________
- [ ] 是否有节点标签 / 污点需要我们在模板里适配（affinity / toleration）？____________

## 6. 风险登记（内部）

| 风险 | 影响 | 处置 |
|------|------|------|
| RuntimeClass 不匹配（kuasar-vmm vs CCE Kata） | 无法启用 microVM 隔离，违背安全前提 | 优先联合决策，见第 0 节 |
| 安全容器节点形态受限 | 容量 / 成本上升 | 评估节点规格与配额 |
| 外网镜像不可达 | 部署失败 | 提前同步镜像到 SWR |
| CRD / ClusterRole 审批 | 上线周期变长 | 提前发起审批 |
| Redis 未持久化 | 会话状态丢失 | 确认 DCS 规格与持久化 |

---

### 给对接团队的一句话口径（待外发时使用）
> AgentCube 对集群的强依赖只有 5 条（K8s≥1.34、可装 CRD/ClusterRole、一个 Redis、Pod 直连网络、镜像可达），CCE Turbo 基本都满足；**唯一需要你们和 CCE 负责人优先敲定的是「安全容器 RuntimeClass」**——这决定我们能否开启 microVM 隔离。

