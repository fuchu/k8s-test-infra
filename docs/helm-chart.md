# nvml-mock Helm Chart

面向 Kubernetes 测试的模拟 GPU 基础设施。通过基于 CGo 的模拟 NVML 库，把任意集群变成多 GPU 环境——无需物理 NVIDIA 硬件。

## 它做什么

部署一个 DaemonSet，在每个节点上创建：
- Mock `libnvidia-ml.so` shared library at `/var/lib/nvml-mock/driver/usr/lib64/`
- Mock device nodes at `/var/lib/nvml-mock/driver/dev/nvidia{N,ctl,-uvm,-uvm-tools}` (consumers see them at `/dev/nvidia*` via CDI bind-mount)
- GPU configuration at `/var/lib/nvml-mock/driver/config/config.yaml`
- An NFD feature file at
  `/etc/kubernetes/node-feature-discovery/features.d/nvml-mock.features`, which
  NFD turns into the node label
  `feature.node.kubernetes.io/pci-10de.present=true` (see [Node Labels](#node-labels))
- A fake InfiniBand sysfs tree at `/var/lib/nvml-mock/ib/sys/class/infiniband/...`
  paired with `libibmocksys.so` (`LD_PRELOAD`) so real `ibstat`, `ibstatus`,
  `iblinkinfo`, ... read mock HCAs
- A fake PCI sysfs tree at `/var/lib/nvml-mock/sys/bus/pci/devices/...` (symlinks
  into `/var/lib/nvml-mock/sys/devices/pciDDDD:BB/...`) so consumers resolve the
  PCIe root complex via a standard `readlink()`. `lspci` and anything else
  reaching it through libc read it via `libmockfs.so`; containers served the
  `nvidia.com/gpu` CDI spec get it bind-mounted at the kernel paths, which is
  what Go consumers need — see [PCI sysfs in containers](#pci-sysfs-in-containers)
- 假的内核模块接口面位于 `/var/lib/nvml-mock/proc/modules` 和 `/var/lib/nvml-mock/sys/module/...`。`/sys/module/nvidia/refcnt` 存在，`lsmod` 列出 `nvidia`、`nvidia_uvm`、`nvidia_modeset`、`gdrdrv` 和 `nvidia_fs`，启用 InfiniBand 时还包括 `nvidia_peermem` 和 `mlx5_core`。这覆盖了 GPU Operator validator 会 grep 的每一个模块以及它检查的 `nvidia` 引用计数。节点自身的模块被镜像在它们旁边。`libmockfs.so` 为 libc 消费者重定向这两个路径。两个 CDI 规格都为 Go 消费者 bind 挂载该树。状态协调会刷新镜像，因此它可能滞后于模块的加载或卸载
- 驱动的逐 GPU procfs 条目位于
  `/var/lib/nvml-mock/driver/proc/driver/nvidia/gpus/<bdf>/information`（型号、UUID、总线位置和设备次设备号），每块 GPU 在 `/var/lib/nvml-mock/sys/bus/pci/drivers/nvidia/<bdf>` 绑定到 `nvidia` 驱动，并带有其 `local_cpulist`。CPU 列表从 profile 中该 GPU 的 CPU 亲和性开始，只保留节点上在线的 CPU；若一个都不存在，则列出所有在线 CPU，正如内核对没有局部性的设备所做的那样。NVML 把同一列表报告为 GPU 的 CPU 亲和性。`libmockfs.so` 在内核路径上提供这两者，这正是 Slurm 的 `AutoDetect=nvidia` 读取的内容——参见 [Slinky 指南](guides/slinky/README.md)

消费者（DRA driver、device plugin）把 `/var/lib/nvml-mock/driver` 作为 NVIDIA 驱动根目录，并通过标准 NVML API 发现 GPU。

默认情况下（`nri.enabled=true`），chart 会向节点 DaemonSet 添加 `nvml-mock-nri`。这个节点本地的 containerd NRI 插件把宿主机覆盖层挂载到新创建容器的 `/opt/nvml-mock`，并在运行时注入模拟环境，因此 Pod 无需针对 Mokka 修改任何 spec。它只注入持有来自 device plugin 或 NVIDIA DRA driver 的 GPU 分配的容器，或其 Pod 带有值为 `"true"` 的 `nvml-mock.nvidia.com/devices`、`nvml-mock.nvidia.com/infiniband` 或 `nvml-mock.nvidia.com/imex-channels` 注解的容器。没有请求 GPU 且不带任何这些注解的 Pod 保持原样、看不到任何 GPU，与真实 GPU 节点一样。完整规则参见[哪些容器会被注入](components/nri-plugin.md#which-containers-are-injected)。[配置 NRI 注入](guides/nri-injection.md)介绍如何验证插件以及如何用 `nri.enabled=false` 关闭它。

**把它安装到自己的命名空间，并传入 `-n`：**

```bash
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  -n mokka --create-namespace
```

插件总是排除自己的 release 命名空间，因此主 nvml-mock DaemonSet 绝不会被自我注入。不使用 `-n` 安装时，release 命名空间就是 `default`——插件随后渲染 `--excluded-namespaces=default,kube-system`，并跳过初次使用者运行的每一个 Pod。没有任何东西会报告这一点：节点 DaemonSet 是 Ready，`/readyz` 返回 200 因为插件*确实*已注册，被跳过的容器在任何级别都不产生日志。Pod 就是简单地在没有模拟 GPU 的情况下启动。

## 前置条件

| 工具 | 版本 | 用途 |
|------|---------|-------------|
| [Docker](https://docs.docker.com/get-docker/) | 20.10+ | 构建镜像 |
| [Kind](https://kind.sigs.k8s.io/) | 0.20+ | 本地集群（或使用你自己的） |
| [kubectl](https://kubernetes.io/docs/tasks/tools/) | 1.31+ | 访问集群 |
| [Helm](https://helm.sh/docs/intro/install/) | 3.x | 安装 chart |
| [Go](https://go.dev/dl/) | 1.25+ | 从源码构建 |
| [jq](https://jqlang.github.io/jq/) | 任意 | 仅用于 DRA 验证 |

**已发布的镜像：** nvml-mock 容器镜像发布在 `ghcr.io/nvidia/nvml-mock`，每次发布时以发布版本打标签，推送到 `main` 时以 chart 的 `appVersion`（下一个 `-dev` 版本，如 `0.7.0-dev`）打标签。chart 默认安装以其 `appVersion` 打标签的镜像，因此已发布的 chart 拉取其自身发布版本，而 `main` 上的 chart 拉取最新的 `main` 构建。若镜像尚不可用，请使用下面快速开始小节中的"方案 B：从源码构建"。

**集群要求：**
- 必须允许特权 Pod（nvml-mock DaemonSet 使用 `privileged: true` 执行 `mknod`，使用 `hostPID: true` 通过 systemd 重启 containerd）
- containerd 启用 NRI，或设置 `nri.enabled=false`。当 `nodeAgent.containerRuntime` 开启时，节点守护进程会自行打开 NRI
- 对于 DRA：Kubernetes 1.32+ 且启用 `DynamicResourceAllocation` 特性门控
- 当 `nodeAgent.containerRuntime` 开启（默认）时：由 systemd 管理的 containerd 1.7+，且节点上 `/usr/bin`、`/usr/local` 和 `/etc` 可写。没有 containerd 的节点（如 CRI-O 或 k3s 节点）会被保持原样。节点守护进程会自行把 `nvidia` 运行时 handler 设为默认；参见[容器运行时设置](components/node-daemon.md#container-runtime-setup)

## 演练指南

以下每一个都端到端地针对真实消费者安装 chart：

| 指南 | 搭建内容 |
|---|---|
| [NVIDIA Device Plugin](guides/device-plugin.md) | 把模拟 GPU 通告为 `nvidia.com/gpu`，并据此调度工作负载 |
| [NVIDIA DRA Driver](guides/dra.md) | 把模拟 GPU 发布为 ResourceSlice，并通过 ResourceClaim 调度 Pod |
| [NVIDIA GPU Operator](guides/gpu-operator.md) | 完整的 operand 技术栈——device plugin、GFD、DCGM 和 validator |

本页其余部分是参考内容：chart 安装什么、它接受的每个取值，以及在你修改之前值得了解的行为。


## 集成：fake-gpu-operator

Run:ai 的 [fake-gpu-operator](https://github.com/run-ai/fake-gpu-operator) 在 Kubernetes API 层面模拟 GPU 以进行规模测试。nvml-mock 可以在真实节点上提供驱动级别的保真度（真实 NVML API），而 fake-gpu-operator 处理 KWOK 虚拟节点。

### 启用 Profile 发现

```bash
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  --set integrations.fakeGpuOperator.enabled=true
```

这会按 fake-gpu-operator 加载器读取的格式创建逐 profile 的 ConfigMap：

```bash
kubectl -n mokka get cm -l fake-gpu-operator/gpu-profile=true
```

```
NAME                              DATA   AGE
gpu-profile-a100                  1      10s
gpu-profile-h100                  1      10s
gpu-profile-b200                  1      10s
gpu-profile-gb200                 1      10s
gpu-profile-gb300                 1      10s
gpu-profile-l40s                  1      10s
gpu-profile-t4                    1      10s
```

FGO 按其自身命名空间中的名称加载这些 ConfigMap，因此请把 `integrations.fakeGpuOperator.targetNamespace` 设为 FGO 的 release 命名空间，以便它们被发现。这要求 FGO 的 `builtinProfiles.enabled=false`，因为其内置集合使用了相同的七个名称。参见 [fake-gpu-operator 指南](guides/runai-fgo/README.md)。

### 自定义标签

```bash
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  --set integrations.fakeGpuOperator.enabled=true \
  --set 'integrations.fakeGpuOperator.profileLabels.my-org/gpu-profile=true'
```

## InfiniBand mocking

Each profile carries an `infiniband:` block alongside the GPU config. When the
DaemonSet starts, `mock-ib` reads it and writes a fake sysfs tree at
`/var/lib/nvml-mock/ib/sys/class/infiniband/...`. Inside the container, three
LD_PRELOAD shims cooperate (preload order
`libibmockumad.so:libibmockverbs.so:libibmocksys.so`):

- `libibmocksys.so` rewrites every access to `/sys/class/infiniband*`,
  `/sys/class/infiniband_mad/`, `/sys/class/infiniband_verbs/` and
  `/dev/infiniband` so sysfs-driven tools read from the rendered tree.
- `libibmockumad.so` proxies `libibumad`'s `umad_send` / `umad_recv` to the
  in-pod `mock-ib` daemon (Unix socket) which handles SA path queries,
  ibping echoes, and SMP synthesis for `iblinkinfo`.
- `libibmockverbs.so` proxies open/read/write on `/dev/infiniband/uverbsN`
  so `libibverbs` consumers can enumerate HCAs.

```bash
POD=$(kubectl -n mokka get pods -l app.kubernetes.io/name=nvml-mock -o jsonpath='{.items[0].metadata.name}')

# sysfs / libibumad (always works):
kubectl -n mokka exec "$POD" -- ibstat
kubectl -n mokka exec "$POD" -- ibstatus

# libibverbs enumeration (modalias matches libmlx5's match table):
kubectl -n mokka exec "$POD" -- ibv_devinfo -l
kubectl -n mokka exec "$POD" -- ibv_devices

# Subnet management direct-route walk (cross-node fabric scan):
kubectl -n mokka exec "$POD" -- iblinkinfo
```

完整的逐设备 `ibv_devinfo`（不带 `-l`）刻意不受支持：libibverbs 认领设备后，`libmlx5` 的 `verbs_open_device` 会发出真实的 uverbs `ioctl()`，而用户态 `LD_PRELOAD` shim 无法伪造它们。同样的端口级信息（state、phys state、GID、LID、rate、link layer）可通过 `ibstatus` 获得，它从渲染好的 sysfs 树中读取。

### 在注入了 NRI 的 Pod 中

在 `nri.enabled=true` 时，同样的工具会被暂存到节点覆盖层中，并可从任何被注入的工作负载中的 `/opt/nvml-mock/driver/usr/bin/<tool>` 访问。它们把共享库（`libibmad`、`libibumad`、`libibverbs`、`libnl`）连同自身放在 `driver/usr/lib64` 下，并使用 RPATH `$ORIGIN/../lib64`，因此它们可以在自身不带 InfiniBand 工具栈的镜像中运行——distroless 或 scratch 工作负载，而不仅仅是完整的发行版镜像。

那里有两条限制，都与暂存无关：

- `ibstatus` 是 `/bin/sh` 脚本而非 ELF 二进制，因此需要一个带 shell 的镜像。
- `ibv_devinfo -l` 在被注入的 Pod 中报告 `0 HCAs found`。枚举需要 libibverbs 把设备匹配到 provider 驱动（`libmlx5`），而该 provider 随 nvml-mock 镜像而非工作负载提供。请改用 `ibstat -l`，它通过 `libibmocksys.so` 读取渲染好的 sysfs 并列出每个模拟 HCA。

这些工具是 glibc 二进制。在 musl 镜像（Alpine）上它们根本无法执行，因为 `PT_INTERP` 以绝对路径指定 `/lib/ld-linux-*.so.*`，任何 RPATH 都无法重定向它。

### 各 profile 的默认值

| Profile | 启用 | HCA | 速率 | 每 GPU 的 HCA 数 |
|---|---|---|---|---|
| `a100`  | 是 | ConnectX-6 (`MT4123`) | HDR 200 Gb/s | 1 |
| `h100`  | 是 | ConnectX-7 (`MT4129`) | NDR 400 Gb/s | 1 |
| `b200`  | 是 | ConnectX-7 (`MT4129`) | NDR 400 Gb/s | 1 |
| `gb200` | 是 | ConnectX-7 (`MT4129`) | NDR 400 Gb/s | 1 |
| `gb300` | 是 | ConnectX-7 (`MT4129`) | NDR 400 Gb/s | 1 |
| `l40s`  | 否 | — | — | — |
| `t4`    | 否 | — | — | — |

### `infiniband:` block schema

| 字段 | 默认值 | 说明 |
|---|---|---|
| `enabled` | `false` | 必须为 `true` 才会渲染任何目录树 |
| `hca_type` | `MT4129` | 在 `ibstat` 输出中显示为 `CA type` |
| `fw_version` | `28.39.2048` | 固件版本 |
| `hw_rev` | `0x0` | 硬件修订版本 |
| `board_id` | `MT_0000000838` | Mellanox 板卡 ID |
| `link_layer` | `InfiniBand` | `InfiniBand` 或 `Ethernet` |
| `rate_gbps` | `400` | 取值为 `100`（EDR）、`200`（HDR）、`400`（NDR）、`800`（XDR）之一 |
| `port_state` | `ACTIVE` | `DOWN`、`INIT`、`ARMED`、`ACTIVE`、`ACTIVE_DEFER` |
| `phys_state` | `LinkUp` | `Disabled`、`Polling`、`Training`、`LinkUp`、…… |
| `hcas_per_gpu` | `1` | HCA 总数 = `gpu.count * hcas_per_gpu` |
| `hca_count` | `0` | 非零时用它代替 `gpu.count * hcas_per_gpu` |
| `guid_prefix` | `a088c20300ab` | 节点/端口 GUID 的十六进制前缀。渲染器固定前 8 位十六进制数字，用低 32 位作为节点/HCA 身份 |
| `node_desc_template` | `{node_name} mlx5_{idx}` | 会插值 `{node_name}` 和 `{idx}` |

### NIC 身份、局部性与计数器

除了 `ibstat` 读取的内容，目录树还带有 NIC 健康监视器和 exporter 用来发现、定位和监视 NIC 的文件：

- `mlx5_N/device/` 是 HCA 的 PCI function：`vendor`（`0x15b3`）、`device`、`numa_node`，以及一个携带 `PCI_SLOT_NAME` 的 `uevent`。每个 HCA 与一块 GPU 配对，占据该 GPU 之后第一个空闲的 PCI 总线，并报告该 GPU 的 NUMA 节点。在 `a100` profile 上，`mlx5_0` 位于 `0000:08:00.0`，紧邻位于 `0000:07:00.0` 的 GPU。没有可配对的 GPU 地址时，`numa_node` 为 `-1`。
- `mlx5_N/ports/1/counters/*` 和 `hw_counters/*` 存放标准 IB 计数器以及 mlx5 传输计数器（`rnr_nak_retry_err`、`local_ack_timeout_err`、`roce_slow_restart`、……）。
- 在 `link_layer: Ethernet` 下，每个 HCA 还会在 `sys/class/net/` 下获得一个 RoCE 网络设备，其命名方式与 systemd 一致（`0000:08:00.0` 对应 `enp8s0np0`）。当 `port_state` 为 `ACTIVE` 时其 `operstate` 为 `up`，否则为 `down`，并带有 `carrier_changes` 和 `statistics/*`。`mlx5_N/device/net/<netdev>` 和 `<netdev>/device/infiniband/mlx5_N` 像内核设备链接那样把两者关联起来。

每个 HCA 都是一个物理 function，因此没有 `device/physfn` 链接。计数器读数为 `0`，且 `link_layer`、`port_state` 和 `phys_state` 适用于节点上的每个 HCA。

### 在某个 profile 上禁用 IB

根据意图有两种选择：

- **在运行时关闭 Pod 内的模拟 IB**（不渲染 sysfs、不运行 `mock-ib` 守护进程、shim 变为空操作），无需编辑 profile：

  ```bash
  helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
    --namespace mokka --create-namespace \
    --set gpu.profile=h100 \
    --set infiniband.mockTier=off
  ```

  这会禁用 Pod 中对 `ibstat` / `ibping` / `iblinkinfo` 的模拟。chart 仍会渲染 IB 的 `Service` 和 `NetworkPolicy`，因为它们跟随 profile 的 `infiniband.enabled` 而非层级——若要一并去掉，请使用下一个选项。

- **通过自定义 profile 文件在 profile 中设置 `infiniband.enabled: false`**（想要完全控制时首选）。注意 `gpu.customConfig` 是*替换整个* profile 配置而非合并，因此内联的 `--set-string 'gpu.customConfig=infiniband: { enabled: false }'` 会丢掉所有 GPU 设置——请改传一份完整的配置文件：

  ```bash
  helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
    --namespace mokka --create-namespace \
    --set-file gpu.customConfig=my-h100-no-ib.yaml
  ```

## PCIe 拓扑模拟

每个 profile 都带有一个 `pcie_topology:` 块，描述宿主机的 PCI root complex 布局。节点守护进程的 `pcibus` 模拟器读取它，并在 `/var/lib/nvml-mock/sys/...` 写入一棵与真实 Linux 内核所暴露内容一致的假 sysfs 树。拓扑感知的消费者（NVIDIA DRA driver、计算 NUMA 提示的 device plugin）通过对渲染出的树执行标准 `readlink()` + 路径解析，来判断"某块 GPU 位于哪个 PCIe root complex 上"：

```bash
$ readlink /var/lib/nvml-mock/sys/bus/pci/devices/0000:07:00.0
../../../devices/pci0000:00/0000:07:00.0

$ cat /var/lib/nvml-mock/sys/devices/pci0000:00/0000:07:00.0/numa_node
0
```

### 各 profile 的默认值

| Profile | Root complex | NUMA 节点 | 每 root 的 GPU 数 | NVSwitch 桥 |
|---|---|---|---|---|
| `a100`  | 2（`pci0000:00`、`pci0000:80`） | 2（双 EPYC） | 4 | 6 |
| `h100`  | 2（`pci0000:00`、`pci0000:80`） | 2（双路） | 4 | 4 |
| `b200`  | 2（`pci0000:00`、`pci0000:80`） | 2（双路） | 4 | 0 |
| `gb200` | 2（`pci0000:00`、`pci0000:40`） | 2（每颗 Grace CPU 一个） | 2 | 0 |
| `gb300` | 2（`pci0000:00`、`pci0000:40`） | 2（每颗 Grace CPU 一个） | 2 | 0 |
| `l40s`  | 2（`pci0000:00`、`pci0000:80`） | 2（双路） | 4 | 0 |
| `t4`    | 1（`pci0000:00`） | 1 | 4 | 0 |

桥接设备位于第一个 root complex 上，这与真实硬件上基板交换机的挂接位置一致。参见 [PCI 总线上的 NVSwitch](#nvswitches-on-the-pci-bus)。

### `pcie_topology:` block schema

```yaml
pcie_topology:
  root_complexes:
    - id: "pci0000:00"            # sysfs root-complex dir, format "pciDDDD:BB"
      numa_node: 0                 # numa_node value for every child device
      devices:
        - "0000:07:00.0"           # canonical 4-digit-domain BDF
        - "0000:0F:00.0"
    - id: "pci0000:80"
      numa_node: 1
      devices:
        - "0000:87:00.0"
        - "0000:90:00.0"
```

没有任何东西会校验该块，因此拼写错误不会在任何地方报告——守护进程的 `/healthz` 也不会。守护进程渲染的是把该块与 NVML 报告的设备对账后的结果，这会静默吸收大多数错误：

- 不是内核 `DDDD:BB:DD.F` 形式地址的 `bus_id` 会被丢弃，该设备也就被排除在树之外。这包括 NVML 通过 `nvmlPciInfo_t.busId` 报告的 8 位域；`bus_id` 携带的是 4 位形式，与 `busIdLegacy` 一样。不是地址的值没有任何消费者能查找，否则它只会变成一个目录名。
- 列在某个 root complex 下但 `devices[]` 中没有任何条目声明的 BDF 会被丢弃，连同因此变空的 root complex 一起。这也是 `gpu.count` 的工作方式：限制设备列表不改变布局，被截掉的 BDF 只是从树中消失，而不是渲染成 NVML 否认存在的 GPU。
- 列在两个 root complex 下的 BDF 保留在第一个，从第二个中丢弃。
- `devices[]` 中其 BDF 未被任何 root complex 列出的设备仍会被渲染，放在其自身地址所隐含的 root（`pciDDDD:BB`）下，并报告 `numa_node` `-1`。树中缺失的 GPU 是任何消费者都无法解析的。

如果 profile 完全省略 `pcie_topology:`，渲染器会退回到扁平的单 root 布局（所有设备位于 `pci0000:00` 下，NUMA 0）。设备完全没有声明 `bus_id` 的 profile 不会渲染任何内容，模拟器会清空前一个 profile 遗留的任何树。

### PCI 总线上的 NVSwitch

GPU 并不是 HGX 节点上唯一的 NVIDIA 设备。基板的 NVSwitch 也是 PCIe 端点，`lspci` 会把它们列为 GPU 旁边的桥接设备：

```console
$ lspci | grep NVIDIA
05:00.0 Bridge: NVIDIA Corporation GH100 [H100 NVSwitch]
...
1a:00.0 3D controller: NVIDIA Corporation GH100 [H100 SXM5 80GB]
```

交换机通过在 profile 的 [`nvlink.switches`](configuration.md#nvswitches) 列表中声明 `device_id` 而进入目录树。把同一 BDF 列在某个 `pcie_topology:` root complex 下，即可让该交换机获得该 root 的 `numa_node`；否则它会落在自身地址隐含的 root 下，`numa_node` 为 `-1`，与未放置的 GPU 完全一样。

交换机的 PCI class 渲染为 `0x068000`（基类 `0x06` 桥，子类 `0x80`），而 GPU 渲染为 `0x030200`（3D 控制器）。这一区别不只是外观：GPU Feature Discovery 会从该 class 派生 `nvidia.com/gpu.mode`，因此
a switch enumerating as a 3D controller would read as an extra GPU.

!!! note "Rack-scale platforms have no PCIe-visible switches"

    `gb200` and `gb300` declare NVSwitches without a `device_id`. On NVL72 the
    switches live in their own switch trays, reached over the NVLink cable
    cartridge, so a compute tray's `lspci` shows its GPUs and no bridges. Those
    switches still act as NVLink remote endpoints, which is what produces the
    `NV18` all-to-all in `nvidia-smi topo -m`.

### 容器中的 PCI sysfs

通过 `MOCK_PCI_ROOT` 访问该树需要 `libmockfs.so` 的 `LD_PRELOAD` shim，而它只对 libc 消费者有效：Go 的 `os` 包直接发出 `openat`，因此 shim 看不到这次打开操作，进程读取的是节点真实的 `/sys`，那里并不存在模拟 GPU。GPU Feature Discovery 和 NVIDIA DRA driver 都是 Go 写的。

因此节点守护进程写入的 `nvidia.com/gpu` CDI 规格会在内核路径上以只读方式 bind 挂载该树：

| Host | Container |
|---|---|
| `/var/lib/nvml-mock/sys/devices` | `/sys/devices` |
| `/var/lib/nvml-mock/sys/bus/pci/devices` | `/sys/bus/pci/devices` |

两者总是一起挂载。`sys/bus/pci/devices` 下的条目是指向 `../../../devices/pciDDDD:BB` 的相对符号链接，因此只挂载该目录会产生能列出、但每次属性读取都以 `ENOENT` 失败的条目。

**权衡：** `/sys/devices` 被整体挂载，这对被服务的容器隐藏了宿主机的其他设备类别——其中包括 CPU 拓扑。无法把它缩小到 profile 的 root complex：运行时必须创建挂载点，而 sysfs 在容器中是只读的，因此容器创建会直接失败而非降级。profile 经常声明节点并不具备的 root complex。[#689](https://github.com/NVIDIA/k8s-test-infra/issues/689) 跟踪移除这一权衡。

需要节点真实设备树的工作负载不得请求 `nvidia.com/gpu`，因为该挂载依附于容器工具链为该资源解析的 CDI 规格。`nri.excludedNamespaces` 不是逃生口：它只作用于 NRI 插件，而插件自身的 `nvml-mock.nvidia.com/gpu` 规格携带模块树却不携带 PCI sysfs 挂载。

### Machine type (`nvidia.com/gpu.machine`)

GFD 从 `--machine-type-file` 派生该标签，它默认是 `/sys/class/dmi/id/product_name`——在 `kind` 下这是 mock 无法拥有的路径：节点镜像在那里写入 `kind`，并在容器自身挂载建立后把它重新 bind 进每个容器，而在没有 DMI 的宿主机（Docker Desktop）上它根本不存在。

因此守护进程把机器类型写入 `driver/config/machine-type`，携带 `config.yaml` 的 CDI 挂载把它提供在 `/etc/nvml-mock/machine-type`。无论是否启用 `nri.enabled`，都必须手工把 GFD 指向它。解析 `nvidia.com/gpu` 的容器工具链会应用规格中的挂载，但会丢弃其环境变量（[#747](https://github.com/NVIDIA/k8s-test-infra/issues/747)）。NRI 插件只在它注入的容器中设置 `GFD_MACHINE_TYPE_FILE`，即持有 GPU 分配或通过注解选择加入的容器（参见[哪些容器会被注入](components/nri-plugin.md#which-containers-are-injected)），而 GFD 两者都不做：

```yaml
gfd:
  env:
    - name: GFD_MACHINE_TYPE_FILE
      value: "/etc/nvml-mock/machine-type"
```

该值是 profile 的 GPU 产品名称，因此 `gpu.machine` 与 `gpu.product`（`NVIDIA-GB300-NVL`）匹配，而不是真实托盘所报告的 `NVIDIA-GB300-NVL72`；这些 profile 没有携带可替代的平台名称。

### 跨节点 `ibping`

仅靠 sysfs 模拟就能让 `ibstat` / `iblinkinfo` 工作，但真正的 `ibping` 需要 UMAD I/O。对启用 IB 的 profile，chart 会在 `libibmocksys.so` 之外预加载 `libibmockumad.so`，在每个 Pod 中启动 `mock-ib`，并在 18515 端口暴露一个 headless Service，用于 nvml-mock Pod 之间的 TCP fabric 中继。

fabric 监听器绑定 `0.0.0.0` 且没有认证，因此在 `full` 层级 chart 还会提供一个 `NetworkPolicy`（`infiniband.ping.networkPolicy.enabled`，默认 `true`），只允许来自该 release 命名空间中对等 nvml-mock Pod 的入站 fabric 流量。跨命名空间拆分的机群可在 `infiniband.ping.networkPolicy.peerNamespaces` 中列出其他命名空间。低于 `full` 时没有任何东西监听该端口，因此不会渲染策略。NetworkPolicy 只由实现它的 CNI 强制执行，包括 Kind 默认的 `kindnet`（自 [Kind v0.24.0](https://github.com/kubernetes-sigs/kind/releases/tag/v0.24.0) 起）、Calico 和 Cilium。较旧的 Kind 版本和不支持 NetworkPolicy 的 CNI 不会强制执行该策略。mock-ib 是测试装置——不要把它部署到共享或生产集群。

```bash
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  --set gpu.profile=a100 \
  --set gpu.count=2 \
  --wait --timeout 120s
```

在多节点集群上，选取位于不同节点上的两个 nvml-mock Pod。从 sysfs 读取服务端的 LID，并从客户端 ping 该 LID：

```bash
SERVER_POD=$(kubectl -n mokka get pods -l app.kubernetes.io/name=nvml-mock \
  -o jsonpath='{.items[0].metadata.name}')
CLIENT_POD=$(kubectl -n mokka get pods -l app.kubernetes.io/name=nvml-mock \
  -o jsonpath='{.items[1].metadata.name}')

LID=$(kubectl -n mokka exec "$SERVER_POD" -- sh -c \
  "tr -d '[:space:]' < /var/lib/nvml-mock/ib/sys/class/infiniband/mlx5_0/ports/1/lid")

kubectl -n mokka exec "$CLIENT_POD" -- ibping -c 3 "$LID"
```

对于自动化的跨节点验证（包括对端重启和重试），运行 [Go e2e 套件](https://github.com/NVIDIA/k8s-test-infra/blob/main/tests/e2e/README.md)中的 `ibping` 和 `ibfabric` 规格。基于 LID 的 ping 是受支持的路径；跨节点 `ibping -G <port_guid>` 也受支持（使用不带冒号的 `0x` 十六进制）。同样的规格还覆盖配套的 fabric 工具：

- `iblinkinfo` 和 `ibnetdiscover`——定向路由遍历到达对端 GUID，无重复端口错误。
- `ibv_devinfo -l` 通过 libmlx5 认领每个渲染出的 HCA，`ibstatus` 确认 ACTIVE / LinkUp 端口状态。
- `sminfo` 报告一个主子网管理器，每个 Pod 都指向同一个。

mock 读取 `MOCK_IB`、`MOCK_IB_PING_FABRIC`、`MOCK_IB_ROOT` 和 `MOCK_IB_PEERS`。chart 从前三个变量派生出[`infiniband` 取值](#infiniband-mocking)。

## 设备注入模式

仅在 `nri.enabled=true` 时适用，且只适用于 `nri.deviceAnnotation` 选择加入的路径。

`nri.deviceInjectionMode` 选择插件如何把模拟 GPU 设备节点交付给带有 `nvml-mock.nvidia.com/devices: "true"` 且不持有 GPU 分配的容器：

| 模式 | 机制 | 需要 |
| --- | --- | --- |
| `raw`（默认） | 插件在 NRI 调整中自行暂存 `/dev/nvidia*` 节点。 | 无。 |
| `cdi` | 插件发出 CDI 设备 `nvml-mock.nvidia.com/gpu=all`，运行时从 `cri` 模拟器暂存在 `<cdiSpecDir>/nvml-mock-nri.yaml` 的规格中解析它。 | 一个启用 CDI 的运行时，以及接受 CDI 设备的 NRI：即非 containerd 1.7。 |

两种模式交付相同的 GPU，且都注入 `libmockfs.so`，因此 libc 读取方无论哪种方式都能找到模拟模块。只有 `cdi` 会 bind 挂载 `/sys/module`，因此 Go 读取方只有在那里才能找到它们。`cdi` 还会设置 `NVML_MOCK_DEVICE_SOURCE=cdi`，这是判断运行了哪种机制的唯一方式。

CDI 在节点上不需要容器工具链。containerd 2.x 默认启用 CDI（`enable_cdi = true`，规格目录 `/etc/cdi` 和 `/var/run/cdi`），这包括原装的 `kindest/node` 镜像。containerd 1.7 即便设置了 `enable_cdi` 也无法使用 `cdi`：它的 NRI 早于 CDI 设备调整，会静默丢弃该引用，于是容器得到模拟库却没有 `/dev/nvidia*` 节点。`raw` 仍是默认值。

若选择了 `cdi` 但没有暂存规格，插件会记录警告并回退到 `raw`。它不会让 Pod 失败：无法解析的 CDI 设备会让 containerd 直接拒绝创建容器。

两种模式都不会改变已分配容器*获得哪些* GPU。持有 device plugin 或 NVIDIA DRA 分配的容器，即使在有注解的情况下，在两种模式中都精确保留该分配，依据 [MEP-0002](https://github.com/NVIDIA/k8s-test-infra/blob/main/enhancements/meps/0002-device-plugin-nri-composition/README.md)。

## NRI Pod 生命周期

仅在 `nri.enabled=true` 时适用。

节点 agent 和 NRI 插件在同一个节点 DaemonSet Pod 中作为独立容器运行。在 Kubernetes 1.29 及以后，节点 agent 是[可重启的 init 容器](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/)，NRI 是普通容器，这会为两者排序：

- **启动。** 节点 agent 的启动探针读取 `/stagedz`，它在其第一波 Stage 写入驱动树后打开，因此 Kubernetes 只在那之后才启动 NRI。`/stagedz` 不等待刻意设置的 `fabricmanager.initDelay`，因此新工作负载仍可能观察到模拟的 `IN_PROGRESS` fabric 状态。启动探针通过之前 Kubernetes 不运行存活探针，因此 `nodeAgent.nriStartupTimeoutSeconds`（默认 120）也是首次暂存失败的节点 agent 在重启前等待的时长。
- **终止。** Kubernetes 在节点 agent 之前停止 NRI，因此插件会在 agent 移除暂存的驱动树之前断开连接。

两者都运行时，NRI 在每次调整前检查节点 agent，并在 agent 重启或重新暂存期间保持新容器不变；参见[失败时开放](components/nri-plugin.md#failing-open)。

如果集群管理员显式禁用了 `SidecarContainers` 特性门控，请设置 `nri.nativeSidecar=false`。该设置以及 Kubernetes 1.28 会把两者都渲染为普通容器，而 Kubernetes 不会为它们排序，因此在节点 agent 启动时暂存文件的过程中，NRI 会保持容器不变。chart 根据 Helm 的 `.Capabilities.KubeVersion` 选择布局：离线渲染时，向 `helm template` 传入 `--kube-version`，或在 GitOps 渲染器中做等价设置。跨越 Kubernetes 1.29 的升级会在下一次 Helm upgrade 时切换布局并滚动每个节点 Pod。

修改任何 `nri.*` 取值都会滚动节点 DaemonSet 并短暂重建暂存的驱动树。chart 没有单独部署插件的选项，因此这就是运行 NRI 的运维成本。

就绪状态也是共享的。未 Ready 的插件，包括其所在节点的容器运行时禁用了 NRI 的插件，会把整个节点 Pod 标记为 NotReady；参见 [NRI 插件失败模式](#nri-plugin-failure-modes)。

对启用 InfiniBand 的 profile，headless `-ibping` Service 即便在共享 Pod 为 NotReady 时也会发布 Pod 地址。中继运行在节点 agent 中，因此未就绪的 NRI 容器不应把一个本来健康的中继对对端发现隐藏起来。Kubernetes 的就绪是 Pod 级的，因此这也会在节点 agent 自身未就绪时发布地址；中继客户端本就会重试不可达的对端。

## NRI 插件失败模式

仅在 `nri.enabled=true` 时适用。

NRI 插件在容器创建时注入模拟 GPU 栈，这让以常规方式获得 GPU 的 Pod 无需修改 Pod spec 就能看到模拟 GPU。这也意味着注入只在创建时一次性写入容器的 OCI spec。已经运行的 Pod 会保留它被给予的一切，无论之后插件发生什么。**只有故障之后创建的 Pod 会受影响**，而且影响是静默的。

正是这一性质让加固变得值得：一个早期创建 Pod、之后才对其断言的测试套件，会在注入数小时前就已停止的节点上继续通过。

### 两种模式

| | 失败即关闭 | 失败即开放 |
|---|---|---|
| 运行时的行为 | 拒绝创建容器 | 在没有插件调整的情况下创建容器 |
| 你看到的现象 | Pod 卡在 `ContainerCreating` / `CreateContainerError` | Pod 正常启动 |
| 工作负载得到什么 | 什么都没有——它根本不运行 | 一个**没有模拟 GPU 栈**的容器 |
| 风险 | 测试运行停止 | 测试继续运行并报告不再具有其所声称含义的结果 |

失败即关闭是响亮且自我宣告的。在专用测试集群上，这可以说是更可取的姿态：损坏的 mock 会停止运行，而不是破坏它。

失败即开放才是危险的那一种，也是本 chart 被设计用来承受的模式。它无法从插件一侧阻止——决定权在 containerd 而非插件——因此本 chart 的姿态是：**假定失败即开放可能发生，并让它不可能静默发生。**

### 本 chart 追求的姿态

**可检测的失败即开放。** 两个探针的存在都是为了把静默窗口变成可见窗口，而非阻止它：

- **就绪（`/readyz`）** 仅在插件已向运行时注册*并且*其 handler 正在应答时报告服务中。节点未进行注入的任何窗口都会表现为 NotReady 的 Pod 和不足的 DaemonSet 计数。就绪不会重启任何东西；它纯粹是检测面。
- **存活（`/healthz`）** 仅在容器创建请求已超过卡死阈值仍在进行中时才失败，并把容器重启到一次全新的注册。

两个探针都不能简化为"进程是否还活着"，因为在每一种要紧的模式下进程都保持存活：

| 状态 | 进程 | 连接 | `/readyz` | `/healthz` |
|---|---|---|---|---|
| 已注册并在服务 | 运行 | 正常 | 200 | 200 |
| 已启动、尚未注册 | 运行 | — | 503 | 200 |
| 被运行时注销 | 运行 | 断开 | 503 | 200 |
| Handler 卡死 | 运行 | **正常** | 503 | **503 → 重启** |

Handler 卡死正是让每一种更简单的检查都失效的情形：`pgrep nri-plugin` 能找到进程，普通 TCP 检查能看到 socket 处于监听状态，而恰恰是在这种状态下什么都没有被注入。

连接丢失被刻意*不*视为存活失败：插件每 500 ms 在进程内重新注册，不受 kubelet 崩溃退避影响，否则在此期间新容器会在没有模拟的情况下启动。

### containerd `plugin_request_timeout`

卡死阈值在 chart 中并非常量。插件从 containerd 在注册时自己报告的请求超时派生它，并在该值的**两倍**处触发。超过一整个超时后，运行时已经放弃该请求，因此它所属的容器无论接下来发生什么，都是在没有注入的情况下创建的；第二个超时是容差，让单个缓慢但会完成的请求不会重启插件。

NRI 的默认值，来自 `containerd/nri/pkg/api`：

| 设置 | 默认值 |
|---|---|
| `plugin_request_timeout` | `2s`（卡死阈值 `4s`） |
| `plugin_registration_timeout` | `5s` | |

两者都在运行时上设置，而非在本 chart 中：

```toml
[plugins."io.containerd.nri.v1.nri"]
  disable = false
  socket_path = "/var/run/nri/nri.sock"
  # 仅当插件确实需要超过 2s 才能应答时才提高。
  plugin_request_timeout = "2s"
```

在[容器运行时设置](components/node-daemon.md#container-runtime-setup)开启时，2.1 之前的 containerd 会从 Mokka 的配置文件获取 NRI 的整个表，因此请把完整的表放在 containerd 配置目录中排序在其之后的文件里，例如 `conf.d/60-nri.toml`。

建议：

- **除非有证据，否则保持默认值。** 插件的 `CreateContainer` 路径只对拓扑文档做一次 `stat`、对设备目录做一次目录读取、对每个设备节点做一次 `stat`——全部针对 hostPath 挂载的覆盖层，别无其他。在健康节点上这远低于 2s。提高超时不会让注入更可靠；它只会拉长每次容器创建阻塞在一个可能已经卡死的插件上的窗口。
- **那些文件系统调用才是真实的卡死点。** 它们是 handler 中唯一的阻塞操作，因此覆盖层背后是一个挂起的挂载，正是该插件在保持存活和连接的同时停止应答的方式。
- **提高它会自动放宽卡死阈值。** 无需修改 chart，也不应修改——`nri.livenessProbe` 的调整与 `plugin_request_timeout` 不是独立的旋钮。
- **不要靠提高它来掩盖卡死。** 需要超过 2s 的插件正是这种加固所要检测的故障，而不是一个调优问题。

### 手工检查节点

```bash
# 当前哪些节点真的在注入
kubectl get pods -n mokka -l app.kubernetes.io/name=nvml-mock -o wide

# 某个节点为什么没有
kubectl describe pod -n mokka <nvml-mock-pod>
```

两个探针端点都在响应正文中给出原因，因此 `kubectl describe` 中的就绪失败会显示为 `not registered with the container runtime; new containers are not being injected`，而不只是一个状态码。

该端口无法从节点访问：节点 DaemonSet 没有设置 `hostNetwork`，因此 `nri.healthPort` 只绑定在 Pod 自身的网络命名空间内，位于 kubelet 访问它的 Pod IP 上。

## 配置

### Values 取值

| 参数 | 默认值 | 描述 |
|-----------|---------|-------------|
| `gpu.profile` | `gb300` | GPU profile：`a100`、`h100`、`b200`、`gb200`、`gb300`、`l40s`、`t4` 或 `vrnvl72` |
| `gpu.count` | `""` | 每节点的模拟 GPU 数量。为空时从 profile 的 `devices:` 列表派生（基板 profile 为 8，`t4`、`gb200`、`gb300` 和 `vrnvl72` 为 4）；较大的值在运行时会被限制到该列表 |
| `gpu.customConfig` | `""` | 用于整体覆盖 profile 配置的内联 YAML |
| `gpu.dynamicMetrics.enabled` | `false` | 让 mock 返回随时间变化的温度/功耗/利用率读数，而非静态 profile 值。参见下文的[动态指标](#dynamic-metrics)。 |
| `gpu.dynamicMetrics.seed` | `0`（基线） | 随机数种子；`0` 使用基于时间的种子，非零值产生可复现的序列。 |
| `gpu.dynamicMetrics.temperature.*` | 基线（`base_c: 55`，……） | GPU 温度生成器的 `base_c`、`variance_c`、`ramp_c`、`ramp_period_sec`。 |
| `gpu.dynamicMetrics.power.*` | profile 默认值，否则基线 `250000`/`25000` | 功耗生成器的 `base_mw`、`variance_mw`（钳制到 profile 的 `min/max_limit_mw`）。解析顺序为**基线 < profile 默认值 < 用户覆盖**；偏离 250W 基线的 profile 自行设置（`t4` 约 65W，`b200`/`gb200` 约 600W，`gb300` 约 800W，`vrnvl72` 约 420W，即其空闲功耗）。参见[动态指标](#dynamic-metrics)。 |
| `gpu.dynamicMetrics.utilization.*` | 基线（`pattern: burst`，……） | `pattern`（`idle` \| `busy` \| `burst` \| `steady`）、`gpu_min/max`、`memory_min/max`、`burst_period_sec`。 |
| `gpu.failureInjection.enabled` | `false` | 启用模拟 GPU 故障（丢失/从总线脱落/不可纠正 ECC）。参见下文的[故障注入](#failure-injection)。 |
| `gpu.failureInjection.mode` | `healthy` | 故障模式：`healthy`（默认，无操作）、`lost`、`fallen_off_bus` 或 `ecc_uncorrectable`。在无效的默认值下，仅 `enabled: true` 只会产生健康设备——必须显式设置 `mode` 才会触发故障。 |
| `gpu.failureInjection.probability` | `0.0` | 随机故障触发的每次调用概率 `[0, 1]`。 |
| `gpu.failureInjection.after_calls` | `0` | 在 N 次受保护 NVML 调用后确定性地触发故障（0 = 禁用）。 |
| `gpu.failureInjection.seed` | `0` | 概率摇号的随机数种子；`0` 使用基于时间的种子。 |
| `gpu.failureInjection.xid.code` | `0` | 触发后通过 NVML 事件集（`NVML_EVENT_TYPE_XID_CRITICAL_ERROR`）传递的 Xid 错误码。`0` = 无 Xid。 |
| `global.imageRegistry` | `""` | 拉取所有 chart 镜像的仓库，例如镜像站。替换 `image.repository`、`nri.image.repository` 和 `controlPlane.image.repository` 的仓库主机，因此 `ghcr.io/nvidia/nvml-mock` 变为 `<registry>/nvidia/nvml-mock`。 |
| `global.imagePullSecrets` | `[]` | 添加到每个 chart Pod（nvml-mock、NRI 和控制面）的拉取密钥，位于 `imagePullSecrets` 之前。每项是一个 Secret 名称或 `{name: <secret>}`。 |
| `imagePullSecrets` | `[]` | 添加到每个 chart Pod 的拉取密钥，位于 `global.imagePullSecrets` 之后；重复项会被丢弃。条目形式相同。 |
| `extraObjects` | `[]` | 随 release 一起渲染的额外 Kubernetes 对象，每个都经过 `tpl` 处理，因此可以使用 `{{ .Release.Name }}` 等 chart 取值。可以是列表，也可以是以名称为键的 map，以便分层 values 文件覆盖某项或用 `""` 丢弃它。每项是一个对象或 YAML 字符串。 |
| `image.repository` | `ghcr.io/nvidia/nvml-mock` | 容器镜像仓库 |
| `image.digest` | `""` | 不可变的 `sha256:...` 摘要。设置后会固定镜像并优先于 `image.tag`。 |
| `image.tag` | `""`（chart `appVersion`） | 容器镜像 tag。为空时使用 chart 的 `appVersion`：已发布 chart 中为发布版本，`main` 上为下一个 `-dev` 版本。 |
| `image.pullPolicy` | `IfNotPresent` | 镜像拉取策略。为空时采用渲染镜像的 Kubernetes 默认值：`latest` tag 为 `Always`，其他为 `IfNotPresent`。 |
| `featureGates` | `{}` | 传递给节点 agent、NRI 插件和控制面的特性门控，格式为 `Name: true\|false`。参见[特性门控](feature-gates.md)。 |
| `driverVersion` | `""`（自动） | 要模拟的 NVIDIA 驱动版本。为空时从解析出的 GPU 配置（所选 `gpu.profile` 文件，或设置后的 `gpu.customConfig`）的 `system.driver_version` 读取，因此 profile 是唯一事实来源（每个随附 profile 都固定为 `580.65.06`）。显式设置可覆盖 profile——该版本还决定存在哪些 NVML 函数，因为 mock 会拒绝其声明驱动之前尚不存在的函数。 |
| `driverBranch` | `""`（派生） | `nvmlSystemGetDriverBranch` 报告的分支。为空时从驱动版本派生（`580.65.06` → `r580_00`）；设置它可模拟某个小版本分支，如 `r570_40` |
| `hic` | `[]` | `nvmlSystemGetHicVersion` 报告的主机接口卡，即 `nvidia-smi -q -u` 打印的内容。条目为 `{id, firmwareVersion}`。真实 HIC 只存在于已退役的 S 级机箱上，因此没有 profile 配置任何 HIC。 |
| `nodeSelector` | `{}` | DaemonSet 的节点选择器 |
| `tolerations` | `[{operator: Exists}]` | Pod tolerations（默认：容忍所有） |
| `priorityClassName` | `""` | 节点 DaemonSet Pod 的 PriorityClass。节点 agent 代替其所在节点的 GPU 驱动，因此 `system-node-critical` 可防止它在依赖它的工作负载之前被饿死或驱逐。 |
| `affinity` | `{}` | 节点 DaemonSet Pod 的 Pod 亲和性 |
| `podAnnotations` | `{}` | 添加到节点 DaemonSet Pod 的注解。chart 自身的注解（`checksum/config`、`checksum/mig-profiles`、`kubectl.kubernetes.io/default-container`）始终优先。 |
| `podLabels` | `{}` | 添加到节点 DaemonSet Pod 的标签。不得设置选择器标签（`app.kubernetes.io/name`、`instance` 或 `component`）。 |
| `nodeAgent.livenessProbe` | `health` 端口上的 `httpGet /healthz` | 节点 agent 的存活探针。设为 `null` 可移除。 |
| `nodeAgent.readinessProbe` | `health` 端口上的 `httpGet /readyz` | 节点 agent 的就绪探针。设为 `null` 可移除。 |
| `nodeLabels.featuresDir` | `/etc/kubernetes/node-feature-discovery/features.d` | NFD local source 读取 feature 文件的宿主目录。仅当 NFD 使用非默认的 `featureFilesDir` 运行时才覆盖 |
| `nodeAgent.nriStartupTimeoutSeconds` | `120` | 在 Kubernetes 1.29+ 上，当 NRI 使用原生 sidecar 时，节点 agent 的 `/stagedz` 门在 Kubernetes 重启它之前，可等待其首次暂存和应用周期的最长时间 |
| `nodeAgent.containerRuntime.enabled` | `true` | 配置节点的容器运行时：安装 NVIDIA 容器运行时并把它设为 containerd 的默认 `nvidia` handler；节点 Pod 停止时撤销。参见[容器运行时设置](components/node-daemon.md#container-runtime-setup)。拒绝 `updateStrategy.rollingUpdate.maxSurge` |
| `nodeAgent.containerRuntime.type` | `containerd` | 节点上的容器运行时；目前仅支持 containerd |
| `nodeAgent.containerRuntime.restartMode` | `systemd` | `systemd` 在变更之后从宿主 PID 命名空间重启运行时的 unit；`none` 只写入配置，把重启留给你 |
| `nodeAgent.containerRuntime.containerd.configPath` | `/etc/containerd/config.toml` | 宿主机上 containerd 的主配置文件。其目录以相同路径挂载到节点 Pod 中 |
| `nodeAgent.containerRuntime.containerd.configDir` | `/etc/containerd/conf.d` | containerd 的配置目录，`config.toml` 会 import 它，Mokka 的配置文件也放在这里；必须位于 `configPath` 的目录之下。在 EKS AL2023 上，`/etc/containerd/config.d` 已被 import |
| `nodeAgent.containerRuntime.containerd.systemdUnit` | `containerd` | 变更后重启的 systemd unit |
| `integrations.fakeGpuOperator.enabled` | `false` | 创建名为 `gpu-profile-<profile>`、键为 `profile.yaml` 的逐 profile ConfigMap，格式遵循 fake-gpu-operator 加载器所读取的形式 |
| `integrations.fakeGpuOperator.targetNamespace` | `""`（release 命名空间） | profile ConfigMap 所在命名空间。设为 FGO 的 release 命名空间以便 FGO 找到它们；要求 FGO 的 `builtinProfiles.enabled=false`，以避免在这七个相同名称上发生 Helm 所有权冲突 |
| `integrations.fakeGpuOperator.profileLabels` | `{"run.ai/gpu-profile": "true"}` | profile ConfigMap 的额外标签。契约标签 `fake-gpu-operator/gpu-profile` 和 `nvml-mock/profile-name` 始终会生成，无法在此移除 |
| `infiniband.mockTier` | `""`（自动） | `MOCK_IB` 层级：`off`、`sysfs` 或 `full`。为空时对启用 IB 的 profile 自动派生 `full`，否则为 `sysfs`（保持 `libibmocksys` 重定向有效，从而屏蔽宿主机上的真实 IB）。`off` 让每个 shim 成为空操作并跳过守护进程。无效值会让 `helm template` 失败 |
| `infiniband.ping.port` | `18515` | nvml-mock Pod 之间 fabric 中继的 TCP 端口（`mock-ib` / `ibping` 始终启用） |
| `infiniband.ping.networkPolicy.enabled` | `true` | 把对 fabric 端口的入站访问限制为对等的 nvml-mock Pod。仅在 `full` 层级渲染。对不强制执行 NetworkPolicy 的 CNI 无效果 |
| `infiniband.ping.networkPolicy.peerNamespaces` | `[]` | 除 release 所在命名空间外，其 nvml-mock Pod 可访问 fabric 端口的命名空间 |
| `nri.enabled` | `true` | 在节点 DaemonSet 中运行 `nvml-mock-nri` containerd NRI 插件。在未排除的命名空间中，它会注入持有 GPU 分配或通过注解选择加入的容器；参见[配置 NRI 注入](guides/nri-injection.md)。在 containerd 禁用 NRI 的地方设为 `false`。始终安装到专用命名空间（`-n mokka`），以避免排除 `default` |
| `nri.nativeSidecar` | `true` | 在 Kubernetes 1.29+ 上，使用有序的 `SidecarContainers` 布局。当该特性门控被显式禁用时设为 `false`；chart 会回退到无序的普通容器 |
| `nri.socketPath` | `/var/run/nri/nri.sock` | 宿主机上的 NRI socket。其目录以 hostPath 挂载到插件中 |
| `nri.pluginName` / `nri.pluginIndex` | `nvml-mock` / `"10"` | NRI 注册身份。索引用于将该插件与其他插件排序 |
| `nri.overlay.hostPath` / `nri.overlay.mountPath` | `/var/lib/nvml-mock` / `/opt/nvml-mock` | 由主 DaemonSet 暂存的宿主机覆盖层，以及它在工作负载内部的注入路径 |
| `nri.optOutAnnotation` | `nvml-mock.nvidia.com/inject` | Pod 注解；值为 `false` 时禁用对该 Pod 的注入 |
| `nri.deviceAnnotation` | `nvml-mock.nvidia.com/devices` | Pod 注解；值为 `true` 时，让没有 GPU 分配的 Pod 获得覆盖层以及节点上所有模拟 GPU。对持有分配的容器会被忽略。由 Pod 作者控制，因此应把它视为演示信任边界的一部分 |
| `nri.deviceInjectionMode` | `raw` | `nri.deviceAnnotation` 交付 GPU 的方式：`raw` 直接暂存设备节点，`cdi` 发出由运行时解析的 CDI 设备引用。参见[设备注入模式](#device-injection-mode) |
| `nri.cdiSpecDir` | `/var/run/cdi` | 存放 CDI 规格的宿主目录，以只读方式挂载到插件中。必须是运行时配置的 `cdi_spec_dirs` 之一 |
| `nri.imexChannelAnnotation` | `nvml-mock.nvidia.com/imex-channels` | Pod 注解；值为 `true` 时，让该 Pod 获得节点 agent 在多节点 NVLink profile 上暂存的模拟 `/dev/nvidia-caps-imex-channels/channel0` 节点（其他 profile 上没有），别无其他。信任边界与 `nri.deviceAnnotation` 相同 |
| `nri.infinibandAnnotation` | `nvml-mock.nvidia.com/infiniband` | Pod 注解；值为 `true` 时，让该 Pod 获得模拟 InfiniBand 工具生效的覆盖层（`MOCK_IB=full`）。除非 Pod 另行选择，否则不含模拟 GPU。信任边界与 `nri.deviceAnnotation` 相同 |
| `nri.excludedNamespaces` | `[]` | 要跳过的额外命名空间。release 所在命名空间和 `kube-system` 始终被排除 |
| `nri.healthPort` | `8080` | 提供 `/healthz` 和 `/readyz` 的端口。仅绑定在 Pod 的网络命名空间内——该 DaemonSet 不使用 `hostNetwork`，因此节点上没有暴露任何东西 |
| `nri.readinessProbe` | `/readyz`、`periodSeconds: 10`、`failureThreshold: 2` | 检测节点是否已停止注入。设为 `null` 可移除。参见[NRI 插件失败模式](#nri-plugin-failure-modes) |
| `nri.livenessProbe` | `/healthz`、`periodSeconds: 10`、`failureThreshold: 3` | 重启卡死的插件。阈值跟随 containerd 的 `plugin_request_timeout`；请勿独立调整两者。设为 `null` 可移除 |
| `nri.resources` | 请求 `cpu: 10m`、`memory: 32Mi` | 插件容器的资源请求/限制。默认无限制 |

### Node 标签

nvml-mock 自身不写任何节点标签。它写一个 feature 文件，由 NFD 转换为一个标签；当守护进程在关闭时删除该文件后，NFD 会在下一个周期撤销该标签：

| 标签 | 写入者 | 移除者 |
|-------|-----------|------------|
| `feature.node.kubernetes.io/pci-10de.present=true` | **NFD**，根据节点守护进程写入的 feature 文件 | NFD，在节点守护进程删除文件之后 |

`nvidia.com/` 下的标签——`gpu.present`、`gpu.count`、`gpu.product`——与真实硬件一样属于 NFD 和 GFD，除非部署了它们，否则这些标签不存在。需要落到模拟节点上的工作负载，应选择其集群为那些节点赋予的 GPU 节点池标签，就像在真实硬件上所做的那样。

上面的标签由 Node Feature Discovery 产生。nvml-mock 只提供输入：`internal/agent/pcibus` 把 `pci-10de.present=true` 写入 `nodeLabels.featuresDir`，NFD 的 local source 读取它并转换为带命名空间的标签。集群上没有 NFD 时，该文件不起作用，标签也不存在——这是诚实的状态，也正是 `tests/e2e/go/scenario_nfd_test.go` 中 e2e 所断言的。

按当前部署方式，NFD 的 *PCI* source 仍看不到模拟 GPU：它读取一个在链接时固定的带宿主机前缀的 sysfs 路径（`/host-sys/bus/pci/devices`，来自 `HOSTMOUNT_PREFIX`），而 nvml-mock 渲染出的 PCI 树位于 `/var/lib/nvml-mock/sys` 下，且只能通过 `LD_PRELOAD` sysfs shim 在规范的 `/sys` 路径上访问——而 `nfd-worker` 是静态链接发布的，因此 `LD_PRELOAD` 在其中不起作用。任一事实本身都足够；两者都成立（针对 `go.mod` 中固定的 NFD v0.19.0 验证）。

这是 NFD 部署方式的限制，而非渲染树的限制。指向 `/var/lib/nvml-mock/sys` 后，NFD v0.19.0 会枚举每块模拟 GPU，并且——借助 GPU Operator 配置的 `sources.pci.deviceLabelFields: [vendor]`——自行派生出精确的 `pci-10de.present`。渲染出的设备已经带有其 PCI source 视为必需的五个属性。只缺可见性，而 nvml-mock 若不修改第三方 DaemonSet 就无法提供它，这就是 chart 采用 local source 路线的原因。

写 feature 文件而非直接写标签，也是该 DaemonSet 完全不需要集群 RBAC 的原因：Pod 中没有任何东西调用 Kubernetes API。

### GPU Profile

内置 profile 为常见数据中心 GPU 提供真实的硬件规格。用 `--set gpu.profile=<name>` 选择 profile：

```bash
# Deploy as an 8-GPU H100 node
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  --set image.repository=nvml-mock \
  --set image.tag=local \
  --set gpu.profile=h100

# Deploy as a 4-GPU B200 node
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  --set image.repository=nvml-mock \
  --set image.tag=local \
  --set gpu.profile=b200 \
  --set gpu.count=4
```

#### Profile 对比

| | A100 | H100 | B200 | GB200 | GB300 | L40S | T4 | VR NVL72 |
|---|---|---|---|---|---|---|---|---|
| **Profile 名称** | `a100` | `h100` | `b200` | `gb200` | `gb300` | `l40s` | `t4` | `vrnvl72` |
| **完整名称** | A100-SXM4-40GB | H100 80GB HBM3 | B200 | GB200 NVL | GB300 NVL | L40S | Tesla T4 | NVIDIA Graphics Device |
| **架构** | Ampere | Hopper | Blackwell | Blackwell | Blackwell Ultra | Ada Lovelace | Turing | Rubin |
| **计算能力** | 8.0 | 9.0 | 10.0 | 10.0 | 10.0 | 8.9 | 7.5 | 10.7 |
| **CUDA 核心** | 6,912 | 16,896 | 18,432 | 18,432 | 21,632 | 18,176 | 2,560 | 27,136 |
| **显存** | 40 GiB HBM2e | 80 GiB HBM3 | 180 GiB HBM3e | 186 GiB HBM3e | 278 GiB HBM3e | 48 GiB GDDR6 | 16 GiB GDDR6 | 280 GiB HBM4 |
| **NVLink** | v3，12 链路 | v4，18 链路 | v5，18 链路 | v5，18 链路 | v5，18 链路 | — | — | v6，36 链路 |
| **NVLink 带宽** | 600 GB/s | 900 GB/s | 1.8 TB/s | 1.8 TB/s | 1.8 TB/s | — | — | 2.95 TB/s |
| **TDP** | 400W | 700W | 1,000W | 1,000W | 1,400W | 350W | 70W | 2,300W |
| **PCIe** | Gen4 | Gen5 | Gen5 | Gen6 | Gen6 | Gen4 | Gen3 | Gen6 |
| **MIG 实例** | 7 | 7 | 7 | 7 | 7 | 0 | 0 | 7 |
| **宿主 CPU** | — | — | — | Grace (NVLink-C2C) | Grace (NVLink-C2C) | — | — | Vera (NVLink-C2C) |
| **FP8** | — | 是 | 是 | 是 | 是 | 是 | — | 是 |
| **FP4** | — | — | 是 | 是 | 是 | — | — | 是 |
| **FP6** | — | — | — | — | 是 | — | — | — |
| **驱动版本** | 580.65.06 | 580.65.06 | 580.65.06 | 580.65.06 | 580.65.06 | 580.65.06 | 580.65.06 | 615.71.09 |

#### 何时使用各 Profile

- **`a100`**——兼容性最广。大多数 NVIDIA 软件在文档和示例中都以 A100 为前提。当测试针对 Ampere，或被更新的架构绊住时使用它。
- **`h100`**——测试 Hopper 特有功能：FP8、Transformer Engine、PCIe Gen5 或 NVLink v4 拓扑。
- **`b200`**——测试下一代 Blackwell 功能：FP4 和 NVLink v5。独立 GPU（无 Grace CPU）。
- **`gb200`**——测试 Grace-Blackwell Superchip：到 Grace CPU 的 NVLink-C2C、统一内存以及 Blackwell 功能。
- **`gb300`**（默认）——测试 Grace-Blackwell Ultra Superchip：每 GPU 278 GiB HBM3e、1.4 kW TDP，以及 FP4/FP8 之外的 FP6。
- **`l40s`**——测试 Ada Lovelace 推理工作负载：FP8、PCIe Gen4、无 NVLink（仅 PCIe 拓扑）。
- **`t4`**——测试 Turing 推理 GPU：低功耗（70W）、小显存（16 GiB）、每节点 4 块 GPU。
- **`vrnvl72`**——测试 Vera Rubin 计算托盘：Rubin 架构，计算能力 10.7，286,524 MiB HBM4，36 条跨节点 NVLink 6 链路（IMEX 和 ComputeDomain），RoCE NIC，MIG，以及一个驱动尚不认识的 GPU 名称（`NVIDIA Graphics Device`）。

### 自定义配置

对于内置 profile 未覆盖的 GPU 类型，请提供你自己的配置 YAML。

#### 方案 A：基于文件（推荐）

按 profile 格式创建一个 YAML 文件，然后在安装时传入：

```bash
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  --set image.repository=nvml-mock \
  --set image.tag=local \
  --set-file gpu.customConfig=my-custom-gpus.yaml
```

#### 方案 B：内联 values

For small overrides, embed the config directly in a values file:

```yaml
# custom-values.yaml
gpu:
  count: 4
  customConfig: |
    version: "1.0"
    system:
      driver_version: "550.163.01"
      nvml_version: "12.550.163.01"
      cuda_version: "12.4"
      cuda_version_major: 12
      cuda_version_minor: 4
    device_defaults:
      name: "NVIDIA L40S"
      architecture: "ada_lovelace"
      compute_capability:
        major: 8
        minor: 9
      num_gpu_cores: 18176
      memory:
        total_bytes: 48318382080
        reserved_bytes: 536870912
        free_bytes: 47781511168
        used_bytes: 0
    devices:
      - index: 0
        uuid: "GPU-14050000-0000-0000-0000-000000000000"
        minor_number: 0
```

```bash
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  --set image.repository=nvml-mock \
  --set image.tag=local \
  -f custom-values.yaml
```

#### 编写自定义 Profile

以一个现有 profile 为起点：

```bash
cp deployments/nvml-mock/helm/nvml-mock/profiles/a100.yaml my-custom-gpus.yaml
```

需要修改的关键字段：

| 字段 | 设置内容 |
|-------|-------------|
| `device_defaults.name` | `nvidia-smi` 中显示的 GPU 名称 |
| `device_defaults.architecture` | 架构字符串（`ampere`、`hopper`、`blackwell` 等） |
| `device_defaults.compute_capability` | `major` / `minor` 版本 |
| `device_defaults.num_gpu_cores` | CUDA 核心数 |
| `device_defaults.memory.total_bytes` | GPU 总显存，单位字节 |
| `devices` | 每块 GPU 一个条目（与 `gpu.count` 匹配），带唯一 UUID |
| `nvlink` | NVLink 版本和链路数（仅 PCIe 的 GPU 可省略） |

完整的 YAML schema 与 `nvidia-smi -x -q` 暴露的字段一致。完整示例参见 `deployments/nvml-mock/helm/nvml-mock/profiles/` 中的内置 profile。

### 动态指标

真实 GPU 报告随时间变化的指标——负载下温度升高、利用率波动、功耗攀升。默认情况下 mock 是完全静态的：profile 的 `thermal`、`power` 和 `utilization` 小节中设置的值在每次调用时都原样返回。

设置 `gpu.dynamicMetrics.enabled=true` 可让渲染出的 ConfigMap 注入一个 `device_defaults.dynamic_metrics` 块。随后 mock 会从 `GetTemperature`、`GetPowerUsage` 和 `GetUtilizationRates` 返回波动的值。每个子小节（`temperature`、`power`、`utilization`）都可独立调整；该覆盖层适用于任何内置 profile 以及 `gpu.customConfig`。

每个字段按三层解析，优先级后者最高：

```
chart baseline  <  GPU profile default  <  user override (values / --set)
```

profile 这一层对**功耗**很重要：一个全局 `base_mw` 无法适配每个 profile 的 `[min_limit_mw, max_limit_mw]` 包络，因此偏离 250W 基线的 profile 通过一个仅用于 Helm 的 `dynamic_metrics_defaults` 键声明自己的基值（`t4` 约 65W，`b200`/`gb200` 约 600W，`gb300` 约 800W，`vrnvl72` 约 420W，即其空闲功耗）。引擎会忽略该键；只有在 `enabled: true` 把它折叠进 `dynamic_metrics` 之后它才生效。

```bash
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  --set image.repository=nvml-mock \
  --set image.tag=local \
  --set gpu.profile=h100 \
  --set gpu.dynamicMetrics.enabled=true \
  --set gpu.dynamicMetrics.utilization.pattern=burst
```

或通过 values 文件：

```yaml
gpu:
  profile: h100
  dynamicMetrics:
    enabled: true
    seed: 0                             # set non-zero for reproducibility
    temperature:
      base_c: 60
      variance_c: 3
      ramp_c: 15
      ramp_period_sec: 120
    power:
      base_mw: 500000
      variance_mw: 50000
    utilization:
      pattern: burst                    # idle | busy | burst | steady
      gpu_min: 0
      gpu_max: 100
      memory_min: 0
      memory_max: 100
      burst_period_sec: 30
```

利用率 pattern 的语义（取值始终钳制到 `0..100`）：

| pattern  | 采样范围 |
| -------- | ----------------------------------------------------- |
| `idle`   | `[gpu_min, gpu_max]` 的底部四分之一 |
| `busy`   | `[gpu_min, gpu_max]` 的顶部四分之一 |
| `burst`  | 每 `burst_period_sec` 在 `idle` / `busy` 之间交替 |
| `steady` | 完整的 `[gpu_min, gpu_max]` 范围（省略时为默认值） |

完整的引擎侧参考参见 [`pkg/gpu/mocknvml/README.md`](https://github.com/NVIDIA/k8s-test-infra/blob/main/pkg/gpu/mocknvml/README.md#dynamic-metrics-optional)。

### 故障注入

真实 GPU 偶尔会从总线脱落、累积不可纠正 ECC 错误，或抛出 Xid 事件。默认情况下 mock 报告健康硬件。设置 `gpu.failureInjection.enabled=true` 可让渲染出的 ConfigMap 注入一个 `device_defaults.failure` 块；随后 mock 会根据你选择的触发条件，把设备切换到配置的故障模式：

```bash
# 确定性：在第 200 次 NVML 调用后设备变为"丢失"
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  --set gpu.profile=h100 \
  --set gpu.failureInjection.enabled=true \
  --set gpu.failureInjection.mode=lost \
  --set gpu.failureInjection.after_calls=200
```

```yaml
# 随机 + Xid：每次调用有 1% 概率抛出 ECC 双比特错误（Xid 64），
# 限制在 1 万次调用内触发，避免 CI 挂起。
gpu:
  profile: h100
  failureInjection:
    enabled: true
    mode: ecc_uncorrectable
    probability: 0.01
    after_calls: 10000
    seed: 12345
    xid:
      code: 64
```

各模式的行为：

| 模式 | 受保护 API 调用返回 | 句柄查找返回 | 身份 getter | ECC 计数器 | 事件集 |
| ------------------- | ------------------------ | --------------------- | ------------------- | -------------------- | ------------------------------- |
| `healthy`（默认） | 正常值 | 正常句柄 | 正常值 | 零 | 空 |
| `lost` | `ERROR_GPU_IS_LOST` | `ERROR_GPU_IS_LOST` | `ERROR_GPU_IS_LOST` | 错误 | `ERROR_GPU_IS_LOST`（若设置 `xid:` 且由 getter 触发，则先发一次 Xid） |
| `fallen_off_bus` | `ERROR_GPU_IS_LOST` | `ERROR_GPU_IS_LOST` | `ERROR_GPU_IS_LOST` | 错误 | `ERROR_GPU_IS_LOST`（若设置 `xid:` 且由 getter 触发，则先发一次 Xid） |
| `ecc_uncorrectable` | 正常值 | 正常句柄 | 正常值 | 严格递增 | 若有 xid 则发一次 `XID_CRITICAL_ERROR` |

配置的 Xid 每次触发只传递一次，通过 `nvmlEventSetWait_v1` 或 `nvmlEventSetWait_v2`。后续等待则遵循该模式其余的行为：`lost` 和 `fallen_off_bus` 立即返回 `NVML_ERROR_GPU_IS_LOST`，正如真实 NVML 在 Xid 79 之后的作为；`ecc_uncorrectable` 报告 `NVML_ERROR_TIMEOUT`（无事件）。没有待处理事件且设备未丢失时，等待会阻塞调用方的超时时长，与真实 NVML 一样——像 device plugin 健康监视器和 dcgm-exporter 这样的客户端会在等待上循环且自身不休眠，因此立即返回 `TIMEOUT` 会空转一个 CPU 核。`ERROR_GPU_IS_LOST` 是例外：真实 NVML 会迅速返回它，而这些客户端本就会在它上面退避。

配置声明为*立即*丢失的设备——没有 `after_calls` 或 `probability` 门槛的裸 `mode: lost` / `fallen_off_bus` 块——自身就会以 `NVML_ERROR_GPU_IS_LOST` 使等待失败，因此只在 `nvmlEventSetWait` 上循环的客户端（DRA driver 的健康监视器）能像真实 NVML 呈现从总线脱落的设备那样看到丢失的 GPU，无需先调用 getter。其他一切仍需要触发：注入器只在受保护的**设备**调用（`GetTemperature`、`GetEccErrors`、……）上触发，从不在等待本身上触发，因此传递配置的 **Xid**、`ecc_uncorrectable` 事件，以及任何 `after_calls` / `probability` 门槛，只有在某物驱动设备 getter（`nvidia-smi -q`、dcgm-exporter 抓取）时才会推进。`nvml-mock-ctl` 只写覆盖文件——它配置故障，但不触发故障。

渲染进 ConfigMap 的取值会在安装/升级时依据 [`values.schema.json`](https://github.com/NVIDIA/k8s-test-infra/blob/main/deployments/nvml-mock/helm/nvml-mock/values.schema.json) 校验：像 `mode: healhty` 这样的拼写错误或 `probability: 1.5` 这样的越界值，会在 chart 渲染之前被 Helm 拒绝，于是配置错误会作为可操作的 schema 错误显现，而不是被静默地在运行时强制转换。

故障注入可与 `gpu.dynamicMetrics` 组合：两者都启用时，设备健康时返回动态读数，触发条件一旦满足便切换到配置的故障模式。一旦触发，设备在该 Pod 的生命周期内保持故障状态，与需要重启才能恢复的真实硬件一致。

#### 用 `nvidia-smi` 验证

每次 `nvidia-smi` 调用都是一个新进程，其调用计数器从 0 开始，因此像 `--query-gpu=ecc.errors.uncorrected.aggregate.total` 这样窄的查询，每次调用每块 GPU 只会发出**一次**受保护调用。（在单个进程内，计数器与设备状态的其他部分一样，在 `nvmlShutdown()` 后接 `nvmlInit()` 时也会保留。）要从一条简短命令中看到故障表现，请设置 `after_calls: 1`，或使用能在单个进程内对每块 GPU 发出多次受保护调用的更丰富查询（如 `nvidia-smi -q`），使触发条件在进程内满足。

```bash
# mode: lost / fallen_off_bus  ─  handle lookup itself fails once tripped.
# nvidia-smi prints "Unable to determine the device handle for GPU ..."
# and exits non-zero.
kubectl -n mokka exec ds/nvml-mock -- nvidia-smi -L
kubectl -n mokka exec ds/nvml-mock -- nvidia-smi --query-gpu=name,uuid --format=csv
kubectl -n mokka exec ds/nvml-mock -- nvidia-smi -q                # "GPU is lost"

# mode: ecc_uncorrectable  ─  device stays addressable; counters grow and
# nvmlEventSetWait_v1/_v2 delivers the configured Xid once per trip.
kubectl -n mokka exec ds/nvml-mock -- nvidia-smi -q -d ECC
kubectl -n mokka exec ds/nvml-mock -- nvidia-smi \
  --query-gpu=ecc.errors.uncorrected.aggregate.total --format=csv
kubectl -n mokka exec ds/nvml-mock -- nvidia-smi \
  --query-gpu=ecc.errors.uncorrected.aggregate.dram  --format=csv

# Any mode  ─  watch the engine trip in real time.
kubectl -n mokka exec ds/nvml-mock -- env MOCK_NVML_DEBUG=1 \
  nvidia-smi -q -d ECC 2>&1 | grep -E 'failure|GPU_IS_LOST|Xid'

# One long-running process so the per-process call counter accumulates
# (useful when after_calls > 1 and you want to see a deterministic trip
# without restarting the daemonset).
kubectl -n mokka exec ds/nvml-mock -- nvidia-smi \
  --query-gpu=ecc.errors.uncorrected.aggregate.total --format=csv -l 1
```

完整的引擎侧参考（包括各模式如何与特定 NVML 调用交互）参见 [`pkg/gpu/mocknvml/README.md`](https://github.com/NVIDIA/k8s-test-infra/blob/main/pkg/gpu/mocknvml/README.md#failure-injection-optional)。

## 工作原理

该 chart 部署：

1. **DaemonSet**——在每个节点上运行一个特权容器，它：
   - 把 `libnvidia-ml.so.{version}` 复制到宿主机的 `/var/lib/nvml-mock/driver/usr/lib64/`
   - 创建符号链接（`libnvidia-ml.so.1` → `libnvidia-ml.so.{version}`）
   - 在 `/var/lib/nvml-mock/driver/dev/nvidia{N,ctl,-uvm,-uvm-tools}` 创建模拟设备节点（CDI 把它们 bind 挂载到消费者容器中的 `/dev/nvidia*`）
   - 在 `/var/lib/nvml-mock/driver/config/config.yaml` 写入 GPU 配置 YAML
   - 写入 NFD feature 文件，使 `feature.node.kubernetes.io/pci-10de.present=true` 出现——参见 [Node 标签](#node-labels)
   - 安装 NVIDIA 容器运行时并向 containerd 注册为其默认 `nvidia` 运行时 handler，Pod 停止时撤销——参见[容器运行时设置](components/node-daemon.md#container-runtime-setup)
2. **ConfigMap**——来自所选 profile 的 GPU 配置
3. **ConfigMap**（`<fullname>-mig-profiles`）——所选板卡的 MIG 分区表，挂载在 `/etc/nvml-mock/mig` 并由 `MOCK_MIG_PROFILES_CONFIG` 指向。仅对五个支持 MIG 的 profile 渲染，且从不与 `gpu.customConfig` 一起渲染（后者的表由用户提供）——参见[表放在哪里](mig.md#where-the-table-lives)
4. **ServiceAccount**——无集群 RBAC；Pod 中没有任何东西调用 API

消费者组件（DRA driver、device plugin）挂载 `/var/lib/nvml-mock`，并使用 `--nvidia-driver-root=/var/lib/nvml-mock/driver` 通过标准 NVML `tryResolveLibrary` 路径发现 GPU。

## 已知限制

模拟 NVML 库覆盖了消费者用于 GPU 发现和监控的 NVML C API 面。某些宿主级子系统没有被模拟：

| 缺失项 | 受影响的消费者 | 影响 |
|----------------|-------------------|--------|
| **在 mock 未服务的容器中**缺少 `/sys/bus/pci/devices/{busID}` sysfs 条目 | 任何 Go 消费者 | Go 的 `os` 包发起原始系统调用，`LD_PRELOAD` shim 无法拦截，因此 Go 二进制读取的是宿主机真实 sysfs。获得 `nvidia.com/gpu` CDI 规格的容器会改为在内核路径上 bind 挂载该树；仅由 NRI 插件触达的容器仍看到宿主机的——参见[容器中的 PCI sysfs](#pci-sysfs-in-containers) |
| **在 mock 已服务的容器中**缺少宿主机的其他设备类别 | 任何读取 `/sys/devices` 的东西 | `/sys/devices` 被整体替换，因此 CPU 拓扑和容器的命名空间 sysfs 在那里不可见。跟踪于 [#689](https://github.com/NVIDIA/k8s-test-infra/issues/689) |
| **在 mock 未服务的容器中**缺少 `/sys/bus/pci/devices/{busID}/numa_node` | Device plugin | 那里无法获得 NUMA 感知的拓扑提示；调度可用但不强制 NUMA 亲和性。被服务的 device plugin 确实能获得提示——渲染器为每个设备写入 `numa_node` |
| `/sys/bus/pci/devices/*/vendor,device,class` **按 NFD 的读取方式**（`/host-sys/…`，在链接时固定） | NFD（Node Feature Discovery） | PCI feature 标签不会被自动检测。`pci-10de.present` 由 NFD 根据 nvml-mock 放在 `nodeLabels.featuresDir` 中的 feature 文件创建——参见 [Node 标签](#node-labels) |

### PCIe Root Complex（DRA driver）

把 DRA driver 与 nvml-mock 一起使用时，你会看到类似这样的警告：

```
W0319 11:41:21.314205       1 nvlib.go:491] error getting PCIe root for device 0,
  continuing without attribute: failed to resolve PCIe Root Complex for PCI Bus ID
  0000:07:00.0: failed to read symlink for PCI Bus ID /sys/bus/pci/devices/0000:07:00.0:
  readlink /sys/bus/pci/devices/0000:07:00.0: no such file or directory
```

driver 通过对 `/sys/bus/pci/devices/{busID}` 执行 `readlink()` 来解析 PCIe root complex 拓扑，而它是 Go 二进制，因此 `libmockfs.so` 无法把该读取重定向到渲染出的树。获得 `nvidia.com/gpu` CDI 规格的容器会在该路径上获得该树并解析出 root；mock 未服务的容器则读取节点真实 sysfs 并记录上面的警告。两种方式下 GPU 都完全可分配——只有 `dra.k8s.io/pcieRoot` 属性受影响。

**缺失它会阻塞什么：** 使用 `pcieRoot` 把设备对齐到同一 PCIe root complex 的 DRA 拓扑感知调度——例如，为了最优数据路径局部性，把一块 GPU 与来自同一 root 的 SR-IOV 虚拟 function（VF）共同调度。没有 `pcieRoot`，表达跨设备拓扑约束的 ResourceClaim 就无法被验证。

针对固定驱动版本端到端确认该属性，跟踪于 [#265](https://github.com/NVIDIA/k8s-test-infra/issues/265)。注意上游对树形形状必须满足的约束：`deviceattribute` 会拒绝解析目标不以 `devices/pci` 开头的符号链接，因此 mock 的条目必须保持为规范的相对符号链接，并在内核路径上提供，而不是重定向到别处。

## 排障

**ImagePullBackOff**：确认镜像可访问。chart 默认拉取 `ghcr.io/nvidia/nvml-mock:<chart appVersion>`；请检查该 tag 是否存在，或设置 `image.tag`。对于本地构建，请确保镜像已加载进集群（参见快速开始）。

**DaemonSet 未就绪**：检查 Pod 日志：`kubectl -n mokka logs -l app.kubernetes.io/name=nvml-mock`。
记录 `container runtime setup failed` 的节点 Pod 无法配置容器运行时；该行说明了原因，[容器运行时设置](components/node-daemon.md#container-runtime-setup)列出了它所需的内容。在 containerd 运行但节点无法配置的场合（如 Bottlerocket），请设置 `nodeAgent.containerRuntime.enabled=false`。

**GPU Operator 的 operand 卡在 `toolkit-validation`**：有六个 operand DaemonSet 会一直阻塞，直到 `/run/nvidia/validations/toolkit-ready` 存在。nvml-mock 刻意不写这个文件——该标记属于 GPU Operator 自己的 `nvidia-validator`，它会删除该标记、针对 mock 驱动运行 `nvidia-smi`，仅在成功时重建它。若在这里写它，会让 operand 在任何检查运行之前就通过关卡。如果 operand 卡在一个永不返回的标记上，而 validator 处于 `CrashLoopBackOff`，请检查 `CLEANUP_ALL`：它让 validator 对输出目录执行 `RemoveAll`，在 bind 挂载上因 `EBUSY` 失败，并在重建任何东西之前退出。默认情况下没有任何东西设置它。

**Device plugin 显示 0 块 GPU**：确认节点上存在 mock 文件：
```bash
NODE_CONTAINER=$(docker ps --filter name=control-plane -q)
docker exec "$NODE_CONTAINER" ls /var/lib/nvml-mock/driver/usr/lib64/libnvidia-ml.so.*
docker exec "$NODE_CONTAINER" cat /var/lib/nvml-mock/driver/config/config.yaml
```

**DRA driver Pod 未就绪**：检查 DRA 日志：
```bash
kubectl -n nvidia logs -l app.kubernetes.io/name=dra-driver-nvidia-gpu --tail=100
```

**DRA driver 的 PCIe root 警告**：参见[已知限制](#known-limitations)。

**特权 Pod 被阻止**：你的集群可能有 PodSecurity 或 OPA/Gatekeeper 策略阻止 `privileged: true` 或 `hostPID: true`。KIND 默认允许两者。对于托管集群，你可能需要为 nvml-mock 的 release 命名空间创建一个 PodSecurity 例外。

## 相关文档

- [模拟 NVML 库文档](index.md)
- [E2E Test Workflow](https://github.com/NVIDIA/k8s-test-infra/blob/main/.github/workflows/nvml-mock-e2e-go.yaml)
- [KIND DRA Config](https://github.com/NVIDIA/k8s-test-infra/blob/main/tests/e2e/kind-dra-config.yaml)
- [Device Plugin Mock Manifest](https://github.com/NVIDIA/k8s-test-infra/blob/main/tests/e2e/device-plugin-mock.yaml)
