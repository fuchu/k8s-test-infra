<!--
Copyright 2026 NVIDIA CORPORATION
SPDX-License-Identifier: Apache-2.0
-->

# NVSentinel 温度余量检测与自动恢复演示

这个演示要证明：[NVSentinel](https://github.com/NVIDIA/nvsentinel)（NVIDIA 开源的 GPU 健康监控与故障修复服务）能够**发现** GPU 温度异常、**修复**受影响的节点（cordon + drain），并在 GPU 降温后**自动恢复**该节点（uncordon）。整个过程跑在本地 Kind 集群上，**没有任何物理 GPU**，GPU 由 `nvml-mock` 模拟。

演示注入的故障是**温度余量越界**。NVSentinel 的 `GpuThermalMarginWatch` 会把每块 GPU 实时的**带符号** T.Limit 余量（DCGM 字段 153）与该 GPU 的**硬件降速偏移值**作比较。这个偏移不是 DCGM 字段：NVSentinel 的 `metadata-collector` 会从 NVML 字段 194（`NVML_FI_DEV_TEMPERATURE_SLOWDOWN_TLIMIT`）读取一次，写进 `gpu_metadata.json`。余量低于偏移值时 GPU 被判为不健康；回升之后检查就会解除。

```
nvml-mock（模拟的 libnvidia-ml.so）
   │  把 GPU 固定在高温 → T.Limit 余量（字段 153）变负
   │  slowdown offset（NVML 字段 194）→ metadata-collector → gpu_metadata.json
   ▼
GPU Operator 独立 DCGM（nv-hostengine :5555）
   ▼
NVSentinel GPU Health Monitor  ──►  platform-connector  ──►  MongoDB（change streams）
   (GpuThermalMarginWatch)                                    │
                                          fault-quarantine ◄──┘  （cordon）
                                                  │
                                          node-drainer            （drain → 工作负载重新调度）
```

**降温时**，DCGM 字段 153 是一个实时读数：下一次健康轮询就会看到余量重新超过降速偏移值，监控随即发出「恢复健康」事件，`fault-quarantine` 便 **uncordon** 该节点——**不需要重启 DCGM**（这一点与会一直锁存、必须重启 hostengine 才清除的 XID/ECC 故障不同）。

## 拓扑

1 个 control-plane + 2 个 worker：

- **Worker**（带 `nvml-mock-gpu=true`）：运行 `nvml-mock` DaemonSet 和 GPU Operator 的全部子组件（DCGM、device plugin、GFD）。每个节点按配置档对外声明模拟 GPU，默认 `h100` 是 8 块。
- **Control-plane**：运行 NVSentinel 的控制面流水线（MongoDB、platform-connector、fault-quarantine、node-drainer）。把它固定在这里，是为了让「驱逐某个 GPU worker」时，不会顺带把正在执行驱逐的流水线也一起赶走。

当一台 worker 的 GPU 过热时，NVSentinel 会 cordon/drain 它，示例 GPU 工作负载则重新调度到另一台健康的 worker 上。

## 前置条件

> **本演示会创建自己的 Kind 集群。** 它依赖固定的双 worker 拓扑和特定节点标签，因此会新建一个专属集群，而不会使用你当前的 `KUBECONFIG` context，也不会改动它。

- [Docker](https://docs.docker.com/get-started/get-docker/)，且守护进程正在运行。
- [Kind](https://kind.sigs.k8s.io/)，用于创建演示专属集群。
- [Helm 3.8 或更高版本](https://helm.sh/docs/intro/install/)——演示从 `oci://ghcr.io/nvidia/nvsentinel` 安装 NVSentinel，而对 OCI 仓库的支持正是 3.8 才有的。
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)。
- `jq`（可选，只用来美化输出节点 condition）。
- 能访问 `ghcr.io`（NVSentinel chart）、`helm.ngc.nvidia.com`（GPU Operator）和 `docker.io`（Percona 镜像）。
- `arm64` 或 `amd64` 主机。演示通过 Percona operator 部署 NVSentinel 的 MongoDB，而不是默认的 Bitnami store，因此 Apple Silicon 上也能跑（原因见下文）。

第一次接触 Mokka？先跑一遍[快速入门](../../quickstart.md)看看模拟 GPU 是什么样，再来做这个演示会更快上手。

全程约 30 分钟；如果本地镜像缓存是空的会更久，因为 GPU Operator、cert-manager 和 NVSentinel 都要拉取。

!!! note "国内镜像拉取"
    本演示会安装 NVSentinel（`ghcr.io`）、GPU Operator（`helm.ngc.nvidia.com` / `nvcr.io`）和 Percona（`docker.io`）的镜像。若这些镜像源拉取缓慢或超时，可给相关 chart 的镜像加私有仓库前缀（如给 GPU Operator 的各镜像加 `harbor.heilansc.com/nvcr.io` 前缀）。具体做法与示例见[国内镜像加速](../README.md#domestic-mirror)。

## 运行

```bash
cd docs/guides/nv-sentinel
./run.sh
```

脚本是幂等的，会复用集群；需要从头重建时设置 `FORCE_RECREATE=true`。常用覆盖项：`GPU_PROFILE`、`HOT_TEMP_C`、`TARGET_GPU`、`NVSENTINEL_VERSION`、`GPU_OPERATOR_VERSION`、`CERT_MANAGER_VERSION`。

## 脚本会做什么

1. **集群** — 用 [`kind.yaml`](kind.yaml) 创建 Kind 集群，并给两个 worker 都打上 `nvml-mock-gpu=true`。
2. **模拟 GPU** — 构建/加载 `nvml-mock` 镜像，把 chart 装到 worker 上。
3. **GPU Operator** — 用 [`gpu-operator-values.yaml`](gpu-operator-values.yaml) 安装：关掉真实 driver/toolkit（模拟层已经提供了），并**启用独立的 DCGM** DaemonSet + Service，供 NVSentinel 轮询。
4. **cert-manager** — 作为 TLS 依赖安装（Percona 和 NVSentinel 都靠它签发 MongoDB 证书）。
5. **NVSentinel** — 先应用 chart 的 Percona CRD 并等它们真正可用（Helm 4 在没加 `--wait` 的安装里不会等 CRD），再用 [`nvsentinel-values.yaml`](nvsentinel-values.yaml) 安装 chart：通过 Percona operator 拉起 MongoDB store，并把流水线接到独立 DCGM 上。这一步会启用 `metadata-collector`（用于提供降速偏移值），同时把温度余量检查从 dry-run 切换成真正会执行修复的动作。
6. **示例工作负载** — [`sample-workload.yaml`](sample-workload.yaml)，一个申请 1 块 `nvidia.com/gpu` 的 Pod，好让 drainer 有东西可以驱逐。
7. **阶段 1 — 检测 + 修复** — 把某台 worker 的 GPU 固定在高温（`HOT_TEMP_C`，默认 142 °C），等待 NVSentinel 把它 cordon；示例工作负载会重新调度到另一台 worker。
8. **阶段 2 — 自动恢复** — 清掉温度覆盖值，等待 NVSentinel 把节点 uncordon。整个过程不涉及重启 DCGM。

只要有一步没按预期发生，脚本就会以非零状态退出：示例工作负载没起来、加热前 worker 就已被 cordon、没有 cordon、没有重新调度、或者没有 uncordon。退出前它会打印每台 worker 的 cordon 状态、以及其上未通过的 NVSentinel 检查项；如果 GPU 还是烫的，也会顺手降温，这样重跑时节点是健康的。

## 故障与恢复过程

给 GPU 加热（脚本会做这一步）：

```bash
MOCK=$(kubectl --context kind-nvml-mock-nvsentinel -n mokka \
  get pod -l app.kubernetes.io/name=nvml-mock -o jsonpath='{.items[0].metadata.name}')
kubectl --context kind-nvml-mock-nvsentinel -n mokka exec "$MOCK" -- \
  nvml-mock-ctl temp --gpu 0 142
```

把温度固定在配置档的降速阈值之上（`h100` 是 87 °C，`gb300` 是 90 °C），会让 T.Limit 余量（DCGM 字段 153）变成负数。此时余量低于该 GPU 的降速偏移值（模拟层为 `0 °C`，来自 NVML 字段 194），GPU Health Monitor 的 `GpuThermalMarginWatch` 便以 `GPU_TEMP_HW_SLOWDOWN_VIOLATION` 失败，并产生类似这样的节点 condition：

```
GpuThermalMarginWatch=True: GPU 0 thermal margin -3°C below HW slowdown T.Limit (slowdown=0°C)
```

这个比较是**严格小于**，所以温度恰好压在阈值上时效余量是 `0 °C`，检查仍算健康——必须超过阈值才会触发。

随后 `fault-quarantine` 把节点 cordon，`node-drainer` 再把它 drain。

给 GPU 降温（同样由脚本完成）：

```bash
kubectl --context kind-nvml-mock-nvsentinel -n mokka exec "$MOCK" -- \
  nvml-mock-ctl reset --gpu 0
```

字段 153 是实时读数，所以下一次健康轮询就会看到余量恢复为正，`fault-quarantine` 随即 uncordon 节点。**不需要重启 DCGM**——这正是它与会锁存的 XID/ECC 故障最关键的区别。

> **关于模拟层能力的说明。** 这条链路依赖专为它新增的两个 `nvml-mock` 行为：一是模拟层会公开 T.Limit 阈值类字段（`NVML_FI_DEV_TEMPERATURE_*_TLIMIT`，id 193–196），好让 metadata-collector 读到降速偏移值；二是 `nvmlDeviceGetMarginTemperature` 返回的是**带符号**余量，超过降速阈值后会变成负数（而不是被截断在 0），这样检查才真的能触发。

## 为什么这些配置不能改

除了数据存储方案之外，`nvsentinel-values.yaml` 还设了几项非默认值，它们对演示能否完整跑通至关重要（该文件里也都有行内注释说明）：

- **`global.metadataCollector.enabled: true`**。`metadata-collector` DaemonSet 会读取每块 GPU 的降速 T.Limit 偏移值（NVML 字段 194）一次，写入 `gpu_metadata.json`。没有它，`GpuThermalMarginWatch` 永远不会进入工作态，只会打印 *"missing slowdown TLIMIT threshold metadata"*。默认关闭。
- **`labeler.assumeDriverInstalled: true`**。metadata-collector 只会调度到带 `nvsentinel.dgxc.nvidia.com/driver.installed=true` 标签的节点上，而 labeler 通常只在看到真实的 `nvidia-driver-daemonset` 时才打这个标签。本演示关掉了真实驱动（由模拟层提供 `libnvidia-ml.so`），所以要让 labeler 直接在每个 `gpu.present` 节点上假定驱动已就绪——这也是 NVIDIA 为预装驱动的主机提供的开关。
- **`gpu-health-monitor.dcgmFieldsMonitoring.gpuTempLimitStoreOnly: false`**。`GpuThermalMarginWatch` 出厂时是 **dry-run**（仅存储）模式：只发事件，绝不动节点。关掉 store-only，收紧的温度余量才真正驱动 cordon/drain 流水线（以及恢复时的 uncordon）。
- **`gpu-health-monitor.dcgmHealthCheck.suppressedErrorCodes`** 里加了 `DCGM_FR_NVLINK_EFFECTIVE_BER_THRESHOLD`。模拟层在启动时会给每块 GPU 都报一次 NVLink effective-BER 阈值越界，这是模拟数据带来的假象，不是本演示要测的故障。若放着不管，它会永远是一项「未通过的检查」，即便 GPU 降温，节点也一直被隔离，恢复永远完不成。屏蔽它，降温后所有检查才能全部通过。
- **`suppressedErrorCodes`** 里还加了 `DCGM_FR_IMEX_UNHEALTHY`。在 NVLink 域跨节点的板子（例如 `gb300`）上，Mokka 会准备 IMEX 工具，于是 DCGM 的健康检查会去执行 `nvidia-imex-ctl -q`。演示里没有跑 IMEX daemon，查询失败，DCGM 报 `IMEX daemon status is -3 (not READY)`，NVSentinel 就会在还没加热任何 GPU 之前把两台 worker 都 cordon 掉。默认的 `h100` 没有 `nvidia-imex-ctl`，DCGM 视其为健康。真实的多节点 NVLink 集群会运行 IMEX daemon，应保留这项检查。
- **`fault-quarantine.circuitBreaker.enabled: false`**。熔断器的规则是：5 分钟内被 cordon 的 GPU 节点达到 50% 就触发，随后**所有**事件处理都会停摆（包括恢复时的 uncordon）。这里只有两台 GPU worker，一次正常的 cordon 就已经达到阈值，所以在这个迷你集群里关掉它。**真实的大集群请保持默认开启。**
- **`node-drainer.userNamespaces[*].mode: Immediate`**。默认的 `AllowCompletion` 会等每个 Pod 自行优雅结束；而示例 GPU 工作负载永远不会自己结束，于是永远不会被驱逐，你也就看不到它迁到健康 worker 上。改成 `Immediate` 才能观察到 drain → 重新调度这一步。真实集群一般仍用 `AllowCompletion`。

## 关于宿主机资源

演示要同时跑 GPU Operator + DCGM + MongoDB + 整套 NVSentinel 流水线。宿主机比较忙的时候（比如同时开着好几个 Kind 集群），GPU worker 在 GPU Operator 拉起期间可能被 CPU 打满，进而拖慢镜像拉取和 Pod 就绪（cert-manager 和 DCGM 尤其明显）。`run.sh` 已经给了很宽松的等待时间；如果某步还是超时，重跑一次（它会复用集群）或者腾掉其他集群通常就能解决。

## 为什么用 Percona MongoDB store

NVSentinel 的 `mongodb-store` 子 chart 默认使用 Bitnami MongoDB chart，它的镜像（`bitnamilegacy/*`）只有 amd64 版本，容器里跑的也是 Bitnami 专属启动脚本，在 arm64 上根本起不来。该子 chart 另外提供了一条 [Percona Server for MongoDB](https://docs.nvidia.com/nvsentinel/runbooks/mongo-db-bitnami-to-percona-migration/) 路径，其涉及的每个镜像——`percona-server-mongodb`、operator、`mongosh`——都同时发布 `linux/amd64` 和 `linux/arm64` 版本，所以演示选了它：

```yaml
mongodb-store:
  useBitnami: false
  usePerconaOperator: true
```

两个开关都必须设，它们决定 Helm 会拉哪些 chart 依赖。

随后演示又按「笔记本能带动的 Kind 集群」把 Percona 默认配置削了一圈：改成**单成员**副本集（`unsafeFlags.replsetSize: true`，否则 Percona 坚持要三个）、调低 CPU/内存请求、去掉 metrics sidecar。NVSentinel 只需要 change streams，单成员副本集就能提供。

## 查看结果

```bash
CTX=kind-nvml-mock-nvsentinel
kubectl --context $CTX get nodes
kubectl --context $CTX -n nvsentinel get pods
# cordon / quarantine / uncordon 相关事件：
kubectl --context $CTX -n nvsentinel logs -l app.kubernetes.io/instance=nvsentinel \
  --prefix --tail=500 | grep -iE 'cordon|quarantin|recovered'
```

## 清理

```bash
kind delete cluster --name nvml-mock-nvsentinel
```
