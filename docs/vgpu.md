# vGPU

profile 中 `virtualization.vgpu` 块的一切含义：GPU 如何成为 vGPU 宿主机或 vGPU 客户机、两侧各自的字段、NVML 和 `nvidia-smi` 如何报告它们，以及加载时哪些内容会被拒绝。

本文是参考文档。关于安装宿主机或客户机节点并用 `nvidia-smi` 读回，参见 [vGPU 宿主机与客户机指南](guides/vgpu/README.md)；profile 的其余部分参见[配置参考](configuration.md)。

## 模拟了什么

NVIDIA vGPU 把一块物理 GPU 拆分成虚拟 GPU，由 hypervisor 交给虚拟机（VM）。涉及两个驱动，各自看到不同的东西。**宿主机**是 hypervisor 的物理 GPU，运行 vGPU manager 和从中切出的 vGPU 实例。**客户机**是 VM 所看到的 vGPU：一块知道自己虚拟身份的 GPU，知道它运行在哪个宿主机驱动上，以及是否持有 vGPU 软件许可证。Mokka 在驱动层模拟任一种视图，因此 `nvidia-smi` 和 go-nvml 消费者读到的内容与真实 hypervisor 或 VM 上一样。不涉及 hypervisor 或 VM。

## 选择哪一侧

`device_defaults.virtualization.mode` 选择 NVML 报告什么：

| `mode` | `nvmlDeviceGetVirtualizationMode` | 其他变化 |
|---|---|---|
| `none` | `NVML_GPU_VIRTUALIZATION_MODE_NONE` | 无：裸机。每个自带 profile 都声明它 |
| `passthrough` | `NVML_GPU_VIRTUALIZATION_MODE_PASSTHROUGH` | 无其他变化 |
| `vgpu` | `NVML_GPU_VIRTUALIZATION_MODE_VGPU` | 有 [`guest` 块](#guest)时：许可信息，以及 GFD 读取的 PCI 能力 |
| `host_vgpu` | `NVML_GPU_VIRTUALIZATION_MODE_HOST_VGPU` | [宿主机](#host)类型和运行中的实例 |

`nvidia-smi -q` 把两种 vGPU 模式打印为 `VGPU` 和 `Host VGPU`。无法识别的模式报告 `NONE` 并忽略 `vgpu` 块。

vGPU 只能在 `device_defaults` 中配置：[逐设备覆盖](configuration.md#per-device-overrides)不会合并 `virtualization`，因此节点上所有 GPU 都从同一侧启动。只有[运行时覆盖](#runtime-overrides)能把某一块 GPU 移到另一侧。

### 在 chart 下

chart 从 `gpu.vgpu` 设置该块，因此安装时既不需要写 NVML 模式也不需要写 profile 键：

| `gpu.vgpu` | 默认值 | 含义 |
|---|---|---|
| `mode` | `""` | `host` 渲染为 `mode: host_vgpu`，`guest` 渲染为 `mode: vgpu`，`""` 保持 profile 的块不变 |
| `hostVgpuMode` | `sriov` | 仅宿主机。变为 `host_vgpu_mode` |
| `types` | `[]` | 仅宿主机。[类型](#types)，使用 profile 的 snake_case 形式 |
| `instances` | `[]` | 仅宿主机。[运行中的实例](#instances) |
| `guest` | `{}` | 仅客户机。[guest 块](#guest)，使用 profile 的 snake_case 形式 |

`mode: host` 只传 `hostVgpuMode`、`types` 和 `instances`；`mode: guest` 只传 `guest`。chart 会忽略另一侧模式的字段，而不是拒绝它们。`mode: guest` 搭配空 `guest` 会渲染为 `mode: vgpu` 且没有 `vgpu` 块，这就是[只有模式的客户机](#a-guest-with-no-block)。chart 拒绝渲染没有 `types` 的 `mode: host`，也拒绝与 `gpu.mig.enabled` 一起渲染；其余规则由节点 agent 执行，它会[校验](#validation)渲染出的块。

### 运行时覆盖

[`nvml-mock-ctl set`](nvml-mock-ctl.md#set--set-arbitrary-fields) 可以在运行中的节点上改变 `virtualization`，针对一块或所有 GPU。覆盖不像 profile 那样被校验：只检查字段名和值类型，不检查[校验规则](#validation)。例如，一块被设为 `host_vgpu` 却没有 `vgpu` 块的 GPU，会把模式报告为 `Host VGPU`，并对所有 vGPU 调用应答 `NVML_ERROR_NOT_SUPPORTED`。客户机的 PCI 能力由节点 agent 根据 profile 写入，因此覆盖不会改变它。

切换客户机许可证是可行的：

```bash
nvml-mock-ctl set --gpu 0 virtualization.vgpu.guest.license.state=unlicensed
```

下一次 `nvidia-smi -q` 会报告 GPU 0 未获许可。

## 宿主机

```yaml
device_defaults:
  virtualization:
    mode: "host_vgpu"
    host_vgpu_mode: "sriov"           # sriov、non_sriov
    vgpu:
      types:
        - id: 474
          name: "NVIDIA A100-4C"
          framebuffer_mib: 4096
          max_instances: 10
          license: "NVIDIA-Virtual-Compute-Server,9.0"
      instances:
        - type: "NVIDIA A100-4C"
          count: 2
```

没有 profile 自带类型列表。一块板卡提供哪些类型取决于 vGPU 软件版本，而不是芯片本身，因此安装要声明自己的类型，ID、名称和大小取自该版本的文档。

`host_vgpu_mode` 是必需的：`sriov` 或 `non_sriov`。`nvidia-smi -q` 把它打印为 `Host VGPU Mode`。

### 类型

| 字段 | 默认值 | 含义 |
|---|---|---|
| `id` | 必需 | NVML 分发的 vGPU 类型 ID。非零，列表内唯一。类型 ID 在每块 GPU 上命名同一类型 |
| `name` | 必需 | `nvidia-smi vgpu` 打印的内容，如 `NVIDIA A100-4C`。列表内唯一；`instances` 引用它 |
| `framebuffer_mib` | 必需 | 每个实例的显存，非零 |
| `max_instances` | 必需 | 一块 GPU 能容纳该类型的实例数，`1`–`999` |
| `class` | `Compute` | 类型的 class 字符串 |
| `max_instances_per_vm` | `1` | 一个 VM 可持有的实例数 |
| `license` | `""` | 该类型需要的许可证字符串 |
| `num_display_heads` | `1` | 显示头数 |
| `resolution` | `{width: 4096, height: 2160}` | 最大分辨率，对每个显示头报告 |
| `frame_rate_limit` | `0` | 帧率限制 |
| `gpu_instance_profile_id` | 非 MIG 支撑（`0xFFFFFFFF`） | 该类型背后的 MIG GPU 实例 profile |
| `device_id` | 板卡的 `pci.device_id` | 类型向客户机呈现的 PCI 设备和厂商 ID，编码方式与 profile 的 `pci` 块相同 |
| `subsystem_id` | 板卡的 `pci.subsystem_id` | 类型向客户机呈现的 PCI 子系统和子系统厂商 ID |

这两个 PCI ID 是 `nvmlVgpuTypeGetDeviceID` 报告的内容。NVML 按类型而非按板卡报告它们，因此每个都可以逐类型设置；未设置的取板卡的值。

### 实例

每个条目请求 `count` 个指定类型的运行中实例。省略 `count` 表示 1 个。实例从库加载那一刻起就运行；没有 NVML 调用能创建或销毁它们，只有[运行时覆盖](#runtime-overrides)的 `instances` 能改变集合。

一块 GPU 一次只运行一种 vGPU 类型，因此所有条目都命名同一类型，其数量总和受该类型 `max_instances` 限制。当 profile 设置了 `memory.total_bytes` 时，它们的显存总和必须能装进去。

### 宿主机身份

实例 ID 是 `(GPU 索引 + 1) × 1000 + 槽位 + 1`，其中槽位按声明顺序从零开始计算该 GPU 的实例：GPU 0 的第一个实例是 `1001`，GPU 1 的第二个是 `2002`。GPU 索引是 GPU 在节点上的物理索引，不是容器看到的枚举索引：在只给了 GPU 3 的容器中，该 GPU 的第一个实例仍是 `4001`。ID 编码了它所属的 GPU，因此在每个进程中都相同。

每个实例的 vGPU UUID 和 VM UUID 由其 GPU 的 UUID 和槽位派生。当 profile 声明了 `devices[].uuid` 时，它们在重启之间保持稳定——每个自带 profile 都这么做了。

每个实例运行在自己的 VM 中。VM 名称为空。客户机驱动版本是 `Not Available`，这是 NVML 在 VM 中加载驱动前报告的值，而模拟 VM 不会加载驱动；`nvidia-smi` 打印为 `N/A`。ECC 模式跟随板卡的 `ecc.mode_current`，GPU 实例 ID 非 MIG 支撑，`nvidia-smi` 打印为 `N/A`。

### 宿主机行为

**支持的类型和可创建的类型。** `nvidia-smi vgpu -s` 列出所有声明的类型。`nvidia-smi vgpu -c` 列出仍可创建的内容：没有实例运行时是所有声明的类型，之后只有正在运行的那种，直到 GPU 持有该类型的 `max_instances` 个实例，然后什么都不列出。

**空闲实例。** 模拟客户机中没有进程运行，因此每个实例都报告零显存使用、零编码器和帧缓冲捕获（FBC）会话，以及禁用的计费。每个实例都报告自己已获许可，且许可证是永久的。

**未实现，`N/A`。** 这些调用返回 `NVML_ERROR_NOT_SUPPORTED`，`nvidia-smi` 打印为 `N/A`：实例的 placement ID、mdev UUID、计费 PID、利用率和客户机 PCI 总线 ID；类型的 BAR1 信息、能力、placement 和每个 GPU 实例的最大实例数；以及设备的 vGPU 能力、vGPU 驱动能力和异构模式。

**错误。** 对不处于 `host_vgpu` 模式的 GPU 发起宿主机侧 vGPU 调用会返回 `NVML_ERROR_NOT_SUPPORTED`，与裸机驱动一样，`nvidia-smi vgpu` 打印 `No supported devices in vGPU mode`。未知类型 ID 返回 `NVML_ERROR_INVALID_ARGUMENT`。对于实例 ID，NVML 为 `0` 保留 `NVML_ERROR_INVALID_ARGUMENT`；其他不匹配运行中实例的 ID，包括调用方看不到的 GPU 上的实例，都返回 `NVML_ERROR_NOT_FOUND`。vGPU 调用根据配置应答，不会推进[故障注入](helm-chart.md#failure-injection)的调用计数器。接受设备句柄或实例 ID 的调用，在故障注入使该 GPU 丢失后会报告丢失；只接受类型 ID 的调用会从任何其他声明该类型的 GPU 应答。

**`nvidia-smi pmon` 拒绝宿主机 GPU**，对每块打印 `The feature is not supported in this configuration`，与 hypervisor 上一样。

## 客户机

```yaml
device_defaults:
  virtualization:
    mode: "vgpu"
    vgpu:
      guest:
        host_driver_version: "550.163.01"
        host_driver_branch: "r550_00"
        license:
          product: "NVIDIA Virtual Compute Server"
          state: "licensed"           # licensed、unlicensed
          expiry: "2027-01-01T00:00:00Z"
```

| 字段 | 默认值 | 含义 |
|---|---|---|
| `host_driver_version` | 必需 | 宿主机的 vGPU manager 版本。最多 10 字节 |
| `host_driver_branch` | 必需 | 宿主机驱动的分支。最多 10 字节 |
| `license` | 缺失：未获许可 | 客户机持有的 vGPU 软件许可证 |
| `license.product` | `""` | 获许可的产品名。最多 127 字节 |
| `license.state` | `licensed` | `licensed` 或 `unlicensed` |
| `license.expiry` | 缺失：永久 | 已获许可客户机的许可证到期时间，RFC 3339 格式 |

两个宿主机驱动字段限制为 10 字节，因为这是承载它们的 PCI 能力字段的宽度。

客户机的变化：

- `nvidia-smi -q` 打印 `Virtualization Mode : VGPU` 和一个 `vGPU Software Licensed Product` 小节，包含产品及其 `License Status`，如 `Licensed (Expiry: Permanent)`。未获许可的客户机报告未获许可状态且没有到期时间。
- 节点 agent 把 vGPU 厂商能力（包含宿主机驱动版本和分支）写入每块 GPU 在 [PCI sysfs 树](helm-chart.md#pci-sysfs-in-containers)中的配置空间。GPU Feature Discovery（GFD）从那里读取并给节点打上 `nvidia.com/vgpu.present`、`nvidia.com/vgpu.host-driver-version` 和 `nvidia.com/vgpu.host-driver-branch` 标签。

许可通过 `nvmlDeviceGetGridLicensableFeatures`（v4，无版本名称解析到该版本）报告；没有 `guest` 块的客户机 GPU 则应答 `NVML_ERROR_NOT_SUPPORTED`。

### 没有块的客户机

`mode: vgpu` 但没有 `guest` 块时，只报告 `Virtualization Mode : VGPU`，别无其他：没有 PCI 能力，许可应答 `NVML_ERROR_NOT_SUPPORTED`。

## 校验

配置加载时会检查该块，每次失败都会点出字段：

- `host_vgpu` 需要 `host_vgpu_mode` 设为 `sriov` 或 `non_sriov`，且至少有一个类型。
- 每个类型需要非零 `id`、`name` 和非零 `framebuffer_mib`，`max_instances` 在 1 到 999 之间。没有两个类型共享 `id` 或 `name`。
- 每个实例都命名一个声明过的类型，其 `count` 不为负。
- 所有实例命名同一类型。
- 实例总数不超过该类型的 `max_instances`。
- 当 profile 设置了 `memory.total_bytes` 时，实例总显存不超过它。
- `guest` 在 `host_vgpu` 下被拒绝；`types` 和 `instances` 在 `vgpu` 下被拒绝。
- 宿主机不运行任何处于 MIG 模式的 GPU：当 `device_defaults.mig` 或任何逐设备 `mig` 块设置 `mode_current: enabled` 时，`host_vgpu` 被拒绝，`nvmlDeviceSetMigMode` 也拒绝在宿主机上启用 MIG，返回 `NVML_ERROR_NOT_SUPPORTED`。真实宿主机启用 MIG 时只运行 MIG 支撑的类型，而那些没有被模拟。
- `guest` 块需要 `host_driver_version` 和 `host_driver_branch`，各最多 10 字节。
- `license.state` 是 `licensed` 或 `unlicensed`，`license.product` 最多 127 字节，`license.expiry` 可按 RFC 3339 解析。

任何其他 `mode` 都会忽略 `vgpu` 块，这就是每个自带 profile 携带的 `host_vgpu_mode: "N/A"` 仍能加载的原因。

节点 agent 拒绝无效的块且不会就绪，把该字段记录为 `state source error`。库同样拒绝该配置，并带警告回退到内置默认值。

!!! note "未模拟"
    - 宿主机的 mdev 和 SR-IOV sysfs 接口面。
    - MIG 支撑的 vGPU 类型，以及启用 MIG 模式的宿主机。
    - 运行时创建或销毁实例。
    - 随附的 vGPU 类型目录：安装要声明自己的类型。
    - 把实例显存计入物理 GPU 内存。
    - 调度器、placement 和异构模式 API。
    - `nvmlGetVgpuVersion` 和 `nvmlSetVgpuVersion`。
    - vGPU 元数据和迁移兼容性。
    - 每个 `Set*` 调用。
    - `nvmlDeviceGetGridLicensableFeatures` v1–v3。
    - KubeVirt 或任何其他 VM 管理器。
