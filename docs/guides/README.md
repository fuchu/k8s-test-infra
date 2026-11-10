# 指南

这里是一组可以照着动手操作的教程，每篇演示 Mokka 的一种具体用法。

## 开始前

所有指南都需要 [Helm 3.8 或更高版本](https://helm.sh/docs/intro/install/) 和 [kubectl](https://kubernetes.io/docs/reference/kubectl/)。大多数教程用 [Docker](https://docs.docker.com/get-started/get-docker/) 加 [Kind](https://kind.sigs.k8s.io/) 在本地起一个集群；Amazon EKS 那篇既可以用现成的兼容集群，也可以用 Terraform 新建一个。每篇教程在安装任何东西之前，都会先写清楚目标和前置条件。

## 国内镜像加速 <a id="domestic-mirror"></a>

指南里大部分 `helm install` 都从境外 registry 拉取镜像：`ghcr.io`（Mokka、Slinky、Run:ai FGO）、`helm.ngc.nvidia.com` / `nvcr.io`（GPU Operator、DRA driver、NVIDIA device plugin）、`quay.io`（cert-manager）、`docker.io`（Percona）等。国内直连这些域名常常很慢甚至超时。

如果你环境里有可用的私有镜像仓库（下文以 `harbor.heilansc.com` 为例，请替换成你自己的域名），可以给 chart 用的镜像统一加上**仓库前缀**，把原 registry 嵌套到前缀之下。两种常见写法：

- **Mokka 的 `nvml-mock` chart（OCI，来自 `ghcr.io`）** 自带 `global.imageRegistry` 开关，直接加前缀即可：

  ```bash
  helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
    --namespace mokka --create-namespace \
    --set global.imageRegistry=harbor.heilansc.com/ghcr.io
  ```

  渲染后实际镜像为 `harbor.heilansc.com/ghcr.io/nvidia/nvml-mock:<tag>`：chart 会先剥掉原 `image.repository` 里的 `ghcr.io`，再用 `global.imageRegistry` 整体前缀替换。因此镜像在仓库里的路径应为 `ghcr.io/nvidia/nvml-mock`。

- **其它 chart（GPU Operator、cert-manager、Slurm、NVSentinel、device plugin、DRA driver、Topograph 等）**：多数不直接暴露 `global.imageRegistry`，需要显式覆盖镜像的 `image.repository`，把原 registry 拼到前缀之后。例如：

  ```bash
  # NVIDIA device plugin（原 registry 为 nvcr.io）
  helm upgrade --install nvidia-device-plugin nvdp/nvidia-device-plugin \
    --set image.repository=harbor.heilansc.com/nvcr.io/nvidia/k8s-device-plugin

  # DRA driver（原 registry 为 nvcr.io）
  helm install nvidia-dra-driver nvidia/dra-driver-nvidia-gpu \
    --set image.repository=harbor.heilansc.com/nvcr.io/nvidia/dra-driver-nvidia-gpu

  # Node Feature Discovery（原 registry 为 registry.k8s.io）
  helm upgrade --install nfd nfd/node-feature-discovery \
    --set image.repository=harbor.heilansc.com/registry.k8s.io/nfd/node-feature-discovery
  ```

  具体覆盖哪个值，取决于各 chart 的 values 结构：有的 chart 所有容器共用同一个 `image` 值（如 NFD、device plugin、DRA driver），覆盖一次即可；有的每个子组件各带一份（如 GPU Operator），需要逐个覆盖或查阅其文档。共同原则是：**把 chart 用到的每个镜像 repository 前面，统一加上 `harbor.heilansc.com/<原registry>`**。

!!! tip "Kind 节点上的两点注意"
    1. **节点镜像本身**：`kind create cluster` 默认拉取 `kindest/node`，国内也可能很慢。可以给 Docker 配置 `--registry-mirror`，或用 `kind create cluster --image` 指定国内镜像。
    2. **kind 节点解析不到私有仓库域名**：宿主机能解析 `harbor.heilansc.com`，但 kind 节点容器默认解析不到，会导致镜像拉取报 `lookup ... connection refused`。持久修复是在 kind 配置里加 `extraHosts`，让节点创建时自动写入：

    ```yaml
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    nodes:
    - role: control-plane
      extraHosts:
      - "harbor.heilansc.com:10.1.1.21"   # 10.1.1.21 换成你环境解析到的真实 IP
    - role: worker
      extraHosts:
      - "harbor.heilansc.com:10.1.1.21"
    ```

## 场景：让 Mokka 对接真实的 GPU 消费方

下面这些教程会把 Mokka 和一个真实的「消费方」组件一起搭起来。大致按上手难度从低到高排列，新手建议从第一条开始。

| 指南 | 演示内容 | 预计耗时 |
|---|---|---|
| [NVIDIA Device Plugin](device-plugin.md) | 把模拟 GPU 以 `nvidia.com/gpu` 资源的形式暴露出来，并让工作负载调度到这些 GPU 上 | 约 5 分钟 |
| [Node Feature Discovery](nfd.md) | NFD 读取 Mokka 生成的特性文件，给节点打上「存在 GPU」的标签 | 约 10 分钟 |
| [Amazon EKS](install/aws/eks/README.md) | 在私有、纯 CPU 的托管工作节点（标准 AL2023 镜像）上跑通 device-plugin 这条路径 | 20–30 分钟 |
| [NVIDIA GPU Operator](gpu-operator.md) | 用真实的 Operator 全家桶（device plugin、GFD、DCGM、validator）对接模拟 GPU | 约 15 分钟 |
| [NVIDIA Dynamo](dynamo/README.md) | 用 Dynamo 提供 OpenAI 兼容的接口，其 worker 经 GPU Operator 调度到模拟 GPU 上 | 约 15 分钟 |
| [Slinky (Slurm)](slinky/README.md) | 搭一个 Slurm 集群，让 `slurmd` 把模拟 GPU 识别为 GRES，并用 `srun --gres=gpu:N` 提交作业 | 约 20 分钟 |
| [NVIDIA DRA Driver](dra.md) | 把模拟 GPU 发布成 ResourceSlice，再通过 ResourceClaim 调度 Pod | 约 10 分钟 |
| [Run:ai fake-gpu-operator](runai-fgo/README.md) | 两个节点池：一个由 Mokka 提供真实的 NVML 垫片，另一个由 FGO 提供 | 约 10 分钟 |
| [Topograph](topograph/README.md) | 把 4 台模拟 GB200 节点划分成两个 NVLink 域，并为每个节点打标签，供拓扑感知调度器使用 | 约 10 分钟 |
| [ComputeDomain](compute-domain/README.md) | 构造 NVLink 网络标识：用真实的 `nvidia-imex` 在模拟 GPU 上组建一个可用的域 | 10–20 分钟 |
| [NVSentinel](nv-sentinel/README.md) | 完整的健康检查闭环：检测温度越界 → 隔离并驱逐节点 → 降温后自动恢复 | 约 30 分钟 |

最后两篇最复杂：ComputeDomain 需要一个 4 个 worker 节点、且启用了 containerd NRI 的集群；NVSentinel 则要先把 GPU Operator、cert-manager 和 NVSentinel 都装好才能开始。

## 任务：用 Mokka 能做什么

这些操作和你接了哪个消费方无关，随时都能用。

| 指南 | 涵盖内容 |
|---|---|
| [配置 NRI 注入](nri-injection.md) | 验证持有 GPU 分配的 Pod 是否拿到了模拟驱动，同时让普通工作负载不受影响 |
| [MIG 切分](mig/README.md) | 把一块 GPU 切成多个 MIG 实例，逐个暴露出来，并把 Pod 调度到单个实例上 |
| [vGPU 宿主机与客户机](vgpu/README.md) | 把节点配置成运行 vGPU 实例的宿主机，或配置成已授权的 vGPU 客户机，并用 `nvidia-smi` 查看两侧状态 |
| [故障注入](failure-injection/README.md) | 模拟一块坏掉的 GPU（不可纠正的 ECC 错误、设备丢失、从总线上掉线），观察各消费方的反应 |
| [在 CI/CD 中使用](ci-cd.md) | 在没有 GPU 的 CPU runner 上跑依赖 GPU 的测试 |
| [运行时控制](../nvml-mock-ctl.md) | 无需重新部署，直接修改运行中节点的温度、功耗、利用率或健康状态 |

## 可观测性（Prometheus + Grafana）

这部分不是独立教程。它是叠加在 GPU Operator 之上的，并不替换 GPU Operator，因此放在 Tilt 环境里，不单独提供集群和 `run.sh`。

Prometheus 抓取的是未经修改的官方 NVIDIA `dcgm-exporter` 指标——只不过 exporter 读的是模拟的 `libnvidia-ml.so`；Grafana 负责展示。整个过程发生在一台 GPU 都没有的集群上。此外还有两个手动触发器，用来注入温度或 Xid 故障：如果指标始终没有出现在 Prometheus 里，触发动作就会失败。也就是说，抓取链路是被「断言」验证的，而不是靠肉眼看图。

```bash
make cluster-create
tilt up -- --observability
```

Tilt 环境及所有可用参数见 [本地开发](../contributing/local-development.md)。
