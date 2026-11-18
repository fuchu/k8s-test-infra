# 配置参考

Mock NVML 配置选项完整指南。

## 配置方式

Mock NVML 支持两种配置方式：

| 方式 | 适用场景 | 灵活性 |
|--------|----------|-------------|
| **YAML 文件** | 完全控制，逐设备设置 | 高 |
| **环境变量** | 简单场景、CI/CD | 低 |

设置 `MOCK_NVML_CONFIG` 时，YAML 配置优先。

## 环境变量

| 变量 | 说明 | 默认值 |
|----------|-------------|---------|
| `MOCK_NVML_CONFIG` | YAML 配置文件路径 | （无）——库会依次尝试：加载它的驱动根目录下的 `config/config.yaml`、CDI 挂载路径 `/etc/nvml-mock/config.yaml`、NRI 挂载路径 `/opt/nvml-mock/driver/config/config.yaml` |
| `MOCK_MIG_PROFILES_CONFIG` | 板卡的 [MIG profile 表](mig.md#declaring-the-profile-table) 路径，该表位于独立文档中 | （无）——改为尝试 `MOCK_NVML_CONFIG` 的同级文件 |
| `MOCK_NVML_NUM_DEVICES` | 要模拟的 GPU 数量 | 8 |
| `MOCK_NVML_DRIVER_VERSION` | NVIDIA 驱动版本字符串 | 550.163.01 |
| `MOCK_NVML_DEBUG` | 开启调试日志（任意值） | （禁用） |

**示例**：

```bash
export MOCK_NVML_NUM_DEVICES=4
export MOCK_NVML_DRIVER_VERSION=550.163.01
LD_LIBRARY_PATH=. nvidia-smi
```

## YAML 配置结构

```yaml
version: "1.0"                    # 配置格式版本（必需）

system:                           # 系统级设置
  driver_version: "550.163.01"    # 必需；同时决定存在哪些 NVML 函数，并选择对应的 nvidia-smi
  nvml_version: "12.550.163.01"
  cuda_version: "12.4"
  cuda_version_major: 12
  cuda_version_minor: 4
  driver_branch: "r550_00"        # 可选；未设置时从 driver_version 推导
  product_name: "VR NVL72"        # 可选；GFD 用该 DMI 产品名作为 nvidia.com/gpu.machine 标签（默认：GPU 名称）
  absent_features: not_supported  # 可选；profile 未包含的功能如何应答（默认：legacy_defaults；见下文）
  hic:                            # 可选；`nvidia-smi -q -u` 打印的主机接口卡
    - id: 0
      firmware_version: "1.2.3.4"

device_defaults:                  # 所有设备的默认设置
  name: "NVIDIA A100-SXM4-40GB"
  # ...（见下方"设备属性"）

devices:                          # 逐设备覆盖
  - index: 0
    uuid: "GPU-12345678-1234-1234-1234-123456780000"
    pci:
      bus_id: "0000:07:00.0"
  - index: 1
    uuid: "GPU-12345678-1234-1234-1234-123456780001"
    pci:
      bus_id: "0000:0F:00.0"
  # ...

nvlink:                           # 可选，NVLink 配置
  version: 4
  links_per_gpu: 18
```

## 设备属性

### 基本标识

```yaml
device_defaults:
  name: "NVIDIA A100-SXM4-40GB"      # nvidia-smi 中显示的 GPU 名称
  brand: "nvidia"                     # nvidia、tesla、quadro、geforce
  serial: "1234567890123"             # 序列号
  board_part_number: "900-21001-0000-000"
  vbios_version: "92.00.45.00.03"
  hostname: "gpu-node-0"              # nvmlDeviceGetHostname_v1；Blackwell 或更新架构

devices:
  - index: 0
    pdi: "0x9c1f4d2e8a7b6c50"         # nvmlDeviceGetPdi；需加引号，0x 开头，最多 16 位十六进制数字
```

PDI（Per Device Identifier，逐设备标识符）是熔丝写入每块 GPU 的 64 位 ID，从 580 版驱动起 `nvidia-smi -q` 将其打印为 `PDI`。未设置 `pdi` 时，mock 会从 GPU 的 UUID 派生一个，因此各 GPU 之间不同，但在每个进程中保持一致。该值也可以逐设备设置，用来报告从真实节点采集或从外部资产清单分配的 ID。MIG 设备报告其所属 GPU 的 PDI。

该值必须以带引号的字符串形式写入。YAML 会把不带引号的 `0x…` 读作数字，配置会被拒绝并提示加引号。两台设备若报告相同 PDI（例如在 `device_defaults` 中设置了同一个 `pdi`），也会被拒绝。

### 架构

```yaml
device_defaults:
  architecture: "ampere"              # kepler、maxwell、pascal、volta、turing、
                                      # ampere、ada、hopper、blackwell、rubin
  compute_capability:
    major: 8
    minor: 0
  num_gpu_cores: 6912
```

名称匹配不区分大小写，并会去除首尾空白；`ada_lovelace` 也可作为 `ada` 的写法被接受。无法识别的名称会记入警告日志，并上报为 `NVML_DEVICE_ARCH_UNKNOWN`，这会让所有受架构门槛限制的功能失败，包括 T.Limit 温度和 GPM 支持。

### 显存

```yaml
device_defaults:
  memory:
    total_bytes: 42949672960          # 40 GiB
    reserved_bytes: 0
    free_bytes: 42949672960
    used_bytes: 0

  bar1_memory:
    total_bytes: 68719476736          # 64 GiB
    free_bytes: 68719476736
    used_bytes: 0
```

### PCI/PCIe

```yaml
device_defaults:
  pci:
    device_id: 0x20B010DE             # A100 设备 ID
    subsystem_id: 0x134F10DE
    bus_id: "0000:07:00.0"        # 通常逐设备设置

  pcie:
    max_link_gen: 4
    current_link_gen: 4
    max_link_width: 16
    current_link_width: 16
    replay_counter: 0
    tx_throughput_kbps: 0
    rx_throughput_kbps: 0
```

`lspci` 通过渲染出的 sysfs 树看到这些信息，并枚举每块 GPU，但有两行与真实硬件相比缺失。`Kernel driver in use: nvidia` 需要一个 `driver` 符号链接，而该树不会渲染它，`libmockfs.so` shim 也无法提供——它拦截的是 `open`/`stat`，而不是 `readlink`。`Kernel modules:` 依赖 libkmod，在没有 `/lib/modules` 的容器中初始化失败；这就是 `lspci -v` 上 `Unable to load libkmod resources: error -2` 的来源，在真实硬件上的容器中同样会打印。两者都只影响显示：`lspci` 仍以 0 退出，从 sysfs 读取身份的消费者拿到的是完整信息。

### 平台标识（机架位置）

节点各单板在机架中的位置，`nvidia-smi -q` 会将其渲染为 `Platform Info` 块。机架级故障关联会读取它，把 GPU 故障转换为物理位置。只有 `gb200`、`gb300` 和 `vrnvl72` 提供该信息；其他所有 profile 上整块显示 `N/A`，对于平台不上报位置的单板，这是正确的读法。

```yaml
device_defaults:
  platform:
    chassis_serial_number: "1822725100200"  # 机箱（即机架）序列号
    slot_number: 21                 # 绝对物理槽位，包含交换机托盘
    tray_index: 11                  # 仅计算托盘中的位置
    host_id: 1                      # 托盘内的操作系统域——即本节点
    peer_type: "switch_connected"    # 或 "direct_connected"
    gpu_fabric_guid: "pdi"          # 或带引号的十六进制值，逐设备设置

devices:
  - index: 0
    platform:
      module_id: 1                  # 该 GPU 在节点内的 ID
  - index: 1
    platform:
      module_id: 2
```

同一节点的各 GPU 之间只有 `module_id` 和 `gpu_fabric_guid` 不同：NVML 把模块 ID 定义为 GPU 在节点内的 ID，而位置字段描述的是节点——节点占据某个机箱某个槽位中的一个托盘。不过每个字段都可以逐设备设置，设备覆盖按字段合并，因此设备只需声明自己的 `module_id`。

`slot_number` 把交换机托盘和计算托盘一起计数，`tray_index` 只计计算托盘，因此在 NVL72 机架（18 个计算托盘、9 个交换机托盘）上，位于交换机组上方的托盘，其槽位号会大于托盘号。

`peer_type` 表示 GPU 如何到达其 NVLink 对等端：`switch_connected` 渲染为 `Switch Connected`，其他任何值（包括缺少该键）渲染为 `Direct Connected`。

与设备覆盖中的其他地方一样，零与未设置的键无法区分，因此 `module_id` 从 1 开始编号。

`gpu_fabric_guid` 是块中的 `GPU Fabric GUID` 行（`nvidia-smi --query-gpu` 中的 `platform.gpu_fabric_guid`）。NVLink fabric 管理器用它标识 GPU，因此各 GPU 之间必须不同。它有三种形式：

| 值 | 上报的 GUID | 用途 |
|-------|---------------|------------|
| 缺失 | `0x0000000000000000` | 没有工具读取 GUID 的单板 |
| `"pdi"` | GPU 自身的 PDI，通过 [`pdi`](#basic-identification) 派生或设置 | Grace-Blackwell（GB200、GB300），其采集数据在此报告 PDI |
| `"0x…"` | 按给定值上报 | 自行报告 GUID 的单板，如 B200 和 Vera Rubin |

`gb200` 和 `gb300` 在 `device_defaults` 中设置 `"pdi"`，因此即便逐节点重写 UUID 的工具也能为每块 GPU 得到不同的 GUID，无需逐一列出。固定值对应某一块 GPU，因此应放在该 GPU 的 `devices` 条目中。两台设备若报告相同 GUID，无论该值是固定的还是取自 PDI，都会被拒绝。检查会覆盖配置加载时已知的所有值；既没有 `uuid` 也没有 `pdi` 的 GPU 要到启动时才获得 PDI，因此不在检查之列。

!!! note "节点内唯一，而非跨节点唯一"
    加载同一 profile 的每个节点都会报告相同的 GPU UUID，因而也报告相同的 PDI 和 fabric GUID。对一次只读取一个节点的软件来说这没有问题。而跨机架比较 fabric GUID 的消费者（例如 NVLink 分区管理器）会看到不同节点上的 GPU 发生冲突。要模拟这样的机架，请给每个节点一份配置，设置各自的 `uuid`、`pdi` 或 `gpu_fabric_guid` 值。

声明该块还要求驱动足够新以具备相应 API：`nvmlDeviceGetPlatformInfo` 在 560 中引入，因此在 `system.driver_version` 更旧的 profile 上始终报告 `N/A`。

### 功耗

```yaml
device_defaults:
  power:
    management_supported: true
    management_mode: "enabled"
    default_limit_mw: 400000          # 400W
    enforced_limit_mw: 400000
    min_limit_mw: 100000              # 100W
    max_limit_mw: 400000              # 400W
    current_draw_mw: 72000            # 72W（空闲）
    power_state: "P0"
```

### 散热

```yaml
device_defaults:
  thermal:
    temperature_gpu_c: 33             # 当前温度
    temperature_memory_c: 31
    shutdown_threshold_c: 92
    slowdown_threshold_c: 87
    max_operating_c: 83
    target_temperature_c: 83
```

这些阈值的上报方式取决于 `device_defaults.architecture`，与真实硬件一致。Ada 及之后架构把它们暴露为有符号的 T.Limit 偏移（`nvmlDeviceGetMarginTemperature` 以及 `NVML_FI_DEV_TEMPERATURE_*_TLIMIT` 字段 ID），`nvidia-smi -q` 将其渲染为 "T.Limit" 行。Ada 之前的架构对两者都报告 `NVML_ERROR_NOT_SUPPORTED`，于是 `nvidia-smi -q` 回退到绝对的 `GPU Shutdown / Slowdown / Max Operating Temp` 行。

默认情况下，T.Limit 视图由绝对阈值推导，以 slowdown 阈值为基准。真实驱动分别报告两种视图，目前二者并不一致，因此从硬件采集的 profile 可以直接声明 T.Limit 视图。上面的绝对阈值仍用于应答 `nvmlDeviceGetTemperatureThreshold`：

```yaml
  thermal:
    tlimit:
      reference_c: 88      # 余量读作 reference_c - 当前温度
      shutdown_c: -5       # NVML_FI_DEV_TEMPERATURE_SHUTDOWN_TLIMIT
      slowdown_c: -2       # NVML_FI_DEV_TEMPERATURE_SLOWDOWN_TLIMIT
      max_operating_c: 0   # NVML_FI_DEV_TEMPERATURE_GPU_MAX_TLIMIT
```

块中省略的距离会按没有该块时的规则推导。

### 风扇

```yaml
device_defaults:
  fan:
    count: 0                          # 0 = 液冷（A100 SXM）
    speed_percent: "N/A"
    target_speed_percent: "N/A"
```

`count` 控制整个功能族。未声明专用风扇的设备对所有风扇查询都应答 `NVML_ERROR_NOT_SUPPORTED`，`nvidia-smi` 将其渲染为 `[N/A]`——这是 Mokka 自带所有 profile 的行为，与仓库中每一次硬件采集一致，包括被动散热的 L40S 和 T4 PCIe 板卡。

声明了风扇的设备会上报块中其余内容。fan 块既可作为默认值也可逐设备设置，因此一个 profile 可以混合带风扇的板卡和无风扇的模组：

```yaml
devices:
  - index: 0
    fan:
      count: 2
      speed_percent: "40"             # 风扇当前转速百分比
      target_speed_percent: "45"      # 驱动曲线目标值
      max_speed_rpm: 5000             # 100% 占空比时的转速表读数
      min_speed_percent: 30           # 手动写入可请求的下限
      max_speed_percent: 90           # 手动写入可请求的上限
```

`speed_percent` 和 `target_speed_percent` 是字符串，因为 `"N/A"` 是合法值：风扇受控但不上报转速的板卡。任何无法解析的值也按同样方式处理。

`max_speed_rpm` 会按当前占空比缩放以应答 `nvmlDeviceGetFanSpeedRPM`，因此上面的风扇读数为 2000 RPM。不设置它表示风扇受控但未测量——转速读数被拒绝，而百分比仍可应答，这是真实硬件上存在的分裂情况。

`min_speed_percent` 和 `max_speed_percent` 是 `nvmlDeviceGetMinMaxFanSpeed` 通告的窗口，也是 `nvmlDeviceSetFanSpeed_v2` 强制的边界；超出范围的写入会被拒绝并返回 `NVML_ERROR_INVALID_ARGUMENT`。未设置上限时按 100 处理。

#### 风扇控制

每个风扇要么遵循驱动的温度曲线，要么遵循消费者请求的占空比。`controls` 用于为这类状态设定初值，适用于启动后应处于手动控制的板卡：

```yaml
fan:
  count: 2
  controls:
    - fan: 0
      policy: manual                  # 或 "temperature"（默认）
      speed_percent: 75
```

`fan` 是 NVML 逐风扇入口点寻址所用的索引，因此指向设备不存在的风扇的条目会被忽略，而不会错误地应用到其他风扇。

四个 setter——`nvmlDeviceSetFanSpeed_v2`、`nvmlDeviceSetDefaultFanSpeed_v2`、`nvmlDeviceSetFanControlPolicy`，以及通过它进行的策略切换——都会写入该状态。与功耗管理限值一样，它们记录在运行时覆盖文档中，而不是记录在执行写入的进程里，因此后续进程能读回该写入，并保持到复位（`nvidia-smi -r` 或 `nvml-mock-ctl reset`）。如果 mock 无处记录写入，就会拒绝该写入并返回 `NVML_ERROR_NO_PERMISSION`，而不是报告一个无人能观察到的成功。

将风扇保持在某个占空比意味着手动控制，因为仍在由驱动按温度控制的风扇不会停在设定位置。`nvmlDeviceSetDefaultFanSpeed_v2` 会让风扇回到温度曲线并同时放弃该占空比。通过 `nvmlDeviceSetFanControlPolicy` 切换到手动控制不会凭空产生占空比：风扇保持原有转速运行，直到有请求要求改变。

其中只有一部分能通过 `nvidia-smi` 看到。575 和 580 版本只调用 `nvmlDeviceGetFanSpeed`，它报告板卡的第一个风扇，因此 `--query-gpu=fan.speed` 和 `<fan_speed>` 元素就是它们暴露的全部风扇接口：对风扇 0 的手动写入会出现在那里，本节其他内容都不会出现。其余入口点服务于直接持有 NVML 的消费者。

`nvml-mock-ctl fan` 是一条独立的便捷路径。它会固定 `speed_percent` 并强制 `count` 至少为 1，从而在默认无风扇的 profile 上也能观察到转速。

#### S 级单元

`nvmlUnitGetFanSpeedInfo` 报告 S 级机箱的风扇。单元是 S870/S1070 代的管理处理器；Mokka 建模的任何设备都没有该单元，现代驱动看到的 GPU 也没有，因此单元枚举为空。`nvidia-smi -q -u` 会报告节点没有单元，而不是失败：

```console
$ nvidia-smi -q -u
HIC Info                                  : N/A
Attached Units                            : 0
```

没有针对单元的配置，也没有单元查询可以接受的句柄。

### 时钟

```yaml
device_defaults:
  clocks:
    graphics_current: 210             # MHz
    graphics_max: 1410
    graphics_app: 1410
    graphics_app_default: 1410
    sm_current: 210
    sm_max: 1410
    memory_current: 1215
    memory_max: 1215
    memory_app: 1215
    memory_app_default: 1215
    video_current: 585
    video_max: 1290
```

`graphics_max` 应答 `nvidia-smi -q` 的两行而非一行：Max Clocks 和 Max Customer Boost Clocks。NVML 将二者区分开——后者是 OEM 定义的上限，合法地可以低于 boost 最大值——但 `tests/e2e/go/assertions/nvidiasmi/testdata/hardware` 中采集的每块真实板卡都报告两者相等，因此 mock 用同一个键解析两者，而不是额外携带一个需要 profile 手工保持同步的键。没有 `graphics_max` 的 profile 两行都报告 `N/A`。

只有 graphics 域应答 customer-boost 行；`nvidia-smi` 在其下不输出其他子项。`nvmlDeviceGetClock` 从上面的键解析完整的"时钟类型 × 时钟 ID"矩阵，对于没有键承载的组合返回 `NVML_ERROR_NOT_SUPPORTED`——即 SM 和 video 域的应用时钟，以及非 graphics 域的 OEM 上限。

#### 支持的时钟

```yaml
device_defaults:
  supported_clocks:
    memory_clocks:                    # 从高到低，NVML 的返回顺序
      - freq_mhz: 2619
        graphics_clocks: [1980, 1965, 1950, 1935]   # 已截断；参见 h100.yaml
      - freq_mhz: 1593
        graphics_clocks: [1980, 1965, 1950, 1935]
```

该表是 `nvidia-smi -q -d SUPPORTED_CLOCKS` 和 `--query-supported-clocks` 打印的内容，也是 `nvidia-smi -ac` 接受的参数对集合。自带 profile 携带其在 `tests/e2e/go/assertions/nvidiasmi/testdata/hardware` 中对应板卡上报的完整表，并有单元测试保证两者相等。没有该块的 profile 会把该节报告为 `N/A`。

mock 只建模一个 P-state：设备的 `performance_state`。该表提供其时钟包络（`nvmlDeviceGetMinMaxClockOfPState`、`nvmlDeviceGetPerformanceModes`），其他任何 P-state 都报告为不支持。

#### 运行时调整时钟

调整命令与 [`nvidia-smi -pl`](#static-held-until-the-config-changes) 一样写入运行时覆盖文档。节点上的每个进程都能看到结果，并保持到对应的复位命令、`nvidia-smi -r` 或 `nvml-mock-ctl reset`。

| 命令 | 对读数的影响 |
|---------|------------------------|
| `-ac <mem,graphics>` / `-rac` | 设置应用时钟。`supported_clocks` 中不存在的参数对会被拒绝；`-rac` 恢复 `*_app_default` 参数对 |
| `-lgc <min,max>` / `-rgc` | 把当前 graphics 和 SM 时钟限制在该范围内 |
| `-lmc <min,max>` / `-rmc` | 把当前显存时钟限制在该范围内 |

锁定的范围存为 `clocks.locked_graphics` 和 `clocks.locked_memory`（`min_mhz`、`max_mhz`），因此 profile 也可以让设备以锁定状态启动。即便如此，`-rgc` 和 `-rmc` 也会解锁 GPU，因为未锁定是驱动默认状态。完全复位（`nvidia-smi -r` 或 `nvml-mock-ctl reset`）则会丢弃所有运行时写入，于是 profile 中的锁定重新生效。NVML 的符号边界行为与 `nvml.h` 一致：两端都是无限制表示解锁，符号边界与 MHz 边界混用会被拒绝，TDP 边界不支持。

时钟偏移（`nvmlDeviceSetClockOffsets` 和 `*ClkVfOffset` 系列调用）是可选开启的，因为没有采集到的板卡报告它们：

```yaml
device_defaults:
  clocks:
    offsets:
      graphics: {min_mhz: -200, max_mhz: 300}
      memory: {min_mhz: 0, max_mhz: 1000}
    adaptive_clocking: "enabled"      # 或 "disabled"；缺失 = 不支持
```

偏移写入会进行范围检查并可被读回，但不会改变上报的时钟。自动 boost（`--auto-boost-default`）被拒绝为不支持，与 Pascal 及更新板卡一律报告 `N/A` 一致。

```yaml
device_defaults:
  clocks_throttle_reasons:
    gpu_idle: true
    applications_clocks_setting: false
    sw_power_cap: false
    hw_slowdown: false
    hw_thermal_slowdown: false
    hw_power_brake_slowdown: false
    sync_boost: false
    sw_thermal_slowdown: false
    display_clocks_setting: false

    # 每种原因已经让 GPU 损失的累计时间，单位为微秒。
    # 上面的标志回答"现在是否受限"，这些回答"已持续多久"，
    # 诊断慢工作负载靠的是后者。对于从未受限的 GPU 可省略该块：
    # 此时每个计数器都报告 0 us，这是一个有效答案，而 #678 之前的 N/A 不是。
    counters:
      sw_power_cap_us: 39595
      sync_boost_us: 0
      sw_thermal_slowdown_us: 0
      hw_thermal_slowdown_us: 0
      hw_power_brake_slowdown_us: 0
```

逐设备的 `clocks_throttle_reasons` 块会整体替换 `device_defaults` 中的块，而不是像 `memory`、`power`、`thermal` 和 `clocks` 那样合并。因此，如果 `devices:` 条目只带 `counters`，就会丢失继承来的标志，需要重新声明仍然适用的项——这也是 `tests/mocknvml/util-test-config.yaml` 中每个逐设备块都重复 `gpu_idle: true` 的原因。

这些计数器在 `nvidia-smi -q` 中出现在 `Clocks Event Reasons Counters` 下，并可通过 `nvmlDeviceGetFieldValues`（DCGM 使用的路径）和 `nvmlDeviceGetViolationStatus` 读取。只要对应标志置位，设备就会在配置的基线之上继续累加时间。累加是逐进程的，与动态指标模拟器一样：像 dcgm-exporter 这样的长驻消费者会看到计数器增长，而每次 `nvidia-smi` 调用只在自身生命周期内累加，因此基本报告基线值。

正是这种逐进程范围决定了标志和计数器必须配置成一致，而不能指望它们天然一致。运行时被置入受限状态的 GPU 没有历史记录：`nvidia-smi` 会把原因渲染为 `Active`，旁边计数器只有一两微秒，那是 `nvidia-smi` 进程的寿命而非受限状态的寿命。要描述已经受限一段时间的 GPU，就把计数器与标志一起预置——两者写入同一个覆盖文档，因此第二条命令会保留第一条：

```bash
nvml-mock-ctl throttle --gpu 3 thermal
nvml-mock-ctl set --gpu 3 clocks_throttle_reasons.counters.hw_thermal_slowdown_us=39595
```

### 性能

```yaml
device_defaults:
  performance_state: "P0"             # P0-P15

  utilization:
    gpu: 0                            # 0-100%
    memory: 0
    encoder: 0
    decoder: 0
    jpeg: 0
    ofa: 0
```

### ECC

```yaml
device_defaults:
  ecc:
    mode_current: "enabled"
    mode_pending: "enabled"
    default_mode: "enabled"
    errors:
      volatile:
        single_bit:
          device_memory: 0
          l1_cache: 0
          l2_cache: 0
          register_file: 0
          texture_memory: 0
          total: 0
        double_bit:
          device_memory: 0
          total: 0
      aggregate:
        single_bit:
          total: 0
        double_bit:
          total: 0
```

#### SRAM ECC

硬件把片上 SRAM 错误与上面的 DRAM 计数器分开统计，并通过独立 API 报告，因此它是一个同级块，而不是 `errors` 下的另一个内存位置。省略该块时报告零，这是健康且启用 ECC 的 GPU 的表现；关闭 ECC 时计数器报告为不支持（`N/A`），与真实硬件一致。

```yaml
device_defaults:
  ecc:
    sram:
      volatile:                         # 驱动重新加载时复位
        correctable: 0
        uncorrectable_parity: 0
        uncorrectable_secded: 0
      aggregate:                        # 持久化在 InfoROM 中
        correctable: 0
        uncorrectable_parity: 0
        uncorrectable_secded: 0
      # 哪个单元报告了聚合不可纠正错误；nvidia-smi
      # 将其渲染为 "Aggregate Uncorrectable SRAM Sources"。
      uncorrectable_sources:
        l2: 0
        sm: 0
        microcontroller: 0
        pcie: 0
        other: 0
      # 累积错误是否超过驱动阈值——
      # 这是 GPU 需要维护而不仅仅是计数增长的信号。
      threshold_exceeded: false
```

可在运行时用 `nvml-mock-ctl sram-ecc` 注入这些错误（参见 [nvml-mock-ctl.md](nvml-mock-ctl.md)）。

`nvidia-smi` 如何渲染这些计数器取决于 profile 的 `architecture`，与真实硬件一致：Ampere 及之后架构把不可纠正计数拆为 `SRAM Uncorrectable Parity` 和 `SRAM Uncorrectable SEC-DED`，并打印来源明细和阈值标志；Ampere 之前（`t4`）只打印一行合并的 `SRAM Uncorrectable`，其余省略。两种情况下配置相同——只是呈现方式不同。

#### 内存修复状态

健康 agent 依据三个查询在复位、行重映射和更换之间决策：修复状态、不可修复内存标志，以及 SRAM 唯一不可纠正错误计数。它们由上面的状态和 [`remapped_rows`](#remapped-rows) 推导，因此注入一次故障会让三者一起变化：

| 查询 | 报告内容 | 架构 |
|---|---|---|
| 修复状态 | 设置 `remapped_rows.pending` 或注入 `ecc_uncorrectable` 故障时，通道修复处于待处理；TPC 修复从不待处理 | Ampere 及之后 |
| 不可修复内存标志 | 设置 `remapped_rows.failure_occurred`、注入 `ecc_uncorrectable` 故障或设置 `ecc.sram.threshold_exceeded` 时置位 | Hopper 及之后 |
| SRAM 唯一不可纠正错误 | 每个非零 `ecc.sram.uncorrectable_sources` 单元一条，带单元名和计数 | Blackwell 及之后，且 ECC 开启 |

更早的架构对每个查询都报告不支持。

同样的 NVML 调用也会写入，结果持久化在运行时覆盖文档中，因此独立进程能看到：

- `nvidia-smi -e` 只设置 `mode_pending`；GPU 通过 [`nvidia-smi -r`](nvml-mock-ctl.md#reset-via-nvidia-smi) 复位后 `mode_current` 才会采用它，与硬件一致。
- 清除易失性或聚合错误计数器会清零该范围的 DRAM 和 SRAM 计数器。清除聚合范围还会清零由计数器推导出的 `uncorrectable_sources` 和 `threshold_exceeded`。
- 设置 DRAM 加密模式会写入 `dram_encryption.mode_pending`。

```yaml
device_defaults:
  ecc:
    dram_encryption:        # Blackwell 及之后；更早芯片报告不支持
      mode_current: "disabled"
      mode_pending: "disabled"
```

只要 profile 有 `inforom` 块，InfoROM 校验查询就会成功。其配置校验和是所声明版本的 CRC-32，因此对同一 profile 稳定，对不同 InfoROM 版本的 profile 则不同。

### 重映射行

`availability_histogram` 表示还有多少内存库留有备用行可用于重映射未来的故障。行重映射是 Ampere 及之后的功能，因此省略该块（如 `t4`）会把直方图报告为不支持。

```yaml
device_defaults:
  remapped_rows:
    correctable: 0
    uncorrectable: 0
    pending: false
    failure_occurred: false
    availability_histogram:
      max: 640                          # 备用容量满的内存库
      high: 0
      partial: 0
      low: 0
      none: 0                           # 已无剩余容量的内存库
```

### 显示

```yaml
device_defaults:
  display:
    mode: "disabled"
    active: "disabled"
```

### 模式

```yaml
device_defaults:
  persistence_mode: "enabled"
  compute_mode: "default"             # default、exclusive_thread、
                                      # prohibited、exclusive_process
  addressing_mode: "hmm"              # hmm、ats、none
  egm: false                          # 扩展 GPU 内存能力
```

`addressing_mode` 是 `nvidia-smi -q` 中的 `Addressing Mode` 行：GPU 如何访问宿主用 `malloc` 或 `mmap` 分配的内存。它与单板一样属于主机：`hmm` 需要内核支持，`ats` 需要 Grace CPU 通过 NVLink-C2C 做转换。同一块 GPU 在同一驱动下可能报告其中任一种，因此每个内置 profile 都声明其采集节点上报的模式。`devices[]` 条目可为单块 GPU 设置该值。不设置键或拼错都会渲染为 `N/A`。该行需要 `580` 或更新的驱动：更旧的 profile 上不存在该查询，与真实硬件一致。

`egm` 设置 `nvmlDeviceGetCapabilities` 的 EGM 位，即 `nvidia-smi -q` 中 `Capabilities` 下的 `EGM` 行。宿主机内核模块开启扩展 GPU 内存，因此默认关闭，除非 profile 设置；`vrnvl72` 按其采集托盘的情况设置了该位。`devices[]` 条目可为单块 GPU 开启它。

### profile 未包含的功能

`system.absent_features` 设置 profile 未包含的功能如何应答：

- `not_supported`：返回 `NOT_SUPPORTED`，`nvidia-smi` 渲染为 `N/A`，与没有该功能的真实 GPU 一致。对照真实硬件采集数据校验过的 profile 会设置它；`vrnvl72` 就是如此。
- `legacy_defaults`（默认）：mock 的内置值，仿佛该功能存在。其他 profile 写于采集数据可得之前，依赖这些值，因此在各自与采集数据校验之前保持原样。未知值也会保留这些值，并记录一条警告。

| 缺失的 profile 键 | 应答 `NOT_SUPPORTED` 的 NVML 调用 |
|---|---|
| `gpu_operation_mode` | `nvmlDeviceGetGpuOperationMode` |
| `retired_pages` | `nvmlDeviceGetRetiredPages`、`nvmlDeviceGetRetiredPages_v2`、`nvmlDeviceGetRetiredPagesPendingStatus` |
| `ecc.dram_encryption` | `nvmlDeviceGetDramEncryptionMode`、`nvmlDeviceSetDramEncryptionMode` |
| `thermal.temperature_memory_c` | 显存最大值对应的 `nvmlDeviceGetTemperatureThreshold` |
| `thermal.target_temperature_c` | 声学阈值（目标温度及其范围）对应的 `nvmlDeviceGetTemperatureThreshold` |
| 无 | GPS 阈值对应的 `nvmlDeviceGetTemperatureThreshold` |

关闭 `accounting` 时，`nvmlDeviceGetAccountingPids` 不列出进程，`nvmlDeviceGetAccountingStats` 应答 `NOT_FOUND`，而不是 `NOT_SUPPORTED`。

### 615 驱动上的 NVLink 遥测

`615` 或更新的驱动只通过字段值报告 NVLink 计数器。与真实 615 托盘一样，以下对每条链路都应答 `NOT_SUPPORTED`：

- `nvmlDeviceGetNvLinkErrorCounter` 和 `nvmlDeviceGetNvLinkUtilizationCounter`
- 逐链路速率字段（`NVML_FI_DEV_NVLINK_SPEED_MBPS_L0`–`L11` 和 `_COMMON`，ID 84–90 与 132–137）
- 数据链路错误字段（`NVML_FI_DEV_NVLINK_ERROR_DL_REPLAY`、`_RECOVERY`、`_CRC`，ID 161–163）

NVLink5 计数字段（`NVML_FI_DEV_NVLINK_COUNT_*`）仍会应答。`nvidia-smi nvlink -e` 读取的正是它们，[NVLink 错误注入](#nvlink-error-injection-per-device)也体现在这里。

与上面的 `system.absent_features` 不同，这取决于驱动版本而非 profile：存在哪些调用由驱动决定。任何 profile 在 `615` 驱动上都会这样应答，包括 `driverVersion: 615.71.09` 的 `gb300`；更旧的驱动仍会应答这些 getter 和字段。

### 排空、移除与排除

这些模拟的是驱动处理 GPU 的方式，而非 GPU 本身：一是在驱动保持加载时把 GPU 退出服务，二是驱动被告知永不管理的 GPU。NVML 都通过 PCI 地址（域、总线和设备号；功能号被忽略）寻址它们，不匹配的 GPU 返回 `NVML_ERROR_INVALID_ARGUMENT`。

**排除**属于配置。标记为 `excluded` 的设备（等同于把它列入内核模块的 `NVreg_ExcludedGpus`）永远不会被枚举：其余 GPU 的编号会像它不存在一样。它只由 `nvmlGetExcludedDeviceCount` 和 `nvmlGetExcludedDeviceInfoByIndex` 报告，附带 PCI 地址和 UUID，这正是 `nvidia-smi -B` 列出的内容。该标记可以逐设备设置，通常在[逐设备覆盖](#per-device-overrides)中完成，也可以在运行时用 `nvml-mock-ctl set --gpu 3 excluded=true` 完成：

```yaml
devices:
  - index: 3
    excluded: true
```

**排空与移除**属于运行时状态，由消费者通过 NVML 设置，并作为 `draining` 和 `removed` 字段记录在[运行时覆盖文档](nvml-mock-ctl.md)中。与功耗上限一样，它们对节点上每个进程都可见，并保持到复位清除（`nvml-mock-ctl reset`）。随附的 `nvidia-smi` 没有 drain 子命令，因此若不想写 NVML 客户端就要排空 GPU，可直接设置该字段：`nvml-mock-ctl set --gpu 1 draining=true`。

| 调用 | 效果 | 以下情况拒绝并返回 `NVML_ERROR_IN_USE` |
|------|--------|---------------------------------------|
| `nvmlDeviceModifyDrainState` | 标记 GPU 正在排空，或清除标记。之后初始化的进程不会枚举正在排空的 GPU；已经看到它的进程保留它 | 对处于 persistence 模式的 GPU 执行排空——先运行 `nvidia-smi -pm 0` |
| `nvmlDeviceQueryDrainState` | 报告 GPU 是否正在排空。 | — |
| `nvmlDeviceRemoveGpu` | 把 GPU 从驱动中摘除。所有进程都停止枚举它，包括调用方，其余 GPU 重新编号。此后调用方持有的该 GPU、其 MIG 设备和实例的句柄都返回 `NVML_ERROR_INVALID_ARGUMENT` | GPU 处于 persistence 模式，或设备配置了 `processes` |
| `nvmlDeviceDiscoverGpus` | 重新挂载已移除的 GPU：域、总线和设备号都为 0 时挂载全部，否则挂载该地址对应的 GPU。与 NVML 一样，`busId` 字符串不被读取。重新发现的 GPU 不再处于排空状态 | — |

!!! note
    已经在运行的进程只有在下次调用 `nvmlInit` 时才能看到其他进程所做的移除或重新发现。做出更改的进程会立即看到。

### MIG

```yaml
device_defaults:
  mig:
    mode_current: "disabled"
    mode_pending: "disabled"
    max_gpu_instances: 7
```

这三个键就是 `device_defaults.mig` 的全部，也是 profile 描述芯片能力所需的全部。单板的分区表、该表所在位置以及声明布局的两种方式有专门一页：[MIG](mig.md)。

### 虚拟化

```yaml
device_defaults:
  virtualization:
    mode: "none"                    # none、passthrough、vgpu、host_vgpu
    host_vgpu_mode: "N/A"           # 仅 host_vgpu：sriov、non_sriov
```

`vgpu` 和 `host_vgpu` 模拟 NVIDIA vGPU 的两侧，需要一个 `virtualization.vgpu` 块；其字段、默认值和校验在专门一页：[vGPU](vgpu.md)。

### InfoROM

```yaml
device_defaults:
  inforom:
    image_version: "G500.0212.00.02"
    oem_object: "2.0"
    ecc_object: "6.16"
    pwr_object: "1.0"
```

### 计费（Accounting）

逐进程计费是 `nvidia-smi --query-accounted-apps` 和 DCGM 作业统计读取的内容。`mode: "enabled"` 时，缓冲区为每条仍在运行的已配置[进程](#processes)保留一条记录，再加上你为已退出进程声明的 `terminated` 记录：

```yaml
device_defaults:
  accounting:
    mode: "enabled"                   # enabled、disabled
    buffer_size: 4000                 # 保留的记录数；最旧的先丢弃
    terminated:                       # 最旧的在前
      - pid: 3131
        gpu_util: 62                  # 生命周期内 GPU 利用率 %
        mem_util: 31                  # 生命周期内显存利用率 %
        max_memory_mib: 1024
        time_ms: 120000               # 运行时长
```

运行中进程的记录从它的 `processes` 条目取利用率和显存，并报告运行时间为 0，这与 NVML 在进程退出前的做法一致。每个进程都算作在驱动加载时启动。如果同一 PID 出现两次，后出现的记录胜出，这与 NVML 在第一个进程退出后复用 PID 的行为一致。

关闭计费时，以及在启用 MIG 的 GPU 上，计费查询返回 `NVML_ERROR_NOT_SUPPORTED`。没有记录的 PID 返回 `NVML_ERROR_NOT_FOUND`。`nvmlDeviceGetAccountingStats_v2` 要求配置的驱动为 `600` 或更新。

`nvidia-smi -am`（`nvmlDeviceSetAccountingMode`）和 `nvidia-smi -caa`（`nvmlDeviceClearAccountingPids`）与功耗上限一样记录在[运行时覆盖文档](nvml-mock-ctl.md)中。清除会丢弃终止记录，保留运行中记录。关闭计费也会丢弃终止记录，因此再次开启时它们不会回来。在启用 MIG 的 GPU 上，两个调用都会被拒绝。

### 编码器/解码器

```yaml
device_defaults:
  encoder_stats:
    session_count: 0
    average_fps: 0
    average_latency_us: 0

  fbc_stats:
    session_count: 0
    average_fps: 0
    average_latency_us: 0
```

### 进程

`processes` 驱动运行中进程查询、`nvmlDeviceGetProcessUtilization`、`nvmlDeviceGetProcessesUtilizationInfo`，以及[计费](#accounting)中的运行中记录。利用率字段（`sm_util`/`mem_util`/`enc_util`/`dec_util`，均为百分比）对所有类型的进程都会上报。

`type` 决定哪个运行中进程查询会报告该进程：

| `type` | 报告者 |
|--------|-------------|
| `C`（或省略） | `nvmlDeviceGetComputeRunningProcesses` |
| `G` | `nvmlDeviceGetGraphicsRunningProcesses` |
| `M` | 仅 `nvmlDeviceGetMPSComputeRunningProcesses` |

`M` 进程是 MPS 客户端。在 MPS（Multi-Process Service）下，客户端的工作运行在 MPS server 的上下文中，因此 NVML 把 server 列入计算进程，而客户端只出现在 MPS 列表中。要建模 MPS 节点，把 server 声明为名为 `nvidia-cuda-mps-server` 的 `C` 进程，其客户端声明为 `M`。`nvmlDeviceGetRunningProcessDetailList` 返回与其模式对应的查询相同的进程，`NVML_PROCESS_MODE_ALL` 则返回全部。mock 不会运行 MPS 守护进程。

它们也会出现在 `nvidia-smi` 中：默认表格的 Processes 框、`-q` 和 `--query-compute-apps`。这里有两点需要注意——`nvidia-smi` 给每一行都标注 `M+C+G`，无论 `type` 如何，因为它枚举进程所用的调用不带类型字段；`nvidia-smi pmon` 使用 mock 未实现的另一个入口点，因此不列出任何内容。

```yaml
device_defaults:
  processes:
    - pid: 1234
      type: "C"                       # C=计算，G=图形
      name: "python"
      used_memory_mib: 1024
      sm_util: 75                     # SM（计算）利用率 %
      mem_util: 40                    # 显存带宽利用率 %
    - pid: 5678
      type: "G"
      name: "ffmpeg"
      used_memory_mib: 512
      enc_util: 60                    # 编码器利用率 %
      dec_util: 30                    # 解码器利用率 %
    - pid: 9012
      type: "M"                       # MPS 客户端
      name: "train.py"
      used_memory_mib: 4096
```

## 逐设备覆盖

覆盖指定设备的任意属性：

```yaml
devices:
  - index: 0
    uuid: "GPU-12345678-1234-1234-1234-123456780000"
    minor_number: 0
    pci:
      bus_id: "0000:07:00.0"
    # 仅覆盖该设备的散热设置
    thermal:
      temperature_gpu_c: 35

  - index: 1
    uuid: "GPU-12345678-1234-1234-1234-123456780001"
    minor_number: 1
    pci:
      bus_id: "0000:0F:00.0"
    thermal:
      temperature_gpu_c: 37
```

`index` 是 NVML 枚举位置，也是覆盖项匹配的键。`minor_number` 是 `/dev/nvidia<N>` 中的 `N`，由 agent 部署，两个 CDI 规格都指向它。它默认为 index——驱动按 PCI 枚举顺序探测时产生的编号——因此只有在描述两者不同的节点时才设置。

次设备号是设备索引的一个排列：重映射一台设备意味着重映射占用了它原来次设备号的那台设备。即使包含默认值，让两台设备使用同一次设备号的 profile 也会被拒绝，超出 0-254 的次设备号同样会被拒绝（255 是 `nvidiactl`）。

## NVLink 配置

```yaml
nvlink:
  version: 4
  links_per_gpu: 18
  bandwidth_per_link_mbps: 26562
  c2c_enabled: false
  multi_node: false
  nvle_enabled: false                # 见"带宽模式、低功耗与 NVLE"
  bw_mode:                           # 见"带宽模式、低功耗与 NVLE"
    scope: system
  links:
    - link: 0
      state: "active"
      remote_device_type: "GPU"
      remote_pci_bus_id: "0000:0F:00.0"
  switches:                          # 见下方"NVSwitch"
    - bdf: "0000:05:00.0"
      uuid: "NVSwitch-H100-0000-0000-0000-000000000000"
      device_id: 0x22A310DE
```

`c2c_enabled` 是节点级设置，驱动所有挂载 GPU 的 `nvidia-smi -q` 中的 `GPU C2C Mode` 行：`true` 报告 `Enabled`，而 `false` 或缺少该键报告 `N/A`——绝不会是 `Disabled`，因为 NVML 应答 `NVML_ERROR_NOT_SUPPORTED`，这对于没有 NVLink-C2C 连接到宿主 CPU 的单板是正确读法。只有 `gb200`、`gb300` 和 `vrnvl72` 启用它。这是驱动该行的唯一键：`device_defaults.features.nvlink_c2c` 只是描述性元数据，不会被读取。

`multi_node` 标记 NVLink 域跨越节点的单板，例如机架级 NVLink 系统，与 HGX H100 或 B200 这类停留在单基板上的 NVLink fabric 相对。自带 profile 中，`gb200`、`gb300` 和 `vrnvl72` 设置了它。在这样的节点上，节点 agent 会部署 DRA 驱动的 ComputeDomain 插件读取的 IMEX 能力接口：一个替代的 `/proc/devices` 条目、`fabric-imex-mgmt` 能力文件和 `channel0` 设备节点——真实节点的驱动只自行创建这一个通道。chart 的 `imex.deviceNumbers` 值设置它们的设备主设备号；节点内核已分配给其他驱动的主设备号会被替换为 384 到 511 之间的空闲号。无法部署该接口的节点仍保持就绪，只是没有 IMEX，节点 agent 会记录原因。

### 带宽模式、低功耗与 NVLE

三个可选的节点级键位于此处。`h100`、`b200`、`gb200` 和 `gb300` 声明 `bw_mode`，`gb300` 还声明 `firmware`；省略的键采用下面的默认值：

```yaml
nvlink:
  nvle_enabled: true                 # 默认 false；`nvidia-smi nvlink --info` 中的 NVLE: 行
  bw_mode:                           # 默认：none；所有带宽模式调用应答 NOT_SUPPORTED
    scope: device                    # system 或 device
    supported: [0, 1, 2, 3, 4]       # 默认：全部五种
    mode: 0                          # 默认：支持的最佳模式
  firmware:                          # 默认：none，渲染为 N/A
    mse: "15:3:0"
```

NVLink 降带宽模式由两个独立的 NVML 函数族提供，真实板卡会对其中一个应答、拒绝另一个。`bw_mode.scope` 选择应答的那一个；另一个返回 `NVML_ERROR_NOT_SUPPORTED`。既不指向其中之一的 `scope` 会收到警告，并视为缺少该块。

| | `scope: system` | `scope: device` |
|---|---|---|
| NVML | `nvmlSystemGetNvlinkBwMode`、`nvmlSystemSetNvlinkBwMode` | `nvmlDeviceGetNvlinkBwMode`、`nvmlDeviceSetNvlinkBwMode`、`nvmlDeviceGetNvlinkSupportedBwModes` |
| `nvidia-smi` | `nvlink -gBwMode`、`nvlink -sBwMode` | 没有参数能到达它——仅供 NVML 调用方使用 |
| 自带 profile | `h100` | `b200`、`gb200`、`gb300`、`vrnvl72` |

无论哪种情况，`supported` 都是 setter 接受的集合，上报的模式是 setter 最后记录的值，否则为 `mode`，否则为支持的最佳模式。

!!! note "为新 profile 选择 scope"
    NVML 参考文档把节点级的函数族记录为 Hopper 或更新，把逐设备的函数族记录为 Blackwell 或更新。然而 `nvidia-smi` 手册把 `-gBwMode` 和 `-sBwMode` 标注为在 Blackwell 上已弃用，真实 GB300 对 `-gBwMode` 的应答是 `Getting nvlink bandwidth mode is not supported`。因此 Hopper 板卡选 `system`，Blackwell 板卡选 `device`，Hopper 之前的板卡省略该块。在 `device` profile 上，该模式只有 NVML 调用方能看到。

`nvidia-smi` 手册记录了五种模式名称及其作用，但没有说明 NVML 调用对某个名称携带哪个索引。下面的映射是通过 mock 镜像随附的 `nvidia-smi` 确定的，集合止于四，因为第六个值会超出其名称表的索引范围。

| 值 | 名称 | 效果 |
|---|---|---|
| 0 | `FULL` | 所有链路以最大带宽运行 |
| 1 | `OFF` | 不使用 NVLink；节点内 GPU 间流量改走 PCIe |
| 2 | `MIN` | 最低速率 |
| 3 | `HALF` | 约为 `FULL` 的一半 |
| 4 | `3QUARTER` | 约为 `FULL` 的 75% |

大于 4 的 `supported` 值会被接受，因为索引由驱动定义，后续版本可能定义更多。在 `scope: system` 下它没有可打印的名称，profile 会收到警告；在 `scope: device` 下没有 `nvidia-smi` 参数显示它，因此不会被显示（`vrnvl72` 列出该托盘的 0、4、……36）。超出有效 `supported` 集合的 `mode` 会被忽略并警告，改为报告支持的最佳模式——`nvmlDeviceSetNvlinkBwMode` 对该值应答 `NVML_ERROR_INVALID_ARGUMENT`，如果照单全收，getter 就会给出一个 setter 拒绝的模式。

`mode` 只是初始值。两个 setter 都写入配置覆盖文档而不是进程内存，因此一个进程设置的模式会被节点上所有其他进程读到，并比写入者活得更久。节点级写入还会清除逐设备记录，否则后者优先级更高。可用 [`nvml-mock-ctl status`](nvml-mock-ctl.md#status--inspect-active-overrides) 查看任一者。

`firmware` 填充 `nvlink --info` 在 `NVLE:` 下打印的 `Firmware Version:` 行。每个值都是 `major:minor:subMinor`——即 `nvidia-smi` 打印的格式——因此从真实硬件上读到的一行可以不加改动地搬进 profile：

```yaml
nvlink:
  firmware:
    mse: "15:3:0"
    netir: "36:2014:4784"
    netir_uphy: "7:0:0"
    netir_cln: "0:10:0"
    netir_dln: "0:10:0"
```

这五个键是 NVML 定义了 ucode 类型的微控制器（从 `NVML_NVLINK_FIRMWARE_UCODE_TYPE_MSE` 到 `..._NETIR_DLN`），无论 profile 按什么顺序列出，`nvidia-smi` 都按该顺序渲染。第六个无法表达，这是有意为之：`nvidia-smi` 遇到无法标注的 ucode 类型会放弃整个 `--info` 渲染。省略某个键就会省掉对应行；省略整个块则表格为空，`nvidia-smi` 渲染为 `Firmware Version: N/A`。不是三段无符号、以冒号分隔的数字的值会被丢弃并警告，而不会导致加载失败。`gb300` 自带上面的集合，读取自一块 GB300 NVL；没有其他 profile 声明它。

其余两个接口面遵循 profile 的声明，因此没有相应硬件的单板会拒绝而不是应答：

| 接口面 | 要求 |
|---|---|
| `nvlink -gLowPwrInfo`、`-sLowPwrThres` | 具有活动 NVLink 的 GPU。阈值为 `1`-`1023`，或 `0xFFFFFFFF` 恢复驱动默认值 |
| `nvlink --info` 的 `NVLE:` 与 `Firmware Version:` 行 | 驱动 580 或更新，且单板具有 NVLink（`links_per_gpu`）。低于 580 时函数本身不存在，因此 `nvidia-smi` 报告 `Function Not Found`，而不是设备不支持。没有架构下限：Hopper 板卡同样打印这些行，只是 NVLE 关闭且没有固件 |

### NVSwitch

`switches:` 列出节点上的 NVSwitch。它们被建模为 NVLink 远端端点：声明它们（且 `links_per_gpu > 0`、没有显式的逐设备 `links:`）会让引擎把每块 GPU 的链路分散到各交换机上，这正是 `nvidia-smi topo -m` 在每对 GPU 之间渲染 `NV<links_per_gpu>` 的原因。NVML 没有提供逐交换机 API，因此 `nvmlUnit*` 机箱调用保持为桩实现，与真实 DGX/HGX 节点一样。

```yaml
nvlink:
  switches:
    - bdf: "0000:05:00.0"
      uuid: "NVSwitch-H100-0000-0000-0000-000000000000"
      device_id: 0x22A310DE         # 10de:22a3 GH100 [H100 NVSwitch]
```

`device_id` 是打包后的 `(device << 16) | vendor` 字，与 GPU 的 `device_defaults.pci.device_id` 采用相同形式。它是可选的，决定节点能否通过 PCIe 看到该交换机：

| `device_id` | 含义 | profile |
|---|---|---|
| 设置 | 节点 PCIe 总线上的基板交换机。渲染为 PCI 树中的桥接设备，因此 `lspci` 会把它与 GPU 一并列出——参见 [PCI 总线上的 NVSwitch](helm-chart.md#nvswitches-on-the-pci-bus) | `a100`、`h100` |
| 未设置 | 仅作为 NVLink 端点。节点从不枚举它，这是机架级交换机托盘的呈现方式 | `gb200`、`gb300` |

### NVLink 错误注入（逐设备）

可以给设备在交换机链路上的 NVLink DL 错误率一个递增设置，使其上行链路报告不断攀升的 replay/recovery/CRC 错误——这是 DCGM 的 `DCGM_HEALTH_WATCH_NVLINK` 读取的 GPU 侧信号（→ `DCGM_FR_NVLINK_*`）。可在设备覆盖中设置，也可在运行时用 [`nvml-mock-ctl nvlink-error`](nvml-mock-ctl.md#nvlink-error--inject-nvlink-dl-errors-on-switch-links) 设置：

```yaml
devices:
  - index: 0
    nvlink_error:
      rate: 250        # 在健康基线之上每秒增加的错误数
      links: [0, 1, 2] # 可选；省略则在所有活动链路上注入
```

计数从共享计数器纪元开始单调累加（与 `nvlink.defaults.error_rate` 模型相同）；`rate: 0`（或省略该块）表示健康。在 `615` 或更新驱动上，它只出现在 NVLink5 计数字段中（参见 [615 驱动上的 NVLink 遥测](#nvlink-telemetry-on-a-615-driver)）。这刻意*不是*"NVSwitch 实体健康"开关——DCGM 的 NVSwitch/SXID 健康来自 NSCQ/内核日志，无法由 `libnvidia-ml.so` mock 驱动。

## Fabric 健康

`fabric:` 描述 GPU 的 NVLink fabric 连接——身份字段（`cluster_uuid`、`clique_id`、`state`）加上 `nvidia-smi -q` 在 `Fabric` → `Health` 下渲染的健康状态。不带健康键的 `fabric:` 块表示 fabric 健康，因此自带的 Grace-Blackwell profile 无需任何配置就报告 `Summary: Healthy` 和 `Bandwidth: Full`：

```yaml
device_defaults:
  fabric:
    cluster_uuid: "00000000-0000-0000-0000-000000000001"
    clique_id: 0
    state: "auto"      # 把注册与模拟 fabricmanager 就绪状态关联
```

只需点名某个状况即可让它故障。其他所有状况保持健康，因此消费者看到的正是你配置的那个故障：

```yaml
devices:
  - index: 3
    fabric:
      health:
        route_unhealthy: true
```

| `health:` 键 | `nvidia-smi -q` 行 | 取值 |
| ------------- | ------------------- | ------ |
| `degraded_bandwidth` | `Bandwidth` | `false` → `Full`，`true` → `Degraded` |
| `route_recovery` | `Route Recovery in progress` | `false` / `true` |
| `route_unhealthy` | `Route Unhealthy` | `false` / `true` |
| `access_timeout_recovery` | `Access Timeout Recovery` | `false` / `true` |
| `incorrect_configuration` | `Incorrect Configuration` | `none`（默认）、`no_partition`、`insufficient_nvlinks`、`incompatible_gpu_fw`、`invalid_location`、`incorrect_sysguid`、`incorrect_chassis_sn`、`gpu_state_invalid` |

### 健康摘要

`Summary` 行由上述状况推导，因此注入故障会让它变化：全部正常 → `Healthy`，仅带宽降级 → `Limited Capacity`，其他任何故障 → `Unhealthy`。当你需要一个与状况不一致的摘要时（例如驱动报告了故障却未分类），可用 `health_summary` 固定它：

```yaml
fabric:
  health_summary: "limited_capacity"   # healthy | unhealthy | limited_capacity
                                       # | not_supported | auto（默认）
```

`not_supported` 复现 #677 之前的渲染：`nvidia-smi` 把未上报的摘要当作"没有健康数据"，整个 `Health` 块打印 `N/A`。

### 原始 `health_mask`

`health_mask` 直接设置 NVML v2/v3 健康位掩码，用于 `health:` 键无法表达的编码。它整体替换推导出的掩码，摘要再由它推导：

```yaml
fabric:
  health_mask: 0x1aa    # `health:` 全部正常时产生的值
```

显式的 `health_mask: 0` 表示"驱动完全没有上报健康状态"，会把整块渲染为 `N/A`——这就是自带 profile 不再设置它的原因。

### `Partition Assigned`

mock 把该字段报告为 `NOT_SUPPORTED`，这是健康机架中真实 GB300 托盘的表现（驱动不应答它）。该行是否出现取决于 `nvidia-smi` 的构建：580.173.02 打印 `Partition Assigned : N/A`，而 mock 镜像为 580 profile 运行的 580.65.06 二进制没有该标签，对所有掩码值都省略该行。

 Fabric 健康也可以在运行时用 [`nvml-mock-ctl fabric-health`](nvml-mock-ctl.md#fabric-health--degrade-nvlink-fabric-health) 降级和恢复，无需重启消费者。

## 可用的 GPU profile

每种支持的 GPU 型号都提供了独立配置文件：

| 文件 | GPU 型号 | 显存 | 架构 |
|------|-----------|--------|--------------|
| `pkg/gpu/mocknvml/configs/mock-nvml-config-gb300.yaml` | NVIDIA GB300 NVL | 278 GiB | Blackwell Ultra |
| `pkg/gpu/mocknvml/configs/mock-nvml-config-gb200.yaml` | NVIDIA GB200 NVL | 186 GiB | Blackwell |
| `pkg/gpu/mocknvml/configs/mock-nvml-config-b200.yaml` | NVIDIA B200 | 180 GiB | Blackwell |
| `pkg/gpu/mocknvml/configs/mock-nvml-config-h100.yaml` | NVIDIA H100 80GB HBM3 | 80 GiB | Hopper |
| `pkg/gpu/mocknvml/configs/mock-nvml-config-a100.yaml` | NVIDIA A100-SXM4-40GB | 40 GiB | Ampere |
| `pkg/gpu/mocknvml/configs/mock-nvml-config-l40s.yaml` | NVIDIA L40S | 48 GiB | Ada Lovelace |
| `pkg/gpu/mocknvml/configs/mock-nvml-config-t4.yaml` | NVIDIA T4 | 16 GiB | Turing |
| `pkg/gpu/mocknvml/configs/mock-nvml-config-vrnvl72.yaml` | NVIDIA Graphics Device（Vera Rubin） | 280 GiB | Rubin |

每个文件都包含完整配置，并配置好所有设备：基板 profile 为 8 块 GPU，机架级 profile（`gb200`、`gb300`、`vrnvl72`）为 4 块——模拟一个计算托盘，即真实节点上报的单元。

## 集成取值

通过 Helm 部署时，还有额外取值控制与外部项目的集成：

| 键 | 默认值 | 说明 |
|-----|---------|-------------|
| `integrations.fakeGpuOperator.enabled` | `false` | 为每个 profile 创建名为 `gpu-profile-<profile>` 的 ConfigMap，键为 `profile.yaml`，格式遵循 fake-gpu-operator 加载器所读取的格式 |
| `integrations.fakeGpuOperator.targetNamespace` | `""`（release 所在命名空间） | profile ConfigMap 所在命名空间。设为 FGO 的 release 命名空间以便 FGO 找到它们；要求 FGO 的 `builtinProfiles.enabled=false` |
| `integrations.fakeGpuOperator.profileLabels` | `run.ai/gpu-profile: "true"` | profile ConfigMap 的额外标签。契约标签始终会生成 |

设置细节参见 [fake-gpu-operator 指南](guides/runai-fgo/README.md)。

## 指标保真度

mock 上报的每个值都来自配置、时钟驱动的模拟器或固定的推导——绝不来自真实 GPU 工作，因为 mock 从不执行 kernel。本节说明各自属于哪一种，以便基于 nvml-mock 构建的仪表盘能以正确的预期来解读。
### 模拟值——调用之间会变化

驱动的依据是**经过时间，而非工作负载**：这些数值在完全空闲的节点上也会变化，在负载下也不会变化得更快。

**所有自带 profile 的利用率都是实时的。** 每个 profile 都有自己的 `device_defaults.dynamic_metrics.utilization` 块——`pattern: steady`，GPU 10–45%，显存 5–25%——因此 `nvmlDeviceGetUtilizationRates` 以及 `DCGM_FI_DEV_GPU_UTIL` 在默认安装下绝不会报告恒定的 0。下限刻意高于 0，使"利用率有上报"成为确定性断言而非偶发断言。由于每个 `DCGM_FI_PROF_*` 活动指标都是 `utilization.gpu` 的固定比例（见下文的*刻意固定*），整个 profiling 接口面也因此非零。

温度和功耗保持**可选开启**：设置 `gpu.dynamicMetrics.enabled=true`（Helm）或在配置中添加 `dynamic_metrics.temperature` / `dynamic_metrics.power`。注意启用 Helm 覆盖会*整体替换* profile 的 `dynamic_metrics` 块为 chart 基线，其利用率默认是 `pattern: burst`，范围 0–100——该区间的空闲阶段确实会报告接近 0。如需下限，请固定 `gpu.dynamicMetrics.utilization.*`。

| 指标 | 配置 | 行为 |
|--------|--------|-----------|
| `temperature.gpu` | `dynamic_metrics.temperature` | `base_c`，加上在 `ramp_period_sec` 内以 `ramp_c` 为界的正弦爬升，再加上 ±`variance_c` 的均匀噪声；已知时钳制到散热关机阈值 |
| `power.draw` | `dynamic_metrics.power` | `base_mw` ± `variance_mw`；设置时钳制到 `min_limit_mw` / `max_limit_mw` |
| `utilization.gpu`、`utilization.memory` | `dynamic_metrics.utilization` | 在配置区间内随机，形状由 `pattern`（`idle` / `busy` / `burst` / `steady`）决定 |
| NVLink 吞吐计数器 | — | 从与进程无关的纪元（`MOCK_NVML_EPOCH`，否则 `/proc/stat` 的 btime）确定性累加，因此在不同 `nvidia-smi` 运行之间单调增长 |

### 静态值——保持到配置改变

其余一切通过有效配置解析：设备显存（`memory.total_bytes` / `free_bytes` / `used_bytes` / `reserved_bytes`）、ECC 模式与计数器、时钟、风扇转速、性能状态、功耗限制、`processes` 以及故障注入。每项保持其配置值，直到 profile 改变或 `nvml-mock-ctl` 写入运行时覆盖，后者在一个覆盖 TTL 内生效——参见 [nvml-mock-ctl](nvml-mock-ctl.md)。

其中三项还接受消费者写入：功耗管理限值、persistence 模式和时钟。时钟调整（`nvidia-smi -ac`、`-lgc`、`-lmc`）与下面的功耗限值一样工作；参见[运行时调整时钟](#tuning-clocks-at-runtime)。

功耗管理限值（`nvidia-smi -pl`，单位毫瓦，包含 `min_limit_mw` / `max_limit_mw`；超出边界的限值会被拒绝）记录在运行时覆盖文档中，因此它的行为与其建模的驱动级写入一致：节点上每个进程都会读到新限值，包括之后启动的进程，并保持到复位清除（`nvidia-smi -r` 或 `nvml-mock-ctl reset`）。设置限值会同时改变功耗管理限值和强制限值，但不会改变 `default_limit_mw`。如果 mock 无处记录写入——没有配置，或覆盖文件不可写——则拒绝该限值并返回 `NVML_ERROR_NO_PERMISSION`，而不是报告一个无人能观察到的成功。

persistence 模式（`nvidia-smi -pm`）以同样方式记录，遵循同样的复位和同样的 `NVML_ERROR_NO_PERMISSION` 拒绝。它的影响超出自身查询：处于 persistence 模式的 GPU 拒绝排空——参见[排空、移除与排除](#drain-removal-and-exclusion)。

### 工作负载功耗 profile

`power.workload_power_profiles` 让设备选择加入 Blackwell 工作负载功耗 profile 功能，`nvidia-smi power-profiles` 可读写该功能。默认不设置，此时设备像所有 Blackwell 之前的板卡一样拒绝该功能。`requested` 用于预置消费者看到的集合，直到某个 setter 覆盖它：

```yaml
power:
  workload_power_profiles:
    supported:
      - id: 0                  # NVML_POWER_PROFILE_MAX_P，渲染为 "Max-P"
        priority: 10           # 数值越小，仲裁优先级越高
        conflicts: [1, 5]      # 不能与这些 profile 同时生效
      - id: 6                  # "LLM Inference"
        priority: 40
    requested: []              # 要请求的 profile ID
```

`id` 是 `NVML_POWER_PROFILE_*` 索引（0-254），同时作为该 profile 在 NVML 255 位掩码中的位位置，因此 ID 必须唯一。它也是 nvidia-smi 渲染的名称：`-l` 列出支持的集合，`-ld` 附加优先级和冲突，`-gr` 与 `-ge` 报告请求的集合和生效的集合，`-sr` / `-cr` 向请求集合添加和移除。

请求与生效不同，因为允许请求互斥的 profile：生效集合是仲裁后保留下来的部分，会丢弃与同样被请求的更高优先级 profile 冲突的那些。设备未通告的请求 ID 在配置中会被忽略，但消费者在运行时请求它时会被拒绝并返回 `NVML_ERROR_INVALID_ARGUMENT`。

自带 profile 的 `requested` 为空，因为每一次真实的 GB200、GB300 和 B200 采集都报告没有请求或生效的 profile——这正是 `nvidia-smi -q -x` 在其 `<power_profiles>` 块中渲染为 `N/A` 的情况。

写入从落地那一刻起优先级高于 `requested`，并且与 `nvidia-smi -pl` 一样记录在运行时覆盖文档而非写入进程中——因此后面的 `nvidia-smi` 会读回它，并保持到复位：

```console
$ nvidia-smi power-profiles -sr 2 -i 0
Successfully set the requested profiles.
$ nvidia-smi power-profiles -gr -i 0
2. Compute
```

在单次调用中只有 `-ge` 能反映写入，因为 nvidia-smi 先计算 `-sr` 和 `-cr`，再计算 `-ge`，但在 `-gr` 之后：

```console
$ nvidia-smi power-profiles -sr 0,2 -cr 0 -ge -i 0
Successfully set the requested profiles.
Successfully cleared the requested profiles.
2. Compute
```

由于请求会持久化，想让板卡回到配置状态的消费者必须清除自己的请求，或复位设备。还有两个 nvidia-smi 行为值得了解，两者都不是 mock 造成的：它会在调用 NVML 之前拒绝板卡未通告的 ID；对于逗号分隔的列表，它只对访问的第一块 GPU 完整应用，其余 GPU 只收到第一个 profile。

有两个维度决定该功能是否应答，且失败方式不同。未声明 `workload_power_profiles` 的设备把该功能报告为不支持；`system.driver_version` 早于 570 的设备不导出相应符号，因此 nvidia-smi 找不到该函数。自带 profile 中只有 `gb200` 和 `gb300` 同时满足两者；其他 profile 运行的驱动足够新，但未声明任何 profile。

### 功耗平滑

功耗平滑是 Blackwell 的一项功能，把 GPU 功耗维持在某个下限之上，使负载波动不会传导回机房供电。它渲染为 `nvidia-smi -q` 的 "Power Smoothing" 块，由 `nvidia-smi power-smoothing` 驱动。Blackwell 及之后设备默认支持它，在自带 profile 中即 `b200`、`gb200` 和 `gb300`；更早架构报告为 `N/A`。它还要求 `system.driver_version` 为 570 或更新：更旧的驱动不报告任何功耗平滑字段，也没有 setter。`power.smoothing` 用于预置其状态：

```yaml
power:
  smoothing:
    supported: true                    # 覆盖架构默认值
    state: "enabled"                   # 默认 "disabled"
    active_profile: 1                  # 预设 0-4，默认 0
    profiles:
      1:
        percent_tmp_floor: 65.5        # 下限占功耗上限的百分比，0-100
        ramp_up_rate_mw_per_s: 30000   # 渲染为 30.00 W/s
        ramp_down_rate_mw_per_s: 15000
        ramp_down_hysteresis_ms: 250
```

设备有五个预设 profile，编号 0 到 4。配置未列出的预设读作全零。功耗上限（"Current TMP"）就是强制功耗限制，因此跟随 `nvidia-smi -pl`。启用平滑时，下限为该上限乘以当前预设的 `percent_tmp_floor`；禁用时下限为 0 W。平滑不会改变模拟的功耗值。

`-s`、`-spp`、`-ptf`、`-rur`、`-rdr` 和 `-rdh` 与 `nvidia-smi -pl` 一样记录在运行时覆盖文档中，因此后面的 `nvidia-smi` 会读回它们，并保持到复位：

```console
$ nvidia-smi power-smoothing -i 0 -s 1
Successfully Enabled Power Smoothing
$ nvidia-smi power-smoothing -i 0 -p 2 -rur 30000
Successfully updated Ramp Up Rate for Preset Profile ID 2
$ nvidia-smi power-smoothing -i 0 -spp 2
Successfully Activated Preset Profile ID: 2
$ nvidia-smi -i 0 -q | grep -A3 "Current Profile"
        Current Profile
            % TMP Floor                   : 0.00%
            Ramp Up Rate                  : 30.00 W/s
            Ramp Down Rate                : 0.00 W/s
```

超出 0-4 的预设、超出 0-100 % 的下限、负的爬升速率或非整数的迟滞都会被拒绝并返回 `NVML_ERROR_INVALID_ARGUMENT`。管理员覆盖未建模，读作 `Not Set`；权限级别读作 `N/A`，立即降功耗为 `Disabled`，剩余硬件寿命为 100 %。

!!! note
    来自真实 B200、GB200 和 GB300 节点的采集会把整块渲染为 `Insufficient Permissions`。mock 不建模 NVML 权限，因此始终渲染完整块。

与工作负载功耗 profile 一样，该功能也要求 `system.driver_version` 为 570 或更新；更旧的驱动不导出这些符号。

### PowerMizer 模式与电源

`power.power_mizer_mode` 设置 `nvmlDeviceGetPowerMizerMode_v1` 报告的模式：`adaptive`、`prefer_maximum_performance`、`auto`（默认）或 `prefer_consistent_performance`。Maxwell 之前两个调用都拒绝。`power.supported_power_mizer_modes` 以相同名称列出设备接受的模式；它会成为 `supportedPowerMizerModes` 掩码。省略、为空或未指向任何 NVML 模式时，四种模式都支持。NVML 未定义的名称会被忽略并警告。`vrnvl72` profile 列出 `adaptive`、`prefer_maximum_performance` 和 `auto`，对应真实托盘上报的掩码 7。

```yaml
power:
  power_mizer_mode: "adaptive"
  supported_power_mizer_modes: ["adaptive", "prefer_maximum_performance", "auto"]
```

`nvmlDeviceSetPowerMizerMode_v1` 把模式写入运行时覆盖文档，因此其他进程能读回。它拒绝设备未列出的模式，返回 `NVML_ERROR_INVALID_ARGUMENT`，未定义的模式同理。两个入口点都要求 `system.driver_version` 为 580 或更新。nvidia-smi 580 不显示该模式，因此只有 NVML 消费者能观察到。

`nvmlDeviceGetPowerSource` 在所有设备上报告交流供电。

### 刻意固定

**Profiling 指标（`DCGM_FI_PROF_*`）。** DCGM 在 Hopper 及之后通过 NVML GPM API 读取这些指标。mock 把每个活动指标都推导为 `utilization.gpu` 的固定比例：

| GPM 指标 | `utilization.gpu` 的比例 |
|------------|-------------------------------|
| SM 利用率、图形利用率 | 1.00 |
| SM 占用率 | 0.75 |
| 任意 tensor 活动 | 0.50 |
| HMMA（FP16 tensor） | 0.40 |
| FP16 | 0.35 |
| FP32 | 0.25 |
| 整数 | 0.10 |
| FP64、DMMA、IMMA | 0.05 |
| DFMA | 0.02 |

DRAM 带宽活动镜像 `utilization.memory`，PCIe / NVLink 吞吐来自计数器快照。这些比例近似于以 tensor 为主的 Hopper 训练步骤的形状；它们**刻意与实际执行脱钩**。SM 占用率和 tensor core 活动是 mock 不运行的 kernel 的属性，因此从配置利用率以外的任何东西推导它们都是捏造而非模拟。它们按设计保持固定，不是待填补的空白。

启用 MIG 时，GPU 实例可通过其 ID（`nvmlGpmMigSampleGet`）或某个 MIG 设备句柄（`nvmlGpmSampleGet`）采样；两者应答相同。mock 把板卡负载均匀分摊到各分片，因此每个 GPU 实例都报告板卡的活动百分比，而 PCIe 和 NVLink 吞吐则是其分片占板卡的比例：七分片板卡上的 `1g` 实例报告板卡 PCIe 速率的七分之一。

**机密计算受保护内存。** `nvidia-smi -q` 的 `Conf Compute Protected Memory Usage` 块在所有 profile 上 Total、Used 和 Free 都报告 `0 MiB`，没有配置键可改变它。在机密计算模式开启之前，没有任何东西会划分受保护内存，而 mock 不建模该模式（见 issue #377）；无论芯片是否支持 CC，NVML 都报告 `0 MiB` 而不是不支持的查询：`tests/e2e/go/assertions/nvidiasmi/testdata/hardware` 中每块板卡都这样报告，包括 A100、L40S 和 T4。唯一例外是固定到早于 `525` 驱动的 profile——当时 CC API 还不存在，因此这些行读作 `N/A`，与那个年代的硬件一致。

**板卡诊断。** 没有配置键能改变这些应答。`nvidia-smi -q` 渲染它们的任何地方，都与刚加载驱动的硬件采集一致：

| 查询 | 应答 |
|-------|--------|
| 总线类型 | PCIe，在所有 profile 上都是，包括 Grace 板卡 |
| EGM（`capabilities` 块） | `disabled` |
| 时钟监控 | 无故障，Ampere 及之后 |
| InfoROM BBX 刷新 | `N/A`：本次驱动运行尚未刷新 |
| 桥接芯片、IRQ 号、PRM 寄存器 | 不支持 |

**采样缓冲区。** `nvmlDeviceGetSamples` 报告 20 个相隔一秒的样本，每个样本都携带对应 getter 的当前读数：功耗、GPU 与显存利用率、graphics 与显存时钟，以及编码器、解码器、JPEG 和 OFA 利用率。mock 不保留历史，因此窗口内的最小值、最大值和平均值都等于当前值，即便在[动态指标](#simulated-changes-between-calls)下也是如此。时间戳位于固定的一秒网格上，因此把上次看到的时间戳传回来的轮询者只会得到此后采集的样本，没有则得到 `NVML_ERROR_NOT_FOUND`。在启用 MIG 的 GPU 上，GPU、显存、编码器和解码器利用率样本不支持。任何 profile 上都不支持单元功耗样本。

**身份与拓扑。** 设备 `name`、`architecture`、`brand`、`compute_capability`、`uuid`、PCI `bus_id` 和 BAR1 窗口（`bar1_memory`）在设备构造时就固定下来，需要重启 Pod 才能改变。

### 感知分配——可选开启

当 `allocationWatcher.enabled=true` 时，`memory.used_bytes` 和 `memory.free_bytes` 会跟随 Kubernetes GPU 分配。nvml-mock DaemonSet 中的一个 sidecar 轮询 kubelet pod-resources API，并写入与 `nvml-mock-ctl` 相同的运行时覆盖文件，引擎在一个覆盖 TTL 内重新读取。调度一个申请 GPU 的 Pod 会改变该 GPU 的数字；删除 Pod 则恢复。

| 取值 | 行为 |
|-------|-----------|
| `allocationWatcher.usedFractionPerClaim` | 每次申请分摊的可用窗口份额（`total - reserved`）。默认 `0.5` |
| 时间切分 GPU | 每个持有容器是一次独立申请；同一设备上的申请会累加并在窗口处钳制 |
| 未申请的 GPU | 报告 `used_bytes: 0` 和 profile 的空闲 `free_bytes` |

上报的字节数是**合成的**。mock 不运行 kernel，也不分配设备显存，因此该数字只表示*存在*一次申请——而不是工作负载碰过什么。它逐设备读取：GPU 3 上的申请只影响 GPU 3。

该设计是电平触发的。每次轮询都根据当前完整分配重新计算每台设备，因此漏掉的事件会在下一个周期自愈，而不会让 GPU 卡在旧值上。失败的轮询不发布任何内容，保留之前的读数，因为瞬时 kubelet 错误时发布零会把整个节点报告为空闲。

监视器运行时**拥有**这两个字段：`nvml-mock-ctl set --gpu N memory.used_bytes=…` 会在下次轮询时被覆盖。`nvml-mock-ctl` 写入的其他字段都会保留——两个写入方使用同一把锁。

默认关闭，因为启用它会挂载 kubelet pod-resources socket（只读，节点本地；无需 RBAC，因为它不是 API server）。

### 仍不感知工作负载——已知空白

`processes` 始终为空，无论是否分配，因此即便 GPU 已被申请，`nvidia-smi` 也不报告运行中进程。跟踪于 [#506](https://github.com/NVIDIA/k8s-test-infra/issues/506) 第 2 项。

这里也要仔细阅读*模拟值*一节：`utilization.gpu` 不再是恒定 0，但它同样**不是**工作证据。它按经过时间变化。空闲节点上看起来忙碌的利用率正是该块按设计产生的结果。

每个容器*能看到*什么一直都是感知分配的，这与数值的含义是两回事：引擎按存在哪些 `/dev/nvidia*` 节点过滤其可见 GPU 集合，因此在八 GPU 节点上分配到一块 GPU 的 Pod，在 `nvidia-smi -L` 中恰好看到一块。

要显式改动其中任何一项，请使用 `nvml-mock-ctl set`。

## 校验

配置在加载时会进行校验：

- `version` 字段必需
- `system.driver_version` 必需
- 设备索引必须唯一
- YAML 语法无效时会回退到默认值

开启调试模式可查看校验错误：

```bash
MOCK_NVML_DEBUG=1 LD_LIBRARY_PATH=. nvidia-smi
```
