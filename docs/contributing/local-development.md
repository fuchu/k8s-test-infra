# 本地开发

Mokka 的本地开发循环是 Kind 集群加上 [Tilt](https://tilt.dev/)。Tilt 负责构建镜像、安装 chart，并在保存时重新部署；集群只需创建一次，之后保持运行。

```bash
make cluster-create          # 仅一次
tilt up -- --gpu-operator    # 开发循环
```

## 前置条件

`docker`、`[kind](https://kind.sigs.k8s.io/)`、`[tilt](https://tilt.dev/)`、`helm`、`kubectl`，以及与 `go.mod` 匹配的 Go 工具链。

## 第 1 步——创建集群

有两种集群形态，通过 `PROFILE` 选择：

| `PROFILE` | 形态 | 集群名称 | 适用场景 |
|------------------|----------------------------------------------------------|------------------------|-----------------------------------|
| `default` | 1 个控制面 + 2 个 worker | `mokka` | 除 compute-domain 之外的所有场景 |
| `compute-domain` | 1 个控制面 + 4 个 worker，组成两个各含两节点的 NVLink clique | `mokka-compute-domain` | `--compute-domain`、`--topograph` |

```bash
make cluster-create                         # 默认
make cluster-create PROFILE=compute-domain

make cluster-delete                         # PROFILE 必须与创建时一致
```

默认形态同时支持单一 release 和按 worker 划分的机群，因此两者之间切换无需重建。

!!! note "控制面节点被刻意排除"
    `local/nvml-mock.values.yaml` 把 DaemonSet 固定到带有 `mokka.nvidia.com/type=sgpu` 标签的节点上。否则 mock 也会落到控制面上——它容忍所有 taint——而 GPU Operator、FGO 和 NFD 的 operand 会停在 `NoSchedule` taint 前。结果就是一个节点通告了驱动却无人消费。

## 第 2 步——启动 Tilt

不带任何 flag 的 `tilt up` 会使用 chart 的[默认 profile](../helm-chart.md#when-to-use-each-profile)，在所有 worker 上安装一个 nvml-mock release。其他一切都是 flag。

### 选择机群形态

| Flag | 效果 |
|---|---|
| `--gpu-profile <name>` | release 使用哪个 GPU profile，替代 chart 默认值。可选 `a100`、`h100`、`b200`、`gb200`、`gb300`、`l40s`、`t4`、`vrnvl72` |
| `--multi-gpu-profile` | 每个 worker 一个 release 而非整个机群一个：`a100` 用于 `worker-0`，`t4` 用于 `worker-1`。忽略 `--gpu-profile` |
| `--compute-domain` | 带 NVLink 拓扑覆盖的 GB200 profile。需要 `compute-domain` 集群 |
| `--nvmlmock-image <ref>` | 跳过镜像构建，把 release 固定到你已构建好的镜像，CI 就是这么做的 |

### 添加消费者

| Flag | 部署内容 |
|---|---|
| `--gpu-operator` | CDI 模式下的 NVIDIA GPU Operator |
| `--dra` | NVIDIA DRA driver |
| `--fgo` | Run:ai 的 fake-gpu-operator 与 Mokka 一起运行，把 worker 分成两个池 |
| `--topograph` | [Topograph](../guides/topograph/README.md)，为每个 worker 打上其 NVLink 域标签 |
| `--observability` | kube-prometheus-stack 和面向模拟机群的 Grafana 仪表盘 |
| `--dynamo` | NVIDIA Dynamo operator 和一个 `qwen3` 推理图，其中 mocker worker 运行在模拟 GPU 上 |
| `--slinky` | cert-manager、Slinky slurm-operator 和一个 Slurm 集群，每个模拟 GPU worker 上有一台 GPU `slurmd` |
| `--control-plane` | Mokka 控制面镜像和 `mokka-crds` chart |

### 哪些能组合，哪些不能

大多数 flag 可以叠加。以下不行：

| 组合 | 原因 |
|---|---|
| `--fgo` 与 `--gpu-operator` | FGO 取代 GPU Operator |
| `--fgo` 与 `--compute-domain` | 机群形态不同 |
| `--compute-domain` 与 `--multi-gpu-profile` 或 `--gpu-profile` | compute-domain 场景固定了这两者 |
| `--slinky` 与 `--dynamo` | `slurmd` 会占用所有模拟 GPU，Dynamo worker 永远无法调度 |

有四个 flag 隐含其他 flag：`--topograph` 隐含 `--compute-domain`，因为 clique 只在那里存在；`--observability` 隐含 `--gpu-operator`，因为 `dcgm-exporter` 是它的 operand 之一；`--dynamo` 和 `--slinky` 隐含 `--gpu-operator`，因为它们的工作负载请求 `nvidia.com/gpu`。

```bash
tilt up -- --gpu-profile gb200
tilt up -- --multi-gpu-profile --gpu-operator
tilt up -- --gpu-operator --dra
tilt up -- --observability
tilt up -- --dynamo
tilt up -- --slinky

make cluster-create PROFILE=compute-domain
tilt up -- --compute-domain --dra
```

## Tilt UI 中的手动触发器

有些场景会添加按钮而不是自动运行。

**`compute-domain-tests`**——`check-fabric` 断言拓扑覆盖给每个节点分配了预期的 clique 和 cluster UUID；`topology-rebind` 实时重新绑定拓扑并重新断言。

**`observability-tests`**——`inject-thermal` 固定一块 GPU 的温度，并断言 Prometheus 恰好提供该值，而其同级 GPU 仍在变化；`inject-xid` 触发一次不可纠正 ECC 故障，并断言该 Xid 到达 `DCGM_FI_DEV_XID_ERRORS`。两者都用 `nvml-mock-ctl` 注入，而不是 `helm upgrade`——后者会回收 exporter，在序列中撕出一个洞。两者都**在故障未到达 Prometheus 时失败**：抓取路径本身受到断言，而不是靠肉眼查看。

Grafana 在其资源就绪后会被端口转发到 <http://localhost:3000/d/mokka-gpu>（`admin` / `mokka`）。

**`dynamo-tests`**——`dynamo-smoke` 断言 `qwen3` worker 恰好看到一块 GPU，其名称与所在节点的 `nvidia.com/gpu.product` 标签匹配，然后通过前端发送一次 chat completion 并检查 token 数。前端被端口转发到 <http://localhost:8000/v1/models>。[NVIDIA Dynamo 指南](../guides/dynamo/README.md)从已发布产物的角度走一遍同样的设置。

**`slinky-tests`**——`slurm-smoke` 断言每个 Slurm 节点都空闲且配备其配置中的 GPU，一个 2-GPU 作业恰好得到 2 块且属于该节点的 profile，以及一个使用每个节点上所有 GPU 的作业能够运行。[Slinky 指南](../guides/slinky/README.md)从已发布产物的角度走一遍同样的设置。

## 覆盖 Helm 取值

取值按以下顺序叠加，最后者胜出：

| 来源 | 已提交 |
|---|---|
| Chart 默认值 | — |
| `local/nvml-mock.values.yaml`——共享的本地基线 | 是 |
| `local/<consumer>/nvml-mock.values.yaml`——按消费者调整 | 是 |
| `local/nvml-mock.values.local.yaml`——你的机器 | **否**，被 gitignore |
| 映射到 chart 取值的 Tilt flag：`--gpu-profile`、`--nvmlmock-image`、`--control-plane` | — |

flag 以 `--set` 的形式传给 Helm，其优先级高于所有文件。因此只有在未传 `--gpu-profile` 时，values 文件中的 `gpu.profile` 才会生效。

```bash
cat > local/nvml-mock.values.local.yaml <<'EOF'
gpu:
  count: 2
infiniband:
  enabled: true
  mockTier: full
EOF
```

Tilt 不监视 values 文件，Tiltfile 只在加载时查找个人文件。请在 `tilt up` 之前创建它；编辑任何 values 文件后，在 Tilt UI 中触发 `nvml-mock` 资源以重新应用。

## 相关链接

| 想了解 | 参见 |
|---|---|
| 运行测试 | [测试](testing.md) |
| profile 定义了什么 | [配置参考](../configuration.md) |
| 在运行中的节点上修改状态 | [运行时控制](../nvml-mock-ctl.md) |
