# mokka-capture

采集真实 GPU 节点、对采集结果脱敏以便提交、从中写出 mock profile，并将其与 Mokka 提供的内容比对。它在你的机器上运行，通过 `kubectl` 访问节点。完整流程见[采集真实节点](../contributing/capturing-a-real-node.md)。

```bash
go build -o bin/mokka-capture ./cmd/mokka-capture
```

| 命令 | 作用 |
|---|---|
| [`collect`](#collect) | 通过 `kubectl` 采集节点与集群 |
| [`scrub`](#scrub) | 替换采集中的标识符，或检查是否还有残留 |
| [`profile`](#profile) | 从脱敏采集写出 mock NVML profile |
| [`diff smi`](#diff-smi) | 比对两份 `nvidia-smi -q -x` 文档 |
| [`diff nvml`](#diff-nvml) | 比对两份 NVML 录制 |
| [`dump`](#dump) | 录制加载路径上的 NVML 库；`collect` 在 driver 容器中运行它 |

## collect

```bash
mokka-capture collect [flags] driver|host|network|all <node>...
mokka-capture collect [flags] cluster
mokka-capture collect [flags] build-dump
```

| 命令 | 运行位置 | 采集内容 |
|---|---|---|
| `driver <node>...` | GPU Operator driver 容器中，或使用 `-mock` 时在替代 Pod 中 | `nvidia-smi`、NVML dump、`/proc/driver/nvidia` |
| `host <node>...` | 临时特权 Pod 中 | PCI、NUMA、设备文件、NIC、CDI 规格 |
| `network <node>...` | 同一 Pod 中 | 仅 NIC、接口和 RDMA 接口面，加上集群网络对象 |
| `cluster` | 你的机器上 | 节点标签、ResourceSlice、`ClusterPolicy`、DCGM 指标 |
| `all <node>...` | 以上全部 | 每个节点的 `driver` 和 `host`，然后 `cluster` |
| `build-dump` | 你机器上的 Docker | 用于 NVML dump 的 Linux 版 `mokka-capture`（见下） |

每条命令都在集群中运行，每个输出都写到你的机器上，在 `-out`/`<node>/` 或 `-out`/`cluster/` 下每条命令一个文件。失败的命令不会中止运行：其错误写到输出旁的 `errors.log`。采集每个节点前都会检查集群连接，因为连接中断会留下看起来像答案的空文件；运行结束时会列出所有空文件。

host Pod 名为 `capture-<node>-<pid>`，在命令完成、中断或失败时删除。它是特权 Pod，挂载节点根文件系统，并用 `apt` 安装 `pciutils`、`rdma-core` 等工具。

NVML dump 必须在 driver 容器内运行，因为那里有驱动的 `libnvidia-ml.so.1`。`build-dump` 在一个 `golang` 容器中构建 Linux 版 `mokka-capture`，写入 `-out/bin/mokka-capture-linux-<arch>`，针对的是运行 Docker 的机器架构（go-nvml 在运行时加载库，因此二进制是动态链接并原生构建的）。`collect` 把 `-dump-binary` 指定的文件复制到 driver 容器，在那里运行 `mokka-capture dump`，然后删除它。没有 `-dump-binary` 时跳过 dump。

### Mokka 节点

`-mock` 以同样方式采集 Mokka 节点，于是可以用 [`diff smi`](#diff-smi) 和 [`diff nvml`](#diff-nvml) 与真实采集比对：

```bash
mokka-capture collect -mock -dump-binary <linux binary> -out mock all <node>
mokka-capture diff smi real/<node>/smi-q.xml mock/<node>/smi-q.xml
```

有两点与真实节点不同，`-mock` 都会处理：

- **没有 driver 容器。** Mokka 的 GPU Operator 模式不运行驱动，因此 `driver` 找不到 driver Pod。使用 `-mock` 时，`collect` 会启动一个替代 Pod `capture-driver-<node>-<pid>`，在其中运行 driver 命令。它具有 host Pod 的 spec（特权、宿主机 PID 命名空间），不安装任何东西，并像 host Pod 一样被删除。
- **mock 是选择加入的。** 模拟 NVML 库、`nvidia-smi`、`/proc/driver/nvidia`、PCI sysfs 中的 GPU 和 InfiniBand 设备，只有在 Mokka 的 NRI 插件注入后才会进入容器，而插件只为通过其注解提出请求的容器注入。使用 `-mock` 时，替代 Pod 和 host Pod 都把 `nvml-mock.nvidia.com/devices`、`nvml-mock.nvidia.com/infiniband` 和 `nvml-mock.nvidia.com/imex-channels` 设为 `"true"`。没有它们，host Pod 看到的是底下的 Kind 节点，没有 GPU。

MIG 表也被部署在别处。Mokka 把 `mig-minors` 写在驱动根目录下（在替代 Pod 中是 `/opt/nvml-mock/driver/proc/driver/nvidia-caps/`），nvidia-container-toolkit 从那里读取，而 `/proc/driver/nvidia-caps` 保持不变，因此使用 `-mock` 时 `proc-*-minors.txt` 文件从那里读取。

有四个文件因 mock 之外的原因输出为空。mock 的库和 `nvidia-smi` 位于 `/opt/nvml-mock` 下，因此 `lib-listing.txt` 和 `bin-listing.txt` 什么也找不到。替代 Pod 的镜像没有 `nm` 或 `modinfo`，因此 `nvml-symbols.txt` 和 `modinfo-nvidia.txt` 也为空。其他任何空文件都是 mock 的空白。

| Flag | 默认值 | 说明 |
|---|---|---|
| `-out` | `./capture` | 输出目录 |
| `-mock` | 关闭 | 节点是 Mokka 节点：在替代 Pod 中运行 driver 命令，并让两个 Pod 都选择加入 mock |
| `-dump-binary` | 无 | 在 driver 容器中运行 `dump` 的 Linux 版 `mokka-capture` |
| `-operator-namespace` | `gpu-operator` | GPU Operator 命名空间 |
| `-driver-container` | `nvidia-driver-ctr` | driver 容器名 |
| `-toolkit-container` | `nvidia-container-toolkit-ctr` | toolkit 容器名 |
| `-toolkit-ctk` | `/usr/local/nvidia/toolkit/nvidia-ctk` | toolkit 容器中的 `nvidia-ctk` |
| `-host-pod-namespace` | `default` | host Pod 的命名空间 |
| `-host-pod-image` | `ubuntu:24.04`（按摘要固定） | host Pod 的镜像；请按摘要固定 |

`kubectl` 使用其自身配置（`KUBECONFIG`、当前 context）。

## scrub

```bash
mokka-capture scrub -out scrubbed \
  -replace worker-7=node-a -replace worker-9=node-b \
  -seed raw/worker-7/smi-L.txt -seed raw/worker-9/smi-L.txt \
  raw/worker-7 raw/worker-9 raw/cluster
mokka-capture scrub -check qx-<node>.xml
```

复制一个采集目录，把所有标识特定机器的内容替换为占位符。输出目录必须为空，且位于所有源目录之外。

### 它替换什么

| 值 | 占位符 |
|---|---|
| GPU UUID、MIG UUID | `GPU-0000000N-0000-0000-0000-00000000000N`、`MIG-…` |
| 其他 UUID（fabric 集群、Kubernetes 对象 UID） | `0000000N-0000-0000-0000-00000000000N` |
| 32 位机器 ID，以及 PCI vital product data 的单元字段 | 以 `N` 结尾的 32 位十六进制数字 |
| 序列号：`<serial>`、`<chassis_serial_number>`、`Serial Number` 行、VPD `[SN]` | `1650000000NNN` 和 `1820000000NNN` |
| PDI、fabric GUID 和其他 16 位十六进制值（下面的掩码除外） | `0x000000000000000N` |
| 冒号分隔的 GUID（`cc30:8903:00cb:be0e`） | `0000:0000:0000:000N`，与该 GUID 的 `0x` 形式使用同一个 `N` |
| 任意写法的 IPv6 地址（`fd00::12`、`::ffff:10.0.0.7`），如节点地址和 RDMA GID | `fe80:0000:…:000N` |
| MAC 地址，以及 `enx<mac>` 接口名 | `02:00:00:00:00:0N` |
| IPv4 地址 | `192.0.2.N` |
| 通过 `-replace old=new` 给出的任何内容 | `new` |

为保持数据可用而遵循的规则：

- **整个运行中每个不同的值只对应一个占位符，跨所有文件和目录。** 某块 GPU 在 `nvidia-smi`、NVML dump 和 CDI 规格中保留同一个占位符。NVML dump 在录制时从 0 给自己的 GPU 编号；脱敏器按 PCI 总线 ID 把其设备与同一目录下 `smi-q.xml` 中的 GPU 配对，并把 dump 的 UUID、序列号、机箱序列号、PDI 和 fabric GUID 重命名以匹配。
- **`-seed` 按索引顺序给 GPU 编号。** 把每个节点的 `nvidia-smi -L` 输出按节点顺序传给它。
- **序列号一旦在指明它的上下文中被发现，就在所有地方脱敏。** NVML dump 或内核日志行中的同一个值，周围没有任何表明"序列号"的内容。
- **不存在的取值保持不变。** 空值、`N/A` 和全零值不会被重写。
- **看起来像标识符但不是的保持不变。** 版本号（`99.00.16.55`）、网络掩码、未指定、回环和广播地址、低于 2^32 或低位全为 1 的 16 位十六进制值，如时钟事件和时间戳掩码（没有 ID 是这些）、内核虚拟地址（`0xffff…`）以及 GID 的链路本地前缀 `fe80::`；IPv6 回环、组播和未指定地址；以及本身不是地址的冒号分隔文本：时间、`std::vector`，或没有数字的 `a::b`。
- **文件和目录名也会被脱敏**，因此 `-replace worker-7=node-a` 会重命名 `worker-7/` 目录。
- **在其自身输出上再运行一次不会改变任何东西。**

二进制文件、`errors.log` 和共享对象（`*.so`、`*.so.*`）会被跳过并在标准错误中点名；用 `-exclude` 覆盖该列表。

### 检查结果

脱敏是尽力而为，不是保证。`-report <file>` 会写出一张包含每次替换及其原值与占位符的表格，供审计：

```bash
cut -f2 report.tsv | sort -u | while read -r v; do
  [ ${#v} -ge 8 ] && grep -rqF -- "$v" scrubbed && echo "LEAK: $v"
done
```

报告包含原值，因此绝不要提交它。模式无法识别的名字（租户、站点和用户名、内部域名）需要你用 `grep` 找出并添加为 `-replace` 值。

`-check FILE...` 读取 `nvidia-smi -q -x` 文件，若 `serial`、`chassis_serial_number`、`uuid`、`clusterUuid`、`pdi` 或 `gpu_fabric_guid` 元素仍持有既非缺失、也非[硬件采集](https://github.com/NVIDIA/k8s-test-infra/blob/main/tests/e2e/go/assertions/nvidiasmi/testdata/hardware/README.md)所用占位符之一的值，则退出 `1`。它使用与 `TestHardwareCapturesAreScrubbed` 相同的模式。

## profile

```bash
mokka-capture profile [-o file] [-base profile] [-explain] <capture node directory>
```

从已提交采集中的一个节点（如 `tests/hardware-captures/vr-nvl72/node-a`）写出 mock NVML profile，使 profile 由证据重新生成而不是手工敲出。格式与 `pkg/gpu/mocknvml/configs/mock-nvml-config-<name>.yaml` 相同；字段见[配置参考](../configuration.md)。

profile 得到的每个值都带一条注释，说明其来源，因此可以把 profile 与采集逐行核对：

```yaml
inforom:
  ecc_object: "7.16"              # GetInforomVersion(1)
  # pwr_object: left out, GetInforomVersion(2) = ERROR_NOT_SUPPORTED
memory:
  total_bytes: 300442189824       # GetMemoryInfo_v2.Total
pci:
  device_id: 0x304110DE           # GetPciInfo.PciDeviceId = 809570526
thermal:
  temperature_gpu_c: 34           # GetTemperature(0) on GPU 0, idle reading; GPUs read 34 33 33 34
  max_operating_c: 90             # tlimit.reference_c - tlimit.slowdown_c = 88 - (-2); GetTemperatureThreshold(3) reads 90 on every GPU
  tlimit:
    reference_c: 88               # GetTemperatureThreshold(1) on GPU 0
    slowdown_c: -2                # GetFieldValues(id=194) on GPU 0
```

像 `GetClockInfo(1)` 这样的名字，是 dump 中以参数 `1` 调用该 getter 的条目；`.Field` 是其应答的一个字段，`[n]` 是一个元素。当注释显示 `= value` 时，采集的原始应答与 profile 键取值不同，例如枚举（`GetBrand = 14` 对应 `"nvidia"`）。驱动不应答的键会被省略，并附一条给出返回码的注释。

### 每个值来自哪里

映射集中在一个文件 `internal/capture/profile/rules.go` 中。每行指明一个 profile 键及其来源；动词说明意见不一致的 GPU 如何处理：

| 类型 | 示例 | 写入的值 |
|---|---|---|
| 板卡属性 | 限制、最大时钟、PCI ID、ECC 模式、MIG 表 | 来自采集。所有 GPU 一致时写入 `device_defaults` 一次，否则写入每个 `devices[]` 条目。引擎会整体替换 `devices[]` 中如 `memory` 或 `power` 的块，因此该块要重复各 GPU 共有的值。在引擎只从 `device_defaults` 读取的键（如 `pcie`）上不一致的 GPU 会让生成器停止 |
| 派生值 | 散热阈值、最大 MIG 实例数 | 由多个采集值计算；注释在每块 GPU 上显示计算过程 |
| 读数 | 温度、功耗、当前时钟与 PCIe 链路、利用率，以及故障计数：ECC 和 SRAM 错误、PCIe replay、retired page、remapped row | 采集的空闲读数，绝不使用 `-base` 的：温度取 GPU 0 的，其他在 GPU 不一致时取较低的中位数，于是一块故障 GPU 既不会让生成器停止，也不会把它的错误带给 mock |
| 总量的一部分 | 空闲显存、空闲 BAR1 | 总量减去写入的保留值和已用值，使之和相符；注释在旁边给出采集的读数。逐 GPU 计算：在保留比同级更多的 GPU 上，放不下的已用读数会被降低到其可用量 |
| 标识符 | UUID、序列号、机箱序列号、fabric 集群 UUID | 来自 `-base`；否则取脱敏采集中的占位符 |
| 宿主 CPU | `cpu` | 仅当 GPU 通过 NVLink-C2C 访问 CPU、因而 CPU 属于板卡时写入：family 和每路核心数来自 `host-lscpu.txt`，`memory_gb` 来自 `host-numactl.txt` 中第一个带 CPU 的 NUMA 节点，其他值取 GPU 0 的 |
| 采集中没有 | `features`、`dynamic_metrics`、NVSwitch、NIC GUID 前缀 | 来自 `-base`；否则省略并附注释 |

NVML dump 提供大多数值。其他文件填补它无法记录的内容：

| 文件 | 提供 |
|---|---|
| `smi-q.xml` | 支持的时钟表（getter 接受缓冲区大小）、NVLink-C2C、目标温度 |
| `smi-nvlink-s.txt` | 每条链路的 NVLink 带宽 |
| `smi-mig-lgip.txt`、`smi-mig-lgipp.txt` | MIG profile 名称和 placement |
| `host-dmi.txt` | 系统产品名 |
| `host-lscpu.txt`、`host-numactl.txt` | CPU 块 |
| `host-ib-sysfs.txt`、`host-ibstat.txt` | NIC 块 |
| `rdma-sysfs.txt`、`pci-nic-functions.txt` | 每个 RDMA 设备的 PCI function，以及哪些 function 是 SR-IOV 虚拟 function |
| `host-pci-hierarchy.txt`、`host-pci-nvidia.txt` | PCIe root complex 和每块 GPU 的 NUMA 节点 |

散热值取 GPU 0 的读数，因此在 mock 中和相符。T.Limit 视图被读取，绝对阈值由其推导：

| 键 | 值 |
|---|---|
| `tlimit.reference_c` | `GetTemperatureThreshold(1)`，驱动的 SLOWDOWN 阈值 |
| `tlimit.shutdown_c`、`slowdown_c`、`max_operating_c` | T.Limit 字段 193、194 和 196 |
| `shutdown_threshold_c` | `tlimit.reference_c - tlimit.shutdown_c` |
| `slowdown_threshold_c` | `tlimit.reference_c` |
| `max_operating_c` | `tlimit.reference_c - tlimit.slowdown_c` |
| `temperature_gpu_c` | `GetTemperature(0)` |

mock 把 `GetMarginTemperature` 应答为 `tlimit.reference_c` 减去温度，`tlimit` 下的注释会把它与 GPU 0 的读数比较。在采集的托盘上两者相差一度：驱动的温度加余量并不等于它自己的参考值。这种配对是驱动的：它用 T.Limit 参考值应答 SLOWDOWN——其中字段 196 读数为 0——用硬件降速点应答 GPU_MAX。每个阈值的注释还给出驱动的 `GetTemperatureThreshold` 应答供比较。没有 T.Limit 字段的板卡保留驱动的阈值。

少数键遵循 profile 的含义而非 NVML 的编码：`nvlink.version` 是 NVLink 代际（NVML 报告其枚举，NVLink 6.0 为 `8`），`power.supported_power_mizer_modes` 命名 `GetPowerMizerMode_v1` 掩码中设置的模式（`7` 是 `adaptive`、`prefer_maximum_performance` 和 `auto`），一个 profile 只建模一种 NIC 类型，因此 NIC 块描述大多数设备所属的类型，注释列出其余类型。只有物理 function 计数：sysfs 通过 `physfn` 链接到父设备的 SR-IOV 虚拟 function，是已计数 NIC 的一部分。两种类型并列最多设备数会让生成器停止，而不是按名称选一个；其他 NIC 值并列也是如此。MIG 关闭时，采集不列出计算实例 profile；MIG 表的每行都假定有一个横跨其 GPU 实例的计算实例。

### 输出

| Flag | 默认值 | 说明 |
|---|---|---|
| `-o` | stdout | 把 profile 写入该文件，并把 MIG 表写在其旁边的 `<name>.mig.yaml`，即 mock 查找的位置。当采集没有 MIG 表时，会移除早前运行留下的 `<name>.mig.yaml`。没有 `-o` 时只写 profile |
| `-base` | 无 | 同型号的 profile，用于取标识符和仅 mock 才有的块。其 `device_defaults.pci.device_id` 与采集不符或缺失的 base 会被拒绝 |
| `-explain` | 关闭 | 打印一张表，列出每个键及其值、规则、来源和每块 GPU 的原始应答，而不是输出 profile |

输出只取决于采集和 `-base`，因此重新生成会得到相同的字节。`internal/capture/profile/testdata/` 保存从每个已提交采集生成的 profile；规则变更改变它时测试会失败，还会检查它能在 mock 引擎中加载，以及每个键都是引擎会读取的。`go test ./internal/capture/profile -update` 会重写它。

## diff smi

```bash
mokka-capture diff smi [-ignore regexp] [-max n] real.xml mock.xml
```

逐元素比对两份 `nvidia-smi -q -x` 文档：真实节点的采集与同一 profile 下 mock 的输出。每份文档都被展平为其叶子元素的 path=value 对。每个元素都按文档顺序在其同名兄弟中编号，因此第一块 GPU 是 `nvidia_smi_log[0]/gpu[0]`。报告说明：

- 真实文档中有多少路径存在于 mock 中；
- 其中多少携带相同的值；
- 只有一方拥有的路径，以及不同的值；
- 每个值未被比对的路径及其双方取值，使保真度分数确切说明它遗漏了什么。

不属于板卡属性的值不参与比对，因为不能期望被采集节点的 mock 复现它：

| 省略 | 元素 |
|---|---|
| 随负载变化的读数 | `gpu_temp`、`gpu_temp_tlimit`、`memory_temp`、`utilization`、功耗（`power_draw`、`average_power_draw`、`instant_power_draw`）、`power_state`、`performance_state`、使用中的时钟（`clocks`、`dla_cur_engine_clock`）、`graphics_volt`、`fan_speed`、显存 `used` 和 `free`、时钟事件原因及其计数器、PCIe `tx_util` 和 `rx_util`、进程 |
| 随时间变化的读数 | `timestamp`、`time_run`、`inforom_bbx_flush` |
| 标识符 | `uuid`、`serial`、`pdi`、`gpu_fabric_guid`、`clusterUuid`、`minor_number`、PCI 总线位置 |

其他一切都会比对，包括散热阈值和 T.Limit 偏移、`max_clocks` 和支持时钟。元素名整体匹配，因此 `clocks` 不会遮蔽 `max_clocks`。

省略的只是读数，不是读数的缺失。当任一方应答 `N/A`、`Not Supported`、`Insufficient Permissions`、`Requested functionality has been deprecated` 或 `Unknown Error` 时，两个应答像任何值一样比对：真实 GPU 应答 `N/A` 而 mock 读出温度算作差异，双方都是 `N/A` 算作匹配。

`-ignore` 用针对路径的正则替换该集合；`-max`（默认 `40`）限制每类列出的条目数。传 `-max 100000` 可看到所有未计入分数的省略值。

## diff nvml

```bash
mokka-capture diff nvml [-ignore regexp] [-max n] real.json mock.json
```

示例输出：

```text
calls compared:        412
return code matches:   398 of 412 (96.6%)
value matches:         301 of 340 (88.5%) (of 412 calls answered SUCCESS on both sides, excluding 72 whose value is a reading or an identifier)
only on real device:   3
only on mock device:   0

...

not compared, a reading or an identifier (88):
  device 0  GetBBXTimeData_v1                        real={"TimeRun":7058642} mock={"TimeRun":45182}
  device 0  GetMemoryInfo_v2.Free                    real=299103551488 mock=299103158272
  ...
```

报告列出返回码不同的调用、不同的值、只有一方拥有的调用，最后列出所有未计入值比率的数值：整个应答，以及被比对应答中属于读数的部分（如上面的 `GetMemoryInfo_v2.Free`）。因此从中取得的保真度分数确切说明它不覆盖什么。

每个调用都会比对返回码，因此真实和 mock GPU 必须就每个 getter 是否受支持达成一致。值只在双方都应答 `SUCCESS` 时比对；双方以相同错误应答的调用算作返回码匹配，仅此而已。不属于板卡属性的值被排除在值比率之外并单独计数：

| 省略 | 调用 |
|---|---|
| 随负载变化的读数 | 温度、功耗和能量、利用率（包括 `GetDynamicPstatesInfo`）、使用中的时钟（`GetClockInfo`、时钟 ID 为 0 的 `GetClock`、`GetCurrentClockFreqs`）、时钟事件原因、风扇转速、进程、PCIe 和 NVLink 吞吐及错误计数器、ECC、retired-page 和 remapped-row 计数 |
| 随时间变化的读数 | `GetBBXTimeData`（GPU 已运行时长）、`GetLastBBXFlushTime`、`GetSamples`、`GetViolationStatus` |
| 宿主机决定的 | `GetCpuAffinity`、`GetCpuAffinityWithinScope`：Kind 节点有自己的 CPU |
| 标识符 | UUID、序列号、PDI、平台和 fabric 信息 |

名称整体匹配：`GetClockInfo` 被排除，`GetMaxClockInfo` 参与比对。有些应答混合了读数与描述板卡的值；只有读数被排除：

- `GetMemoryInfo` 和 `GetMemoryInfo_v2`：`Free` 和 `Used`。`Total` 和驱动的 `Reserved` 参与比对。`GetBAR1MemoryInfo` 和 `GetConfComputeProtectedMemoryUsage` 同理。
- `GetThermalSettings`：每个传感器的 `CurrentTemp`。
- 每个应答的 `Version`，即调用带回的结构体版本，属于 API 而非 GPU。
- `GetFieldValues`：作为计数器、能量、功耗或温度的字段。描述板卡的字段（ECC 模式、NVLink 数量、速率和版本、功耗限制、T.Limit 偏移、能力）参与比对。字段 ID 遵循 go-nvml 的头文件，580.95 及之后的驱动使用它；更旧的驱动对 251 到 273 的字段编号不同。

`-ignore <regexp>` 替换以上全部。它针对完整调用名和应答的各部分匹配，结构体字段命名为 `Call.Field`，数组元素为 `Call[i]`，如 `GetMemoryInfo_v2.Free` 或 `GetThermalSettings(0).Sensor[0].CurrentTemp`。`-max <n>` 设置每类列出的条目数（`-max 100000` 列出所有省略值）；列出的差异只显示被比对的部分。设备按索引配对，这在双方以相同顺序暴露相同 GPU 时成立。gzip 压缩的录制（`.json.gz`）可直接读取，这也是已提交采集的存储方式。

## dump

```bash
mokka-capture dump > nvml-dump.json
```

初始化动态加载器找到的任意 `libnvidia-ml.so.1`，然后在每台设备上调用 getter（`Get*` 和 `Is*`），在进程级调用驱动、NVML 和 CUDA 版本 getter 以及设备数量。它通过对 go-nvml 设备句柄的反射找到设备 getter，因此新版 go-nvml 添加的 getter 会自动收录，无需修改工具。不会调用任何改变状态的东西。"每个 getter"受下面的列表限制：dump 无法调用的内容会列在 `skipped` 中。

- 无参数的 getter 记录为 `Name`。
- 参数全为整数或枚举的 getter，在固定值范围内调用并记录为 `Name(a,b)`：时钟类型、NVLink 索引 0 到 39、ECC 计数器种类、温度传感器等。超出枚举末尾的值应答 `ERROR_INVALID_ARGUMENT`，也会被记录，因此 dump 还显示每个范围的终点。
- `GetFieldValues` 一次探测一个字段 ID（0 到 399），仅 scope 0，记录为 `GetFieldValues(id=N)` 并带逐字段返回码。不探测逐链路字段值（每条 NVLink 的 scope ID）。驱动定义的 ID 因版本而异，因此这是驱动分支变化显现的地方。
- 参数属于其他类型（结构体、句柄、字符串）的 getter 列在 `skipped` 下且不调用，因此 dump 的覆盖率在 dump 本身中可见。

每次调用连同其 NVML 返回码（以 `ERROR_NOT_SUPPORTED` 之类的固定名称）和值一起记录。库以 `ERROR_NOT_SUPPORTED` 应答的 getter 就如实记录：在真实硬件上，这是 mock 也必须给出的应答。

写 dump 时会遮盖真实节点的标识符：GPU UUID、序列号、机箱序列号、PDI、IB 和机架 GUID 以及 fabric 集群 UUID 替换为占位符，每个不同值一个。NVML 以 C 字节数组和整数返回其中大多数，没有文本脱敏器能识别，因此遮盖依据字段或 getter 的名称而非值的形状。dump 一次写一台设备，因此后续设备上的驱动崩溃会保留之前的设备（此时文件是被截断的 JSON，已写入的设备完好）。命名标识符的新字段会被覆盖；不命名的不会，因此审阅者仍需在提交前阅读 dump。

要 dump mock，请用相同的 `-dump-binary` 运行 `collect -mock`（见 [Mokka 节点](#a-mokka-node)）。

## 退出码

| 码 | 含义 |
|---|---|
| `0` | 命令完成。`collect` 即使单个节点上的命令失败也算成功；那些在 `errors.log` 中 |
| `1` | 集群不可达、文件无法读写、输出目录非空、NVML 初始化失败、`scrub -check` 发现标识符，或 `profile` 发现 GPU 在某个没有逐设备形式的键上不一致 |
| `2` | 用法错误 |
| `130` | `collect` 被中断；它启动的 Pod 会先被删除 |
