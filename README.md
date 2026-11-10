<div align="center">
    <img src="./docs/img/logo.png" width="180px" alt="Mokka" />
    <h1>Mokka</h1>
    <p>在纯 CPU 节点上模拟你的 GPU 基础设施。</p>
    <a href="https://github.com/NVIDIA/k8s-test-infra/actions/workflows/ci.yaml">
        <img src="https://github.com/NVIDIA/k8s-test-infra/actions/workflows/ci.yaml/badge.svg" alt="CI pipelines" />
    </a>
    <a href="https://nvidia.github.io/k8s-test-infra/">
        <img src="https://github.com/NVIDIA/k8s-test-infra/actions/workflows/deploy-pages.yaml/badge.svg" alt="Documentation" />
    </a>
    <a href="https://scorecard.dev/viewer/?uri=github.com/NVIDIA/k8s-test-infra">
        <img src="https://api.scorecard.dev/projects/github.com/NVIDIA/k8s-test-infra/badge" alt="OpenSSF Scorecard" />
    </a>
    <a href="https://www.bestpractices.dev/projects/14445">
        <img src="https://www.bestpractices.dev/projects/14445/badge" alt="OpenSSF Best Practices" />
    </a>
    <a href="LICENSE">
        <img src="https://img.shields.io/badge/License-Apache_2.0-blue.svg" alt="License" />
    </a>
</div>

---

Mokka 模拟的是 NVIDIA 设备周围的**软件契约**，而不是设备本身。判断标准很简单：当你的系统**读取**硬件状态并据此做出反应时，用 Mokka；当系统要**执行**计算、搬运数据或衡量性能时，用真实硬件。

因此它适合以下场景：

- Kubernetes 的发现与分配（通过 device plugin 或 DRA）——调度、ResourceClaims、CDI 可见性、拓扑属性。
- 消费 NVML、`nvidia-smi`、DCGM 或 dcgm-exporter 的软件。
- 监控看板、解析器、告警，以及执行 cordon、drain、重新调度和恢复的修复逻辑。
- 可复现的故障注入——设备丢失、Xid 错误、ECC 错误、温度、功耗、利用率、时钟，以及 GPU 侧的 NVLink 错误。
- 解读所声明的 PCI、NUMA、NVLink、fabric UUID 或 clique 拓扑的代码。
- 经 Pod 网络的 IMEX 节点间就绪与存活探测。

以下场景请用真实硬件：CUDA 执行、NCCL、GPUDirect 与 RDMA 数据通路、任何吞吐或温度测量、驱动与固件生命周期、交换机管理平面、物理上忠实的故障时序、MIG 分区生命周期，以及机密计算。

## 快速开始

```bash
kind create cluster --name mokka

helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
    --namespace mokka --create-namespace
```

现在每个节点都会报告 4 块模拟的 GB300 GPU。用 `--set gpu.profile=<name>` 换成 `a100`、`b200`、`gb200`、`h100`、`l40s`、`t4` 或 `vrnvl72`。

[按领域的模拟深度](https://nvidia.github.io/k8s-test-infra/#simulation-depth-by-area) 逐层拆解了每个模拟面——GPU 与 NVML、Kubernetes 分配、指标、PCI 与 NUMA、NVLink、NVSwitch 与 Fabric Manager，以及 InfiniBand——分别说明了它们能证明什么、不能证明什么。

## 已验证的下游消费者

| 消费者 | 可用能力 |
|---|---|
| Node Feature Discovery | 由 Mokka 写入的特征文件派生出 PCI 厂商标签 |
| GPU Feature Discovery | 由 NVML 派生的节点标签 |
| NVIDIA Device Plugin | 可分配的 `nvidia.com/gpu` 与配置档一致，且工作负载能据此调度 |
| NVIDIA DRA Driver | ResourceSlices 上报正确的 GPU，且 `ResourceClaimTemplate` Pod 达到 `Running` |
| NVIDIA GPU Operator | 完整的 operand 技术栈能安装，其 validator 能够启动 |
| DCGM / dcgm-exporter | 遥测、随时间变化的功耗，以及注入的 Xid 错误 |
| Run:ai fake-gpu-operator | 以 FGO 期望的形态发布 profile ConfigMap |

## 文档

访问 **[我们的文档站点](https://nvidia.github.io/k8s-test-infra/)** 了解 Mokka 工作原理的全部细节。

|                                                                         |                                                                          |
|-------------------------------------------------------------------------|--------------------------------------------------------------------------|
| [快速开始](https://nvidia.github.io/k8s-test-infra/quickstart/)      | 安装并看到模拟的 GPU                                           |
| [架构](https://nvidia.github.io/k8s-test-infra/architecture/)   | 各个组成部分以及系统的行为方式                              |
| [指南](https://nvidia.github.io/k8s-test-infra/guides/)                 | Device plugin、DRA、GPU Operator、故障注入、节点级注入 |
| [配置](https://nvidia.github.io/k8s-test-infra/configuration/) | 每个配置档开关                                                       |
| [常见问题](https://nvidia.github.io/k8s-test-infra/faq/)                     | Mokka 模拟什么、不模拟什么                               |

## 参与贡献

本地开发（Tilt、测试套件、如何提交改动）见[贡献指南](https://nvidia.github.io/k8s-test-infra/contributing/)。较大的改动应始于一份 [Mokka 增强提案](enhancements/)。

报告安全漏洞请走私密渠道——见 [SECURITY.md](SECURITY.md)，不要用 issue 跟踪器。

## 致谢

- Logo 由 [Roman Hlushko](https://github.com/roma-glushko) 在 OpenAI ChatGPT 协助下设计。

## 许可证

Apache License 2.0 —— 见 [LICENSE](LICENSE)。
