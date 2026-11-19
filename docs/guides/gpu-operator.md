# NVIDIA GPU Operator

让真实的 GPU Operator 跑在 Mokka 之上：device plugin、GPU Feature Discovery、DCGM 和 validator 都会启动，行为与在有硬件的节点上一样。

## 为什么需要覆盖配置

GPU Operator 的职责是安装和管理驱动。Mokka 已经提供了一个，因此必须让 operator 只消费驱动而不安装驱动——并把它指向 Mokka 部署驱动的位置。

下面的取值就是做这件事。用默认配置对 Mokka 节点安装 operator 会失败：driver DaemonSet 会尝试构建一个不可能存在的内核模块。

## 前置条件

- [Docker](https://docs.docker.com/get-started/get-docker/) 和 [Kind](https://kind.sigs.k8s.io/)
- [Helm 3.8 或更新版本](https://helm.sh/docs/intro/install/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)

约需 15 分钟，大部分时间用于拉取 operator 镜像。

## 第 1 步——创建集群

```bash
kind create cluster --name mokka-operator
```

operator 通过容器设备接口（CDI）解析 GPU，其自身 Pod 运行在 `nvidia` 运行时 handler 下。Mokka 的节点守护进程在第 2 步启动时会在 containerd 中把两者都配置好，因此集群无需其他准备。

## 第 2 步——安装 Mokka

```bash
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  --wait --timeout 120s
```

!!! note "国内镜像拉取"
    国内环境可给 Mokka chart 加 `--set global.imageRegistry=shharbor.heilansc.com/ghcr.io`。GPU Operator 的 operand 各自使用独立的镜像值，需在 `gpu-operator-values.yaml` 中逐个改为 `shharbor.heilansc.com/nvcr.io/...`；完整做法见[国内镜像加速](README.md#domestic-mirror)。

## 第 3 步——安装 GPU Operator

```bash
cat > gpu-operator-values.yaml <<'EOF'
driver:
  enabled: false
toolkit:
  enabled: false
dcgm:
  enabled: false
mig:
  strategy: none
migManager:
  enabled: false
nodeStatusExporter:
  enabled: false

cdi:
  enabled: true
  default: true

devicePlugin:
  enabled: true
  config:
    name: ""

gfd:
  enabled: true
  env:
    - name: GFD_MACHINE_TYPE_FILE
      value: "/etc/nvml-mock/machine-type"

dcgmExporter:
  enabled: true
  env:
    - name: DCGM_EXPORTER_COLLECT_INTERVAL
      value: "5000"

validator:
  driver:
    env:
      - name: DISABLE_DEV_CHAR_SYMLINK_CREATION
        value: "true"
  toolkit:
    env:
      - name: NVIDIA_VISIBLE_DEVICES
        value: "all"
  cuda:
    env:
      - name: WITH_WORKLOAD
        value: "false"
EOF

helm repo add nvidia https://helm.ngc.nvidia.com/nvidia && helm repo update

helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator --create-namespace \
  --version v26.3.3 \
  -f gpu-operator-values.yaml \
  --wait --timeout 600s
```

版本是刻意选定的。GPU Operator `v26.3.3` 是 CI 针对此流程验证过的版本。更新固定版本前，请先用 Mokka 验证新版本，因为它可能加入硬件检查或改变 chart schema。

!!! warning "用 values 文件，不要用 `--set`"

    每个 operand 都通过嵌套的 `env` 列表接收驱动根目录，而这种结构无法在命令行中存活：链接的 `--set gfd.env[0].name=…` 会静默丢弃值，`--set validator.cuda.env[0].value="false"` 会把本应是字符串的字段当成 bool，未加引号的 `env[0]` 还会在 Helm 看到之前被 zsh 做 glob 展开。values 文件是这一配置唯一可用的形式。

## 第 4 步——验证

```bash
kubectl -n gpu-operator wait --for=condition=ready pod --all --timeout=300s

kubectl get node "$NODE" -o jsonpath='{.status.allocatable.nvidia\.com/gpu}'
# 4 —— gb300 默认 profile 有四台设备

kubectl get node "$NODE" -o json \
  | jq '.metadata.labels | with_entries(select(.key | startswith("nvidia.com")))'
```

validator Pod 进入 `Completed` 才是有意义的信号：它会探测驱动根目录、验证 CDI 注入，并检查节点是否通告了 GPU。

## 每个取值在做什么

| 取值 | 原因 |
|---|---|
| `driver.enabled: false` | Mokka 就是驱动。真实的 DaemonSet 会试图构建内核模块 |
| `toolkit.enabled: false` | Mokka 的节点守护进程会安装 NVIDIA 容器运行时并注册 operator Pod 使用的 `nvidia` handler。operator 的 toolkit 在这里本来也跑不了：它需要 NVIDIA 内核模块 |
| `cdi.enabled` / `cdi.default` | 运行时读取 Mokka 生成的 `/var/run/cdi/nvidia.yaml`。这就是替代 toolkit operand 的东西 |
| `dcgm.enabled: false` | 单独的 nv-hostengine DaemonSet 是多余的：`dcgm-exporter` 已在进程内嵌入 host engine |
| `dcgmExporter.enabled: true` | 刻意保留——它通过 libdcgm 读取 mock，这正是要证明的内容之一 |
| `GFD_MACHINE_TYPE_FILE` | GFD 默认读取 `/sys/class/dmi/id/product_name`，在这里是 `kind`，而在没有 DMI 的宿主机上根本不存在。Mokka 会写自己的文件 |
| `mig.strategy: none` | 上面的安装没有对板卡分区，因此没有分片可通告，整卡才是正确的视图。MIG 本身是模拟的：[MIG 切分](mig/README.md)会切分板卡，并通过 `migStrategy=single` 的 device plugin 提供分片 |
| `validator.cuda.WITH_WORKLOAD: false` | CUDA 验证步骤会启动 kernel，而 [CUDA 未模拟](../faq.md#can-i-run-cuda-workloads-against-mokka) |
| `DISABLE_DEV_CHAR_SYMLINK_CREATION` | `/dev/char` 符号链接步骤会运行 `modprobe nvidia`，这在 Kind 容器中无法工作。Mokka 已部署这些节点 |

## 三个看起来不对但其实没问题的标签

运行后检查节点标签，会发现三个似乎与覆盖配置矛盾的标签。它们都不会造成问题。

- `nvidia.com/gpu.deploy.driver=true` 和 `nvidia.com/gpu.deploy.container-toolkit=true` 是 operator 自己的调度提示，由 GFD 为它管理的 operand 写入。它们的意思是"该节点有资格运行该 operand"，而不是"该 operand 在这里"。由于 `driver.enabled=false` 和 `toolkit.enabled=false`，对应的 DaemonSet 从未创建，因此这些标签没有可选对象。用 `kubectl -n gpu-operator get ds` 确认，两者都不存在。
- `nvidia.com/mig.capable=true` 看起来与 `mig.strategy: none` 矛盾，其实不然。板卡确实支持 MIG，mock 确实能分区；只是这次安装没有切分它，所以策略保持 `none`。切分请见 [MIG 切分](mig/README.md)指南。

## 排障

**validator 不断 crash-loop。** 它是静态链接的 Go 二进制，没有 shell，因此部署好的驱动根目录中少一个文件，看起来像是崩溃而不是报错。检查实际内容：

```bash
docker exec mokka-operator-control-plane ls -la /run/nvidia/driver/usr/lib64/libnvidia-ml.so*
```

**device plugin 报告 0 块 GPU。** 几乎总是 `driver.enabled` 或 `toolkit.enabled` 留了默认值——operator 在管理一个不存在的驱动。用 `helm get values gpu-operator` 确认。

**GFD 把节点标为 `kind`。** 覆盖配置中缺少 `GFD_MACHINE_TYPE_FILE`。

更多症状见[排障](../troubleshooting.md)。

## 从仓库检出运行

上面的步骤就是完整场景，从不需要仓库。如果你已克隆仓库，`run.sh` 会执行同样的安装并断言结果：

```bash
cd docs/guides/with-gpu-operator && ./run.sh
```

它安装 Mokka，检查 `nvidia-smi -L` 能在 mock Pod 中运行，用[`gpu-operator-values.yaml`](with-gpu-operator/gpu-operator-values.yaml)中同样的覆盖配置安装 operator，然后断言两件事——只有真实 operand 读取了 mock 才可能成立：某节点把 `nvidia.com/gpu` 列为可分配，且某节点带有 `nvidia.com/gpu.product` 标签。两者都会扫描所有节点，因为 operand 不容忍控制面的 `NoSchedule` taint。

| 变量 | 默认值 | 效果 |
|---|---|---|
| `GPU_PROFILE` | `gb300` | chart `profiles/` 目录下的任意 profile |
| `NAMESPACE` | `mokka-operator` | Mokka release 的命名空间 |
| `OPERATOR_NAMESPACE` | `gpu-operator` | GPU Operator release 的命名空间 |
| `GPU_OPERATOR_VERSION` | `v26.3.3` | 要安装的 GPU Operator chart 版本 |
| `NVML_MOCK_IMAGE` | `ghcr.io/nvidia/nvml-mock:latest` | 要安装的已发布镜像 |
| `BUILD_LOCAL` | `false` | 从源码构建镜像并用 `kind load` 旁加载 |
| `HELM_TIMEOUT` | `15m` | 每次 Helm 安装的等待预算 |
| `DEMO_ASSUME_YES` | `false` | 跳过确认提示和共置拒绝 |

!!! warning "每个集群一次只能有一个 Mokka release"

    chart 的 hostPath 挂载是固定的、与 release 无关（`/var/lib/nvml-mock`、`/var/run/cdi`、`/run/nvidia` 和 NFD features 目录），而且它容忍所有 taint。因此两个 Mokka release 会落到每个节点上，写入相同的逐节点状态，无论它们叫什么——也覆盖这些节点上真实 GPU 工作负载随后会读取的共享配置。如果集群中任何位置已经安装了另一个 Mokka release，`run.sh` 会以退出码 `4` 拒绝执行。

## 清理

```bash
kind delete cluster --name mokka-operator
```

## 相关链接

| 想了解 | 参见 |
|---|---|
| 所有 chart 取值 | [安装](../helm-chart.md) |
| 在 operator 监视时修改 GPU 状态 | [运行时控制](../nvml-mock-ctl.md) |
| 在 CI 中驱动这一切 | [在 CI/CD 中使用](ci-cd.md) |
| 建立在此技术栈之上的完整健康循环 | [NVSentinel](nv-sentinel/README.md) |
