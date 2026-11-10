# nvml-mock 故障注入演示

这是一篇端到端演练，会在你当前 context 指向的集群上跑一遍 GPU 故障注入功能。它通过 `helm upgrade --reuse-values` 在各个场景之间重新部署 chart，依次走遍所有支持的故障模式（`healthy` → `ecc_uncorrectable` → `lost` → `fallen_off_bus`），并针对 Pod 内的 `nvidia-smi` 输出，逐一断言每种模式的预期表现。

设置 `BUILD_LOCAL=true` 时，演示会创建自己专属的集群 `nvml-mock-failure-demo`，这也是唯一能把它与其他 Mokka release 完全隔离的方式。

默认路径则直接使用你当前指向的集群：它会把名为 `nvml-mock-failure` 的 release 装进独立命名空间 `mokka-failure`（可用 `NAMESPACE=...` 覆盖）。这两个名字都区别于快速入门里的 `nvml-mock` / `mokka`，因此两套安装不会误把对方的对象当成自己的。

> **不要把 `NAMESPACE=` 指向另一个 Mokka release 的命名空间。** 如果那里已经有别的 Mokka release，演示会拒绝安装并以 `4` 退出。之所以硬性拒绝，是因为**无论命名空间叫什么，两个 release 都共享同一份节点级宿主机状态**。两者的 DaemonSet 都挂载同样的 hostPath：`/var/lib/nvml-mock`、`/var/run/cdi`、`/run/nvidia` 以及 NFD 特性目录，而这些路径都不按 release 或命名空间做隔离。
>
> 在真实集群上实测过：把两个 release 放在一起跑过一轮后，唯一的共享文件 `/var/lib/nvml-mock/driver/config/config.yaml`（CDI 规格 `/var/run/cdi/nvidia.yaml` 用 `MOCK_NVML_CONFIG` 指向它）被留成了 `failure: {after_calls: 1, mode: fallen_off_bus, code: 79}`。
>
> 两个演示自己的 Pod 察觉不到，因为它们各自挂载自己的 ConfigMap。**但任何调度到这些节点上的真实 GPU 工作负载都会受影响**：它会不知不觉读到这份注入了故障的配置——而这恰恰是这个模拟层应当如实提供的行为。发生这种情况时，两个演示仍然都退出 `0`，所以这里必须是硬性终止，而不是给个警告。
>
> `DEMO_ASSUME_YES=true` 可以绕过这个检查。一旦用了它，请在单独重装其中一个演示之前，把受影响的节点视为「已污染」。

**名字不同只是记账方便，并不是隔离。** chart 的 hostPath 挂载路径是固定的、与 release 无关（`/var/lib/nvml-mock`、`/var/run/cdi`、`/run/nvidia`、NFD 特性目录），而且它默认带着 `nodeSelector: {}` 和 `tolerations: [{operator: Exists}]`，因此两个 release 的 DaemonSet 都会落到每个节点上，写出同一份节点级宿主机状态。请不要在同一个集群上把本演示和另一个 Mokka release 并排运行。

至于 Kind 集群拓扑，直接复用共享的 [`../kind.yaml`](../kind.yaml)（1 个 control-plane + 3 个带 FGO 风格标签的 worker）即可，故障注入不需要单独的拓扑。

## 前置条件

- **一个 Kubernetes 集群和有效的 `KUBECONFIG`。** 演示会装进当前 context 指向的集群，可用 `kubectl config current-context` 确认。
- **克隆好的本仓库。** chart 是从本地检出目录安装的，不从镜像仓库拉。
- [Helm 3.8 或更高版本](https://helm.sh/docs/intro/install/)——这是演示预检强制要求的最低版本。
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)，版本与集群匹配。

> **还没有集群？** [快速入门](../../quickstart.md) 能用 Kind 在一分钟左右建一个一次性的，建完再回来。

[Docker](https://docs.docker.com/get-started/get-docker/) 和 [Kind](https://kind.sigs.k8s.io/) 只在 `BUILD_LOCAL=true` 时需要：那种模式会从源码构建镜像、旁路加载进名为 `nvml-mock-failure-demo` 的 Kind 集群（集群不存在就创建）。默认路径拉的是已发布的镜像，不会创建或删除任何集群。

### 冷集群上第一次跑会比较慢

镜像约 100 MB。在一个从没拉过它的集群上，实测冷启动拉取**每个节点大约要 8 分钟**，所以演示现在最多等 **15 分钟**（以前是 2 分钟）。安装前它会打印提示，那段时间里长时间没输出是在拉镜像，不是卡死。

首次安装时所有节点会自然并发拉取，因为新建 DaemonSet 会立刻创建全部 Pod。演示还额外设置了 `updateStrategy.rollingUpdate.maxUnavailable=100%`，这在**重跑**时很关键：chart 默认 25%，会让已有 release 的滚动更新退化成一次只更新一个节点。

链路慢的话可以调大等待预算：

```bash
HELM_TIMEOUT=30m ./run.sh
```

注意 `ghcr.io/nvidia/nvml-mock:latest` 是浮动标签，集群若缓存了旧的 `:latest`，跑起来的构建版本可能和这份 chart 对不上。需要固定配对时，请用 `NVML_MOCK_IMAGE` 指定一个已发布的版本标签。

!!! note "国内镜像拉取"
    演示默认从 `ghcr.io/nvidia/nvml-mock:latest` 拉取镜像。若 `ghcr.io` 拉取缓慢或超时，可用 `NVML_MOCK_IMAGE` 指向镜像仓库里带前缀的版本（如 `harbor.heilansc.com/ghcr.io/nvidia/nvml-mock:<tag>`）。具体做法与示例见[国内镜像加速](../README.md#domestic-mirror)。

## 脚本会做什么

1. **先报出目标**：context、API server、节点数，以及工作负载所在的命名空间。除非目标是一次性 Kind 集群（判断依据是 context 名为 `kind-*` **且** API server 是回环地址——光看名字说明不了什么），否则它会要求你把 context 名字原样输入一遍，确认后才开始安装。由于 chart 会跑特权 DaemonSet 并挂载 hostPath，目标没确认前不会安装任何东西。

   如果没人能应答，脚本会**安全地失败**：非交互模式下，只要目标不是一次性集群，就直接以 `4` 退出，不会在无人值守的情况下装东西。确实要在无终端环境下继续时，设置 `DEMO_ASSUME_YES=true`。

   退出码含义：

   - `2`：没有可用的集群；
   - `3`：工具缺失或版本过旧；
   - `4`：确认被拒绝 / 无法进行确认 / 集群上已装了另一个 nvml-mock 演示而拒绝执行 / helm 无法列出 release 因而无法排除该情况；
   - `5`：演示无法表达的配置（例如用 digest 固定的 `NVML_MOCK_IMAGE`）。

   之后每一次 `kubectl` 和 `helm` 调用，都会同时锁定它报出的 **context** 和 **命名空间**。

2. 仅当 `BUILD_LOCAL=true` 时：集群 `nvml-mock-failure-demo` 不存在就创建，然后构建并加载 `nvml-mock:failure-demo` 镜像。否则使用 `ghcr.io/nvidia/nvml-mock:latest`（可用 `NVML_MOCK_IMAGE` 覆盖）。

3. **场景 1 — healthy 基线**。以 `gpu.failureInjection.enabled=false` 安装，并断言：

   - 渲染出的 ConfigMap 里**没有** `failure:` 块；
   - `nvidia-smi -L` 至少列出一块 GPU。脚本会记下实际观测到的数量，供后续场景复用——Pod 内的 GPU 数来自 chart 渲染出的 ConfigMap（也就是所选配置档），不受 `--set gpu.count` 影响，后者只作用于宿主机侧的 CDI 规格；
   - 聚合的不可纠正 ECC 计数初始为 `0`。

4. **场景 2 — `ecc_uncorrectable` + Xid 79**。以 `mode=ecc_uncorrectable, after_calls=1, xid.code=79` 升级 release、重建 DaemonSet，并断言：

   - ConfigMap 里现在有 `mode: ecc_uncorrectable`；
   - `nvidia-smi -L` **仍然**列出所有 GPU（这是该模式的约定：设备保持可寻址）；
   - `nvidia-smi --query-gpu=ecc.errors.uncorrected.aggregate.total --format=csv,noheader,nounits` 至少为一块 GPU 输出一个严格大于 0 的整数。每块设备各自维护调用计数器，所以这一次查询（每块 GPU 一次受保护的 ECC 读取）会在同一次 `nvidia-smi` 调用里，让每块 GPU 在首次读取时触发。

5. **场景 3 — `lost`**。以 `mode=lost, after_calls=1` 升级、重建 DaemonSet，并断言 `nvidia-smi --query-gpu=temperature.gpu --format=csv` 输出的是错误标记（`[N/A]`、`[Unknown Error]` 等），而不是正常的整数。第一次受保护的指标调用会让设备「掉线」，此后每一次 NVML 调用（指标、身份查询、句柄查找）都会返回 `ERROR_GPU_IS_LOST`。

6. **场景 4 — `fallen_off_bus` + Xid 79**。NVML 层面的表现与 `lost` 相同，但额外把 `xid.code=79` 排入 NVML 事件集。断言同样的 `nvidia-smi` 错误标记行为，并确认 ConfigMap 同时带有 `mode: fallen_off_bus` 和 `code: 79`。

每个场景都用 `helm upgrade --reuse-values`，这样各轮之间只改动故障注入相关的开关，其他配置（镜像、配置档、数量）都沿用场景 1 的设置。

## 快速开始

```bash
./run.sh
```

脚本是幂等的，可以反复运行：已有集群会被复用，`helm upgrade --install` 同时覆盖首次安装和后续升级。

## 注意事项

### Xid 事件通过 NVML 事件集传递，不会出现在 `nvidia-smi` 里

`nvidia-smi` 并不订阅 `nvmlEventSetWait`，因此永远不会打印 `Xid 79`。模拟层是按标准 NVML 事件集（`NVML_EVENT_TYPE_XID_CRITICAL_ERROR`）投递配置好的 Xid，且每个引擎生命周期内只投一次，与真实 NVML 的语义一致。真正的消费方通过以下途径看到它：

- **NVIDIA device plugin** 的健康检查（把该 GPU 标记为 `Unhealthy`）；
- **dcgm-exporter**（`DCGM_FI_DEV_XID_ERRORS` 指标）；
- 一段调用 `nvml.EventSetCreate` / `RegisterEvents(EventTypeXidCriticalError)` / `EventSetWait` 的小程序，适合临时验证：

  ```go
  set, _ := nvml.EventSetCreate()
  dev, _ := nvml.DeviceGetHandleByIndex(0)
  dev.RegisterEvents(nvml.EventTypeXidCriticalError, set)
  for i := 0; i < 5; i++ { _, _ = dev.GetTemperature(nvml.TEMPERATURE_GPU) }
  ev, _ := nvml.EventSetWait(set, 1000)
  // ev.EventType == 0x8 (XID_CRITICAL_ERROR), ev.EventData == 79
  ```

  `nvmlEventSetWait_v1` 和 `nvmlEventSetWait_v2` 都已导出且行为一致。和真实 NVML 一样，没有待处理事件时，等待会阻塞满整个超时（上例是 1000 ms），然后返回 `NVML_ERROR_TIMEOUT`。

  上面那个 `GetTemperature` 循环是**不能省**的：等待每 100 ms 复查一次，但只认领**已经**处于 pending 的 Xid；而设备只在受保护的**设备类**调用上触发注入器，绝不会在 wait 里触发。去掉这些调用，无论 `failure.xid` 怎么配，wait 都会一直阻塞到超时。

### 注入计数器按进程隔离，进程内再按设备隔离

每次 `kubectl exec ... -- nvidia-smi` 都是一个新进程，带着全新的 `failureInjector`，计数从 0 开始。因此阈值必须在**同一次调用内**达到，否则永远不会达到。计数还是按设备分开的：每块 GPU 只累计自己的受保护调用，而不是共用一个进程级总数。

这就是为什么场景 2、3、4 都用 `after_calls: 1`。像场景 2 的 `--query-gpu=ecc.errors.uncorrected.aggregate.total` 这种「每块 GPU 恰好发起一次受保护调用」的查询，依然能让每块 GPU 都触发，因为各自在自己的首次读取上就达到了阈值。如果把 `after_calls` 调大，就需要单条命令对同一块设备重复命中那么多次，而演示里的一次性查询做不到。

想交互式观察的话，可以用一个长驻进程，例如：

```bash
kubectl --context kind-nvml-mock-failure-demo -n mokka-failure exec -it "$POD" -- nvidia-smi \
  --query-gpu=index,ecc.errors.uncorrected.aggregate.total \
  --format=csv -l 1
```

然后就能看到计数器每次轮询都在加一。

各模式的完整行为约定见 [`pkg/gpu/mocknvml/README.md#failure-injection-optional`](https://github.com/NVIDIA/k8s-test-infra/blob/main/pkg/gpu/mocknvml/README.md#failure-injection-optional)。

## 清理

```bash
# 默认路径：集群本来就是你的，所以只删掉这个 release。
# -n 和 --kube-context 都不能省。少了它们，helm 会按当前碰巧生效的
# context 和命名空间去解析 release，在共享集群上就可能删掉另一个 Mokka release。
helm uninstall nvml-mock-failure -n mokka-failure --kube-context <your-context>

# BUILD_LOCAL=true 还额外创建了集群，所以也要拆掉：
kind delete cluster --name nvml-mock-failure-demo
```
