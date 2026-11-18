# check-fabric

打印每个可见 GPU 的 NVLink fabric 身份，并把结果编码进退出码。它是证明 ComputeDomain 模拟有效的消费者：只要动态加载器能够解析到模拟 NVML 库，它就会报告拓扑覆盖分配给本节点的 cluster UUID、clique ID 和注册状态。

每块 GPU 的输出是一个四行块，标签宽度固定。调用方会 grep 这些字面字符串，因此对齐也是契约的一部分：

```text
Discovered 8 GPU(s)
GPU 0 (GPU-...)
  clusterUuid : 00000000-0000-0000-0000-0000000000ab
  cliqueId    : 1
  state       : completed (3)
```

`state` 映射原始 NVML 值：`not_supported` (0)、`not_started` (1)、`in_progress` (2)、`completed` (3)，其他值为 `unknown`。

没有 fabric 的 GPU 会打印 `GPU N (uuid): fabric NOT SUPPORTED`，不会中止运行——该计数会在最后折进退出码。

## 退出码

| 码 | 含义 |
|------|---------|
| `0` | 每个可见 GPU 都报告了 fabric 信息 |
| `1` | 某次 NVML 调用失败：init、设备数量、句柄或 fabric 查询 |
| `2` | 至少一块 GPU 返回 `ERROR_NOT_SUPPORTED` |

## 用法

它不接受任何 flag 或参数。

```bash
kubectl -n mokka exec <nvml-mock-pod> -- check-fabric
```

针对输出做断言是各场景使用它的方式：

```bash
out=$(kubectl exec "${pod}" -- check-fabric 2>&1 || true)
grep -q  "cliqueId    : ${expected_clique}" <<<"$out"
grep -qi "clusterUuid : ${expected_uuid}"   <<<"$out"
grep -q  'state       : completed'          <<<"$out"
```

以 IMEX 通道设备为门槛，使其只在存在 fabric 的地方运行：

```bash
sh -c 'test -c /dev/nvidia-caps-imex-channels/channel0 && check-fabric | head -6'
```

!!! note "它如何找到模拟库"

    程序本身不定位库——go-nvml 会 `dlopen` `libnvidia-ml.so.1`，该库通过标准的 `/usr/local/lib` 安装路径，或通过 NRI 插件注入的 `LD_LIBRARY_PATH` 解析。两者都不适用时，运行会停在 `nvmlInit`，退出码为 `1`。

## 它来自哪里

它在构建时被放进 nvml-mock 镜像的 `/usr/local/bin/check-fabric`。随后[节点守护进程](../components/node-daemon.md)把它复制到驱动根目录下的覆盖层 `$DRIVER_ROOT/usr/bin/check-fabric`，这才使得从未挂载镜像的 NRI 注入工作负载 Pod 也能运行它。没有该二进制的镜像会跳过复制，而不是失败。

有三个调用方会运行它，且从不作为长驻进程：Go e2e 套件在每个 worker 的 NRI 注入 Pod 内 exec 它；ComputeDomain 指南的 `run.sh` 在每个 worker 的节点 Pod 中运行它，遇到非零状态或 clique 不匹配即失败；以及开发者通过手动触发的 `check-fabric` Tilt 资源运行它。

## 另见

- [命令行工具](README.md)
- [ComputeDomain 指南](../guides/compute-domain/README.md)
- [NRI 插件](../components/nri-plugin.md)——把库注入 Pod 的组件
