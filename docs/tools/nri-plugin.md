# nri-plugin

Mokka 节点 DaemonSet 中 NRI sidecar 背后的二进制文件。它通过 [NRI](https://github.com/containerd/nri) socket 向 containerd 注册，只订阅 `CreateContainer`，并在容器创建时编辑容器，使以常规方式获得 GPU 的负载，或通过 `nvml-mock.nvidia.com/devices` 注解显式加入的负载，能够看到模拟 GPU。

[NRI 插件](../components/nri-plugin.md)介绍它决定什么以及为什么——哪些容器会被注入、它如何识别 device plugin 或 DRA 分配、何时放过容器，以及为什么失败时采用 fail-open。本页只讲命令行。

chart 默认运行它；`nri.enabled=false` 会移除该 sidecar。sidecar 以 root 运行，`allowPrivilegeEscalation: false`，且没有 service account token。它读写挂载 NRI socket 目录，并只读共享节点 Pod 的覆盖层和 CDI 规格目录。启动、关闭以及 Kubernetes 1.28 兼容性行为，参见 [NRI Pod 生命周期](../helm-chart.md#nri-pod-lifecycle)。

## 探针

fail-open 的插件会无声地失败，因此 `--health-addr` 提供两个探针。空值会禁用它们。

插件尚未向运行时注册时，`GET /readyz` 返回 `503`，这会把注入悄然停止的每一段时间都变成 NotReady 的 Pod。containerd 调用 `Configure` 时设置注册状态，运行时关闭连接时清除。

只有 `CreateContainer` handler 的持续时间超过注册时运行时所报请求超时的**两倍**时，`GET /healthz` 才返回 `503`。这个乘数在要求 kubelet 重启插件之前多留了一整个超时周期，因此单个缓慢但会完成的请求不会引发重启。运行时未报超时时，采用 NRI 客户端库的 2 秒回退值，阈值即为 4 秒。

两者都把失败原因写进响应正文，因此在 `kubectl describe pod` 中可读。

## Flags

每个 flag 也读取一个环境变量；两者都设置时以 flag 为准。

| Flag | 环境变量 | 默认值 | 说明 |
|------|----------------------|---------|-------------|
| `--feature-gates` | `MOKKA_FEATURE_GATES` | 空 | 逗号分隔的 `Name=true\|false` 对。参见[特性门控](../feature-gates.md) |
| `--log-level` | `MOKKA_LOG_LEVEL` | `info` | `debug`、`info`、`warn` 或 `error`。`warning` 是 `warn` 的别名；空值回退到 `info` |
| `--log-format` | `MOKKA_LOG_FORMAT` | `json` | `json` 或 `plain`；空值回退到 `json` |
| `--health-addr` | `MOKKA_NRI_HEALTH_ADDR` | `:8080` | `/healthz` 和 `/readyz` 的监听地址；空则禁用 |
| `--agent-staged-url` | `MOKKA_NRI_AGENT_STAGED_URL` | 空 | 节点 agent 的 `/stagedz` URL。设置后，仅在 agent 报告其目录树已部署时才调整容器，否则保持容器不变并把每次跳过记为警告。与 `--staging-lock-path` 一起设置 |
| `--staging-lock-path` | `MOKKA_NRI_STAGING_LOCK_PATH` | 空 | 节点 agent 的部署锁，在决定每次调整时以共享方式持有。必须位于注入的覆盖层之外。与 `--agent-staged-url` 一起设置 |
| `--socket-path` | `MOKKA_NRI_SOCKET_PATH` | `/var/run/nri/nri.sock` | NRI socket 路径 |
| `--plugin-name` | `MOKKA_NRI_PLUGIN_NAME` | `mokka-nri-plugin` | 插件向运行时注册所用的名称 |
| `--plugin-index` | `MOKKA_NRI_PLUGIN_INDEX` | `10` | 与其他已注册插件的相对顺序；索引较大者在较小者之后调整容器 |
| `--excluded-namespaces` | `MOKKA_NRI_EXCLUDED_NAMESPACES` | `kube-system` | 要跳过的命名空间，逗号分隔；空表示不跳过任何命名空间 |
| `--opt-out-annotation` | `MOKKA_NRI_OPT_OUT_ANNOTATION` | `nvml-mock.nvidia.com/inject` | Pod 注解键；值为 `false` 时禁用注入 |
| `--overlay-host-path` | `MOKKA_NRI_OVERLAY_HOST_PATH` | `/var/lib/nvml-mock` | 覆盖层的宿主机路径 |
| `--overlay-mount-path` | `MOKKA_NRI_OVERLAY_MOUNT_PATH` | `/opt/nvml-mock` | 覆盖层在容器内的路径 |
| `--ld-preload-shims` | `MOKKA_NRI_LD_PRELOAD_SHIMS` | `driver/usr/local/lib` 下的 `libibmockumad`、`libibmockverbs`、`libibmocksys` 和 `libmockfs` | 逗号分隔的 shim 路径，相对覆盖层挂载路径或绝对路径。预加载顺序即列表顺序，因此多个 shim 定义同一符号时以第一个为准 |
| `--node-name` | `NODE_NAME` | 空 | 覆盖层中已部署拓扑文档时，启用 ComputeDomain 拓扑注入 |
| `--topology-host-path` | `MOKKA_NRI_TOPOLOGY_HOST_PATH` | `<overlay-host-path>/topology/topology.yaml` | 检查已部署拓扑文档的宿主机路径 |
| `--topology-mount-path` | `MOKKA_NRI_TOPOLOGY_MOUNT_PATH` | `<overlay-mount-path>/topology/topology.yaml` | 作为 `MOCK_TOPOLOGY_CONFIG` 注入的容器路径 |
| `--device-annotation` | `MOKKA_NRI_DEVICE_ANNOTATION` | `nvml-mock.nvidia.com/devices` | Pod 注解键；值为 `true` 时添加 `/dev/nvidia*` 节点 |
| `--device-host-path` | `MOKKA_NRI_DEVICE_HOST_PATH` | `<overlay-host-path>/driver/dev` | 包含模拟 `/dev/nvidia*` 节点的宿主机路径 |
| `--device-injection-mode` | `MOKKA_NRI_DEVICE_INJECTION_MODE` | `raw` | `raw`（设备节点）或 `cdi`（CDI 引用）。其他值在启动时被拒绝 |
| `--cdi-device-name` | `MOKKA_NRI_CDI_DEVICE_NAME` | `nvml-mock.nvidia.com/gpu=all` | `cdi` 模式下注入的完整限定 CDI 设备名 |
| `--cdi-spec-host-path` | `MOKKA_NRI_CDI_SPEC_HOST_PATH` | `/var/run/cdi/nvml-mock-nri.yaml` | 发出 CDI 引用前检查的规格文件；规格缺失时回退到原始注入 |
| `--infiniband-annotation` | `MOKKA_NRI_INFINIBAND_ANNOTATION` | `nvml-mock.nvidia.com/infiniband` | Pod 注解键；值为 `true` 时启用模拟 InfiniBand 工具和 shim，与 GPU 访问无关 |
| `--imex-channel-annotation` | `MOKKA_NRI_IMEX_CHANNEL_ANNOTATION` | `nvml-mock.nvidia.com/imex-channels` | Pod 注解键；值为 `true` 时添加 `/dev/nvidia-caps-imex-channels/*` 节点 |
| `--imex-channel-host-path` | `MOKKA_NRI_IMEX_CHANNEL_HOST_PATH` | `<overlay-host-path>/driver/dev/nvidia-caps-imex-channels` | 包含节点 agent 在多节点 NVLink profile（`nvlink.multi_node`）上部署的模拟 IMEX 通道节点的宿主机路径 |

三个 `<overlay-...>` 派生项按覆盖层 flag 的最终值解析，而不是按打包默认值。插件返回的任何错误都会以 `nri-plugin: <error>` 的形式打印到 stderr 并退出 `1`。

!!! note "CDI vendor 刻意不是 `nvidia.com`"

    `nvml-mock.nvidia.com` 让我们的设备引用避开 device plugin 和容器工具链所拥有的命名空间，这使得"恰好有一个组件为该容器发出 CDI 设备引用"成为可以在 OCI 规格中观察到的事实，而不是只能假设。

!!! note "注册名称不是二进制名称"

    `--plugin-name` 是 containerd 认识该插件所用的名称。chart 从 `nri.pluginName` 设置它，值为 `nvml-mock`；编译内置的 `mokka-nri-plugin` 只适用于独立运行。

## 用法

chart 渲染这条命令行，在排除的命名空间前加上 release 命名空间，并通过 downward API 提供 `NODE_NAME`：

```text
/usr/local/bin/nri-plugin \
  --socket-path=/var/run/nri/nri.sock \
  --plugin-name=nvml-mock \
  --plugin-index=10 \
  --overlay-host-path=/var/lib/nvml-mock \
  --overlay-mount-path=/opt/nvml-mock \
  --device-host-path=/var/lib/nvml-mock/driver/dev \
  --opt-out-annotation=nvml-mock.nvidia.com/inject \
  --device-annotation=nvml-mock.nvidia.com/devices \
  --device-injection-mode=raw \
  --cdi-spec-host-path=/var/run/cdi/nvml-mock-nri.yaml \
  --imex-channel-annotation=nvml-mock.nvidia.com/imex-channels \
  --imex-channel-host-path=/var/lib/nvml-mock/driver/dev/nvidia-caps-imex-channels \
  --infiniband-annotation=nvml-mock.nvidia.com/infiniband \
  --excluded-namespaces=<release-namespace>,kube-system \
  --node-name=$(NODE_NAME) \
  --health-addr=:8080 \
  --log-level=info \
  --log-format=json
```

`--cdi-device-name`、两个拓扑 flag 和 `--ld-preload-shims` 没有模板化，因此部署后的插件以编译内置默认值运行。

工作负载作者写入的显式加入项，彼此独立：

```yaml
metadata:
  annotations:
    nvml-mock.nvidia.com/devices: "true"
    nvml-mock.nvidia.com/infiniband: "true"
    nvml-mock.nvidia.com/imex-channels: "true"
```

从 debug Pod 手动探测：

```bash
curl -sS http://<pod-ip>:8080/readyz    # 未注册时 503 加原因
curl -sS http://<pod-ip>:8080/healthz   # 仅当 handler 卡住时 503
```

该二进制安装在 nvml-mock 镜像的 `/usr/local/bin/nri-plugin`。

## 另见

- [命令行工具](README.md)
- [NRI 插件](../components/nri-plugin.md)——它注入什么，跳过什么
- [配置 NRI 注入](../guides/nri-injection.md)——检查插件是否注入
- [安装](../helm-chart.md)——所有 `nri` chart 取值
