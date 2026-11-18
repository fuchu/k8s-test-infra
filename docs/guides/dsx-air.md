# DSX Air 上的 Mokka

使用 DSX Air 上托管的 **Kubernetes 入门 – Mokka + KWOK** 演示，无需物理 GPU 或本地仓库检出即可探索 Kubernetes GPU 技术栈。选择已配置好的环境来探索该技术栈，或选择未配置的环境来动手完成其设置。

## DSX Air 提供什么

[NVIDIA DSX Air](https://docs.nvidia.com/dsx-air)是一个云托管的数据中心仿真平台，用于验证基础设施配置、功能和自动化。它让你构建基础设施环境的数字孪生，并试验网络拓扑和操作系统。

它的 [Demo Marketplace](https://docs.nvidia.com/dsx-air/pre-built-demos)提供现成的仿真及配套说明。启动一个演示会在你的账户中创建一份副本，你可以修改和重建。Mokka + KWOK 演示采用这种交付模式，提供一个托管环境来探索 Kubernetes GPU 技术栈。

## 集成如何组合

这些演示把三层组合在一起：

| 层 | 角色 |
|---|---|
| DSX Air | 托管演示环境及其实验说明。 |
| Mokka | 在运行容器的 CPU 节点上提供模拟 GPU 驱动接口，使 `nvidia-smi` 等真实工具和 GPU 管理组件能够发现模拟设备。参见 [Mokka 架构](../architecture.md)。 |
| Kubernetes WithOut Kubelet（KWOK） | 表示虚拟 Kubernetes 节点，无需在每个节点上运行 kubelet 或工作负载容器，从而可以用更大规模的节点数量进行控制面演练。 |

这些层分别演练技术栈的不同部分。KWOK 节点的 Kubernetes 状态并不能证明容器执行。用 `nvidia-smi` 检查驱动接口时，请使用由 Mokka 支撑且运行容器的节点。

## 访问 DSX Air

DSX Air 使用 NVIDIA NGC 做认证和组织管理。你需要一个 NGC 账户和组织；用工作邮箱登录并选择要使用的组织。每个用户都需要组织级 **Air User** 或 **Air Org Admin** 角色。入门和访问排障参见[账户设置](https://docs.nvidia.com/dsx-air/account-setup)。

## 选择演示

| 演示 | 用途 |
|---|---|
| [Kubernetes 入门 – Mokka + KWOK 已配置](https://dsx-air.nvidia.com/demos/07891765-a16f-4fae-8f36-21c2babd739f) | 探索配置好的环境。 |
| [Kubernetes 入门 – Mokka + KWOK 未配置](https://dsx-air.nvidia.com/demos/23ba81c5-8180-4075-90bb-7c502fb559a8) | 按照演示的实验说明完成配置。 |

1. 打开选定的演示，点击 **Launch** 创建你的副本。部署可能需要几分钟；可在 **Simulations** 列表中找到副本。
2. 打开演示的 **Documentation**，查看凭据、集群访问方式、组件版本和设置步骤。启动流程参见 [Pre-built Demos](https://docs.nvidia.com/dsx-air/pre-built-demos)。
3. 在仿真的 **Topology** 标签页中，等状态变为 **Active**，然后双击节点打开其控制台。选择实验说明中指定的节点进行 Kubernetes 访问。[DSX Air 快速开始](https://docs.nvidia.com/dsx-air/quick-start)还介绍节点属性以及通过 **Services** 标签页进行的外部连接。

对于**已配置**的演示，先检查现有部署再改动。对于**未配置**的演示，先完成实验设置再检查模拟 GPU。如果这些步骤需要更多背景，[安装参考](../helm-chart.md)介绍 Mokka 的部署选项。

## 检查环境

在实验终端中运行以下命令，`kubectl` 已配置好指向演示集群：

```bash
kubectl config current-context
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get nodes \
  -o custom-columns='NODE:.metadata.name,GPUS:.status.allocatable.nvidia\.com/gpu'
```

节点和 Pod 列表有助于确定各组件运行在哪里。最后一条命令显示通告的 `nvidia.com/gpu` 资源（如果存在）。该资源计数是分配视图；它不能证明节点是否运行容器，也不能证明是否提供了 Mokka 的驱动接口。

Mokka 配置完成后，在 Pod 列表中找到它的某个节点守护进程 Pod。把下面的 `<namespace>` 和 `<mokka-pod>` 替换为它的命名空间和 Pod 名称：

```bash
kubectl -n <namespace> exec <mokka-pod> -c node-agent -- nvidia-smi -L
```

这会检查该 Pod 所在节点上的驱动视图。上报的设备来自当前生效的 [GPU 配置](../configuration.md)；预期型号和数量请使用演示的说明。如果命令失败，请查阅[排障](../troubleshooting.md)。

这些检查能证明的范围，参见 Mokka 的[模拟边界](../index.md#simulation-depth-by-area)。GPU 资源分配请使用 [device plugin 指南](device-plugin.md)，修改模拟设备状态请使用[运行时控制](../nvml-mock-ctl.md)。

## 保存与恢复实验

完成后，选择 **Stop Simulation and Store Checkpoint** 保存节点状态并释放计算资源。再次启动仿真时，默认从最新检查点恢复。这样你以后可以继续设置练习。检查点选择、自动休眠和生命周期细节参见[仿真管理](https://docs.nvidia.com/dsx-air/simulation-management)。
