---
# 页面以 hero <div> 开头，MkDocs 无法从下方的 H1 推断标题，会回退到文件名。此处显式指定。
title: 概览
---

<div class="mokka-hero" markdown>
![Mokka](img/logo.png)

<p class="mokka-tagline">在 CPU 节点上模拟你的 GPU 基础设施。</p>
</div>

# Mokka

Mokka 模拟的是 NVIDIA 设备周围的软件契约，而不是设备本身。判断标准很简单：当你的系统**读取**硬件状态并据此做出反应时，用 Mokka；当它**执行**计算、搬运数据或测量性能时，用真实硬件。

因此它非常适合：

- 通过 device plugin 或 DRA 进行的 Kubernetes 发现与分配——调度、ResourceClaim、CDI 可见性、拓扑属性。
- 消费 NVML、`nvidia-smi`、DCGM 或 DCGM Exporter 的软件。
- 监控仪表盘、解析器、告警，以及执行隔离（cordon）、驱逐（drain）、重新调度和恢复的修复逻辑。
- 可重复执行的故障注入——设备丢失、Xid 错误、ECC 错误、温度、功耗、利用率、时钟频率，以及 GPU 侧 NVLink 错误。
- 解析声明式 PCI、NUMA、NVLink、fabric UUID 或 clique 拓扑的代码。
- IMEX 对等节点就绪状态与存活性。对等协议是真实运行的，走 Pod 网络；不涉及任何 GPU 或 NVLink 流量。

## 快速体验

```bash
kind create cluster --name mokka

helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
    --namespace mokka --create-namespace
```

国内环境可加 `--set global.imageRegistry=<私有仓库>/ghcr.io`，把 Mokka 的镜像统一改从私有仓库拉取；镜像需预先同步到对应路径。完整做法见[国内镜像加速](guides/README.md#domestic-mirror)。

现在每个节点都会上报模拟 GPU。[快速开始](quickstart.md)会带你继续往下操作。

## 各领域的模拟深度

Kubernetes 控制流程和客户端程序是真实的；硬件身份、拓扑、计数器和故障是合成的。

| 领域 | Mokka 模拟的内容 | 适用场景 | 不能证明的内容 |
|---|---|---|---|
| **GPU 与 NVML** | 一个可配置的、NVML 可见的 GPU。profile 和逐设备覆盖项定义身份、型号、显存大小、PCI 与 NUMA 身份、时钟、温度、功耗、利用率、ECC 状态、进程记录和故障响应。[`nvml-mock-ctl`](nvml-mock-ctl.md) 可在运行时修改其中大多数。MIG 切分通过 `nvidia-smi mig` 工作——参见 [MIG 切分](guides/mig/README.md)。 | 资产清点、监控、解析器、DCGM 集成、告警处理、故障恢复、MIG 分区 | 没有内核驱动、固件、CUDA kernel、context、stream 或数据通路。未实现的 NVML 函数返回 `NVML_ERROR_NOT_SUPPORTED`，因此新消费者应确认自己调用了哪些函数。运行时重新分区只改变 NVML 视图；节点可分配的是启动时采用的布局。 |
| **Kubernetes 分配** | 基于合成设备的真实分配流程。device plugin 和 DRA driver 通告并分配 GPU；Kubernetes 调度 Pod，并创建 ResourceClaim、ResourceSlice、CDI 分配和拓扑属性。 | 调度器、operator、准入控制、claim 生命周期和控制器恢复测试 | claim 只能证明分配成功，不能证明工作负载使用了显存或做了实际工作。进程记录不会随分配而更新。安装或升级驱动、固件和 toolkit 的路径不在模拟范围内。 |
| **指标** | 利用率、温度、功耗、分析活动以及 NVLink 计数器，可配置，也可按经过时间生成。参见 [指标保真度](configuration.md#metric-fidelity) 了解各自对应哪种方式。 | 仪表盘、阈值、告警、扩缩容输入、确定性故障场景 | 没有任何指标与工作负载相关联，因此无法证明吞吐量、效率、散热行为、时序或性能。 |
| **PCI 与 NUMA** | 一棵合成发现树：PCI BDF、root complex 路径、NUMA node 值、选中的设备节点，以及提供给容器的 sysfs 内容。 | 发现、放置、拓扑解析，以及由声明的局部性驱动的行为 | 没有 PCIe 事务、DMA、IOMMU 或 ACS 行为，没有带宽、时延或硬件错误，也没有真实的 CPU / 内存局部性。未通过 CDI 或 NRI 接收该树的消费者，读到的将是宿主机自身的 sysfs。 |
| **NVLink** | 链路代际、数量、状态、远端 GPU 或 NVSwitch 端点、能力以及线速。`nvidia-smi topo -m` 渲染声明的本地矩阵，计数器以合成方式累加，GPU 侧链路错误会进入 DCGM 健康状态。 | 拓扑解析、策略决策、本地链路健康、故障处理、clique 逻辑 | 没有 CUDA 对等访问、GPU 流量、跨节点 NVLink 路径、集合通信、NCCL、拥塞、带宽或时延。计数器不反映工作负载流量。 |
| **NVSwitch 与 Fabric Manager** | GPU 可见的 fabric 状态：声明拓扑中的 NVSwitch 端点，以及 fabric UUID、clique、注册和[健康状态](configuration.md#fabric-health)。基板上的交换机也会作为[节点 PCI 总线上的桥接设备](helm-chart.md#nvswitches-on-the-pci-bus)出现。Fabric Manager 的替代实现会发布节点本地就绪标记，并可延迟注册。 | 等待 fabric 注册、读取 fabric 身份，或响应 GPU 可见 fabric 健康状况的软件 | 没有 NVSwitch ASIC、转发、NSCQ、SXID 模型、交换机控制 API 或 CLI、固件、分区或路由。 |
| **InfiniBand HCA** | 合成 HCA 的 sysfs、UMAD 和 verbs 接口，型号、固件、GUID、LID、GID、链路层、状态和速率均可配置。`ibstat`、`ibstatus`、`ibv_devices` 以及 `ibv_devinfo` 的大部分功能都可使用。 | 发现、解析器、资产清点、链路状态处理、部分管理面工作流 | 没有内核 provider、队列对、完成队列、内存注册、verbs 执行、RDMA 流量或性能。 |
| **InfiniBand fabric** | 跨 Pod 注册的对等 HCA、SA 与 SMP 应答、确定性的合成子网管理器身份、`iblinkinfo`、`ibnetdiscover`，以及通过 Pod 网络进行的跨节点 `ibping`。 | 工具兼容性、解析器行为、对等发现、简单的控制面故障处理 | 没有 IB 交换机、子网管理器选举、交换机 NOS、固件、路由、自适应路由、拥塞、QoS、PKey、线缆行为或 IB 数据通路。fabric 视图由全互联 HCA 模型合成。 |

## 应使用真实硬件的场景

- CUDA 应用的正确性、兼容性、kernel、内存访问和 CUDA 对等访问。
- NCCL 集合通信、GPUDirect、RDMA，以及 NVLink、NVSwitch 或 InfiniBand 数据通路。
- 吞吐量、时延、超额订阅、拥塞、扩展效率和散热行为。
- GPU、HCA、NVSwitch 或 IB 交换机驱动与固件的安装、升级、复位和恢复。
- 通过管理面、CLI、遥测、路由或固件接口运维交换机。
- 需要物理保真度地复现硬件故障、竞态条件和时序问题。
- 在 MIG 分片**上**运行工作负载，以及机密计算。创建、销毁和枚举分区是模拟的——参见 [MIG 切分](guides/mig/README.md)——但没有 CUDA，因此分片可以接纳 Pod，却不会为它运行 kernel。

## 已验证的消费者

| 消费者 | 可用能力 |
|---|---|
| Node Feature Discovery | 根据 Mokka 写入的 feature 文件生成 PCI vendor 标签 |
| GPU Feature Discovery | 根据 NVML 生成节点标签 |
| NVIDIA Device Plugin | 可分配的 `nvidia.com/gpu` 与 profile 一致，工作负载可据此调度 |
| NVIDIA DRA Driver | ResourceSlice 上报正确的 GPU，使用 `ResourceClaimTemplate` 的 Pod 可进入 `Running` 状态 |
| NVIDIA GPU Operator | 完整的 operand 技术栈可安装，其 validator 可启动 |
| DCGM / dcgm-exporter | 遥测、随时间变化的功耗，以及注入的 Xid 错误 |
| Run:ai fake-gpu-operator | 按其发现逻辑期望的格式发布 profile ConfigMap |
| 基于 Slinky 的 Slurm | `slurmd` 自动将 GPU 识别为 GRES，`srun --gres=gpu:N` 作业可据此调度 |
| Topograph | NVLink clique 转换为每节点的 accelerator-domain 标签，供拓扑感知调度器使用 |

## 接下来去哪里

<div class="grid cards" markdown>

-   **让它跑起来**

    安装到 KIND 集群，五分钟内看到模拟 GPU。

    [快速开始](quickstart.md)

-   **理解工作原理**

    各组成部件、它们如何连接，以及系统行为方式。

    [架构](architecture.md)

-   **完成具体任务**

    面向任务的演练：device plugin、DRA、GPU Operator、故障注入、节点级注入。

    [指南](guides/README.md)

-   **修改 Mokka**

    使用 Tilt 进行本地开发、测试套件，以及如何提交变更。

    [贡献指南](contributing/index.md)

</div>

[常见问题](faq.md)回答接下来通常会遇到的问题：包含哪些 GPU 型号、哪些接口完全没有模拟，以及为什么 `nvidia-smi` 报告的是这些数字。
