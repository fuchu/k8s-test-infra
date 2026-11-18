# node-agent

nvml-mock DaemonSet 运行的进程。单个 `start` 命令把 GPU profile 编译成宿主机上的状态，在该 profile 变化时保持节点收敛，并提供 kubelet 读取的两个探针。

[节点守护进程](../components/node-daemon.md)介绍它在节点上做什么——它驱动的模拟器、协调（reconcile）的阶段波次和拆除过程。本页只讲命令行。

## Flags

所有 flag 都属于 `start` 子命令。每个 flag 也读取一个环境变量；两者都设置时以 flag 为准。

| Flag | 环境变量 | 默认值 | 说明 |
|------|----------------------|---------|-------------|
| `--feature-gates` | `MOKKA_FEATURE_GATES` | 空 | 逗号分隔的 `Name=true\|false` 对。参见[特性门控](../feature-gates.md) |
| `--config` | `MOKKA_AGENT_CONFIG` | 无 | 模拟 NVML YAML profile 的路径。必需——未设置时启动失败并报 `--config is required` |
| `--topology` | `MOKKA_AGENT_TOPOLOGY` | 空 | 集群 ComputeDomain 拓扑文档的路径 |
| `--host-root` | `MOKKA_AGENT_HOST_ROOT` | `/host` | 宿主机文件系统挂载到本进程命名空间的位置 |
| `--health-addr` | `MOKKA_AGENT_HEALTH_ADDR` | `:9090` | `/healthz`、`/readyz` 和 `/stagedz` 的监听地址；空则三者全部禁用。最新一轮 Stage 波次写入驱动树后 `/stagedz` 通过；NRI 插件以此为门槛 |
| `--shutdown-timeout` | `MOKKA_AGENT_SHUTDOWN_TIMEOUT` | `30s` | SIGINT/SIGTERM 时拆除过程的时间预算 |
| `--resync-interval` | `MOKKA_AGENT_RESYNC_INTERVAL` | `1m` | 无论文件系统事件如何，按此频率重新读取 `--config` 和 `--topology`；`0` 表示只依赖事件 |
| `--staging-lock-path` | `MOKKA_AGENT_STAGING_LOCK_PATH` | 空 | 部署或拆除时独占持有的锁文件，NRI 插件在决定每次调整时也共享该锁。必须位于注入的覆盖层之外。关闭时 agent 最多等待 `--shutdown-timeout` 的一半。空表示禁用该门槛；chart 在 `nri.enabled` 时设置它 |
| `--imex-download-timeout` | `MOKKA_IMEX_DOWNLOAD_TIMEOUT` | `1m` | `IMEXDownload` 特性门控开启时，单次下载固定版本 IMEX 归档的时间限制。chart 从 `imex.binaries.download.timeout` 设置它 |
| `--imex-provided-dir` | `MOKKA_IMEX_PROVIDED_DIR` | `/opt/mokka/imex` | 存放固定版本 IMEX 归档的目录，文件名与 NVIDIA 发布时一致。在多节点 NVLink profile 上，agent 会用固定的 SHA-256 校验它，并把 IMEX 部署进驱动树。若这里和节点缓存中都没有匹配的归档，则在 `IMEXDownload` 门控开启时下载一份，否则不部署任何 IMEX 并保持就绪。chart 在设置 `imex.binaries.image` 时设置该目录，并从该镜像把归档复制过去 |
| `--log-level` | `MOKKA_LOG_LEVEL` | `info` | `debug`、`info`、`warn` 或 `error`。`warning` 是 `warn` 的别名；空值回退到 `info` |
| `--log-format` | `MOKKA_LOG_FORMAT` | `json` | `json` 或 `plain`；空值回退到 `json` |
| `--ib-mode` | `MOCK_IB` | `off` | InfiniBand 层级：`off`、`sysfs`（仅渲染）或 `full`（增加 mock-ib 守护进程）。空值读作 `off` |
| `--ib-fabric` | `MOCK_IB_PING_FABRIC` | `false` | 跨 Pod fabric 中继；多节点 `ibping` 和 `iblinkinfo` 必需 |
| `--ib-fabric-port` | `MOCK_IB_PING_PORT` | `18515` | 该中继的 TCP 端口 |
| `--fabricmanager-init-delay` | `MOCK_FABRICMANAGER_INIT_DELAY` | `0` | 延迟 fabric 就绪这么长时间，模拟 NVSwitch 注册延迟 |
| `--kernel-log` | `MOCK_NVML_KMSG` | `/dev/kmsg` | 注入的 Xid 发布到哪个内核日志，方式与驱动 printk 相同。空表示不发布任何地方，这是 chart 的默认设置，除非 `nodeAgent.kernelLog.enabled` 授予该设备。不以 `--host-root` 为根 |
| `--container-runtime` | `MOKKA_AGENT_CONTAINER_RUNTIME` | 空 | 要配置的容器运行时：安装 NVIDIA 容器运行时并设为默认 `nvidia` handler，agent 停止时撤销。目前只支持 `containerd`；空表示不改动节点的容器运行时。参见[容器运行时设置](../components/node-daemon.md#container-runtime-setup)。chart 会设置它 |
| `--container-runtime-restart` | `MOKKA_AGENT_CONTAINER_RUNTIME_RESTART` | `systemd` | 容器运行时如何感知配置变更：`systemd`，或 `none` 把重启交给运维。宿主机的 `systemctl` 拒绝与看不见的 systemd 通信，因此 `systemd` 要求 agent 位于宿主机 PID 命名空间；chart 为此设置 `hostPID` |
| `--container-runtime-config` | `MOKKA_AGENT_CONTAINER_RUNTIME_CONFIG` | 空 | 宿主机上运行时的主配置文件；空则取运行时默认 `/etc/containerd/config.toml`。`nvidia-ctk` 会在 agent 容器内的相同路径编辑它，因此宿主机目录必须挂载到那里，chart 就是这么做的 |
| `--container-runtime-config-dir` | `MOKKA_AGENT_CONTAINER_RUNTIME_CONFIG_DIR` | 空 | 宿主机上运行时的配置目录，其配置文件会 import 该目录，Mokka 的配置文件也放在那里；必须位于配置文件目录之下。空则取 `/etc/containerd/conf.d` |
| `--container-runtime-systemd-unit` | `MOKKA_AGENT_CONTAINER_RUNTIME_SYSTEMD_UNIT` | 空 | 变更时重启的 systemd unit；空则取 `containerd` |
| `--container-runtime-nri-socket` | `MOKKA_AGENT_CONTAINER_RUNTIME_NRI_SOCKET` | 空 | 在宿主机该 socket 上开启容器运行时的 NRI，NRI 插件在此连接；空则保持节点原有的 NRI 设置 |
| `--host-fs` | `MOKKA_AGENT_HOST_FS` | `/host` | 宿主机完整根文件系统，宿主机的 `systemctl` 会 chroot 进其中来重启容器运行时。chart 以只读方式把它挂载到 `/host`，与 `--host-root` 相同路径 |

无法识别的 `--log-level`、`--log-format`、`--ib-mode` 或 `--container-runtime-restart`、相对的容器运行时路径，或不支持的 `--container-runtime` 都会导致启动失败，而不是静默回退，因此 Helm 取值中的拼写错误会停住 Pod，而不是让它以错误模式运行。`start` 返回的任何错误都会以 `node-agent: <error>` 的形式打印到 stderr 并退出 `1`。

### 没有 flag 的输入

`GPU_COUNT`、`DRIVER_VERSION`、`NODE_NAME`、`HOSTNAME`、`MOCK_FABRICMANAGER_STATE_DIR`、`IMEX_CHANNEL_MAJOR` 和 `IMEX_CAPS_MAJOR`（仅用于多节点 NVLink profile）在 profile 被编译为状态的地方读取，而不是由任何模拟器读取，因此没有 flag 覆盖它们。chart 会设置它们——参见[配置参考](../configuration.md)。

## flag 列表看不到的行为

- `--config` 和 `--topology` 每 5 秒重新读取一次。状态只在启动时生成一次，之后仅当任一文档的字节发生变化时才生成。
- 未设置 `--topology`，或它指向的文件不存在，都读作*本集群未声明拓扑*。其他读取失败则是错误，因为空文档会撤回节点上已部署的拓扑。
- 任一文档读取出错都会保留缓存状态，因此短暂不可读的 ConfigMap 不会拆除节点的模拟 GPU。
- 第一次协调之前，`nvml-mock-ctl` 写入的运行时覆盖会在模拟 NVML 引擎解析的两个位置被清除：`<host-root>/var/lib/nvml-mock/config/overrides.yaml` 和 `<host-root>/var/lib/nvml-mock/driver/config/overrides.yaml`。因此重启 Pod 是回到原始 profile 的方式。

## 用法

针对源码树运行，使用仓库中的 profile：

```bash
go run ./cmd/node-agent start \
  --config ./deployments/nvml-mock/helm/nvml-mock/profiles/a100.yaml \
  --host-root /tmp/mokka-host \
  --health-addr :9091
```

chart 会把这条命令行渲染进 nvml-mock DaemonSet。当 `topology.enabled` 为 false 时去掉 `--topology`；当 `nodeAgent.containerRuntime.enabled` 为 false 时去掉从 `--container-runtime` 开始的所有 flag。有 `nri.enabled` 时会加上 `--staging-lock-path`，并把 `--container-runtime-nri-socket` 设为 `nri.socketPath`：

```text
/usr/local/bin/node-agent start \
  --config=/etc/nvml-mock/config.yaml \
  --topology=/etc/nvml-mock/topology/topology.yaml \
  --host-root=/host \
  --health-addr=:9091 \
  --log-level=info \
  --log-format=json \
  --shutdown-timeout=5s \
  --resync-interval=1m \
  --container-runtime=containerd \
  --container-runtime-restart=systemd \
  --container-runtime-config=/etc/containerd/config.toml \
  --container-runtime-config-dir=/etc/containerd/conf.d \
  --container-runtime-systemd-unit=containerd \
  --host-fs=/host
```

其中三项与二进制自身默认值有重要差异，因此 chart 安装的行为不同于裸跑 `node-agent start`：探针监听 `:9091`，拆除使用 `nodeAgent.shutdownTimeout`（默认 5 秒）而不是 30 秒，并且会配置容器运行时。

InfiniBand 和 fabricmanager 的 flag 没有模板化。chart 通过 `MOCK_IB`、`MOCK_IB_PING_FABRIC`、`MOCK_IB_PING_PORT` 和 `MOCK_FABRICMANAGER_INIT_DELAY` 驱动这些模拟器，它们到达的是同样的字段。

该二进制安装在 nvml-mock 镜像的 `/usr/local/bin/node-agent`。

## 另见

- [命令行工具](README.md)
- [节点守护进程](../components/node-daemon.md)——它在节点上做什么
- [配置参考](../configuration.md)——它编译的每个 profile 字段
- [运行时控制](../nvml-mock-ctl.md)——无需重启即可修改状态
