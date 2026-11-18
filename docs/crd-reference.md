<!--
SPDX-License-Identifier: Apache-2.0
SPDX-FileCopyrightText: Copyright 2026 NVIDIA CORPORATION
-->

# API 参考

## 包
- [mokka.nvidia.com/v1alpha1](#mokkanvidiacomv1alpha1)


## mokka.nvidia.com/v1alpha1

v1alpha1 包包含 mokka.nvidia.com API 组的类型。


### 资源类型
- [SGPUInventory](#sgpuinventory)
- [SGPURack](#sgpurack)
- [SGPURackProfile](#sgpurackprofile)
- [SGPURuntimePolicy](#sgpuruntimepolicy)



#### CapabilityAttribute



CapabilityAttribute 是一种带类型的变体；只设置一个字段。



_出现于：_
- [GPUCapabilities](#gpucapabilities)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `bool` _boolean_ | Bool 存储布尔型能力值。 |  | Optional: \{\} <br /> |
| `int` _integer_ | Int 存储整型能力值。 |  | Optional: \{\} <br /> |
| `string` _string_ | String 存储字符串型能力值。 |  | Optional: \{\} <br /> |
| `strings` _string array_ | Strings 存储一组字符串型能力值。 |  | Optional: \{\} <br /> |


#### ClockRates



ClockRates 是以 MHz 为单位的 graphics/SM/memory/video 时钟元组。



_出现于：_
- [GPUClocks](#gpuclocks)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `graphics` _integer_ | Graphics 是 graphics 时钟频率，单位 MHz。 |  | Optional: \{\} <br /> |
| `sm` _integer_ | SM 是流多处理器时钟频率，单位 MHz。 |  | Optional: \{\} <br /> |
| `memory` _integer_ | Memory 是显存时钟频率，单位 MHz。 |  | Optional: \{\} <br /> |
| `video` _integer_ | Video 是视频时钟频率，单位 MHz。 |  | Optional: \{\} <br /> |


#### ClocksTelemetry



ClocksTelemetry 报告当前时钟频率。



_出现于：_
- [RuntimeTelemetry](#runtimetelemetry)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `graphicsMHz` _integer_ | GraphicsMHz 是模拟的 graphics 时钟频率，单位 MHz。 |  | Minimum: 0 <br />Optional: \{\} <br /> |
| `smMHz` _integer_ | SMMHz 是模拟的流多处理器时钟频率，单位 MHz。 |  | Minimum: 0 <br />Optional: \{\} <br /> |
| `memoryMHz` _integer_ | MemoryMHz 是模拟的显存时钟频率，单位 MHz。 |  | Minimum: 0 <br />Optional: \{\} <br /> |
| `videoMHz` _integer_ | VideoMHz 是模拟的视频时钟频率，单位 MHz。 |  | Minimum: 0 <br />Optional: \{\} <br /> |


#### ComputeCapability



ComputeCapability 是 CUDA 计算能力。



_出现于：_
- [GPUModel](#gpumodel)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `major` _integer_ | Major 是 CUDA 计算能力的主版本号。 |  |  |
| `minor` _integer_ | Minor 是 CUDA 计算能力的次版本号。 |  |  |


#### DeviceState

_底层类型：_ _string_

DeviceState 是模拟的 GPU 健康状态。

_校验：_
- Enum: [Healthy Degraded Failed]

_出现于：_
- [RuntimeState](#runtimestate)

| 字段 | 描述 |
| --- | --- |
| `Healthy` |  |
| `Degraded` |  |
| `Failed` |  |


#### FabricDomain



FabricDomain 是 fabric 计算域。



_出现于：_
- [GPUFabric](#gpufabric)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `scope` _string_ | Scope 是 GPU 共享 fabric 域的层级。 |  | Enum: [Node Rack Cluster] <br />Optional: \{\} <br /> |
| `gpuCount` _integer_ | GPUCount 是该域中的 GPU 数量。 |  | Optional: \{\} <br /> |


#### FabricSwitches



FabricSwitches 是每个节点可见的 fabric 交换机。



_出现于：_
- [GPUFabric](#gpufabric)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `visiblePerNode` _integer_ | VisiblePerNode 是从每个节点可见的 fabric 交换机数量。 |  | Optional: \{\} <br /> |


#### GPUBoard



GPUBoard 是板卡级标识。



_出现于：_
- [GPUModel](#gpumodel)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `partNumber` _string_ | PartNumber 是为 GPU 报告的板卡料号。 |  | Optional: \{\} <br /> |


#### GPUCapabilities



GPUCapabilities 是 GPU 功能开关和可扩展属性。



_出现于：_
- [SGPUGPUs](#sgpugpus)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `mig` _[MIGCapability](#migcapability)_ | MIG 描述对多实例 GPU 的支持。 |  | Optional: \{\} <br /> |
| `attributes` _object (keys:string, values:[CapabilityAttribute](#capabilityattribute))_ | 以限定名作为键的可扩展能力标志。 |  | Optional: \{\} <br /> |


#### GPUClocks



GPUClocks 是时钟上限和按显存时钟排列的调度表。



_出现于：_
- [SGPUGPUs](#sgpugpus)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `maximumMHz` _[ClockRates](#clockrates)_ | MaximumMHz 给出 graphics、SM、memory 和 video 的最大时钟频率。 |  | Optional: \{\} <br /> |
| `supported` _[SupportedClocks](#supportedclocks) array_ | Supported 列出每种显存时钟频率下可用的 graphics 时钟频率。 |  | Optional: \{\} <br /> |


#### GPUCores



GPUCores 是可编程核心数量。



_出现于：_
- [GPUModel](#gpumodel)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `cuda` _integer_ | CUDA 是每块 GPU 的 CUDA 核心数。 |  | Optional: \{\} <br /> |


#### GPUFabric



GPUFabric 是 NVLink/NVSwitch fabric。



_出现于：_
- [SGPUTopology](#sgputopology)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `type` _string_ | Type 命名 GPU 互连 fabric。 |  | Optional: \{\} <br /> |
| `generation` _integer_ | Generation 是 fabric 的代际。 |  | Optional: \{\} <br /> |
| `linksPerGPU` _integer_ | LinksPerGPU 是每块 GPU 可用的 fabric 链路数。 |  | Optional: \{\} <br /> |
| `bandwidthPerLinkMBps` _integer_ | BandwidthPerLinkMBps 是每条链路的带宽，单位 MB/s。 |  | Optional: \{\} <br /> |
| `c2cSupported` _boolean_ | C2CSupported 表示是否支持 CPU 到 GPU 的一致性链路。 |  | Optional: \{\} <br /> |
| `domain` _[FabricDomain](#fabricdomain)_ | Domain 描述 GPU fabric 域的范围和大小。 |  | Optional: \{\} <br /> |
| `switches` _[FabricSwitches](#fabricswitches)_ | Switches 描述每个节点可见的 fabric 交换机。 |  | Optional: \{\} <br /> |


#### GPUFirmware



GPUFirmware 是固件版本。



_出现于：_
- [GPUModel](#gpumodel)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `vbiosVersion` _string_ | VBIOSVersion 是视频 BIOS 版本。 |  | Optional: \{\} <br /> |
| `gspVersion` _string_ | GSPVersion 是 GPU 系统处理器固件版本。 |  | Optional: \{\} <br /> |
| `infoROM` _[InfoROM](#inforom)_ | InfoROM 包含 GPU 信息 ROM 对象的版本。 |  | Optional: \{\} <br /> |


#### GPUMemory



GPUMemory 是设备上显存。



_出现于：_
- [SGPUGPUs](#sgpugpus)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `capacity` _[Quantity](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#quantity-resource-api)_ | Capacity 是设备显存总容量。 |  |  |
| `reserved` _[Quantity](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#quantity-resource-api)_ | Reserved 是为工作负载预留的显存量。 |  | Optional: \{\} <br /> |
| `bar1Capacity` _[Quantity](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#quantity-resource-api)_ | Bar1Capacity 是设备的 BAR1 窗口大小。 |  | Optional: \{\} <br /> |
| `busWidthBits` _integer_ | BusWidthBits 是显存总线宽度，单位比特。 |  | Optional: \{\} <br /> |


#### GPUModel



GPUModel 是厂商/产品标识。



_出现于：_
- [SGPUGPUs](#sgpugpus)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `vendor` _string_ | Vendor 是 GPU 制造商名称。 |  | Optional: \{\} <br /> |
| `product` _string_ | Product 是厂商的产品标识符。 |  | Optional: \{\} <br /> |
| `productName` _string_ | ProductName 是可读的产品名称。 |  | Optional: \{\} <br /> |
| `architecture` _string_ | Architecture 是 GPU 架构名称。 |  | Optional: \{\} <br /> |
| `computeCapability` _[ComputeCapability](#computecapability)_ | ComputeCapability 是 CUDA 计算能力版本。 |  | Optional: \{\} <br /> |
| `cores` _[GPUCores](#gpucores)_ | Cores 包含可编程核心数量。 |  | Optional: \{\} <br /> |
| `board` _[GPUBoard](#gpuboard)_ | Board 包含板卡级标识。 |  | Optional: \{\} <br /> |
| `firmware` _[GPUFirmware](#gpufirmware)_ | Firmware 包含固件和信息 ROM 版本。 |  | Optional: \{\} <br /> |


#### GPUPCI



GPUPCI 是 PCI 标识和链路特性。



_出现于：_
- [SGPUGPUs](#sgpugpus)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `vendorID` _string_ | VendorID 是十六进制的 PCI 厂商标识符。 |  | Optional: \{\} <br /> |
| `deviceID` _string_ | DeviceID 是十六进制的 PCI 设备标识符。 |  | Optional: \{\} <br /> |
| `subsystemVendorID` _string_ | SubsystemVendorID 是 PCI 子系统厂商标识符。 |  | Optional: \{\} <br /> |
| `subsystemDeviceID` _string_ | SubsystemDeviceID 是 PCI 子系统设备标识符。 |  | Optional: \{\} <br /> |
| `maxLink` _[PCILink](#pcilink)_ | MaxLink 是支持的最大 PCIe 代际和宽度。 |  | Optional: \{\} <br /> |


#### GPUPower



GPUPower 是功耗包络。



_出现于：_
- [SGPUGPUs](#sgpugpus)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `managementSupported` _boolean_ | ManagementSupported 表示是否可用功耗管理。 |  | Optional: \{\} <br /> |
| `limitsMilliWatts` _[PowerLimits](#powerlimits)_ | LimitsMilliWatts 给出最小、默认和最大功耗限制。 |  | Optional: \{\} <br /> |


#### GPUSlot



GPUSlot 把一块 GPU 固定到 PCI 地址 + NUMA/root complex/宿主 CPU。



_出现于：_
- [SGPUTopology](#sgputopology)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `index` _integer_ | Index 是节点上从零开始的 GPU 索引。 |  | Maximum: 63 <br />Minimum: 0 <br /> |
| `pciAddress` _string_ | PCIAddress 是该 GPU 的 PCI 总线地址。 |  | Pattern: `^[0-9a-f]\{4\}:[0-9a-f]\{2\}:[0-9a-f]\{2\}\.[0-7]$` <br />Required: \{\} <br /> |
| `rootComplex` _string_ | RootComplex 标识连接到该 GPU 的 PCI root complex。 |  | Pattern: `^pci[0-9a-f]\{4\}:[0-9a-f]\{2\}$` <br />Required: \{\} <br /> |
| `numaNode` _integer_ | NumaNode 是与该 GPU 关联的宿主 NUMA 节点。 |  | Optional: \{\} <br /> |
| `hostProcessorIndex` _integer_ | HostProcessorIndex 是与该 GPU 关联的宿主 CPU 索引。 |  | Optional: \{\} <br /> |


#### GPUThermal



GPUThermal 是散热阈值。



_出现于：_
- [SGPUGPUs](#sgpugpus)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `targetCelsius` _integer_ | TargetCelsius 是目标 GPU 温度，单位摄氏度。 |  | Optional: \{\} <br /> |
| `maxOperatingCelsius` _integer_ | MaxOperatingCelsius 是额定最高工作温度。 |  | Optional: \{\} <br /> |
| `slowdownThresholdCelsius` _integer_ | SlowdownThresholdCelsius 是开始散热降速的温度。 |  | Optional: \{\} <br /> |
| `shutdownThresholdCelsius` _integer_ | ShutdownThresholdCelsius 是触发关机的温度。 |  | Optional: \{\} <br /> |


#### HostCPU



HostCPU 是宿主 CPU 标识。



_出现于：_
- [SGPUHost](#sgpuhost)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `vendor` _string_ | Vendor 是处理器制造商。 |  | Optional: \{\} <br /> |
| `product` _string_ | Product 是处理器产品标识符。 |  | Optional: \{\} <br /> |
| `architecture` _string_ | Architecture 是宿主处理器架构。 |  | Optional: \{\} <br /> |
| `cores` _integer_ | Cores 是宿主 CPU 核心数。 |  | Optional: \{\} <br /> |


#### HostMemory



HostMemory 是宿主内存容量。



_出现于：_
- [SGPUHost](#sgpuhost)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `capacity` _[Quantity](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#quantity-resource-api)_ | Capacity 是宿主内存容量。 |  |  |
| `coherentWithGPU` _boolean_ | CoherentWithGPU 表示宿主内存与 GPU 显存是否一致。 |  | Optional: \{\} <br /> |


#### InfoROM



InfoROM 是 NVML inforom 子对象的版本。



_出现于：_
- [GPUFirmware](#gpufirmware)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `imageVersion` _string_ | ImageVersion 是信息 ROM 镜像版本。 |  | Optional: \{\} <br /> |
| `oemObjectVersion` _string_ | OEMObjectVersion 是 OEM 信息 ROM 对象版本。 |  | Optional: \{\} <br /> |
| `eccObjectVersion` _string_ | ECCObjectVersion 是 ECC 信息 ROM 对象版本。 |  | Optional: \{\} <br /> |
| `powerObjectVersion` _string_ | PowerObjectVersion 是功耗信息 ROM 对象版本。 |  | Optional: \{\} <br /> |


#### InventoryCapacity



InventoryCapacity 是已实现的机架/节点/GPU 数量。



_出现于：_
- [RackGroupStatus](#rackgroupstatus)
- [SGPUInventoryStatus](#sgpuinventorystatus)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `racks` _integer_ | Racks 是已物化的机架数量。 |  |  |
| `nodes` _integer_ | Nodes 是这些机架中的逻辑节点数量。 |  |  |
| `gpus` _integer_ | GPUs 是这些节点中的模拟 GPU 数量。 |  |  |


#### InventoryUsage



InventoryUsage 汇总符合条件的放置节点、已绑定的机架槽位和待处理的分配。



_出现于：_
- [RackGroupStatus](#rackgroupstatus)
- [SGPUInventoryStatus](#sgpuinventorystatus)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `requestedNodes` _integer_ | RequestedNodes 统计匹配放置选择器的符合条件的节点。清单总量<br />在各组之间对匹配去重；每组的数值统计该组自身的匹配。 |  |  |
| `allocatedNodes` _integer_ | AllocatedNodes 统计具有有效匹配的 Kubernetes 节点绑定的机架节点槽位。 |  |  |
| `availableNodes` _integer_ | AvailableNodes 是扣除已绑定机架槽位后的剩余容量，永不低于零。 |  |  |
| `pendingNodes` _integer_ | PendingNodes 统计在某一匹配组中等待容量的符合条件的放置请求。 |  |  |


#### MIGCapability



MIGCapability 是 MIG 分区支持。



_出现于：_
- [GPUCapabilities](#gpucapabilities)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `supported` _boolean_ | Supported 表示该 GPU 上是否可用 MIG。 |  | Optional: \{\} <br /> |
| `maxGPUInstances` _integer_ | MaxGPUInstances 是支持的最大 GPU 实例数。 |  | Optional: \{\} <br /> |


#### NetworkTopology



NetworkTopology 是带外适配器。



_出现于：_
- [SGPUTopology](#sgputopology)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `type` _string_ | Type 命名带外网络适配器类型。 |  | Optional: \{\} <br /> |
| `adapterModel` _string_ | AdapterModel 是适配器型号名称。 |  | Optional: \{\} <br /> |
| `firmwareVersion` _string_ | FirmwareVersion 是适配器固件版本。 |  | Optional: \{\} <br /> |
| `linkSpeedGbps` _integer_ | LinkSpeedGbps 是适配器链路速率，单位 Gbit/s。 |  | Optional: \{\} <br /> |
| `adaptersPerGPU` _integer_ | AdaptersPerGPU 是与每块 GPU 关联的网络适配器数量。 |  | Optional: \{\} <br /> |


#### PCILink



PCILink 是 PCIe 最大链路代际和宽度。



_出现于：_
- [GPUPCI](#gpupci)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `generation` _integer_ | Generation 是最大 PCIe 代际。 |  | Optional: \{\} <br /> |
| `width` _integer_ | Width 是最大 PCIe 链路宽度，单位通道。 |  | Optional: \{\} <br /> |


#### PercentRange



PercentRange 是最小/最大百分比边界。两个边界都是必需的，因此覆盖会整体替换继承的范围。



_出现于：_
- [UtilizationPattern](#utilizationpattern)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `minimum` _integer_ | Minimum 是利用率百分比下界。 |  | Maximum: 100 <br />Minimum: 0 <br /> |
| `maximum` _integer_ | Maximum 是利用率百分比上界。 |  | Maximum: 100 <br />Minimum: 0 <br /> |


#### PolicyTargetRef



PolicyTargetRef 选择策略所适用的模拟 GPU。列出的每个维度都会收窄其上一级，省略的维度则选择所有索引。列出的最深维度即策略的作用域：清单、机架组、机架、节点或 GPU。较窄的作用域按字段覆盖较宽的。列出的每个机架组都必须存在于清单中，且每个列出的索引都必须至少存在于一个所选机架组中。



_出现于：_
- [SGPURuntimePolicySpec](#sgpuruntimepolicyspec)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `group` _string_ | Group 是包含目标资源的 API 组。 |  | Enum: [mokka.nvidia.com] <br /> |
| `kind` _string_ | Kind 是目标资源的种类。 |  | Enum: [SGPUInventory] <br /> |
| `name` _string_ | Name 是目标 SGPUInventory 的名称。 |  | MinLength: 1 <br /> |
| `rackGroups` _string array_ | RackGroups 可选地把策略限制到这些机架组 ID。 |  | MaxItems: 64 <br />MinItems: 1 <br />Optional: \{\} <br /> |
| `rackIndexes` _integer array_ | RackIndexes 可选地把策略限制到这些机架索引。 |  | MaxItems: 64 <br />MinItems: 1 <br />items:Minimum: 0 <br />Optional: \{\} <br /> |
| `nodeIndexes` _integer array_ | NodeIndexes 可选地把策略限制到这些逻辑节点索引。 |  | MaxItems: 64 <br />MinItems: 1 <br />items:Minimum: 0 <br />Optional: \{\} <br /> |
| `gpuIndexes` _integer array_ | GPUIndexes 可选地把策略限制到这些 GPU 索引。 |  | MaxItems: 64 <br />MinItems: 1 <br />items:Minimum: 0 <br />Optional: \{\} <br /> |


#### PowerLimits



PowerLimits 是以毫瓦为单位的最小/默认/最大功耗上限。



_出现于：_
- [GPUPower](#gpupower)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `minimum` _integer_ | Minimum 是以毫瓦为单位的最小功耗限制。 |  | Optional: \{\} <br /> |
| `default` _integer_ | Default 是以毫瓦为单位的默认功耗限制。 |  | Optional: \{\} <br /> |
| `maximum` _integer_ | Maximum 是以毫瓦为单位的最大功耗限制。 |  | Optional: \{\} <br /> |


#### PowerTelemetry



PowerTelemetry 驱动合成功耗。



_出现于：_
- [RuntimeTelemetry](#runtimetelemetry)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `mode` _string_ | Mode 选择恒定或生成的功耗。 |  | Enum: [Fixed Pattern] <br />Optional: \{\} <br /> |
| `drawMilliWatts` _integer_ | DrawMilliWatts 是模拟的 GPU 功耗，单位毫瓦。 |  | Minimum: 0 <br />Optional: \{\} <br /> |


#### ProfileReference



ProfileReference 按名称指向一个 SGPURackProfile。



_出现于：_
- [RackGroup](#rackgroup)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `name` _string_ | Name 是要使用的 SGPURackProfile 名称。 |  | MinLength: 1 <br /> |


#### RackGroup



RackGroup 是共享一个 profile 的同构机架组。



_出现于：_
- [SGPUInventorySpec](#sgpuinventoryspec)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `id` _string_ | ID 在其清单中唯一标识该组。 |  | MaxLength: 63 <br />MinLength: 1 <br /> |
| `count` _integer_ | Count 是从该组物化的机架数量。 |  | Maximum: 100000 <br />Minimum: 1 <br /> |
| `profileRef` _[ProfileReference](#profilereference)_ | ProfileRef 命名用于物化每个机架的机架 profile。 |  |  |
| `placement` _[RackPlacement](#rackplacement)_ | Placement 可选地限制该机架组可用的节点。 |  | Optional: \{\} <br /> |


#### RackGroupStatus



RackGroupStatus 是每组的容量 + 使用量投影。



_出现于：_
- [SGPUInventoryStatus](#sgpuinventorystatus)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `id` _string_ | ID 标识此状态所汇总的机架组。 |  |  |
| `profileName` _string_ | ProfileName 是用于物化该组的 profile 名称。 |  |  |
| `capacity` _[InventoryCapacity](#inventorycapacity)_ | Capacity 是该组已实现的容量。 |  |  |
| `usage` _[InventoryUsage](#inventoryusage)_ | Usage 汇总该组的节点分配情况。 |  |  |


#### RackPlacement



RackPlacement 约束机架组可以在哪些节点上物化。



_出现于：_
- [RackGroup](#rackgroup)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `nodeSelector` _[LabelSelector](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#labelselector-v1-meta)_ | NodeSelector 把放置限制到匹配这些标签的节点。 |  | Optional: \{\} <br /> |


#### RuntimeModes



RuntimeModes 是持久化的 NVML/CUDA 模式设置。



_出现于：_
- [RuntimeState](#runtimestate)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `persistence` _string_ | Persistence 控制模拟的 persistence 模式。 |  | Enum: [Enabled Disabled] <br />Optional: \{\} <br /> |
| `compute` _string_ | Compute 选择模拟的 compute 模式。 |  | Enum: [Default Exclusive Prohibited] <br />Optional: \{\} <br /> |
| `mig` _string_ | MIG 控制是否启用多实例 GPU 模式。 |  | Enum: [Enabled Disabled] <br />Optional: \{\} <br /> |
| `ecc` _string_ | ECC 控制是否启用纠错码。 |  | Enum: [Enabled Disabled] <br />Optional: \{\} <br /> |
| `accounting` _string_ | Accounting 控制是否启用 GPU 计费模式。 |  | Enum: [Enabled Disabled] <br />Optional: \{\} <br /> |


#### RuntimeState



RuntimeState 是模拟 GPU 的有效运行时设置。稀疏：省略的字段继承；显式零是设定值。



_出现于：_
- [SGPURackProfileDefaults](#sgpurackprofiledefaults)
- [SGPURuntimePolicySpec](#sgpuruntimepolicyspec)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `deviceState` _[DeviceState](#devicestate)_ | DeviceState 是 GPU 的模拟健康状态。 |  | Enum: [Healthy Degraded Failed] <br />Optional: \{\} <br /> |
| `modes` _[RuntimeModes](#runtimemodes)_ | Modes 包含持久化的 NVML/CUDA 模式设置。 |  | Optional: \{\} <br /> |
| `telemetry` _[RuntimeTelemetry](#runtimetelemetry)_ | Telemetry 包含合成的利用率、功耗、温度和时钟值。 |  | Optional: \{\} <br /> |


#### RuntimeTelemetry



RuntimeTelemetry 是合成的 NVML 遥测。



_出现于：_
- [RuntimeState](#runtimestate)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `performanceState` _string_ | NVML P-state，例如 "P0"。 |  | Pattern: `^P[0-9]+$` <br />Optional: \{\} <br /> |
| `utilization` _[UtilizationTelemetry](#utilizationtelemetry)_ | Utilization 定义合成的 GPU 和显存利用率值。 |  | Optional: \{\} <br /> |
| `power` _[PowerTelemetry](#powertelemetry)_ | Power 定义合成的功耗值。 |  | Optional: \{\} <br /> |
| `temperature` _[TemperatureTelemetry](#temperaturetelemetry)_ | Temperature 定义合成的 GPU 和显存温度值。 |  | Optional: \{\} <br /> |
| `clocks` _[ClocksTelemetry](#clockstelemetry)_ | Clocks 定义当前模拟的时钟频率。 |  | Optional: \{\} <br /> |


#### SGPUGPUs



SGPUGPUs 是节点上每块 GPU 的共享模板。



_出现于：_
- [SGPUNode](#sgpunode)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `count` _integer_ | Count 是每个节点上暴露的 GPU 数量。 |  | Maximum: 64 <br />Minimum: 1 <br /> |
| `model` _[GPUModel](#gpumodel)_ | Model 描述 GPU 的厂商、产品和架构。 |  |  |
| `memory` _[GPUMemory](#gpumemory)_ | Memory 描述设备显存容量和总线宽度。 |  |  |
| `pci` _[GPUPCI](#gpupci)_ | PCI 描述设备标识符和最大链路属性。 |  |  |
| `power` _[GPUPower](#gpupower)_ | Power 描述功耗管理支持和设备限制。 |  | Optional: \{\} <br /> |
| `thermal` _[GPUThermal](#gputhermal)_ | Thermal 描述设备温度阈值。 |  | Optional: \{\} <br /> |
| `clocks` _[GPUClocks](#gpuclocks)_ | Clocks 描述最大和支持的设备时钟频率。 |  | Optional: \{\} <br /> |
| `capabilities` _[GPUCapabilities](#gpucapabilities)_ | Capabilities 描述 MIG 支持和其他命名能力。 |  | Optional: \{\} <br /> |


#### SGPUHost



SGPUHost 是宿主 CPU + 内存。



_出现于：_
- [SGPUNode](#sgpunode)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `cpu` _[HostCPU](#hostcpu)_ | CPU 描述模拟节点所呈现的宿主处理器。 |  | Optional: \{\} <br /> |
| `memory` _[HostMemory](#hostmemory)_ | Memory 描述模拟节点所呈现的宿主内存。 |  | Optional: \{\} <br /> |


#### SGPUInventory



SGPUInventory 是要分布到 CPU 节点上的一组模拟 GPU 机架。





| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `apiVersion` _string_ | `mokka.nvidia.com/v1alpha1` | | |
| `kind` _string_ | `SGPUInventory` | | |
| `metadata` _[ObjectMeta](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#objectmeta-v1-meta)_ | 有关 `metadata` 的字段，请参阅 Kubernetes API 文档。 |  |  |
| `spec` _[SGPUInventorySpec](#sgpuinventoryspec)_ | Spec 定义该清单管理的模拟机架组。 |  |  |
| `status` _[SGPUInventoryStatus](#sgpuinventorystatus)_ | Status 报告控制器当前已实现的容量和使用量。 |  | Optional: \{\} <br /> |


#### SGPUInventorySpec



SGPUInventorySpec 是期望的机架组成。



_出现于：_
- [SGPUInventory](#sgpuinventory)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `rackGroups` _[RackGroup](#rackgroup) array_ | RackGroups 声明同构的机架集合。一个组可以展开为<br />多个机架；控制器在所有清单中最多接纳 64 个组。 |  | MaxItems: 64 <br />MinItems: 1 <br /> |


#### SGPUInventoryStatus



SGPUInventoryStatus 是控制面对已实现容量和使用量的视图。



_出现于：_
- [SGPUInventory](#sgpuinventory)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `rackGroupsSummary` _string_ | RackGroupsSummary 是机架组 ID 的逗号连接呈现。 |  | Optional: \{\} <br /> |
| `capacity` _[InventoryCapacity](#inventorycapacity)_ | Capacity 是该清单已实现的机架、节点和 GPU 总容量。 |  | Optional: \{\} <br /> |
| `usage` _[InventoryUsage](#inventoryusage)_ | Usage 汇总整个清单中请求和已分配的节点。 |  | Optional: \{\} <br /> |
| `rackGroups` _[RackGroupStatus](#rackgroupstatus) array_ | RackGroups 包含每组的容量和使用量汇总。 |  | Optional: \{\} <br /> |
| `conditions` _[Condition](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#condition-v1-meta) array_ | Conditions 报告最近的清单接纳和编程结果。 |  | Optional: \{\} <br /> |


#### SGPUNode



SGPUNode 是机架中的一个同构逻辑节点。



_出现于：_
- [SGPURackProfileSpec](#sgpurackprofilespec)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `gpus` _[SGPUGPUs](#sgpugpus)_ | GPUs 定义每个节点上的同构模拟 GPU 集合。 |  |  |
| `host` _[SGPUHost](#sgpuhost)_ | Host 描述每个节点呈现的 CPU 和内存。 |  | Optional: \{\} <br /> |
| `topology` _[SGPUTopology](#sgputopology)_ | Topology 描述 PCIe 插槽和可选的 GPU/网络 fabric。 |  | Required: \{\} <br /> |


#### SGPUNodeReference



SGPUNodeReference 标识一个确切的 Kubernetes 节点实例。



_出现于：_
- [SGPURackNode](#sgpuracknode)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `name` _string_ | Name 是 Kubernetes 节点名称。 |  | MinLength: 1 <br /> |
| `uid` _[UID](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#uid-types-pkg)_ | UID 用于把该节点实例与同名替代者区分开。 |  |  |


#### SGPURack



SGPURack 是由控制器拥有的单个清单机架的物化。其控制器所有者引用标识由 spec.inventoryRef 固定的清单。





| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `apiVersion` _string_ | `mokka.nvidia.com/v1alpha1` | | |
| `kind` _string_ | `SGPURack` | | |
| `metadata` _[ObjectMeta](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#objectmeta-v1-meta)_ | 有关 `metadata` 的字段，请参阅 Kubernetes API 文档。 |  |  |
| `spec` _[SGPURackSpec](#sgpurackspec)_ | Spec 是渲染后的机架定义及其逻辑节点绑定。 |  |  |
| `status` _[SGPURackStatus](#sgpurackstatus)_ | Status 汇总该机架的持久节点分配。 |  | Optional: \{\} <br /> |


#### SGPURackGPU



SGPURackGPU 包含确定性的标识和结构性放置。



_出现于：_
- [SGPURackNode](#sgpuracknode)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `index` _integer_ | Index 是逻辑节点内从零开始的 GPU 索引。 |  | Maximum: 63 <br />Minimum: 0 <br /> |
| `uuid` _string_ | UUID 是稳定的 GPU 标识符。 |  | Pattern: `^GPU-[0-9a-f]\{8\}-[0-9a-f]\{4\}-[0-9a-f]\{4\}-[0-9a-f]\{4\}-[0-9a-f]\{12\}$` <br /> |
| `serial` _string_ | Serial 是 GPU 序列号。 |  | MinLength: 1 <br /> |
| `minorNumber` _integer_ | MinorNumber 是向工作负载暴露的设备次设备号。 |  | Minimum: 0 <br /> |
| `pciAddress` _string_ | PCIAddress 是 GPU 的 PCI 总线地址。 |  | Pattern: `^[0-9a-f]\{4\}:[0-9a-f]\{2\}:[0-9a-f]\{2\}\.[0-7]$` <br /> |
| `rootComplex` _string_ | RootComplex 标识连接到该 GPU 的 PCI root complex。 |  | Pattern: `^pci[0-9a-f]\{4\}:[0-9a-f]\{2\}$` <br /> |
| `numaNode` _integer_ | NUMANode 是与该 GPU 关联的宿主 NUMA 节点。 |  | Minimum: 0 <br /> |
| `hostProcessorIndex` _integer_ | HostProcessorIndex 是与该 GPU 关联的宿主 CPU 索引。 |  | Minimum: 0 <br /> |


#### SGPURackIdentity



SGPURackIdentity 包含确定性的机架坐标和 fabric 身份。



_出现于：_
- [SGPURackSpec](#sgpurackspec)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `rackGroup` _string_ | RackGroup 是源清单中的机架组标识符。 |  | Format: dns1123Label <br />MaxLength: 63 <br />MinLength: 1 <br /> |
| `rackIndex` _integer_ | RackIndex 是其组内从零开始的机架索引。 |  | Minimum: 0 <br /> |
| `fabricUUID` _string_ | FabricUUID 标识与该机架关联的 fabric 域。 |  | Format: uuid <br /> |
| `cliqueID` _integer_ | CliqueID 保留给未来的多 clique 布局，当前保持为零。 |  | Maximum: 0 <br />Minimum: 0 <br /> |


#### SGPURackInventoryReference



SGPURackInventoryReference 把机架固定到确切的清单实例。



_出现于：_
- [SGPURackSpec](#sgpurackspec)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `name` _string_ | Name 是清单名称。 |  | MinLength: 1 <br /> |
| `uid` _[UID](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#uid-types-pkg)_ | UID 把该引用固定到确切的清单实例。 |  |  |


#### SGPURackNode



SGPURackNode 是一个逻辑节点及其可选的 Kubernetes 节点绑定。



_出现于：_
- [SGPURackSpec](#sgpurackspec)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `index` _integer_ | Index 是机架内从零开始的逻辑节点索引。 |  | Maximum: 1023 <br />Minimum: 0 <br /> |
| `nodeRef` _[SGPUNodeReference](#sgpunodereference)_ | 逻辑节点未绑定时 NodeRef 不存在。其 UID 用于区分<br />复用同一名称的替代 Kubernetes 节点。 |  | Optional: \{\} <br /> |
| `gpus` _[SGPURackGPU](#sgpurackgpu) array_ | GPU 为逻辑节点渲染一次，不依赖其绑定。 |  | MaxItems: 64 <br />MinItems: 1 <br /> |


#### SGPURackProfile



SGPURackProfile 是模拟 GPU 机架的静态形态。





| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `apiVersion` _string_ | `mokka.nvidia.com/v1alpha1` | | |
| `kind` _string_ | `SGPURackProfile` | | |
| `metadata` _[ObjectMeta](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#objectmeta-v1-meta)_ | 有关 `metadata` 的字段，请参阅 Kubernetes API 文档。 |  |  |
| `spec` _[SGPURackProfileSpec](#sgpurackprofilespec)_ | Spec 定义硬件形态以及默认的软件/运行时元数据。 |  |  |


#### SGPURackProfileDefaults



SGPURackProfileDefaults 是新 sGPU 的初始运行时状态。



_出现于：_
- [SGPURackProfileSpec](#sgpurackprofilespec)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `runtime` _[RuntimeState](#runtimestate)_ | Runtime 为新物化的 GPU 提供初始的稀疏运行时设置。 |  | Optional: \{\} <br /> |


#### SGPURackProfileReference



SGPURackProfileReference 把渲染数据固定到确切的 profile 修订。



_出现于：_
- [SGPURackSpec](#sgpurackspec)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `name` _string_ | Name 是 profile 名称。 |  | MinLength: 1 <br /> |
| `uid` _[UID](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#uid-types-pkg)_ | UID 把该引用固定到确切的 profile 实例。 |  |  |
| `generation` _integer_ | Generation 记录用于渲染的 profile generation。 |  | Minimum: 1 <br /> |
| `revision` _string_ | Revision 标识渲染出的 profile 内容，与其名称无关。 |  | Pattern: `^[0-9a-f]\{64\}$` <br /> |


#### SGPURackProfileSpec



SGPURackProfileSpec 是一个逻辑机架 profile。



_出现于：_
- [SGPURackProfile](#sgpurackprofile)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `rack` _[SGPURackShape](#sgpurackshape)_ | Rack 是该 profile 描述的逻辑机架形态。 |  |  |
| `node` _[SGPUNode](#sgpunode)_ | Node 描述每个机架节点共享的模拟硬件。 |  |  |
| `software` _[SGPUSoftware](#sgpusoftware)_ | Software 记录可选的驱动、NVML 和 CUDA 版本。 |  | Optional: \{\} <br /> |
| `defaults` _[SGPURackProfileDefaults](#sgpurackprofiledefaults)_ | Defaults 为新物化的 GPU 提供初始运行时设置。 |  | Optional: \{\} <br /> |


#### SGPURackShape



SGPURackShape 定义逻辑机架模板的尺寸。它是嵌套的 profile 数据，而非已物化的 SGPURack 资源。



_出现于：_
- [SGPURackProfileSpec](#sgpurackprofilespec)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `nodesPerRack` _integer_ | NodesPerRack 是每个机架中的逻辑节点数量。 |  | Maximum: 1024 <br />Minimum: 1 <br /> |


#### SGPURackSpec



SGPURackSpec 是具有持久逻辑节点绑定的渲染机架。



_出现于：_
- [SGPURack](#sgpurack)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `inventoryRef` _[SGPURackInventoryReference](#sgpurackinventoryreference)_ | InventoryRef 把所有权固定到确切的清单实例，使同名替代者无法继承该机架。 |  |  |
| `profileRef` _[SGPURackProfileReference](#sgpurackprofilereference)_ | ProfileRef 记录用于渲染该机架的确切 profile 修订。 |  |  |
| `identity` _[SGPURackIdentity](#sgpurackidentity)_ | Identity 是 agent 和拓扑消费者使用的稳定坐标。 |  |  |
| `nodes` _[SGPURackNode](#sgpuracknode) array_ | 当 Kubernetes 节点分配发生变化时，Nodes 保留稳定的设备身份。<br />Fabric 和网络能力保留在所引用的 profile 中。 |  | MaxItems: 1024 <br />MinItems: 1 <br /> |


#### SGPURackStatus



SGPURackStatus 汇总持久的逻辑节点分配。



_出现于：_
- [SGPURack](#sgpurack)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `observedGeneration` _integer_ | ObservedGeneration 是此状态所反映的最新 spec generation。 |  | Minimum: 0 <br />Optional: \{\} <br /> |
| `assignedNodes` _integer_ | AssignedNodes 是具有确切 Kubernetes 节点绑定的逻辑节点数量。 |  | Minimum: 0 <br />Optional: \{\} <br /> |
| `conditions` _[Condition](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#condition-v1-meta) array_ | Conditions 报告渲染出的分配是否已为 agent 就绪。 |  | Optional: \{\} <br /> |


#### SGPURuntimePolicy



SGPURuntimePolicy 对一个 SGPUInventory 的子集应用稀疏的 RuntimeState 覆盖。





| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `apiVersion` _string_ | `mokka.nvidia.com/v1alpha1` | | |
| `kind` _string_ | `SGPURuntimePolicy` | | |
| `metadata` _[ObjectMeta](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#objectmeta-v1-meta)_ | 有关 `metadata` 的字段，请参阅 Kubernetes API 文档。 |  |  |
| `spec` _[SGPURuntimePolicySpec](#sgpuruntimepolicyspec)_ | Spec 定义目标选择和要应用的运行时设置。 |  |  |
| `status` _[SGPURuntimePolicyStatus](#sgpuruntimepolicystatus)_ | Status 汇总为该策略选择的目标作用域。 |  | Optional: \{\} <br /> |


#### SGPURuntimePolicySpec



SGPURuntimePolicySpec 把目标选择器与要应用的 RuntimeState 配对。



_出现于：_
- [SGPURuntimePolicy](#sgpuruntimepolicy)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `targetRef` _[PolicyTargetRef](#policytargetref)_ | TargetRef 选择清单以及可选的机架、节点和 GPU 子集。 |  |  |
| `runtime` _[RuntimeState](#runtimestate)_ | Runtime 包含要应用于所选 GPU 的稀疏设置。 |  | Optional: \{\} <br /> |


#### SGPURuntimePolicyStatus



SGPURuntimePolicyStatus 报告策略是否被接受，并保存 spec.targetRef 各维度的逗号连接反规范化值，用于打印列。



_出现于：_
- [SGPURuntimePolicy](#sgpuruntimepolicy)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `rackGroupsSummary` _string_ | RackGroupsSummary 是以逗号分隔的所选机架组 ID。 |  | Optional: \{\} <br /> |
| `rackIndexesSummary` _string_ | RackIndexesSummary 是以逗号分隔的所选机架索引。 |  | Optional: \{\} <br /> |
| `nodeIndexesSummary` _string_ | NodeIndexesSummary 是以逗号分隔的所选逻辑节点索引。 |  | Optional: \{\} <br /> |
| `gpuIndexesSummary` _string_ | GPUIndexesSummary 是以逗号分隔的所选 GPU 索引。 |  | Optional: \{\} <br /> |
| `conditions` _[Condition](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.32/#condition-v1-meta) array_ | Conditions 报告策略是否被接受。 |  | Optional: \{\} <br /> |


#### SGPUSoftware



SGPUSoftware 是驱动/NVML/CUDA 版本。



_出现于：_
- [SGPURackProfileSpec](#sgpurackprofilespec)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `driverVersion` _string_ | DriverVersion 是 NVIDIA 驱动版本。 |  | Optional: \{\} <br /> |
| `nvmlVersion` _string_ | NVMLVersion 是 NVIDIA 管理库版本。 |  | Optional: \{\} <br /> |
| `cudaVersion` _string_ | CUDAVersion 是 CUDA 版本。 |  | Optional: \{\} <br /> |


#### SGPUTopology



SGPUTopology 是节点可见的 PCIe 插槽、GPU fabric 和网络。



_出现于：_
- [SGPUNode](#sgpunode)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `gpuSlots` _[GPUSlot](#gpuslot) array_ | GPUSlots 把每个 GPU 索引映射到其 PCI 和宿主拓扑坐标。 |  | MaxItems: 64 <br />MinItems: 1 <br /> |
| `gpuFabric` _[GPUFabric](#gpufabric)_ | GPUFabric 描述可选的 NVLink/NVSwitch 连接。 |  | Optional: \{\} <br /> |
| `network` _[NetworkTopology](#networktopology)_ | Network 描述可选的带外网络适配器。 |  | Optional: \{\} <br /> |


#### SupportedClocks



SupportedClocks 把显存时钟与支持的 graphics 时钟配对。



_出现于：_
- [GPUClocks](#gpuclocks)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `memoryMHz` _integer_ | MemoryMHz 是与 GraphicsMHz 值配对的显存时钟频率。 |  |  |
| `graphicsMHz` _integer array_ | GraphicsMHz 列出在 MemoryMHz 下支持的 graphics 时钟频率。 |  |  |


#### TemperatureTelemetry



TemperatureTelemetry 驱动合成的 GPU/显存温度。



_出现于：_
- [RuntimeTelemetry](#runtimetelemetry)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `mode` _string_ | Mode 选择恒定或生成的温度值。 |  | Enum: [Fixed Pattern] <br />Optional: \{\} <br /> |
| `gpuCelsius` _integer_ | GPUCelsius 是模拟的 GPU 温度，单位摄氏度。 |  | Optional: \{\} <br /> |
| `memoryCelsius` _integer_ | MemoryCelsius 是模拟的显存温度，单位摄氏度。 |  | Optional: \{\} <br /> |


#### UtilizationPattern



UtilizationPattern 塑造生成的利用率曲线。



_出现于：_
- [UtilizationTelemetry](#utilizationtelemetry)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `type` _string_ | Type 选择生成的利用率曲线的形状。 |  | Enum: [Steady Bursty Wave] <br />Optional: \{\} <br /> |
| `gpuPercent` _[PercentRange](#percentrange)_ | GPUPercent 设置 GPU 利用率的最小和最大百分比。 |  | Optional: \{\} <br /> |
| `memoryPercent` _[PercentRange](#percentrange)_ | MemoryPercent 设置显存利用率的最小和最大百分比。 |  | Optional: \{\} <br /> |


#### UtilizationTelemetry



UtilizationTelemetry 驱动合成的 GPU/显存利用率。



_出现于：_
- [RuntimeTelemetry](#runtimetelemetry)

| 字段 | 描述 | 默认值 | 校验 |
| --- | --- | --- | --- |
| `mode` _string_ | Mode 选择按模式生成或固定的利用率值。 |  | Enum: [Pattern Fixed] <br />Optional: \{\} <br /> |
| `pattern` _[UtilizationPattern](#utilizationpattern)_ | Pattern 定义随时间生成的利用率值。 |  | Optional: \{\} <br /> |
