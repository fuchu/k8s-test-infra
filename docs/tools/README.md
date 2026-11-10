# 命令行工具

除了模拟库之外，Mokka 还在 `cmd/` 下提供了一组小型工具二进制。下面四个里，有三个是 chart 替你跑的工作负载；只有 `nvml-mock-ctl` 是设计来手动调用的。

| 组件 | 用途 |
|------|---------|
| [`node-agent`](node-agent.md) | 把模拟的 NVML 配置档编译成宿主机上的状态并持续对齐。即 nvml-mock DaemonSet 运行的进程。 |
| [`nri-plugin`](nri-plugin.md) | containerd 的 NRI 插件，在容器创建时把模拟驱动覆盖层、LD_PRELOAD 垫片和设备节点注入容器。 |
| [`control-plane`](control-plane.md) | 运行 [Mokka 控制面](../control-plane.md)，并响应存活探针与控制器就绪探针。 |
| [`nvml-mock-ctl`](../nvml-mock-ctl.md) | 无需重启即可修改运行中节点的模拟 GPU 状态。 |

另外三个在[贡献指南](../contributing/index.md)下记录：[`check-fabric`](check-fabric.md) 打印每块可见 GPU 的 NVLink 交换网身份（cluster UUID、clique ID、状态），[`generate-bridge`](generate-bridge.md) 生成模拟 NVML 的桥接桩代码，[`mokka-capture`](mokka-capture.md) 采集真实 GPU 节点并为仓库脱敏。

## 构建

`make build` 会把每个 `cmd/*/main.go` 编译进 `dist/`，然后对 `shims/` 下的 Go 垫片做同样的事。

nvml-mock 镜像在 `/usr/local/bin` 下安装其中四个：`node-agent`、`nri-plugin`、`nvml-mock-ctl` 和 `check-fabric`。`control-plane` 有自己单独的镜像，由 `deployments/control-plane/Dockerfile` 构建；`generate-bridge` 与 `mokka-capture` 是开发者工具，两者都不随镜像发布。
