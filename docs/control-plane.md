<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- SPDX-FileCopyrightText: Copyright 2026 NVIDIA CORPORATION -->

# Mokka 控制面

Stage 1 控制面把 `SGPURackProfile` 和 `SGPUInventory` 资源转换为由控制面拥有的 `SGPURack` 资源，把符合条件的 Kubernetes Node 分配给逻辑机架 Node，并把分配结果投影到这些 Node 上。它还决定哪些 [`SGPURuntimePolicy` 覆盖](#override-runtime-state)适用于每块模拟 GPU。它建模容量、拓扑和运行时状态；本身不向工作负载提供 GPU。

## 安装

先安装 CRD，再在已有 chart 中启用控制面：

```bash
helm upgrade --install mokka-crds deployments/mokka-crds/helm/mokka-crds
helm upgrade --install nvml-mock deployments/nvml-mock/helm/nvml-mock \
  --namespace mokka --create-namespace \
  --set controlPlane.enabled=true \
  --set controlPlane.image.tag=latest
```

默认情况下，控制面运行带有 chart `appVersion` 标签的 `ghcr.io/nvidia/mokka-control-plane`，在已发布的 chart 中，该标签就是同一版本发布的镜像。在 `main` 上 `appVersion` 是下一个 `-dev` 版本，但控制面镜像的 `main` 构建只以 `latest` 发布，因此上面的命令设置了 `controlPlane.image.tag=latest`；安装已发布的 chart 时请去掉它。控制器具有集群级权限，生产中请按摘要固定：把 `controlPlane.image.digest` 设为控制面镜像发布的 `sha256:...` 摘要。摘要优先于 `controlPlane.image.tag`。

控制面 Pod 的调度和标签独立于节点 DaemonSet，通过 `controlPlane.nodeSelector`、`controlPlane.tolerations`、`controlPlane.affinity`、`controlPlane.topologySpreadConstraints`、`controlPlane.priorityClassName`、`controlPlane.podLabels` 和 `controlPlane.podAnnotations` 控制。它们的形式与[节点 DaemonSet 对应取值](helm-chart.md#values)及 Kubernetes Pod spec 相同。`controlPlane.replicas` 大于 1 时，基于 `kubernetes.io/hostname` 的 Pod 反亲和让备用副本避开 leader 所在节点，`topology.kubernetes.io/zone` 上的分布约束让它待在另一个可用区：

```yaml
controlPlane:
  replicas: 2
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            topologyKey: kubernetes.io/hostname
            labelSelector:
              matchLabels:
                app.kubernetes.io/name: nvml-mock-control-plane
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: nvml-mock-control-plane
```

`controlPlane.pdb.enabled` 会添加一个 [PodDisruptionBudget](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/)（PDB），限制节点驱逐或集群升级一次能逐出多少副本。PDB 只在 `controlPlane.replicas` 大于 1 时渲染：对单副本来说，它要么阻塞所有驱逐，要么什么都不保护。`minAvailable` 或 `maxUnavailable` 可设为数量或百分比；两者都设置时以 `maxUnavailable` 为准，都没设置时 `minAvailable` 为 1。`minAvailable` 等于 `controlPlane.replicas` 会阻塞对运行副本的节点的任何驱逐。

卸载 `mokka-crds` release 会保留 CRD 和已有的 Mokka 资源。移除 Mokka API 及其资源需要显式删除 CRD。

一个集群中只能有一个 Helm release 启用集群级控制面。固定的 `control-plane.mokka.nvidia.com` ClusterRoleBinding 由该 release 拥有，充当单例守卫：其他 release 必须等所有者卸载后才能启用。拥有者 release 的升级和回滚仍使用同一守卫。

## 安全停用或卸载

当 `SGPUInventory` 仍存在时，不要禁用 `controlPlane.enabled`、卸载其 `nvml-mock` release，或删除 Mokka CRD。禁用或卸载会移除控制面 Deployment 和 RBAC；删除 CRD 会移除其 API 对象。任一顺序都会阻止控制面释放 `mokka.nvidia.com/inventory-cleanup` 和 `mokka.nvidia.com/rack-cleanup` finalizer，或移除其 Node 投影元数据。CRD chart 卸载时保留每个 CRD，因此卸载任一 Helm release 都不是清理手段。

移除控制面前请先排空它。所有 Mokka 自定义资源和 Node 都是集群作用域的；以下命令刻意省略它们的 `--namespace`，并从单例 ClusterRoleBinding 中发现唯一拥有者 Helm release 的命名空间。

```bash
MOKKA_RELEASE="$(kubectl get clusterrolebinding \
  control-plane.mokka.nvidia.com \
  -o jsonpath='{.metadata.annotations.meta\.helm\.sh/release-name}')"
MOKKA_NAMESPACE="$(kubectl get clusterrolebinding \
  control-plane.mokka.nvidia.com \
  -o jsonpath='{.metadata.annotations.meta\.helm\.sh/release-namespace}')"
test -n "$MOKKA_RELEASE"
test -n "$MOKKA_NAMESPACE"

kubectl --namespace "$MOKKA_NAMESPACE" rollout status deployment \
  --selector="app.kubernetes.io/component=control-plane,app.kubernetes.io/instance=$MOKKA_RELEASE" \
  --timeout=2m

kubectl delete sgpuinventories.mokka.nvidia.com \
  --all --wait=true --timeout=15m
```

删除必须成功完成。它会等待资产清单 finalizer，而这些 finalizer 又会等待每个由控制面拥有的机架和精确的 Node 投影完成排空。如果超时，请保持控制面及其 RBAC 权限继续运行，检查剩余的 finalizer 和控制面日志，并解决报告的清理冲突。不要强制移除 finalizer。

在改动 Helm release 之前先验证排空结果。第一条命令不应产生任何对象。第二条不应产生任何 Node 名称：它检查两个投影身份标记，以及仍由 `mokka-controller` 服务端 apply manager 拥有的字段。由其他 manager 拥有的仅 clique 标签，刻意不被视为 Mokka 投影状态。

```bash
kubectl get \
  sgpuinventories.mokka.nvidia.com,sgpuracks.mokka.nvidia.com \
  --no-headers \
  -o 'custom-columns=KIND:.kind,NAME:.metadata.name,FINALIZERS:.metadata.finalizers'

kubectl get nodes --show-managed-fields=true \
  -o go-template='{{range .items}}{{$node := .metadata.name}}{{if or (index .metadata.labels "mokka.nvidia.com/sgpu-assigned") (index .metadata.annotations "mokka.nvidia.com/sgpu-assignment")}}{{printf "%s\n" $node}}{{else}}{{range .metadata.managedFields}}{{if eq .manager "mokka-controller"}}{{printf "%s\n" $node}}{{end}}{{end}}{{end}}{{end}}'
```

只有两项检查都为空时，才能禁用控制面或卸载其拥有者 release：

```bash
# 只禁用控制面；从仓库根目录运行。
helm upgrade "$MOKKA_RELEASE" deployments/nvml-mock/helm/nvml-mock \
  --namespace "$MOKKA_NAMESPACE" --reuse-values \
  --set controlPlane.enabled=false

# 或者改为卸载整个 nvml-mock release。
helm uninstall "$MOKKA_RELEASE" --namespace "$MOKKA_NAMESPACE"
```

Profile、运行时策略和保留的 CRD 可以留待以后安装控制面时使用。要完全移除 API，请卸载 `mokka-crds` Helm release，然后显式删除全部四个保留的 CRD，但只能在完成上述排空之后：

```bash
# 使用该 release 安装时所在的命名空间。
MOKKA_CRD_NAMESPACE=default
helm uninstall mokka-crds --namespace "$MOKKA_CRD_NAMESPACE"
kubectl delete customresourcedefinitions.apiextensions.k8s.io \
  sgpuinventories.mokka.nvidia.com \
  sgpuracks.mokka.nvidia.com \
  sgpurackprofiles.mokka.nvidia.com \
  sgpuruntimepolicies.mokka.nvidia.com
```

本地构建镜像：

```bash
docker build -f deployments/control-plane/Dockerfile \
  -t REGISTRY/mokka-control-plane:TAG .
```

本地 Tilt 开发时传入 `--control-plane`；否则控制面及其 CRD 都是禁用的，现有 Tilt 默认值不变。

## 声明容量

应用[示例 profile](https://github.com/NVIDIA/k8s-test-infra/blob/main/examples/controlplane-crds/sgpu-rack-profile.yaml)和[资产清单](https://github.com/NVIDIA/k8s-test-infra/blob/main/examples/controlplane-crds/sgpu-inventory.yaml)，然后让一个 Node 符合条件并匹配示例组选择器：

```bash
kubectl apply -f examples/controlplane-crds/sgpu-rack-profile.yaml
kubectl apply -f examples/controlplane-crds/sgpu-inventory.yaml
kubectl label node NODE \
  mokka.nvidia.com/sgpu-node=true \
  mokka.nvidia.com/pool=example
```

只有带有 `mokka.nvidia.com/sgpu-node=true` 的 Node 才会进入控制面缓存。空的组选择器匹配所有符合条件的 Node；否则条件与选择器都必须匹配。放置选择器不能引用 `mokka.nvidia.com/sgpu-assigned` 或 `nvidia.com/gpu.clique`，因为这些标签是放置的结果。这样的资产清单会被拒绝，且不会改变其上一次物化的机架或绑定。

支持的拓扑上限是 100,000 个符合条件的 Node、100,000 个生成的机架，以及所有 `SGPUInventory` 资源中 64 个声明的机架组。机架组是同构声明，而不是单个机架：用它的 `count` 把一个组展开成多个机架。资产清单按创建顺序整体准入，前提是聚合限制仍然满足。超出上限的资产清单会报告 `CapacityExceeded`；删除或缩小更早的资产清单后，下一个声明才可被准入。

持久分配是 `SGPURack.spec.nodes[].nodeRef`。当 Node、机架或 profile 被添加或编辑时，已有的有效绑定不会移动。新 Node 按创建时间、名称、再按 UID 排序，并按机架/索引顺序填充逻辑 Node 坐标。GPU、序列号、fabric 和机架身份由资产清单 UID 和坐标派生，因此重试、重启和 leader 变更不会改变未变动的坐标。

对于每个成功投影的绑定，控制面只拥有：

- `mokka.nvidia.com/sgpu-assigned=true`；
- `nvidia.com/gpu.clique=<fabric UUID>.<clique ID>`；
- `mokka.nvidia.com/sgpu-assignment`，紧凑 JSON，包含精确的资产清单、机架、profile 修订、坐标和 Node UID 数据。

## 覆盖运行时状态

`SGPURuntimePolicy` 覆盖某个资产清单中部分 GPU 的模拟运行时状态：它们的健康状态、设备模式和遥测。profile 的 `defaults.runtime` 设置该 profile 渲染的每块 GPU 的起始状态，策略只改变它设置的字段。省略的字段保持原本的值；显式的零则代表一个值：`drawMilliWatts: 0` 把功耗设为零。Stage 1 会评估策略并报告每条是否被接受，但还不会把它们应用到模拟 GPU 上。

[示例策略](https://github.com/NVIDIA/k8s-test-infra/blob/main/examples/controlplane-crds/sgpu-runtime-policy.yaml)把示例机架逻辑 Node 1 上的 GPU 标记为故障，并把功耗降为零：

```bash
kubectl apply -f examples/controlplane-crds/sgpu-runtime-policy.yaml
```

`spec.targetRef` 指定资产清单，并可用 `rackGroups`、`rackIndexes`、`nodeIndexes` 和 `gpuIndexes` 缩小选择范围；省略某个列表即选择该维度上的所有索引。最深层的列表决定策略的作用域：

| `targetRef` 中最深的列表 | 作用域 |
| --- | --- |
| 无 | 资产清单 |
| `rackGroups` | 机架组 |
| `rackIndexes` | 机架 |
| `nodeIndexes` | Node |
| `gpuIndexes` | GPU |

一块 GPU 的有效状态从 profile 默认值开始，按作用域从宽到窄应用选中它的已接受策略，因此较窄的作用域按字段覆盖较宽的。

列出的每个机架组都必须由资产清单声明。当策略选择的机架组形态不同时，每个列出的索引必须至少存在于其中一个机架组里，且每个机架组只应用它拥有的索引。Node 和 GPU 索引来自机架组的 profile，因此缺少 profile 的机架组没有任何索引。

两条策略在作用域相同、至少选中一块共同 GPU、且设置了共同字段时冲突。`creationTimestamp` 更早的策略保持生效，UID 用于打破平局，另一条整体被拒绝。删除胜出者后，剩余最早的策略被接受。

`Accepted` condition 报告结果：

| 原因 | 状态 | 含义 |
| --- | --- | --- |
| `Accepted` | `True` | 策略适用于其目标。 |
| `TargetNotFound` | `False` | 目标资产清单不存在。 |
| `InvalidTarget` | `False` | 列出的机架组未声明，或列出的索引没有选中任何 GPU。 |
| `Conflicted` | `False` | 同作用域的更早策略为部分相同 GPU 设置了它的某个字段。 |

`kubectl get sgpuruntimepolicies` 显示每条策略的目标和接受状态，condition 消息解释拒绝原因：

```bash
kubectl get sgpuruntimepolicies
kubectl get sgpuruntimepolicy example-node-1-failed \
  -o jsonpath='{.status.conditions[?(@.type=="Accepted")].message}'
```

## 观察与排障

```bash
kubectl get sgpuinventories,sgpuracks,sgpuruntimepolicies
kubectl get sgpuinventory example -o yaml
kubectl get node NODE -o jsonpath='{.metadata.annotations.mokka\.nvidia\.com/sgpu-assignment}'
kubectl -n mokka logs -l app.kubernetes.io/component=control-plane
kubectl -n mokka get lease control-plane.mokka.nvidia.com -o yaml
```

资产清单 condition 区分无效输入或 profile 引用、物化失败、待处理或冲突的放置，以及投影失败。缺失或无效的 profile 会保留上一次物化的机架，但阻止该组的新分配。选择器重叠会让某个未分配的 Node 处于冲突状态。外来的机架所有权和不兼容的 Node 元数据会被报告，且绝不会被覆盖。运行时策略的 `Accepted` condition 会指出它不适用的原因。

容量缩小、删除资产清单或机架、失去资格，以及替换 Node UID，都会在清除活动绑定之前先移除精确旧投影。清理只移除控制面拥有的键，且其分配注解仍指向该绑定；不兼容的值会被保留并重试。如果精确 Node UID 已不存在，清理可以在不触碰同名替代者的情况下继续。单例 ClusterRoleBinding 确保只有拥有者 release 具有集群权限。在该 release 内，其命名空间本地的 Lease 防止副本并发协调；在单安装前提下不需要集群级 Lease 权限。每个副本都保持对 profile、资产清单、机架、运行时策略、符合条件的 Node 和持久机架槽位分配的同步 informer 视图。只有当选出的 leader 会挂接协调 handler 并运行 worker。备用副本在其缓存同步并观察到选出的 leader 后报告就绪；leader 还要等待其 handler 重放当前缓存并启动 worker。升级使用 `RollingUpdate` Deployment 策略，`maxUnavailable: 0`、`maxSurge: 1`，以保持服务重叠，同时由 leader 选举维持单写者变更。

## Stage 1 排除项

Stage 1 没有 agent 或驱动、工作负载 GPU 注入、REST API、Redis、基于心跳的回收、Node lease 或跨机架交换机图。唯一的 Lease 是 Kubernetes leader 选举。
