# 快速开始

在 Kubernetes 集群上安装 Mokka，然后看 `nvidia-smi` 报告并不存在的 GPU。五分钟搞定，无需 NVIDIA 硬件。

## 前置条件

- [Helm 3.8 或更高版本](https://helm.sh/docs/intro/install/)——chart 从 OCI 仓库提供。国内安装二进制时优先用 `https://get.helm.sh/helm-<版本>-linux-amd64.tar.gz`，备用[华为云镜像](https://repo.huaweicloud.com/helm/)；`get-helm-3` 脚本和 GitHub release 直链在国内常受限。
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)，指向你要使用的集群。
- Docker 和 [Kind](https://kind.sigs.k8s.io/)——仅当你需要一个临时的本地集群时才需要。

## 安装

```bash
kind create cluster --name mokka

helm install nvml-mock \
    oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
    --namespace mokka \
    --create-namespace
```

已经有集群？跳过 `kind` 那一行即可。chart 会自行准备每个节点的 containerd，因此只要节点在 systemd 下运行 containerd 1.7 或更高版本，任何集群都可以；托管集群可参见 [Amazon EKS 指南](guides/install/aws/eks/README.md)。

在默认的 `gb300` profile 下，每个节点还会从 `developer.download.nvidia.com` 下载 NVIDIA 的 IMEX 归档包，用于 DRA ComputeDomain。无法访问该地址的节点仍可在没有 IMEX 的情况下工作；添加 `--set featureGates.IMEXDownload=false` 可跳过下载。

### 国内环境镜像加速

国内直连 `ghcr.io` 较慢时，给 `nvml-mock` chart 设置 `global.imageRegistry`：

```bash
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
    --namespace mokka --create-namespace \
    --set global.imageRegistry=shharbor.heilansc.com/ghcr.io
```

chart 会剥掉原 repository 的 `ghcr.io`，最终拉取 `shharbor.heilansc.com/ghcr.io/nvidia/nvml-mock:<tag>`。请把镜像预先同步到该路径；后续 `helm upgrade` 也要带上相同的覆盖，或写入 values 文件。私有仓库需要认证时，可加 `--set global.imagePullSecrets[0]=<secret>`。

其它 chart（DRA driver、NFD、GPU Operator 等）通常需要覆盖 `image.repository`；kind 节点还要能解析私有仓库域名。完整示例见[国内镜像加速](guides/README.md#domestic-mirror)。

## 验证

```bash
kubectl exec -n mokka ds/nvml-mock -- nvidia-smi -L
```

```text
GPU 0: NVIDIA GB300 NVL (UUID: GPU-...)
GPU 1: NVIDIA GB300 NVL (UUID: GPU-...)
GPU 2: NVIDIA GB300 NVL (UUID: GPU-...)
GPU 3: NVIDIA GB300 NVL (UUID: GPU-...)
```

这是未经修改的真实 `nvidia-smi` 二进制文件，它读取的是 Mokka 的驱动，而不是物理驱动。`nvidia-smi -q` 同样可用，会报告完整的 profile。

## 修改 GPU 型号

`gb300` 是默认型号。集群中的每个节点都采用同一个 profile：

```bash
helm upgrade nvml-mock \
    oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
    --namespace mokka \
    --set gpu.profile=a100
```

chart 自带八种 profile：`a100`、`b200`、`gb200`、`gb300`、`h100`、`l40s`、`t4` 和 `vrnvl72`。[配置参考](configuration.md)介绍每种 profile 定义了什么，以及如何修改单个值。

## 清理

```bash
helm uninstall nvml-mock --namespace mokka
kind delete cluster --name mokka          # 如果上面创建过集群
```

## 下一步

你现在拥有了一个*看起来*有 GPU 的节点。有意思的部分是把真实软件指向它。

| 要做的事 | 前往 |
|---|---|
| 使用 device plugin、DRA 或 GPU Operator 调度 GPU 工作负载 | [安装](helm-chart.md) |
| 让一块 GPU 发生故障，观察消费者如何反应 | [故障注入](guides/failure-injection/README.md) |
| 检查 GPU 工作负载是否获得模拟驱动 | [配置 NRI 注入](guides/nri-injection.md) |
| 在运行中的节点上修改温度、功耗或健康状态 | [运行时控制](nvml-mock-ctl.md) |
| 理解实际发生了什么 | [架构](architecture.md) |

节点守护进程会在每个节点上安装 NVIDIA 容器运行时，并将其设为 containerd 的默认 handler `nvidia`，同时启用容器设备接口（CDI）。这是 [device plugin](guides/device-plugin.md)、[DRA](guides/dra.md) 和 [GPU Operator](guides/gpu-operator.md) 路径所使用的运行时配置；参见[容器运行时设置](components/node-daemon.md#container-runtime-setup)。
