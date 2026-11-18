# nvml-mock

用于 Kubernetes 测试的 Mock NVIDIA 驱动基础设施

本页由 `values.yaml` 生成。安装指南和运行行为参见 [Helm chart 指南](helm-chart.md)。

## 要求

Kubernetes：`>= 1.28.0-0`

## Values

| Key | Type | Default | 说明 |
|-----|------|---------|-------------|
| affinity | object | `{}` | 调度 node-agent Pod 的亲和性规则。 |
| allocationWatcher.enabled | bool | `false` | 启用基于分配的合成显存指标。 |
| allocationWatcher.interval | string | `"2s"` | 轮询 kubelet pod-resources socket 的周期。 |
| allocationWatcher.podResourcesSocket | string | `"/var/lib/kubelet/pod-resources/kubelet.sock"` | kubelet pod-resources socket 路径。 |
| allocationWatcher.resources.limits.cpu | string | `"100m"` | 分配监视器的 CPU 限制。 |
| allocationWatcher.resources.limits.memory | string | `"64Mi"` | 分配监视器的内存限制。 |
| allocationWatcher.resources.requests.cpu | string | `"10m"` | 分配监视器的 CPU 请求。 |
| allocationWatcher.resources.requests.memory | string | `"32Mi"` | 分配监视器的内存请求。 |
| allocationWatcher.usedFractionPerClaim | float | `0.5` | 每次分配分摊的可用 GPU 显存比例。 |
| controlPlane.affinity | object | `{}` | 调度控制面 Pod 的亲和性规则。 |
| controlPlane.enabled | bool | `false` | 启用 Mokka 控制面服务。 |
| controlPlane.image.digest | string | `""` | 不可变镜像摘要；设置时优先于 tag。 |
| controlPlane.image.pullPolicy | string | `"IfNotPresent"` | 控制面的 Kubernetes 镜像拉取策略。 |
| controlPlane.image.repository | string | `"ghcr.io/nvidia/mokka-control-plane"` | 控制面镜像仓库。 |
| controlPlane.image.tag | string | `""` | 控制面镜像 tag；默认为 chart 的 appVersion。 |
| controlPlane.kubeAPIBurst | int | `100` | 控制面的 Kubernetes API 突发额度。 |
| controlPlane.kubeAPIQPS | int | `50` | 控制面的 Kubernetes API 请求速率。 |
| controlPlane.leaderElection.name | string | `"control-plane.mokka.nvidia.com"` | 控制面 leader 选举使用的 Lease 名称。 |
| controlPlane.logging.format | string | `"json"` | 控制面日志编码。 |
| controlPlane.logging.level | string | `"info"` | 控制面日志级别。 |
| controlPlane.nodeSelector | object | `{}` | 控制面 Pod 的节点选择器。 |
| controlPlane.pdb.enabled | bool | `false` | 启用 PodDisruptionBudget。仅在 replicas > 1 时渲染。 |
| controlPlane.podAnnotations | object | `{}` | 添加到控制面 Pod 的额外注解。 |
| controlPlane.podLabels | object | `{}` | 添加到控制面 Pod 的额外标签。 |
| controlPlane.priorityClassName | string | `""` | 控制面 Pod 的 PriorityClass。 |
| controlPlane.replicas | int | `1` | 控制面副本数。 |
| controlPlane.resources.limits.cpu | string | `"500m"` | 控制面的 CPU 限制。 |
| controlPlane.resources.limits.memory | string | `"1Gi"` | 控制面的内存限制。 |
| controlPlane.resources.requests.cpu | string | `"50m"` | 控制面的 CPU 请求。 |
| controlPlane.resources.requests.memory | string | `"64Mi"` | 控制面的内存请求。 |
| controlPlane.service.port | int | `8080` | 控制面的 Kubernetes Service 端口。 |
| controlPlane.service.type | string | `"ClusterIP"` | 控制面的 Kubernetes Service 类型。 |
| controlPlane.shutdownTimeout | string | `"5s"` | 控制面优雅关闭超时。 |
| controlPlane.terminationGracePeriodSeconds | int | `30` | 控制面 Pod 的终止宽限期。 |
| controlPlane.tolerations | list | `[]` | 控制面 Pod 的 tolerations。 |
| controlPlane.topologySpreadConstraints | list | `[]` | 控制面 Pod 的拓扑分布约束。 |
| controlPlane.workers | int | `2` | 控制面 worker goroutine 数量。 |
| driverBranch | string | `""` | `nvmlSystemGetDriverBranch` 报告的驱动分支；空则从驱动版本推导（`"580.65.06"` → `"r580_00"`）。 |
| driverVersion | string | `""` | 模拟驱动报告的驱动版本；空则从 profile 推导。 |
| extraObjects | list | `[]` | 在 tpl 求值后随 release 渲染的额外 Kubernetes 对象。 |
| fabricmanager.enabled | string | `""` | 启用或禁用假 fabric manager；空则从 profile 推导。 |
| fabricmanager.initDelay | string | `""` | 发布 fabric 就绪前的延迟。 |
| fabricmanager.stateDir | string | `"/var/lib/nvml-mock/fabric-state"` | fabric-manager 就绪状态的宿主机目录。 |
| featureGates | object | `{}` |  |
| global.imagePullSecrets | list | `[]` | 添加到本 chart 创建的所有 Pod 的拉取密钥。 |
| global.imageRegistry | string | `""` | 应用于所有镜像的仓库主机覆盖，如内部镜像仓库。 |
| gpu.count | string | `""` | 模拟 GPU 数量；空则从所选 profile 推导。 |
| gpu.customConfig | string | `""` | 替换所选 GPU profile 的完整 YAML 配置。 |
| gpu.dynamicMetrics.enabled | bool | `false` | 启用随时间变化的合成温度、功耗和利用率数值。 |
| gpu.failureInjection.after_calls | int | `0` | 经过指定次数的受保护调用后触发所选故障；零表示禁用。 |
| gpu.failureInjection.enabled | bool | `false` | 启用确定性或概率性的模拟 GPU 故障。 |
| gpu.failureInjection.mode | string | `"healthy"` | 启用故障注入时暴露的故障模式。 |
| gpu.failureInjection.probability | float | `0` | 每次受保护 NVML 调用触发所选故障的概率。 |
| gpu.failureInjection.seed | int | `0` | 随机种子；零表示使用基于时间的种子。 |
| gpu.failureInjection.xid.code | int | `0` | 故障模式发出 Xid 事件时报告的 Xid 代码。 |
| gpu.mig.enabled | bool | `false` | 为支持的 profile 启用 MIG 分区。 |
| gpu.mig.gpuInstances | list | `[]` | 在每台模拟设备上创建的 GPU 实例布局。 |
| gpu.profile | string | `"gb300"` | GPU profile 及逐设备模拟设置。 |
| hic | list | `[]` | `nvidia-smi -q -u` 报告的主机接口卡，以 `{id, firmwareVersion}` 条目形式。 |
| image.digest | string | `""` | 不可变镜像摘要；设置时优先于 tag。 |
| image.pullPolicy | string | `"IfNotPresent"` | Kubernetes 镜像拉取策略。 |
| image.repository | string | `"ghcr.io/nvidia/nvml-mock"` | 节点 agent 的容器镜像仓库。 |
| image.tag | string | `""` | 镜像 tag；默认为 chart 的 appVersion。 |
| imagePullSecrets | list | `[]` | nvml-mock、NRI 和控制面 Pod 的拉取密钥。 |
| imex.binaries.download.timeout | string | `"1m"` | IMEXDownload 特性门控开启时，单次 IMEX 归档下载的时间限制。 |
| imex.binaries.image.digest | string | `""` | IMEX 二进制镜像的不可变摘要；设置时优先于 tag。 |
| imex.binaries.image.pullPolicy | string | `""` | IMEX 二进制镜像拉取策略；空则选 IfNotPresent，"latest" tag 选 Always。 |
| imex.binaries.image.repository | string | `""` | IMEX 二进制镜像仓库；空（默认）不渲染 init 容器。 |
| imex.binaries.image.tag | string | `""` | IMEX 二进制镜像 tag，即其携带的 IMEX 版本；除非设置了 digest，否则必填。 |
| imex.binaries.resources.requests.cpu | string | `"10m"` | IMEX 二进制 init 容器的 CPU 请求。 |
| imex.binaries.resources.requests.memory | string | `"32Mi"` | IMEX 二进制 init 容器的内存请求。 |
| imex.deviceNumbers.capsMajor | int | `236` | IMEX 能力设备使用的主设备号。 |
| imex.deviceNumbers.channelMajor | int | `235` | IMEX 通道设备使用的主设备号。 |
| infiniband.mockTier | string | `""` | InfiniBand 模拟层级；空则从所选 profile 推导。mockTier 设置容器的 `MOCK_IB` 值：off \| sysfs \| full。启用 InfiniBand 的 profile 默认为 "full"，非 IB profile 为 "sysfs"（sysfs 保持 libibmocksys 重定向有效，以屏蔽宿主机真实 IB，即 ibstat 报告 0 个 HCA）。留空即按 profile 自动推导；显式设置可覆盖（无效值会使 `helm template` 失败）。 |
| infiniband.ping.networkPolicy.enabled | bool | `true` | 把模拟 InfiniBand 流量限制在对等 nvml-mock Pod。仅在 `full` 层级渲染，该层级才会监听端口。 |
| infiniband.ping.networkPolicy.peerNamespaces | list | `[]` | 除 release 所在命名空间外，可访问该端口的 nvml-mock Pod 所在命名空间，用于跨命名空间拆分的机群。空则只允许 release 的命名空间，这可以避免 OVN-Kubernetes 在每个 Pod 事件上重新列出集群中的所有 Pod。 |
| infiniband.ping.port | int | `18515` | 模拟 InfiniBand ping 服务使用的 TCP 端口。 |
| integrations.fakeGpuOperator.enabled | bool | `false` | 启用 fake-gpu-operator profile 集成。 |
| integrations.fakeGpuOperator.profileLabels."run.ai/gpu-profile" | string | `"true"` | 添加到生成的 fake-gpu-operator profile 的额外标签。 |
| integrations.fakeGpuOperator.targetNamespace | string | `""` | fake-gpu-operator 读取 profile ConfigMap 的命名空间。 |
| nodeAgent.kernelLog.enabled | bool | `false` | 通过宿主机内核日志公布注入的 Xid。 |
| nodeAgent.livenessProbe | object | `{"httpGet":{"path":"/healthz","port":"health"}}` | 存活探针；最近一次 Stage 波次失败时 /healthz 失败。设为 null 可禁用。 |
| nodeAgent.livenessProbe.httpGet.path | string | `"/healthz"` | 存活探针端点路径。 |
| nodeAgent.livenessProbe.httpGet.port | string | `"health"` | 存活探针端点端口名或端口号。 |
| nodeAgent.logging | object | `{"format":"json","level":"info"}` | 节点 agent 的日志设置与生命周期行为。 |
| nodeAgent.logging.format | string | `"json"` | 节点 agent 的日志编码。 |
| nodeAgent.logging.level | string | `"info"` | 节点 agent 的日志级别。 |
| nodeAgent.nriStartupTimeoutSeconds | int | `120` |  |
| nodeAgent.readinessProbe | object | `{"httpGet":{"path":"/readyz","port":"health"}}` | 就绪探针；/readyz 报告每个模拟器是否都在服务。设为 null 可禁用。 |
| nodeAgent.readinessProbe.httpGet.path | string | `"/readyz"` | 就绪探针端点路径。 |
| nodeAgent.readinessProbe.httpGet.port | string | `"health"` | 就绪探针端点端口名或端口号。 |
| nodeAgent.resources | object | `{"requests":{"cpu":"10m","memory":"32Mi"}}` | 节点 agent 的资源请求。 |
| nodeAgent.resources.requests.cpu | string | `"10m"` | 节点 agent 的 CPU 请求。 |
| nodeAgent.resources.requests.memory | string | `"32Mi"` | 节点 agent 的内存请求。 |
| nodeAgent.resyncInterval | string | `"1m"` | 节点 agent 的 profile 与拓扑重新同步周期。 |
| nodeAgent.shutdownTimeout | string | `"5s"` | 节点 agent 的优雅关闭超时。 |
| nodeLabels.featuresDir | string | `"/etc/kubernetes/node-feature-discovery/features.d"` | NFD 读取生成 feature 文件的宿主机目录。 |
| nodeSelector | object | `{}` | 调度模拟 DaemonSet 所需的现有节点标签。 |
| nri.cdiSpecDir | string | `"/var/run/cdi"` | 包含已部署 CDI 规格的宿主机目录。 |
| nri.deviceAnnotation | string | `"nvml-mock.nvidia.com/devices"` | 启用模拟 GPU 设备注入的 Pod 注解。 |
| nri.deviceInjectionMode | string | `"raw"` | 设备注入模式：原始节点或 CDI 引用。 |
| nri.enabled | bool | `true` | 通过 NRI 插件启用节点级容器注入。把 mock 注入持有 GPU 分配或通过注解选择加入的容器。需要节点上 containerd 启用 NRI：containerd 2.x 默认启用，1.7 不启用。未启用时请设为 false，否则节点 Pod 会保持 NotReady。 |
| nri.excludedNamespaces | list | `[]` | 排除在 NRI 注入之外的命名空间。 |
| nri.healthPort | int | `8080` | NRI 插件的健康与就绪端口。 |
| nri.image | object | `{}` | NRI 插件的可选容器镜像覆盖。 |
| nri.imexChannelAnnotation | string | `"nvml-mock.nvidia.com/imex-channels"` | 启用 IMEX 通道注入的 Pod 注解。 |
| nri.infinibandAnnotation | string | `"nvml-mock.nvidia.com/infiniband"` |  |
| nri.livenessProbe.failureThreshold | int | `3` | 重启插件前的连续失败次数。 |
| nri.livenessProbe.httpGet.path | string | `"/healthz"` | 存活探针端点路径。 |
| nri.livenessProbe.httpGet.port | string | `"nri-health"` | 存活探针端点端口名。 |
| nri.livenessProbe.periodSeconds | int | `10` | 存活探针间隔。 |
| nri.livenessProbe.timeoutSeconds | int | `2` | 存活探针超时。 |
| nri.logging.format | string | `"json"` | NRI 插件的日志编码。 |
| nri.logging.level | string | `"info"` | NRI 插件的日志级别。 |
| nri.nativeSidecar | bool | `true` |  |
| nri.optOutAnnotation | string | `"nvml-mock.nvidia.com/inject"` | 设为 `"false"` 时禁用默认 mock 注入的 Pod 注解。 |
| nri.overlay.hostPath | string | `"/var/lib/nvml-mock"` | 包含已部署模拟驱动覆盖层的宿主机路径。 |
| nri.overlay.mountPath | string | `"/opt/nvml-mock"` | 模拟驱动覆盖层的容器挂载路径。 |
| nri.pluginIndex | string | `"10"` | NRI 插件排序索引。 |
| nri.pluginName | string | `"nvml-mock"` | NRI 插件名称。 |
| nri.readinessProbe.failureThreshold | int | `2` | 标记插件未就绪前的连续失败次数。 |
| nri.readinessProbe.httpGet.path | string | `"/readyz"` | 就绪探针端点路径。 |
| nri.readinessProbe.httpGet.port | string | `"nri-health"` | 就绪探针端点端口名。 |
| nri.readinessProbe.periodSeconds | int | `10` | 就绪探针间隔。 |
| nri.readinessProbe.timeoutSeconds | int | `2` | 就绪探针超时。 |
| nri.resources | object | `{"requests":{"cpu":"10m","memory":"32Mi"}}` | NRI 插件资源请求；默认不设限制，仅设置 requests。插件默认在每个节点上运行，在测得实际负载下的用量之前不设 limits。 |
| nri.socketPath | string | `"/var/run/nri/nri.sock"` | 运行时 NRI socket 路径。 |
| podAnnotations | object | `{}` | 添加到节点 agent Pod 的额外注解。 |
| podLabels | object | `{}` | 节点 agent Pod 的额外标签；请勿设置 app.kubernetes.io 选择器标签。 |
| priorityClassName | string | `""` | 节点 agent DaemonSet Pod 的 PriorityClass。 |
| terminationGracePeriodSeconds | int | `10` | 节点 agent DaemonSet 的 Pod 终止宽限期。 |
| tolerations | list | `[{"operator":"Exists"}]` | 模拟 DaemonSet 的 Pod tolerations。 |
| topology.domains | list | `[]` | 合成拓扑域与 clique 成员关系。 |
| topology.enabled | bool | `false` | 启用多节点 GPU fabric 拓扑模拟。 |
| updateStrategy.rollingUpdate.maxUnavailable | string | `"25%"` | 更新期间不可用的 DaemonSet Pod 最大数量。 |
| updateStrategy.type | string | `"RollingUpdate"` | DaemonSet 更新策略。 |
