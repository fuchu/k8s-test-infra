# MIG

profile 中 `mig:` 块的一切含义：什么让板卡支持 MIG、说明可切分方式的分区表、该表放在哪里，以及声明布局的两种方式。

本文是参考文档。关于安装已分区的节点并把 Pod 调度到分片上，参见 [MIG 切分指南](guides/mig/README.md)；profile 的其余部分参见[配置参考](configuration.md)。

## 什么让板卡支持 MIG

```yaml
device_defaults:
  mig:
    mode_current: "disabled"
    mode_pending: "disabled"
    max_gpu_instances: 7
```

这三个键就是 `device_defaults.mig` 的全部——它们是芯片的属性，而且很小。板卡的分区表 `supported_profiles` 有数百行几何数据，存放在[独立文档](#declaring-the-profile-table)中。逐设备的 `mig:` 块声明的是布局而非能力，见[按数量声明布局](#declaring-a-layout-by-count)；它继承板卡的表和实例上限，因此设备只需声明自己的模式和分区。

没有自带 profile 声明分区。`max_gpu_instances` 是板卡的能力；如何切分是部署选择，在 chart 下即 `gpu.mig.gpuInstances`——只要设置了 `gpu.mig.enabled`，任何板卡都必须提供它。完全没有 `mig` 块的板卡——`t4`、`l40s`——不支持 MIG，NVML 对它们应答 `NVML_ERROR_NOT_SUPPORTED`，与真实硬件一样。

## 声明 profile 表

`supported_profiles` 是板卡的 MIG profile 表，即 `nvidia-smi mig -lgip` 打印的那些行。它用 YAML 声明，而不是在 Go 中按设备名解析，因此让 mock 支持新板卡只需改 YAML。该表是独立文档，不是 profile 的一部分——引擎如何找到它，参见[表放在哪里](#where-the-table-lives)。每一行完整描述其分区——跨越的分片、板卡公布的 ID、可占用的槽位以及提供的计算实例——引擎直接报告这些值，而不是相互推导。仍有一项几何数据是合成的；见下文的[计算实例 placement](#compute-instance-placements)。下面是 `h100` 表文档的全部内容，以及其七行中的完整一行：

```yaml
version: 1               # 会被解码，目前还没有任何东西读取它
supported_profiles:
  - name: "1g.10gb"
    nvml_profile: "1_SLICE"
    slices: 1
    profile_id: 19
    instances: 7
    memory_mb: 10240
    multiprocessors: 16
    copy_engines: 1
    decoders: 1
    jpeg: 1
    placements:            # 内存单元，不是计算分片
      - {start: 0, size: 1}
      - {start: 1, size: 1}
      - {start: 2, size: 1}
      - {start: 3, size: 1}
      - {start: 4, size: 1}
      - {start: 5, size: 1}
      - {start: 6, size: 1}
    compute_instances:     # 该行的 `mig -lcip` 列表
      - nvml_profile: "1_SLICE"
        slices: 1
        instances: 1
        multiprocessors: 16
        shared_copy_engines: 1
        decoders: 1
        jpeg: 1
      - nvml_profile: "1_SLICE_REV1"
        slices: 1
        instances: 1
        multiprocessors: 16
        shared_copy_engines: 1
        decoders: 1
        jpeg: 1
```

`supported_profiles` 位于表文档的根部，引擎在校验前把它挂到 `device_defaults.mig.supported_profiles` 上，因此无论写在哪里，行都按同样方式检查。

| 字段 | 含义 |
|---|---|
| `name` | 列表打印的内容以及集群中的写法，如 `1g.10gb`、`1g.10gb+me`。必需，两行同名会被拒绝——go-nvlib 从它派生 `nvidia.com/mig-<name>` 资源名 |
| `nvml_profile` | 该行是哪个 NVML profile，取 `NVML_GPU_INSTANCE_PROFILE_*` 常量的后缀：`1_SLICE`、`1_SLICE_REV1`、`2_SLICE`、`7_SLICE`。它是 `nvmlDeviceGetGpuInstanceProfileInfo` 的 `profile` 参数，也是所有查找的键——而下面的 `profile_id` 是同一次调用*返回*的数字。未知后缀或重复声明会在加载时被拒绝 |
| `slices` | 分区跨越多少个计算分片。枚举本身带有相同宽度，两者不一致的行会被拒绝，因此这是对同一事实的第二次读取，而不是第二个来源。省略则取枚举的宽度 |
| `profile_id` | 板卡为该 profile 公布的 ID：`mig -lgip` 的 `ID` 列，也是 `mig -cgi <id>` 接受的值。它不是 NVML 枚举——A100 的 `1g.5gb` 绑定 `1_SLICE` 却公布 `19`。两行公布同一 ID 会被拒绝，这也能抓到省略该键的行，因为省略的 `profile_id` 会公布 `0` |
| `instances` | 板卡一次提供多少个该 profile。加载时受该宽度分区能装进板卡的数量限制：2 分片 profile 在 7 分片板卡上只能放三个，而不是七个 |
| `memory_mb` | 分区的显存。加载时会与 `name` 宣称的大小宽松比对——真实分配会略少于名称，越宽的 profile 差得越多 |
| `placements` | 该分区在板卡上可占用的槽位，以内存单元计。必需——见下文 |
| `compute_instances` | 该行的 `nvidia-smi mig -lcip` 列表。必需——见下文 |
| `multiprocessors`、`copy_engines`、`decoders`、`encoders`、`jpeg`、`ofa` | 列表报告的引擎数量。省略为零 |

### REV 后缀区分什么

三个枚举后缀命名相同计算宽度的变体，区别它们的不是计算能力。

`1_SLICE` 是普通的分片分区。`1_SLICE_REV1` 是**媒体扩展**，列表中写作 `+me`：相同的分片和显存，但持有板卡的视频引擎。这就是为什么 `a100` 的 `1g.5gb` 不声明 `decoders`，而 `1g.5gb+me` 在计算和显存完全相同的情况下声明了一个 decoder、一个 JPEG 引擎和一个 OFA，也是为什么它提供 `instances: 1`——板卡只有一套这样的引擎可以给出。`1_SLICE_REV2` 是**双倍显存**变体：仍是一个计算分片，但占两个内存单元而不是一个，因此按其容纳的更大显存命名——`a100` 上是 `1g.10gb`，`b200` 上是 `1g.45gb`——七分片板卡只能放四个。

计算实例枚举把 `REV1` 后缀复用于同一思路但不同对象，因此 `compute_instances` 中命名它的条目指的是具备媒体能力的计算实例形态，而不是 `+me` GPU 实例。它只在所处的 GPU 实例有视频引擎可共享时才报告视频引擎：`a100` 的 `1g.5gb` 下的条目不声明任何视频引擎，而 `1g.5gb+me` 下同名条目声明全部三个。

### Placements

`placements` 以**内存单元而非计算分片**计量，这是在新板卡上最容易搞错的一点。板卡的内存单元数等于大于或等于其 `max_gpu_instances` 的下一个二的幂——7 分片板卡为 8，4 分片板卡为 4。因此占板卡一半内存的 `3g` 分区占用八个单元中的四个，整板的 `7g` 是一个 `{start: 0, size: 8}` 而不是 size 7。7 分片板卡上的 `1g` 行是七个 size 1 的 start，而占八分之二内存的单分片行——`h100` 的 `1g.20gb`——是四个 size 2 的 start。

加载时会拒绝不是二的幂的 `size`，因为单元恰好划分板卡且每个 start 都按自身 size 对齐；这让 `1g`、`2g` 和 `3g` 的 placement 能共存而不部分重叠，也排除了 size 为零。还会拒绝 `start` 加 `size` 超出板卡内存单元，以及两个 placement 共享一个 `start`——那会把同一个槽位通告两次。

### 计算实例

`compute_instances` 是该行的 `mig -lcip` 列表。GPU 实例并不提供所有能装进去的宽度——7 分片实例提供 `3c` 和 `4c`，然后跳到 `7c`——因此列表按行声明，而不是按宽度枚举。

| 字段 | 含义 |
|---|---|
| `nvml_profile` | 与上面相同的键，绑定到另一个枚举族：`NVML_COMPUTE_INSTANCE_PROFILE_*` 常量的后缀，`nvmlGpuInstanceGetComputeInstanceProfileInfo` 接受它。两族共享后缀词汇，因此一个值指的是哪一族由它所在的块决定，而不是由值本身决定。未知或在单行内重复声明会被拒绝 |
| `slices` | 与 GPU 实例的 `slices` 一样按枚举交叉检查，并且受它所处 GPU 实例宽度的限制 |
| `instances` | 该计算实例在 GPU 实例中能放多少个：7g 中能放三个 2c，而不是七个 |
| `multiprocessors` | 该计算实例在 GPU 实例 SM 中的自身份额 |
| `shared_copy_engines`、`decoders`、`encoders`、`jpeg`、`ofa` | NVML 的 `Shared*` 计数。GPU 实例内的每个计算实例都能看到其所有固定功能引擎，因此这些重复 GPU 实例的计数 |

### 计算实例 placements

有一项几何数据没有被声明而仍是合成的：计算实例在其 GPU 实例*内部*可占用的计算分片偏移，由 `nvmlGpuInstanceGetComputeInstancePossiblePlacements` 报告。`compute_instances` 条目没有对应字段，因此引擎从条目本身推导——`instances` 个 placement，每个占 `slices` 个分片，首尾相接——这正是 NVIDIA 发布列表的所有板卡遵循的布局。

对新板卡的后果是：如果计算实例 placement 不是首尾相接均布，这样的板卡目前无法用 YAML 表达。GPU 实例自身的 `placements` 不受影响——那些是按上述方式以内存单元声明的。

声明几何数据每块板卡约需 400 行 YAML，添加板卡的贡献者要全部写出来。这是用冗长换取表达力的刻意权衡，而非简化：算法只能产生它学过的几何，布局不匹配的板卡根本无法用 YAML 表达——而过去在 Go 中推导这些字段的地方，正是该领域缺陷的集中处。

行内容抄自 NVIDIA MIG 用户指南中的 [Supported MIG Profiles](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/latest/supported-mig-profiles.html)，每个自带 profile 的行都来自那里——各自在 YAML 注释中标明其来源表。该文档没有 GB200 或 GB300 的表，因此这两者取自板卡自身报告的内容。NVIDIA 只以图的形式发布 placement，因此这些是从图所示的布局抄录的。

## 表放在哪里

表是 profile 的同级文档，而不是其中一节。六块支持 MIG 的板卡各把表放两份，每个配置树一份：

| | 文件 |
|---|---|
| chart profile | `deployments/nvml-mock/helm/nvml-mock/profiles/mig/<board>.yaml` |
| 独立配置 | `pkg/gpu/mocknvml/configs/mock-nvml-config-<board>.mig.yaml` |

引擎在加载时按以下顺序解析表：

1. **`MOCK_MIG_PROFILES_CONFIG`。** 显式路径，也是 chart 设置的方式。
2. **从配置路径派生的同级文件**，替换扩展名：`config.yaml` → `config.mig.yaml`。这让本地运行和独立配置在完全不设置环境变量的情况下也能工作，也是五块板卡的表能共享一个目录而不冲突的原因。

在 chart 下无需设置任何东西。所选板卡的表渲染到自己的 ConfigMap `<fullname>-mig-profiles` 中，键为 `mig-profiles.yaml`，以只读方式挂载在 `/etc/nvml-mock/mig`；容器获得 `MOCK_MIG_PROFILES_CONFIG=/etc/nvml-mock/mig/mig-profiles.yaml`。chart 的 `mig/` 布局并不与所属 profile 同级，因此环境变量是在那里找到它的唯一途径。修改表会通过其自身的 `checksum/mig-profiles` 注解滚动 DaemonSet。

### 消费者 Pod 无需任何环境变量就能获得表

加载模拟库的 Pod——首先是 device plugin——读取[节点 agent](tools/node-agent.md)在模拟根目录下部署的配置，且不带任何指定表的环境变量。因此 agent 会把表写到它部署的每份配置旁边，命名为 `config.mig.yaml`，这些进程便通过上面的规则 2 解析它。无需向消费者挂载任何东西。

这就是为什么板卡的表必须到达 agent 而不只是库：配置旁边没有表的消费者会看到一块无法分区的板卡，device plugin 会在 NVML 报告分区的节点上通告整卡。

!!! warning "内联*且*外部声明的表会被拒绝"

    如果配置在 `device_defaults.mig` 下保留 `supported_profiles`，同时又有外部表被解析，加载会失败并点出两个来源。这是歧义而不是优先级问题：无论哪份表落败，都是某人编写却被 mock 静默忽略的表，而消费者通过 NVML 看到的任何内容都无法说明哪份文件在生效。

    对于按旧的单文件布局编写的外部配置，这就是迁移指引：**移动**表到同级文档，不要复制。chart 交付的内容没有内联声明表的，因此 chart 安装不会遇到这种情况。

### 无法解析到表的板卡不支持 MIG

表缺失不是加载错误。板卡只是没有分区表，这与声明没有 profile 的板卡含义相同——`t4` 和 `l40s` 一直就是这种板卡——因此 NVML 对其上的每个 MIG 调用应答 `NVML_ERROR_NOT_SUPPORTED`，配置的其余部分正常加载和提供服务。唯一痕迹是一条 debug 级别日志：

```text
[CONFIG] MIG profiles: no table at /etc/nvml-mock/mig/mig-profiles.yaml, board is not MIG-capable
```

这是刻意的：可能丢失的是 Kubernetes 挂载，因为一个挂载没到就让整块板卡的配置失效，比一块报告不支持 MIG 的板卡更糟。代价是*本应*能分区的板卡和真正不能的板卡从外部看起来完全一样。

因此，如果一块本应能分区的板卡却不能——`nvidia-smi mig` 拒绝它，`NVML_ERROR_NOT_SUPPORTED` 到达消费者——首先要检查的就是表，而那条 debug 日志就是方法。每个进程都自行加载引擎，因此开一个带调试日志的进程：

```bash
kubectl -n mokka exec ds/nvml-mock -- env MOCK_NVML_DEBUG=1 \
  nvidia-smi mig -lgip 2>&1 | grep 'MIG profiles'
```

该行会打印尝试过的确切路径，从而判断是环境变量、挂载还是文件名出错。完全没有这一行意味着没有路径解析成功：`MOCK_MIG_PROFILES_CONFIG` 和配置路径都没有设置。

其他情况都是运维编写错误，确实会导致加载失败，并给出点名文件的错误：不可读、无法解析、表文件声明零行、某一行未通过上述校验，或内联与外部冲突。

## 按数量声明布局

`gpu_instances` 请求若干个相同的实例。按名称或板卡公布的 ID 指定每个 profile：

```yaml
devices:
  - index: 0
    mig:
      mode_current: "enabled"
      gpu_instances:
        - profile: "3g.20gb"   # 按名称
          count: 2
        - profile_id: 19       # 按 ID：A100 上的 1g.5gb
          count: 1
```

| 字段 | 含义 |
|---|---|
| `profile` | 集群中的 profile 名称写法，如 `1g.10gb`、`1g.5gb+me` |
| `profile_id` | 用原始 ID 代替名称。两者必须恰好指定一个 |
| `count` | 相同实例的数量。默认为 1 |
| `compute_instances` | 每个 GPU 实例内的计算分片，采用相同的 `profile`/`profile_id`/`count` 结构。默认为一个横跨整个 GPU 实例的实例，这也是 `nvidia-mig-parted` 创建的形态 |

`profile_id` 是 `nvidia-smi mig -lgip` 的 `ID` 列中的数字，因此可以直接从该列表复制。那里出现哪些数字，取决于板卡自己的行声明的 `profile_id`：在 `a100` 和 `h100` 上，那些是 NVIDIA 公布的 ID，其中 `19` 是 1 分片分区，而 NVML 对同一分区的 profile 枚举完全是另一个数字。NVIDIA 没有为 Blackwell 发布 `mig -lgip` 列表，因此 `b200`、`gb200` 和 `gb300` 改而声明每行自己的 NVML 枚举——在那里 `19` 不指向任何东西，`0` 才是 1 分片 profile 而不是整板。在这些板卡上请用 `profile` 按名称指定分区，ID 则从 mock 自身的列表中取，而不是从真实板卡取。`vrnvl72` 的数据读自真实托盘，因此它的 ID 是托盘自身的，与 `a100` 和 `h100` 一样。

## 显式声明布局

`instances` 精确说明存在哪些 GPU 实例及其创建时使用的 ID。数量无法表达带空洞的布局——删除三个实例中的实例 1，剩下的是 0 和 2，而 `count: 2` 会重新加载为 0 和 1——因此这是运行时变更记录的形态，也是实例需要固定 ID 时应使用的形态：

```yaml
device_defaults:
  mig:
    mode_current: "enabled"
    max_gpu_instances: 7
    instances:
      - id: 0
        profile: "1g.10gb"
        compute_instances:
          - id: 0
            profile: "1c"
      - id: 2                  # 刻意省略 1
        profile: "1g.10gb"
        placement_start: 2     # 可选；省略则取第一个空闲槽位
```

固定 ID 让非创建者进程也能按 ID 删除分区，并让 `nvidia-smi -L` 跨进程报告相同的 MIG UUID。布局变更按差量应用：只创建缺失的实例，只销毁多余的实例，因此已持有句柄的消费者能在重新分区后继续持有，而不是丢失句柄。

空的 `instances: []` 表示已启用 MIG 但删除了所有实例的板卡，这与省略该键不同；GPU 实例的 `compute_instances` 同理，因为删除最后一个计算实例是硬件确实存在的状态。
