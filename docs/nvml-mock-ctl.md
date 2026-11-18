# 使用 `nvml-mock-ctl` 进行运行时控制

`nvml-mock-ctl` 可以在**不**执行 Helm upgrade、重建镜像或重启 Pod 的情况下，修改正在运行的 nvml-mock 节点的模拟 GPU 状态。用它可以在测试进行中即时注入故障、翻转 ECC 状态或调整指标（温度、功耗、利用率、时钟、风扇等）。

- **启动时状态**来自 Helm profile / `config.yaml`（参见 [Helm chart README](helm-chart.md) 和[配置参考](configuration.md)）。
- **运行时状态**由 `nvml-mock-ctl` 叠加在其上，本文所述即此。

## 工作原理

nvml-mock 是**逐进程**的共享库（`libnvidia-ml.so`），而**不是**守护进程。没有可发送命令的服务器。取而代之的是：

1. `nvml-mock-ctl` 原子性地写入一个节点本地的**配置覆盖**文件 `overrides.yaml`，它位于原始的 `config.yaml` 旁边。
2. 加载在每个消费者进程内的模拟引擎，按较短的 TTL（默认 **1 秒**，`MOCK_NVML_OVERRIDES_TTL`）重新读取该配置覆盖，并把它**深度合并**到原始基础配置之上。基础 `config.yaml` 从不被修改。
3. 由于配置覆盖通过 CDI bind 挂载进消费者容器，已经在运行的进程和新启动的进程都会在一个 TTL 内收敛到新状态。

合并顺序为：

```
base config.yaml  <  配置覆盖的 "all:" 块  <  配置覆盖的 "devices[<idx>]:" 块
```

因此逐设备覆盖优先于 `all` 覆盖，`all` 覆盖又优先于基础 profile。未知字段和错误类型在写入时会被**拒绝**，因此拼写错误会让命令失败，而不是静默地什么都不做。

### v1 范围——什么可以热重载，什么不可以

配置覆盖驱动每一个*由配置派生的 getter*：故障注入、ECC 模式/计数器、温度、功耗、利用率、时钟、风扇、性能状态、设备显存（`memory.total_bytes` / `free_bytes` / `used_bytes` / `reserved_bytes` / `memory_bus_width`）等。这些在一个 TTL 内变化。

MIG 分区在 **NVML 视图**层面可热重载，通过 `nvidia-smi` 而非本 CLI：`nvidia-smi -mig` 和 `nvidia-smi mig -cgi/-cci/-dgi/-dci` 对 mock 的作用与对驱动一样，库会记录每次变更，使其比发起变更的进程存活更久。它们改变的是 `nvidia-smi`、DCGM 和 GFD 报告的内容，以及 device plugin *发现*的内容——参见 [MIG 切分指南](guides/mig/README.md)。

驱动能力接口面不可热重载——`/dev/nvidia-caps` 和 `/proc/driver/nvidia-caps/mig-minors` 在 Pod 启动时部署一次，因此要分配运行时重新分区产生的资源仍需 Helm 变更（`gpu.mig.enabled` 配合 `gpu.mig.gpuInstances`）。重新分区也不是完全可逆的：清除覆盖——包括 `nvml-mock-ctl reset`——会恢复 NVML 报告的布局，但不会恢复它报告时使用的实例 ID，于是部署好的能力接口面与 CDI 条目不再匹配，只有重启 `nvml-mock` Pod 才能让节点回到可分配状态。

显存数值从有效配置**原样**报告——只设置 `memory.used_bytes` 不会重新计算 `memory.free_bytes`。要在一条命令中同时设置两者，以保持报告的三元组一致：

```bash
nvml-mock-ctl set --gpu 0 memory.used_bytes=1073741824 memory.free_bytes=41264611328
```

少数**身份/拓扑**字段在设备构造时就固化到设备上，在 v1 中**不可**热重载——修改它们需要重启 Pod 或变更 Helm：

- 设备 `name`
- `architecture`
- `brand`
- `compute_capability`
- `uuid`
- PCI `bus_id`
- BAR1 窗口——`bar1_memory`（在构造时固化，因此 `set --gpu 0 bar1_memory.total_bytes=...` 之类要到 Pod 重启后才会生效）

如需修改其中任何一项，请编辑 profile / Helm values 并重启 DaemonSet（或受影响的 Pod）。

关于哪些报告值是模拟的、哪些是静态的、哪些是刻意固定的——包括为什么 `DCGM_FI_PROF_*` 不跟踪真实工作，以及为什么没有任何值会响应持有 `nvidia.com/gpu` claim 的 Pod——参见[指标保真度](configuration.md#metric-fidelity)。

## 运行位置

`nvml-mock-ctl` 随 nvml-mock DaemonSet 镜像一起交付，通过 `kubectl exec` 进入**目标节点**上的 DaemonSet Pod 运行。其作用域是**逐节点**的：它只影响你 exec 进入的那个节点。要修改多个节点，请对每个节点的 Pod 重复执行该命令。

在 Pod 内，配置路径由 `MOCK_NVML_CONFIG` 指定，配置覆盖默认指向宿主机挂载的驱动配置目录（`/var/lib/nvml-mock/driver/config/overrides.yaml`），因此通常无需任何路径 flag 即可运行子命令。（只有在覆盖该默认值时才设置 `MOCK_NVML_OVERRIDES` / `--file`。）

```bash
# 选择特定节点上的 nvml-mock Pod。
# 把 <node> 替换为节点名；如果安装在别处请调整 -n。
POD=$(kubectl -n mokka get pod -l app.kubernetes.io/name=nvml-mock \
  --field-selector spec.nodeName=<node> -o jsonpath='{.items[0].metadata.name}')
```

## 命令参考

`nvml-mock-ctl --help` 列出各命令，`nvml-mock-ctl <command> --help` 说明该命令自身的 flag、默认值和位置参数。

```text
NAME:
   nvml-mock-ctl - mutate the simulated GPU state of a running nvml-mock node

USAGE:
   nvml-mock-ctl [global options] [command [command options]]

COMMANDS:
   fail               inject a device failure, or clear one with --mode healthy
   temp, temperature  pin reported GPU temperature
   power              pin reported power draw
   fan                pin reported fan speed (forces fan count >= 1)
   util, utilization  pin reported GPU + memory utilization
   clocks             pin reported SM + graphics clocks
   throttle           set the active throttle reasons ('none' clears them)
   pstate             pin the reported performance state (P-state)
   nvlink-error       inject NVLink DL errors at a rate in errors/second (0 heals)
   sram-ecc           inject SRAM ECC errors (0 heals)
   fabric-health      degrade NVLink fabric health ('healthy' clears it)
   set                write arbitrary schema fields, as key.path=value
   status             print the overrides currently in effect
   reset              clear the targeted overrides, returning the device(s) to the pristine profile
   watch-allocations  mirror each pod's nvidia.com/gpu claim into memory.used_bytes/free_bytes
   help, h            Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --file string    config override path (default: "/var/lib/nvml-mock/driver/config/overrides.yaml") [$MOCK_NVML_OVERRIDES]
   --config string  config path for UUID resolution and validation (default: "/var/lib/nvml-mock/driver/config/config.yaml") [$MOCK_NVML_CONFIG]
   --help, -h       show help
```

每个变更类命令都接受 `--gpu <idx|all|uuid>`；`reset` 不带它时作用于所有设备，`status` 报告所有覆盖。全局 flag 可以放在命令的任一侧，但 `--gpu` 和命令自身的 flag 必须跟在命令之后。

```text
nvml-mock-ctl fail --gpu <t> --mode <healthy|lost|fallen_off_bus|ecc_uncorrectable> [--after-calls N] [--xid CODE]
nvml-mock-ctl temp --gpu <t> celsius
nvml-mock-ctl power --gpu <t> watts
nvml-mock-ctl fan --gpu <t> percent
nvml-mock-ctl util --gpu <t> percent
nvml-mock-ctl clocks --gpu <t> mhz
nvml-mock-ctl throttle --gpu <t> reason [reason ...]
nvml-mock-ctl pstate --gpu <t> pstate
nvml-mock-ctl nvlink-error --gpu <t> [--links a,b,c] errors_per_sec
nvml-mock-ctl sram-ecc --gpu <t> [--type correctable|parity|secded]
              [--source l2|sm|microcontroller|pcie|other] [--threshold-exceeded] count
nvml-mock-ctl fabric-health --gpu <t> condition [condition ...]
nvml-mock-ctl set --gpu <t> key.path=value [key.path=value ...]
nvml-mock-ctl status [--gpu <idx>]
nvml-mock-ctl reset [--gpu <t>]
nvml-mock-ctl watch-allocations [--socket PATH] [--interval D] [--used-fraction F]
```

### 目标选择：`--gpu <idx|all|uuid>`

- **索引**——设备索引，如 `--gpu 0`。
- **`all`**——共享的 `all` 桶；作用于所有设备，除非逐设备覆盖优先。
- **UUID**——GPU UUID，如 `--gpu GPU-12345678-...`。按 UUID 定位会用 profile 配置把 UUID 解析为索引，因此**只对在 YAML 中显式声明了 `uuid:` 的设备有效**。UUID 自动生成的设备（profile 中没有 `uuid:`）无法按 UUID 定位——请改用索引。

### `fail`——注入或清除故障

为目标设置 `failure` 块。模式有：

| 模式 | 效果 |
| --- | --- |
| `healthy` | **移除**故障覆盖（恢复设备） |
| `lost` | 受保护调用、句柄查找和事件集等待返回 `ERROR_GPU_IS_LOST` |
| `fallen_off_bus` | 与 `lost` 相同的接口面（模拟从 PCIe 总线脱落的 GPU） |
| `ecc_uncorrectable` | 设备仍可寻址；不可纠正 ECC 计数攀升 |

- `--after-calls N`——在 `N` 次受保护 NVML 调用后确定性触发（省略则在第一次受保护调用时触发）。
- `--xid CODE`——设备触发后通过 NVML 事件集呈现该 Xid 代码（任何配置了 Xid 的触发故障模式都会传递，如 `--mode ecc_uncorrectable --xid 79`），**并且**让节点 agent 在节点内核日志上公布它——见下文。

#### 内核日志上的 Xid

真实驱动通过内核 printk 抛出 Xid，而监视 Xid 的 agent 读取的是那一行而非 NVML：有的直接扫描 `/dev/kmsg`，NVSentinel 的 syslog 监视器读取 journal，只接纳内核传输的条目。因此带 `--xid` 的注入也会抵达内核日志，每个目标 GPU 一次：

```text
kernel: NVRM: Xid (PCI:0000:1a:00): 79
```

写入这行的是节点 agent，而不是本 CLI。agent 监视 `fail` 写入的同一个覆盖文档，并公布其中出现的每个 Xid，因此内核日志跟随的是模拟 GPU 状态本身，而非修改它的某一种方式——来自分配监视器的注入、手工编辑的覆盖或未来的控制面都会同样抛出这行。这也意味着公布是节点自己的职责：agent 按其通过 NVML 服务的地址命名每台设备，且只针对该节点实际运行的设备（`gpu.count`），而 CLI 只能从 profile 推断。

该地址——profile 的 `pci.bus_id`，或未设置时 mock 自己的地址——的拼写方式与驱动一致：小写且不带 function 号，因此内核行、NVML 事件和模拟的 `/sys/bus/pci` 树都命名同一台设备。恢复不会公布任何内容：`--mode healthy` 清除注入，但内核日志从不撤回 Xid。恢复后再次注入会再次公布，即便代码相同，因为那是一次新故障。

这行出现在注入后约一秒内，与 NVML 事件同时：双方都轮询覆盖文档。使用 `--after-calls N` 时，内核行会先于 NVML 故障，因为延迟故障在第 N 次受保护调用时于某个消费者进程内触发——那个计数器存在于该进程中且从不落盘，因此外部没有任何东西能等待它触发。如果消费者不能在 NVML 报告设备故障前看到 Xid，请省略 `--after-calls`。

公布是尽力而为的，绝不会让注入失败：内核日志缺失或不可写的节点会保留 NVML 一侧，agent 记录跳过。
有两个节点级要求，mock 自身都无法安排：

- agent 必须能写 `/dev/kmsg`，这需要 `nodeAgent.kernelLog.enabled=true`。它默认关闭，因为这会挂载设备并**以特权模式**运行 `node-agent`：仅挂载是不够的，即便以 root 挂载了节点上全局可写的 `/dev/kmsg`，容器设备 cgroup 仍会拒绝写入（`operation not permitted`），没有任何更小的 capability 能解除这一点。不接受特权 Pod 的集群，或没有 `/dev/kmsg` 的节点，只能保持关闭；此时 DaemonSet 设置 `MOCK_NVML_KMSG=""`，agent 保持安静，而不是对一个未给它的设备发出警告。`local/nvml-mock.values.yaml` 和 nv-sentinel 演示启用了它，因此那里注入的 Xid 会落到内核日志。在强制执行 PodSecurity 的地方，命名空间必须接纳该 Pod：`kubectl label namespace mokka pod-security.kubernetes.io/enforce=privileged`——在 `baseline` 或 `restricted` 下，DaemonSet 会被直接拒绝而不是降级；
- 对基于 journal 的消费者，journald 必须摄取内核环形缓冲区（`ReadKMsg=yes`），并把 journal 留在消费者查找的位置。Kind 的节点镜像设置 `ReadKMsg=no` 且保留易失 journal，因此 Kind 集群需要一个 `journald` drop-in，设置 `Storage=persistent` 和 `ReadKMsg=yes`。启动后应用它需要两次重启——`systemd-journald` 无法重新加载，也不会自行创建 `/var/log/journal`；`systemd-journal-flush` 才会创建该目录并把 journal 移过去。

在 Kind 集群上，内核行不是节点本地的。内核环形缓冲区没有命名空间，因此同一宿主机上的每个节点容器共享它：在一个节点上写入的 Xid 会出现在所有节点的 `dmesg` 中，也会出现在宿主机上。请只在一个节点上摄取它（NVSentinel 演示的 `run.sh` 只配置一个 worker），否则每个节点的监视器都会报告同一故障，修复器会隔离 GPU 完好的节点。真实节点各有自己的内核，因此这是共享宿主机的产物，不是消费者需要处理的。

`fail --mode healthy` 是*恢复*单个设备的方式（它从该桶中删除 `failure` 块）。各模式的完整语义参见 [mock NVML README 的故障注入小节](https://github.com/NVIDIA/k8s-test-infra/blob/main/pkg/gpu/mocknvml/README.md#failure-injection-optional)。

### `temp` / `power` / `fan` / `util` / `clocks` / `throttle` / `pstate`——固定常见指标

这些便捷命令把最常调整的读数固定为一个值。除 `throttle` 外，它们都接受单个位置参数：

| 命令 | 参数 | 固定项 |
| --- | --- | --- |
| `temp --gpu <t> <celsius>` | 0–200 °C | `nvidia-smi ... temperature.gpu` |
| `power --gpu <t> <watts>` | 瓦特（≥0） | `nvidia-smi ... power.draw`（转换为 mW） |
| `fan --gpu <t> <percent>` | 0–100 % | `nvidia-smi ... fan.speed` |
| `util --gpu <t> <percent>` | 0–100 % | `nvidia-smi ... utilization.gpu,utilization.memory` |
| `clocks --gpu <t> <mhz>` | 0–100000 MHz | `nvidia-smi ... clocks.sm,clocks.gr` |
| `throttle --gpu <t> <reason>...` | 原因名称 | `nvidia-smi ... clocks_throttle_reasons.*` |
| `pstate --gpu <t> <0-15>` | P-state 编号 | `nvidia-smi ... pstate` |

它们存在的原因是手工固定这些字段很繁琐（见下文的[动态指标遮蔽其静态对应项](#dynamic-metrics-mask-their-static-counterparts)）。无论 profile 如何配置，每条命令都能得到预期的结果：

- **`temp`** 和 **`power`** 同时写入*静态块*和零变化的动态块（`ramp_c`/`variance_c` = 0，`variance_mw` = 0），因此无论 profile 是否运行动态指标模拟器，读数都是确定的。引擎在下一个 TTL 重建模拟器，因此运行中的消费者无需重启即可收敛。`power` 接受**瓦特**（`nvidia-smi` 显示的单位）并转换为 NVML 使用的毫瓦；该值仍会被钳制在 profile 的 `[min_limit_mw, max_limit_mw]` 包络内。
- **`fan`** 设置 `fan.speed_percent` 并强制 `fan.count` 至少为 1（保留更大的基线 count）。液冷/被动散热 profile 自带 `fan.count: 0`，使 `fan.speed` 报告 `[N/A]`；强制 count 让固定转速可观察。没有动态风扇模拟器，因此这只触及静态风扇块。
- **`util`** 把 GPU **和**显存利用率固定为同一百分比。它设置静态 `utilization` 块并**禁用**动态利用率子模拟器（写入 `dynamic_metrics.utilization: null`），因此对任意百分比——包括 `0`——读数都是确定的（零变化动态块无法表达 `0`，因为模拟器把 `min==max==0` 视为"无界"）。
- **`clocks`** 固定报告的 SM 和 graphics 时钟（`clocks.sm_current` 和 `clocks.graphics_current`）。没有动态时钟模拟器，因此它直接热重载。显存/video 时钟保持其 profile 基线——要修改它们请用 `set clocks.memory_current=<mhz>`。
- **`throttle`** 设置激活的时钟降速原因。它是*权威的*：请求的原因被打开，其他所有原因被关闭，因此重复调用是替换而非累积。传入一个或多个原因名称，或单独传 `none` 以清除全部。接受的名称是 `clocks_throttle_reasons` 的字段名加上短别名：`thermal`（`hw_thermal_slowdown`）、`sw_thermal`（`sw_thermal_slowdown`）、`power`（`sw_power_cap`）、`power_brake`（`hw_power_brake_slowdown`）、`idle`（`gpu_idle`）、`app_clocks`（`applications_clocks_setting`）、`display_clocks`（`display_clocks_setting`），以及 `hw_slowdown` 和 `sync_boost`。
- **`pstate`** 把性能状态固定为 `P<n>`，其中 `n` 在 `0–15`。

`reset` 清除这些覆盖，让指标回到 profile 基线（如果 profile 是动态驱动的，则再次变化）。这些命令未覆盖的任何内容，请使用下面的 `set`。

### `nvlink-error`——在交换机链路上注入 NVLink DL 错误

在目标设备到其 NVSwitch 的链路上注入逐链路 NVLink 数据链路错误累积，使 GPU 的上行链路报告*不断攀升*的 DL 错误。位置参数是错误**速率，单位为 errors/second**（0–1e9）；`0` 表示恢复（不注入）。

```bash
# ramp 250 NVLink errors/sec on every active link of GPU 0
kubectl -n mokka exec "$POD" -- nvml-mock-ctl nvlink-error --gpu 0 250
# restrict to specific link ids
kubectl -n mokka exec "$POD" -- nvml-mock-ctl nvlink-error --gpu 0 250 --links 0,3,7
# heal
kubectl -n mokka exec "$POD" -- nvml-mock-ctl nvlink-error --gpu 0 0
```

计数从共享计数器纪元开始单调累加（与 profile 的 `nvlink.defaults.error_rate` 采用相同累积模型），因此随时间采样的消费者看到的是*上升的错误率*而不是一次性的跳变。它同时出现在逐计数器的直接 API（`nvmlDeviceGetNvLinkErrorCounter`）和 DL 错误字段值（`NVML_FI_DEV_NVLINK_ERROR_DL_{REPLAY,RECOVERY,CRC}`，字段 ID 161–163，如 `dcgmi dmon -e 161,162,163`），以及 `nvidia-smi nvlink -e` 读取的 NVLink5 错误计数器（`NVML_FI_DEV_NVLINK_COUNT_*`）中。在 615 或更新驱动上，只有 NVLink5 计数器承载它：直接 API 和字段 161–163 应答 `NOT_SUPPORTED`，与真实的 615 托盘一样（参见[615 驱动上的 NVLink 遥测](configuration.md#nvlink-telemetry-on-a-615-driver)）。注入只落在设备实际**活动**的链路上——不存在的链路不会被凭空变成错误链路——而 `--links`（逗号分隔的 ID）会进一步收窄；省略则以所有活动链路为目标（"GPU 失去交换机上行链路"故障）。

> **为什么是这个，而不是"NVSwitch 健康"注入？** DCGM 的 NVSwitch 实体健康（`DCGM_HEALTH_WATCH_NVSWITCH_*`）和 SXID 错误来自 NSCQ（`libnvidia-nscq.so`）和内核日志，**而非** NVML，因此 `libnvidia-ml.so` mock 无法驱动它们。本命令所馈送的 GPU 侧 NVLink 错误接口面，才是 DCGM 的 `DCGM_HEALTH_WATCH_NVLINK` 读取的内容（→ `DCGM_FR_NVLINK_*`），也是 NVSentinel 的 gpu-health-monitor 真正能检测并修复的交换机链路故障。

### `sram-ecc`——注入片上 SRAM ECC 错误

设置目标的 `ecc.sram` 计数器，使 GPU 像刚发生 SRAM 故障一样报告 SRAM 错误。位置参数是错误**计数**（0–1e9）；`0` 表示恢复。

```bash
# 4 uncorrectable SEC-DED errors on GPU 0's SM, past the service threshold
kubectl -n mokka exec "$POD" -- nvml-mock-ctl sram-ecc --gpu 0 \
  --type secded --source sm --threshold-exceeded 4
# correctable errors (no source attribution on real hardware)
kubectl -n mokka exec "$POD" -- nvml-mock-ctl sram-ecc --gpu 0 --type correctable 12
# heal
kubectl -n mokka exec "$POD" -- nvml-mock-ctl sram-ecc --gpu 0 0
```

| flag | 含义 |
| --- | --- |
| `--type` | `correctable`、`parity` 或 `secded`（不可纠正 SEC-DED，默认） |
| `--source` | *不可纠正*错误归属的单元；默认 `other` |
| `--threshold-exceeded` | 提升 `SRAM Threshold Exceeded`，即驱动"该 GPU 需要维护"的信号 |

计数同时落到**易失性和聚合性**两个范围：刚发生 SRAM 故障的硬件会在两者中都报告，只固定其中一个会让 GPU 的历史与现状不一致。在 `ecc.sram` 内，该命令是*权威的*——写入每个计数器和来源——因此重复调用是替换而非累积，而 `--source` 会把整个计数移到指定单元。ECC 模式保持不变，因为关闭 ECC 的 GPU 根本不报告 SRAM 计数器，注入会自我抹除。

数值出现在 `nvidia-smi -q` 的 `ECC Errors` 下（两个范围、`SRAM Threshold Exceeded` 和 `Aggregate Uncorrectable SRAM Sources`），以及 DCGM 读取的逐位置字段值中。

### `fabric-health`——降低 NVLink fabric 健康度

降低目标的 NVLink *fabric* 健康度——即 `nvidia-smi -q` 在 `Fabric` → `Health` 下渲染的块，也是故障处理控制器判断 GPU 的 fabric 连接是否可用时首先读取的内容。位置参数指明要报告的状况：

```bash
# GPU 0 上一条路由不健康
kubectl -n mokka exec "$POD" -- nvml-mock-ctl fabric-health --gpu 0 route_unhealthy
# 路由恢复期间 fabric 带宽降级
kubectl -n mokka exec "$POD" -- nvml-mock-ctl fabric-health --gpu 0 degraded_bandwidth route_recovery
# fabric manager 没有给这块 GPU 分配分区
kubectl -n mokka exec "$POD" -- nvml-mock-ctl fabric-health --gpu 0 no_partition
# 恢复
kubectl -n mokka exec "$POD" -- nvml-mock-ctl fabric-health --gpu 0 healthy
```

| 状况 | 报告为 |
| --- | --- |
| `degraded_bandwidth` | `Bandwidth: Degraded`、`Summary: Limited Capacity` |
| `route_recovery` | `Route Recovery in progress: True` |
| `route_unhealthy` | `Route Unhealthy: True` |
| `access_timeout_recovery` | `Access Timeout Recovery: True` |
| `no_partition`、`insufficient_nvlinks`、`incompatible_gpu_fw`、`invalid_location`、`incorrect_sysguid`、`incorrect_chassis_sn`、`gpu_state_invalid` | `Incorrect Configuration: <该错误配置>` |
| `healthy` | 清除所有状况 |

与 `throttle` 一样，它是*权威的*：指定的状况被打开，其他所有状况被关闭，因此重复调用是替换而非累积。单独使用 `healthy` 会清除全部——恢复不需要 `reset`，因此设备的其他覆盖会保留。

`Summary` 行跟随状况（有故障时 `Unhealthy`，仅带宽降级时 `Limited Capacity`，清除时 `Healthy`），即便 profile 固定了 `fabric.health_summary` 也是如此：注入故障会解除该固定，因为摘要保持 `Healthy` 而某状况报告故障，不是硬件可能处于的状态。静态等价配置参见[fabric 健康配置](configuration.md#fabric-health)。

### MIG 分区——使用 `nvidia-smi`

重新分区不是本 CLI 的命令之一，这是刻意的：那会成为 `nvidia-smi` 已覆盖接口的第二种非标准拼写，也是同一状态的第二个写入者。请使用 `nvidia-smi`，它对 mock 的作用与对驱动一样：

```bash
# 打开 MIG，然后把 GPU 0 切成七个 1g.5gb 分区
nvidia-smi -i 0 -mig 1
nvidia-smi mig -i 0 -cgi 1g.5gb,1g.5gb,1g.5gb,1g.5gb,1g.5gb,1g.5gb,1g.5gb -C

# 拆除一个分区，或全部拆除
nvidia-smi mig -i 0 -dci -ci 0 -gi 3
nvidia-smi mig -i 0 -dgi -gi 3
```

完整工作流——安装已分区的节点、device plugin 随后通告什么，以及运行时重新分区——见 [MIG 切分指南](guides/mig/README.md)。

与本 CLI 相关的是：每次变更都记录在它写入的同一个覆盖文档中，因此分区比发起它的 `nvidia-smi` 存活更久。`nvml-mock-ctl status` 显示记录的布局，`nvml-mock-ctl reset` 清除它。无法记录的变更会以 `NVML_ERROR_NO_PERMISSION` 失败，而不是只在一个进程中成功——这正是驱动在缺少所需权限时对被禁止调用的应答。

**这只会改变 NVML 视图。** `/dev/nvidia-caps` 和 `mig-minors` 表保持节点 agent 在 Pod 启动时部署的样子，因此重新分区创建的分区永远*不可分配*——见上面的 v1 范围说明。清除覆盖——包括 `nvml-mock-ctl reset`——会让 NVML 回到安装时的布局，使用其此前报告的实例 ID，也就是那些部署文件写入时所针对的布局。

### `set`——设置任意字段

`set` 接受一个或多个 `key.path=value` 对。路径是进入设备配置的 YAML/JSON 路径；值按 YAML 标量解析（因此数字、布尔值和字符串获得其天然类型）。示例路径：`thermal.temperature_gpu_c`、`utilization.gpu`、`ecc.mode_current`、`power.current_draw_mw`、`remapped_rows.availability_histogram.low`、`clocks_throttle_reasons.counters.sw_power_cap_us`。

#### 动态指标遮蔽其静态对应项

当 profile 启用**动态指标**时（`gpu.dynamicMetrics.enabled=true`，演示/e2e chart 就是如此），模拟器驱动 `temperature`、`power` 和 `utilization`，并**遮蔽这些字段的静态块**。在该模式下：

- `set thermal.temperature_gpu_c=<n>` **没有可见效果**——模拟器继续产生自己的读数。
- 要固定温度，请覆盖动态块并把其变化归零，使读数确定：

  ```bash
  nvml-mock-ctl set --gpu 0 \
    dynamic_metrics.temperature.base_c=85 \
    dynamic_metrics.temperature.ramp_c=0 \
    dynamic_metrics.temperature.variance_c=0
  ```

  引擎在下一个 TTL 重建模拟器，因此运行中的消费者无需重启即可看到固定值；`reset` 让它回到变化的基线。上面的 **`temp` 命令正是为你完成这一切**——除非你需要便捷命令未覆盖的字段，否则优先使用它（以及 `power`/`util` 命令）。

如果动态指标**关闭**，静态的 `thermal.temperature_gpu_c`（以及 `power.*`、`utilization.*`）是权威的并直接热重载。同样的遮蔽也适用于 `power`（`dynamic_metrics.power`）和 `utilization`（`dynamic_metrics.utilization`）。注意 `util` 的处理方式是*禁用*动态利用率子模拟器而非把其变化归零，因此即便在 `0%` 也能正确固定。

要一次修改多个字段，请在一次 `set` 调用中传入多个 `key.path=value` 对。

### `status`——查看生效的覆盖

`status` 打印当前的 `overrides.yaml`。`status --gpu <idx>` 只筛选单个设备的桶加上共享的 `all` 桶。**`status --gpu` 只接受整数索引**（不接受 `all` 或 UUID）。没有生效覆盖时打印 `no active overrides`。

### `reset`——移除覆盖

`reset --gpu <t>` 移除目标的覆盖桶。**不带** `--gpu` 的 `reset` 清除**所有内容**（等价于 `reset --gpu all`）。状态在一个 TTL 内回退到原始 profile。

### 通过 `nvidia-smi` 复位

`nvidia-smi --gpu-reset`（`-r`）执行同样的复位，因此已经使用标准 GPU 复位的修复控制器或 runbook 无需修改即可对 mock 工作：

```bash
kubectl -n mokka exec "$POD" -- nvidia-smi -r -i 0   # 等同于: reset --gpu 0
kubectl -n mokka exec "$POD" -- nvidia-smi -r        # 所有 GPU，逐块进行
```

它逐 GPU 报告（`GPU 00000000:0A:00.0 was successfully reset.`）并以 0 退出。

由于 nvidia-smi 逐设备驱动复位，有两点与 CLI 不同：

- 裸 `-r` 逐块复位每个 GPU，因此它**不会**清除共享的 `all:` 桶。用 `--gpu all` 注入的状态需要 `nvml-mock-ctl reset --gpu all`。
- 复位一个没有注入任何内容的 GPU 不会改动 `overrides.yaml`（不写、不加锁），因此在健康 GPU 上复位从不依赖覆盖文件可写。

与 CLI 不同，它还会应用 `nvidia-smi -e` 留下的待生效 ECC 模式，正如硬件上的复位所做的：GPU 回来后报告该模式为当前和待生效。`nvml-mock-ctl reset` 会连同其他覆盖一起丢弃它。

复位同样可以从被注入的消费者容器执行，也可以从 nvml-mock Pod 执行，这让修复控制器能在工作负载看到 GPU 的地方运行它。清除桶会在 flock 下重写 `overrides.yaml`，因此配置目录以可写方式 bind 挂载进被注入的容器（驱动库和 `nvidia-smi` 本身保持只读）。若以只读方式挂载，恰好在有状态要清除的 GPU 上会失败并报 `GPU Reset couldn't run`，而健康的 GPU 则通过上述无写入路径报告成功。

## 复位语义

| 操作 | 对运行时覆盖的影响 | 结果 |
| --- | --- | --- |
| `nvml-mock-ctl reset [--gpu <t>]` | 从 `overrides.yaml` 清除目标桶 | 设备在 TTL 内回退到原始 profile |
| `nvidia-smi --gpu-reset [-i <idx>]` | 清除同样的逐设备桶并应用待生效的 ECC 模式——见[通过 nvidia-smi](#reset-via-nvidia-smi) | 设备在 TTL 内回退到原始 profile |
| `nvml-mock-ctl fail --gpu <t> --mode healthy` | 只移除目标的 `failure` 块 | 该设备在 TTL 内恢复；其他覆盖保留 |
| `nvml-mock-ctl fabric-health --gpu <t> healthy` | 只清除目标的 fabric 健康状态 | 该设备的 fabric 在 TTL 内报告健康；其他覆盖保留 |
| DaemonSet Pod 重启 | 节点 agent 在启动时删除 `overrides.yaml` | **所有**覆盖被清除；回到原始 profile |
| 消费者 Pod 重启 | 无——配置覆盖在节点上，不在消费者中 | 消费者重新读取并拾取*当前*配置覆盖（**不会**重置它） |
| `helm upgrade`（profile/values 变更） | 滚动 DaemonSet Pod（配置校验和 + `RollingUpdate`），于是节点 agent 在新 Pod 上清除 `overrides.yaml` | **所有**覆盖重置为新的原始配置；只有不重建 nvml-mock Pod 的升级才会留下配置覆盖 |

## 完整示例

以下示例都假定 `$POD` 如[运行位置](#where-it-runs)所示已设置。

```bash
# 1) 在 GPU 0 上强制不可纠正 ECC，传递 Xid 79
kubectl -n mokka exec "$POD" -- nvml-mock-ctl fail --gpu 0 --mode ecc_uncorrectable --after-calls 1 --xid 79
# 从任意消费者 Pod 验证：
kubectl exec <consumer> -- nvidia-smi --query-gpu=ecc.errors.uncorrected.aggregate.total --format=csv,noheader
# 同一个 Xid 出现在节点内核日志上（在 Kind 上，在节点自身读取）：
docker exec <node> sh -c 'dmesg | grep "NVRM: Xid"'
```

```bash
# 2) 把所有 GPU 标记为丢失
kubectl -n mokka exec "$POD" -- nvml-mock-ctl fail --gpu all --mode lost
```

```bash
# 3) 把 GPU 3 报告的温度固定为 85 摄氏度（无论 profile 是否动态驱动温度都有效——temp 命令处理两者）
kubectl -n mokka exec "$POD" -- nvml-mock-ctl temp --gpu 3 85
# 从任意消费者 Pod 验证：
kubectl exec <consumer> -- nvidia-smi --id=3 --query-gpu=temperature.gpu --format=csv,noheader,nounits
```

```bash
# 3b) 把所有 GPU 的功耗固定为 350 W，把 GPU 0 的风扇转速固定为 60%
kubectl -n mokka exec "$POD" -- nvml-mock-ctl power --gpu all 350
kubectl -n mokka exec "$POD" -- nvml-mock-ctl fan --gpu 0 60
# 从任意消费者 Pod 验证：
kubectl exec <consumer> -- nvidia-smi --query-gpu=index,power.draw,fan.speed --format=csv,noheader
```

```bash
# 3c) 在 GPU 0 上固定利用率、时钟、降速原因和 P-state
kubectl -n mokka exec "$POD" -- nvml-mock-ctl util --gpu 0 90
kubectl -n mokka exec "$POD" -- nvml-mock-ctl clocks --gpu 0 1200
kubectl -n mokka exec "$POD" -- nvml-mock-ctl throttle --gpu 0 thermal
kubectl -n mokka exec "$POD" -- nvml-mock-ctl pstate --gpu 0 8
# 从任意消费者 Pod 验证：
kubectl exec <consumer> -- nvidia-smi --id=0 \
  --query-gpu=utilization.gpu,clocks.sm,clocks_throttle_reasons.hw_thermal_slowdown,pstate \
  --format=csv,noheader
# 再次清除降速原因：
kubectl -n mokka exec "$POD" -- nvml-mock-ctl throttle --gpu 0 none
```

```bash
# 3d) 在工作负载持续运行时降低 GPU 0 的 NVLink fabric 路由健康度
kubectl -n mokka exec "$POD" -- nvml-mock-ctl fabric-health --gpu 0 route_unhealthy
# 从任意消费者 Pod 验证（Fabric -> Health 块）：
kubectl exec <consumer> -- nvidia-smi --id=0 -q | grep -A 7 Health
# 恢复，保留设备的其他覆盖：
kubectl -n mokka exec "$POD" -- nvml-mock-ctl fabric-health --gpu 0 healthy
```

```bash
# 4) 一次调用在 GPU 0 上设置多个字段
kubectl -n mokka exec "$POD" -- nvml-mock-ctl set --gpu 0 \
  ecc.mode_current=disabled \
  utilization.gpu=100
```

```bash
# 5) 按 UUID 定位（要求该设备在 profile 中有显式 uuid:）
kubectl -n mokka exec "$POD" -- nvml-mock-ctl fail --gpu GPU-12345678-1234-1234-1234-123456780000 --mode fallen_off_bus
```

```bash
# 6) 查看生效的覆盖
kubectl -n mokka exec "$POD" -- nvml-mock-ctl status
# 或只看 GPU 0（仅限整数索引）：
kubectl -n mokka exec "$POD" -- nvml-mock-ctl status --gpu 0
```

```bash
# 7) 恢复一块 GPU，然后重置所有内容
kubectl -n mokka exec "$POD" -- nvml-mock-ctl fail --gpu 0 --mode healthy
kubectl -n mokka exec "$POD" -- nvml-mock-ctl reset --gpu all
```

```bash
# 8) 通过重启 Pod 完全复位（节点 agent 在启动时清除 overrides.yaml）
kubectl -n mokka delete pod "$POD"
```

## 排障

- **更改没有立即可见。** 传播受配置覆盖 TTL 限制（默认约 1 秒，`MOCK_NVML_OVERRIDES_TTL`）。等待一个 TTL 后重新检查。`nvidia-smi` 每次调用都会启动新进程，因此 TTL 过后它总能反映当前配置覆盖；长驻的进程内 NVML 客户端会在 TTL 后的下一次 getter 时拾取更改。
- **确认实际应用了什么。** 运行 `nvml-mock-ctl status`（可加 `--gpu <idx>`），或直接在节点上读取 `overrides.yaml`。
- **命令被拒绝。** 未知字段或错误的值类型会使写入失败（不应用任何内容）。请对照[配置参考](configuration.md)中的 profile schema 检查字段路径——例如应是 `thermal.temperature_gpu_c`，而不是 `temperature.gpu_temp_c`。
- **UUID 目标无法解析。** 该设备很可能使用自动生成的 UUID（profile 中没有 `uuid:`）。请改用索引定位。
- **其他节点上没有变化。** 作用域是逐节点的。请对每个节点的 DaemonSet Pod 重复该命令。
- **Xid 不在内核日志中。** 沉默通常意味着公布从未打开：它需要 `nodeAgent.kernelLog.enabled=true`，没有它时 DaemonSet 设置 `MOCK_NVML_KMSG=""`。否则节点 agent 的日志会说明原因——它负责写入那行，因此失败在那里报告：`kubectl -n mokka logs "$POD" -c node-agent | grep kernellog`。写入失败意味着容器无法访问 `/dev/kmsg`。如果 `dmesg` 显示了该行但基于 journal 的消费者看不到，说明节点的 journald 正在丢弃内核消息——参见[内核日志上的 Xid](#xid-on-the-kernel-log)。
- **身份字段没有变化。** 设备 `name`、`architecture`、`brand`、`compute_capability`、`uuid` 和 PCI `bus_id` 在构造时固化，v1 中不可热重载。请修改 profile/Helm values 并重启 Pod。
