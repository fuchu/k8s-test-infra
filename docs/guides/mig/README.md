# MIG 切分

多实例 GPU（MIG）能把一块物理 GPU 拆成几个互相隔离的分区。Mokka 在驱动层面模拟了这次拆分，因此 `nvidia-smi mig` 的行为和对着真实驱动时一样，NVIDIA device plugin 也无需任何改动就能跑。整个过程不涉及真的 A100、H100 或 B200。

人们说「把一块卡切开」时，其实指的是两件不同的事，二者不能混用：

| | 安装时切分 | 运行时切分 |
|---|---|---|
| 由谁决定 | chart values | `nvidia-smi mig` |
| 改变的是 | 节点**对外声明**的资源，以及 NVML 的报告内容 | 只改变 NVML 的报告内容 |
| Pod 能否调度到切片上 | 能 | 不能 |
| 适合验证 | 调度器、device plugin，以及任何涉及资源分配的逻辑 | 读取 NVML 的消费方，例如 DCGM 和 GFD |

!!! warning "节点启动时是什么布局，就只能分配什么布局"

    device plugin 是依据「驱动能力面」来分配的，而这份能力面由 Mokka 在 `nvml-mock` Pod 启动时准备一次。运行时重新切分只会改变它下面的 NVML 视图，不会改变这份能力面，所以用 `nvidia-smi` 新建的分区**看得见、但分配不了**。如果你打算往上调度东西，请在安装时就把布局定好。

## 安装时切分

每次启用 MIG 都要明确给出布局。只写 `gpu.mig.enabled=true` 会被拒绝，因为「怎么切一块卡」是部署决策，不是芯片本身的属性；而且一块支持 MIG 的卡如果开了 MIG 却不给布局，就一个 GPU 资源都不会发布出来。

```bash
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  --set gpu.profile=h100 \
  --set gpu.count=2 \
  --set gpu.mig.enabled=true \
  --set 'gpu.mig.gpuInstances[0].profile=1g.10gb' \
  --set 'gpu.mig.gpuInstances[0].count=7' \
  --wait
```

上面是两块板子、每块 7 个切片，所以节点上一共 14 个分区。

!!! note "国内镜像拉取"
    若 `ghcr.io` 等镜像源拉取缓慢或超时，可给 `nvml-mock` 加 `--set global.imageRegistry=harbor.heilansc.com/ghcr.io`。具体做法与示例见[国内镜像加速](../README.md#domestic-mirror)。

用最小的切片把一块板子填满，所有分区就会共用同一个规格名——这正是 device plugin 的 `migStrategy=single` 所要求的：

| 配置档 | 最小切片 | 每块板的实例数 |
|---|---|---|
| `a100` | `1g.5gb` | 7 |
| `h100` | `1g.10gb` | 7 |
| `b200` | `1g.23gb` | 7 |
| `gb200` | `1g.23gb` | 7 |
| `gb300` | `1g.35gb` | 7 |
| `vrnvl72` | `1g.35gb` | 7 |

`l40s` 和 `t4` 不支持切分。chart 会拒绝对它们启用 MIG，NVML 也会像在真实硬件上一样返回 `NVML_ERROR_NOT_SUPPORTED`。

每块板子支持的完整规格表（不只是最小切片）用 YAML 声明，每一行写明该分区的切片数、对外发布的 profile id、可放置位置和 compute instance。它是一份独立文档，由 chart 从专属的 ConfigMap 挂载进来；表结构以及引擎如何找到它，见 [`mig:` 参考](../../mig.md#declaring-the-profile-table)。如果某块板子的规格表没能送达，它就会以[「不支持 MIG」](../../mig.md#a-board-with-no-resolvable-table-is-not-mig-capable)的状态启动——支持 MIG 的板子却意外拒绝启用时，这是第一个要查的地方。

混合布局和固定实例 id 的写法见 [MIG](../../mig.md)。

## 查看分区

```bash
POD=$(kubectl -n mokka get pod -l app.kubernetes.io/name=nvml-mock \
  -o jsonpath='{.items[0].metadata.name}')

kubectl -n mokka exec "$POD" -- nvidia-smi -L
kubectl -n mokka exec "$POD" -- nvidia-smi mig -lgi
```

`nvidia-smi -L` 会用规格名和各自的 `MIG-…` UUID 来命名每个分区——这是最终交给工作负载的身份，不是母板 GPU 的。`mig -lgi` 列出这些分区背后的 GPU instance，`mig -lgip` 则报告每种规格还剩多少个可用。

## 运行时调整布局

这一步由 `nvidia-smi` 驱动，不是 `nvml-mock-ctl`。作用范围是单个节点，所以要在目标节点的 `nvml-mock` Pod 里执行；想改多个节点，就对每个 Pod 各来一遍。

安装时没给布局的板子，MIG 初始是关闭的，所以第一步要先启用。下面这些操作都不要求安装时就切好分。

```bash
SMI="kubectl -n mokka exec $POD -- nvidia-smi"

$SMI -i 0 -mig 1                   # 让 GPU 0 支持 MIG；此时还没有分区
$SMI mig -i 0 -cgi 1g.10gb -C      # 切出一个切片，并带上它的 compute instance
$SMI mig -lgi                      # 在另一个进程里读回结果
$SMI mig -i 0 -dci -gi 0 -ci 0     # 先删 compute instance
$SMI mig -i 0 -dgi -gi 0
$SMI -i 0 -mig 0                   # 恢复成一整块 GPU
```

改造这段流程之前，有四点值得先弄清楚。

`-C` 实际上是必需的。少了它，GPU instance 会出现在 `mig -lgi` 里，却没有对应的 compute instance，于是 `nvidia-smi -L` 里不会生成 MIG 设备，谁也用不了它。

删除要「由内向外」。NVML 拒绝销毁还挂着 compute instance 的 GPU instance，会返回 `NVML_ERROR_IN_USE`，所以得先删 compute instance。`nvidia-mig-parted` 和 MIG 用户指南用的也是这个顺序。

`-i` 是**限定范围**，不是「先选中再操作」。省略它，命令就会作用于节点上的每一块 GPU——在 8 块板的节点上，`mig -dgi -gi 0` 会一次删掉 8 个分区，而不是 1 个。

每次变更都会被记录下来，因此它不会随着发出命令的 `nvidia-smi` 进程退出而消失，并会在一个 TTL 周期内同步给节点上的其他消费方。正因为每个进程都有自己独立的引擎，这份记录才让分区对**另一个** `nvidia-smi`、对 DCGM、对 GFD 都可见。如果某次变更无法被记录，调用会直接以 `NVML_ERROR_NO_PERMISSION` 失败，而不是「只在一个进程里生效」——真实驱动在权限不足时也是这么报的。

`nvml-mock-ctl status` 可以查看已记录的布局，`nvml-mock-ctl reset` 可以清空它。这里刻意没有提供 `nvml-mock-ctl mig`：那相当于给 `nvidia-smi` 已有的接口再造一套非标准写法，还会变成同一份状态的第二个写入方。

## 把 Pod 调度到某个切片上

切分后的节点按「每个切片一个资源」而不是「每块板一个资源」来对外声明，所以上面的安装把 2 块 GPU 变成了 14 个可分配资源：

```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,GPUS:.status.allocatable.nvidia\.com/gpu'
```

对插件之上的各层来说，真正重要的是这个形态变化：同样一句 `nvidia.com/gpu: 1`，现在买到的是七分之一块 H100；Pod 会在 `NVIDIA_VISIBLE_DEVICES` 里拿到某个分区的 `MIG-…` UUID，外加两个 `/dev/nvidia-caps` 设备节点（一个对应 GPU instance，一个对应 compute instance）——支持 MIG 的运行时正是靠它们来打开分区的。

要做到这一点，需要按 [NVIDIA Device Plugin](../device-plugin.md) 部署 device plugin，并做两处调整：

- 加 `--mig-strategy=single`，把所有分区都发布成 `nvidia.com/gpu`。如果节点上各分区的规格不统一，它会直接拒绝该节点，这也是前面要用同一种切片尺寸填满每块板子的原因。
- 插件在构建设备映射时，会通过一个**硬编码的绝对路径** `/proc/driver/nvidia-caps/mig-minors` 解析 MIG 能力。读不到这个文件，插件就什么都不发布，而不会回退到整块 GPU。普通卷挂载也送不进去，因为 runc 拒绝任何目标位于 `/proc` 内的挂载。可行的办法是让一个特权容器把 Mokka 准备好的副本 bind-mount 到它自己的 `/proc/driver` 上。

`tests/e2e/go/assets/device-plugin-mock-mig.yaml` 里保留了一份可正常工作的 DaemonSet，两件事都做了，也包含等待节点代理完成准备的逻辑。

## 切片是怎么命名的

切片的名称表示它占**自己那块板子**的份额。项目内置的每个配置档都声明了 NVIDIA 命名时所依据的产品容量——`b200` 是 180 GiB、`gb200` 是 186、`gb300` 是 278——所以模拟层报出的名字和这些板子真实报出的一致。

某块板子接受哪些名字，请以 `nvidia-smi mig -lgip` 的输出为准，而不要查 NVIDIA 的 [Supported MIG Profiles](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/latest/supported-mig-profiles.html)：那份文档没有 GB200 或 GB300 的表格。不过对于真实 A100、H100、B200 逐规格支持哪些 MIG profile，它仍是权威参考，内置表格也正是据此整理的。

在 Blackwell 上请按 profile 名而不是 profile id 来指定分区：NVIDIA 没有为这些板子公布 profile id，因此模拟层返回的是 NVML 的枚举值，和真实板子列出的 id 对不上。

## 限制 <a id="limits"></a>

运行时重新切分只改 NVML 视图。`/dev/nvidia-caps` 和 `mig-minors` 表都是在 `nvml-mock` Pod 启动时准备一次的，所以由重新切分产生的分区永远不可分配，放多久都没用。清掉覆盖配置后，NVML 会回到安装时的布局，GPU instance id 和 MIG UUID 也与之前一致——那些暂存文件本来就是按这组身份写的。

项目不模拟 CUDA，因此 MIG 切片可以完成调度、Pod 也能被准入，但上面不会真的跑 kernel。

## 排查问题

**`helm install` 报 `gpu.mig.gpuInstances` 没有声明任何分区。**
说明只设了 `gpu.mig.enabled=true` 却没给布局，补上一个即可。

**`helm install` 报该配置档不是支持 MIG 的板子。**
该配置档没有声明 `mig.max_gpu_instances`；`l40s` 和 `t4` 无法切分。

**`nvidia-smi mig` 回答 `No MIG-enabled devices found`。**
所有 GPU 的 MIG 都是关着的（安装时不给布局就是这个状态）。先用 `nvidia-smi -i 0 -mig 1` 启用。注意 MIG 状态是按节点算的：如果启用成功了、下一条命令却说没启用，说明两次调用打到了不同的 Pod 上。

**`mig -dgi` 报 GPU instance 仍在占用。**
它还挂着 compute instance。和真实硬件上一样，先用 `mig -dci -gi <id> -ci <id>` 把 compute instance 删掉。

**启用 MIG 后可分配数仍是 0。**
插件在跑，但没找到能力表，或者找到了它不愿服务的分区。先确认布局是统一的（`migStrategy=single` 会拒绝混合规格），再看插件日志。

**`nvidia-smi` 能列出的分区却分配不了。**
它是运行时创建的，参见[限制](#limits)。清掉覆盖配置以恢复安装时的布局，或者重启该节点上的 `nvml-mock` Pod。

## 延伸阅读

| 想了解 | 参见 |
|---|---|
| `mig:` 配置结构，含按设备指定布局和固定实例 id | [MIG](../../mig.md) |
| 全部 chart 配置项 | [安装](../../helm-chart.md) |
| 整块 GPU 的分配，以及不带 MIG 的插件 | [NVIDIA Device Plugin](../device-plugin.md) |
| 运行时调整温度、功耗或健康状态 | [运行时控制](../../nvml-mock-ctl.md) |
