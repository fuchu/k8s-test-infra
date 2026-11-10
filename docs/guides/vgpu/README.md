# vGPU 宿主机与客户机

NVIDIA vGPU 会把一块物理 GPU 拆成多个虚拟 GPU，再由 hypervisor 分配给虚拟机。Mokka 能在一台 CPU 节点上模拟拆分的**任意一侧**：作为**宿主机**时，它的 GPU 上运行着 vGPU manager 和一组 vGPU 实例；作为**客户机**时，它的 GPU 就是虚拟机看到的、已授权的 vGPU。`nvidia-smi` 读取两侧的方式和在真实硬件上一样，全程不需要 hypervisor 或虚拟机。

一个节点一次只能扮演其中一侧，由 `gpu.vgpu.mode` 决定。下面两个 values 文件对 A100 节点来说是完整可用的；其中每个字段的含义，以及每个配置块必须遵守的规则，都写在 [vGPU 参考](../../vgpu.md) 里。

## 模拟 vGPU 宿主机 <a id="simulate-a-vgpu-host"></a>

把下面内容存为 `host-values.yaml`。它声明了一种计算类型 `NVIDIA A100-4C`，并在每块 GPU 上跑两个该类型的实例。每次安装都必须自己声明类型，原因见[参考文档](../../vgpu.md#host)。

```yaml title="host-values.yaml"
gpu:
  profile: a100
  vgpu:
    mode: host
    hostVgpuMode: sriov
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

```bash
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  -f host-values.yaml \
  --wait

POD=$(kubectl -n mokka get pod -l app.kubernetes.io/name=nvml-mock \
  -o jsonpath='{.items[0].metadata.name}')
kubectl -n mokka exec "$POD" -- nvidia-smi vgpu
```

每块 GPU 都会列出自己的两个实例（下面输出只截取 8 块 GPU 中的前两块）：

!!! note "国内镜像拉取"
    若 `ghcr.io` 等镜像源拉取缓慢或超时，可给 `nvml-mock` 加 `--set global.imageRegistry=harbor.heilansc.com/ghcr.io`。具体做法与示例见[国内镜像加速](../README.md#domestic-mirror)。

```text
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 580.65.06              Driver Version: 550.163.01                |
|---------------------------------+------------------------------+------------+
| GPU  Name                       | Bus-Id                       | GPU-Util   |
|      vGPU ID     Name           | VM ID     VM Name            | vGPU-Util  |
|=================================+==============================+============|
|  0   NVIDIA A100-SXM4-40GB      | 00000000:07:00.0             |  50%       |
|      1001        NVIDIA A100-4C | f641...                      |   N/A      |
|      1002        NVIDIA A100-4C | 2385...                      |   N/A      |
+---------------------------------+------------------------------+------------+
|  1   NVIDIA A100-SXM4-40GB      | 00000000:0F:00.0             |  50%       |
|      2001        NVIDIA A100-4C | ff61...                      |   N/A      |
|      2002        NVIDIA A100-4C | 10bc...                      |   N/A      |
+---------------------------------+------------------------------+------------+
```

vGPU ID 里编码了它属于哪块 GPU：GPU 0 的实例是 `1001`、`1002`，GPU 1 的是 `2001`、`2002`。具体规则见[参考文档](../../vgpu.md#host-identities)。

`nvidia-smi vgpu -q -i 0` 会展开某块 GPU 上的实例详情（下面只截取第一个实例的身份与授权相关行）：

```text
GPU 00000000:07:00.0
    Active vGPUs                          : 2
    vGPU ID                               : 1001
        VM UUID                           : f641e950-f095-42e5-aaed-85f4222ef023
        VM Name                           : 
        vGPU Name                         : NVIDIA A100-4C
        vGPU Type                         : 474
        vGPU UUID                         : 0f7566ae-0643-78b0-b8ab-4d9865eb8ff4
        Guest Driver Version              : N/A
        License Status                    : Licensed (Expiry: Permanent)
        GPU Instance ID                   : N/A
        Placement ID                      : N/A
        Accounting Mode                   : Disabled
        ECC Mode                          : Enabled
```

这些 UUID 由 GPU 自身的 UUID 推导而来，所以每次运行结果都一样。有些行显示 `N/A`，是因为模拟层没有实现对应调用（例如 `Placement ID`），完整清单见[参考文档](../../vgpu.md#host-behaviour)。`GPU Instance ID` 显示 `N/A` 则是另一个原因：该实例是时间片复用的，背后并没有 MIG GPU instance。

`nvidia-smi vgpu -s` 列出支持的类型，`nvidia-smi vgpu -c` 列出当前还能创建的类型。10 个槽位用了 2 个时，两条命令都会输出 `NVIDIA A100-4C`。

## 模拟 vGPU 客户机

把下面内容存为 `guest-values.yaml`。它会让每块 GPU 都成为已授权的 vGPU，宿主机驱动版本 `550.163.01`、分支 `r550_00`。

```yaml title="guest-values.yaml"
gpu:
  profile: a100
  vgpu:
    mode: guest
    guest:
      host_driver_version: "550.163.01"
      host_driver_branch: "r550_00"
      license:
        product: "NVIDIA Virtual Compute Server"
        state: licensed
```

这份配置会**替换**宿主机那份，所以是对同一个 release 做升级：

```bash
helm upgrade --install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  -f guest-values.yaml \
  --wait

POD=$(kubectl -n mokka get pod -l app.kubernetes.io/name=nvml-mock \
  -o jsonpath='{.items[0].metadata.name}')
kubectl -n mokka exec "$POD" -- nvidia-smi -q -i 0
```

GPU 会把自己报告成 vGPU，并展示授权信息（下面只截取虚拟化和授权相关行）：

```text
    GPU Virtualization Mode
        Virtualization Mode               : VGPU
        Host VGPU Mode                    : N/A
        vGPU Heterogeneous Mode           : N/A
    vGPU Software Licensed Product
        Product Name                      : NVIDIA Virtual Compute Server
        License Status                    : Licensed (Expiry: Permanent)
```

宿主机驱动版本和分支不会出现在 `nvidia-smi` 里，它们是通过[参考文档](../../vgpu.md#guest)描述的 PCI capability 传给 GPU Feature Discovery 的。

## 排查问题

**`helm install` 报 `gpu.vgpu.mode is host but gpu.vgpu.types declares no vGPU types`。**
宿主机至少要有一种类型，缺少时 chart 会拒绝渲染，也就不会部署任何东西。像[上面的 `host-values.yaml`](#simulate-a-vgpu-host) 那样补上 `gpu.vgpu.types`。

**`helm install --wait` 超时，`nvml-mock` Pod 一直停在 `Init:0/1` 并反复重启。**
说明节点代理（node agent）拒绝了 `gpu.vgpu` 配置块，始终无法就绪，启动探针因此失败；而 `--wait` 只会把它报成超时。节点代理的日志里会点名具体字段：

```bash
kubectl -n mokka logs "$POD" -c node-agent | grep 'state source error'
```

```text
{"level":"warn","time":"2026-10-14T07:05:29.955Z","msg":"state source error; keeping cached state","error":"/etc/nvml-mock/config.yaml: device_defaults.virtualization.vgpu.instances[0].type: \"NVIDIA A100-8C\" is not a declared vGPU type"}
```

对照[校验规则](../../vgpu.md#validation)修正该字段，再执行一次 `helm upgrade`。

**`nvidia-smi vgpu` 提示 `No supported devices in vGPU mode`。**
该节点不是 vGPU 宿主机，即 `gpu.vgpu.mode` 不是 `host`。客户机节点也会输出这句话，因为 `nvidia-smi vgpu` 是宿主机侧的视图。

**在宿主机节点上执行 `nvidia-smi pmon` 失败。**
属于预期行为：`pmon` 会拒绝运行着 vGPU manager 的 GPU，真实 hypervisor 上也是如此。

## 延伸阅读

| 想了解 | 参见 |
|---|---|
| 全部 `gpu.vgpu` chart 配置项 | [vGPU：chart 之下](../../vgpu.md#under-the-chart) |
| 配置档的每个字段、身份规则以及未模拟的部分 | [vGPU](../../vgpu.md) |
| 配置档里 `virtualization` 块的其余内容 | [配置](../../configuration.md#virtualization) |
