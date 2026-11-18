# 常见问题

## 哪些内容尚未模拟？

读取 Mokka 未部署的接口面，得到的应答与没有 GPU 的机器相同，因此依赖这些接口的消费者会拒绝启动，而不是以有趣的方式失败：

| 接口面 | 读取它的消费者 |
|---|---|
| PCIe 配置空间——`/sys/bus/pci/devices/<BDF>/config`、`/proc/bus/pci/devices` | `lspci -vv`、底层探测工具 |
| fabricmanager 遥测 socket、`nvswitch-audit` | DCGM-Exporter 的 fabric-manager 采集器、operator 诊断 |
| NVLink 链路状态变化事件、RDMA netlink 事件 | 对链路降级做出反应的工具 |

## 能否针对 Mokka 运行 CUDA 工作负载？

不能。目前不支持在 CPU 上模拟 CUDA。

## 需要真实 GPU 或 GPU 驱动吗？

不需要。Mokka 运行在普通 CPU 节点上。没有内核模块，也没有数据通路——这正是它的意义所在。

## 这能用于生产环境吗？

不能，它也不是为此设计的。Mokka 是面向 CI 和测试集群的测试替身。它上报并不存在的 GPU，这对测试调度和故障处理很有用，但在期望真实工作落地的环境中则是有害的。

## 可以模拟哪些 GPU 型号？

chart 自带八种 profile：`a100`、`b200`、`gb200`、`gb300`、`h100`、`l40s`、`t4` 和 `vrnvl72`。`gb300` 是默认型号。用 `--set gpu.profile=<name>` 切换。

各 profile 定义了什么，参见[配置参考](configuration.md)。

## 一个集群能否在不同节点上使用不同 GPU 型号？

可以。每个 release 面向一组节点，因此集群可以同时呈现 A100 和 T4 worker。CI 中就是按异构机群来演练的。参见[不同节点使用不同 GPU 型号](guides/device-plugin.md#different-gpu-models-on-different-nodes)。

## `nvidia-smi` 真的能用吗？

能用——是未经修改的真实二进制文件。它加载 Mokka 的 `libnvidia-ml.so` 而不是厂商库，并报告 profile 描述的任何内容。

## 为什么 `nvidia-smi` 总是报告 0 MiB 已用？

因为 profile 就是这么写的，而且没有东西会改变它。Mokka 不运行 kernel，因此没有工作负载消耗设备显存；无论调度了什么，所有消费者读到的都是 profile 里的 `memory.used_bytes`——在每个自带 profile 中都是 `0`。

设置 `allocationWatcher.enabled=true` 可让已用和空闲显存跟随 Kubernetes GPU 分配：一个 sidecar 轮询 kubelet pod-resources API，随申请的出现和消失更新数字。这些值仍然是合成的——它们报告的是申请*存在*，而不是工作负载碰过什么——足以演练读取显存压力的消费者。参见[感知分配的显存](configuration.md#allocation-aware--opt-in)。

## `lsmod` 会显示哪些内核模块？

`nvidia`、`nvidia_uvm`、`nvidia_modeset`、`gdrdrv` 和 `nvidia_fs`，启用 InfiniBand 时还有 `nvidia_peermem` 和 `mlx5_core`。`/sys/module/nvidia/refcnt` 也存在，节点自身的模块会与模拟模块一并可见。

这覆盖了 GPU Operator validator 会 grep 的模块。状态协调会刷新镜像，因此该镜像可能滞后于节点上的模块加载或卸载。接口面如何进入容器，参见 [Helm chart 参考](helm-chart.md)。

## 我的 Pod 申请了 GPU，却没有 `nvidia-smi`。为什么？

有两种机制会把模拟驱动放入持有 GPU 的容器，且默认都开启。节点守护进程把 `nvidia` 运行时 handler 设为 containerd 默认值，该 handler 会在任何 `NVIDIA_VISIBLE_DEVICES` 指定了 GPU 的容器中挂载 `nvidia-smi` 和 `libnvidia-ml.so`——device plugin 会为它分配的 GPU 这样设置。[NRI 插件](components/nri-plugin.md)也会把模拟驱动注入持有 GPU 分配的容器。

如果 Pod 里没有 `nvidia-smi`，说明它所在的位置两种机制都没能覆盖：容器运行时配置[失败](troubleshooting.md#pods-get-no-mock-driver-or-the-node-pod-is-not-ready)或未启用（`nodeAgent.containerRuntime.enabled=false`），且 NRI 插件已禁用（`nri.enabled=false`）或节点上的 containerd 禁用了 NRI；参见[排障](troubleshooting.md#the-node-pod-is-notready-and-nvml-mock-nri-crash-loops)。

否则，可以像 [`local/gpu-validator.k8s.yaml`](https://github.com/NVIDIA/k8s-test-infra/blob/main/local/gpu-validator.k8s.yaml) 那样自行挂载驱动根目录：一个位于 `/run/nvidia/driver` 的 hostPath，加上相应的 `LD_LIBRARY_PATH`。

## 可以使用 MIG 吗？

可以，在支持 MIG 的 profile 上——`a100`、`h100`、`b200`、`gb200` 和 `gb300`。`nvidia-smi -mig` 和 `nvidia-smi mig -cgi/-cci/-dgi/-dci` 对 mock 的作用与对驱动一样，节点可以在启动时就已分区，此时 device plugin 在 `migStrategy=single` 下按每个分片通告一个资源。

有一个边界：运行时重新分区只改变 NVML 视图。节点能*分配*的是启动时的布局，因为驱动能力接口面在 Pod 启动时才部署。

参见 [MIG 切分指南](guides/mig/README.md)。

## 可以模拟故障 GPU 吗？

可以。故障是一种配置状态，而不是特殊代码路径，因此每个接口面都会一致地报告：`nvidia-smi`、DCGM 和 device plugin 保持一致，就像在真实硬件上一样。ECC 错误、GPU 丢失和从总线脱落的 GPU 都可以注入。

参见[故障注入指南](guides/failure-injection/README.md)。
