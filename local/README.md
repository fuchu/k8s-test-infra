# 本地开发环境

在笔记本上运行 Mokka 所需的 Tiltfile、Helm 值覆盖与 Kind 集群配置。具体工作流（创建集群、Tilt 标志、哪些组件被组合、如何覆盖 Helm 值）见[本地开发](../docs/contributing/local-development.md)。本页只收录与本目录相关的内容。

## 集群

两种集群形态都使用 Makefile 固定的标准节点镜像（`KIND_NODE_IMAGE`）：Mokka 节点 daemon 会安装 NVIDIA 容器运行时，并向 containerd 自身注册 `nvidia` handler（见[容器运行时设置](../docs/components/node-daemon.md#container-runtime-setup)）。

用 `nvml-mock-ctl fail --xid` 注入的 Xid 可以从节点 journal 中读到。Kind 把节点容器命名为 `<cluster>-worker[N]`，而不是配置里固定的节点名，因此 `worker-0` 实际是 `mokka-worker`：

```bash
docker exec mokka-worker journalctl -k | grep 'NVRM: Xid'
```

## Compute-domain 场景

加上 `--dra` 后，DRA driver 会用其原版镜像运行 ComputeDomains。节点代理会从 `local/nvml-mock.values.local.yaml` 里设置的 `imex.binaries.image` 为它们准备 IMEX；在 `IMEXDownload` feature gate 开启时，则从 NVIDIA 下载。Tilt 会在每个节点都具备 IMEX 之后才安装 driver。

## 下游消费者

本地开发页未覆盖的注意事项。

### FGO

`--fgo` 在 `integration` worker 上跑模拟、在 `scale` worker 上跑 FGO 的 fake 后端；`--multi-gpu-profile --fgo` 产生同样的形态。检查方式：

```bash
kubectl get pods -n mokka -l app.kubernetes.io/name=nvml-mock -o wide   # 模拟，在 integration worker 上
kubectl get configmaps -A -l run.ai/gpu-profile=true                    # 为 FGO 的 loader 塑形的 profile
kubectl get pods -n gpu-operator -o wide                                # FGO，在 scale worker 上
```

### topograph

见 [topograph/README.md](topograph/README.md)。

### 可观测性

见 [observability/README.md](observability/README.md)。

### Dynamo

worker 只能经由 nvml-mock 的 NRI 插件（默认开启）看到 `--gpu-profile` 选定的 GPU 配置档；原因见 [NVIDIA Dynamo 指南](../docs/guides/dynamo/README.md)。一旦 `dynamo-frontend` 就绪，OpenAI API 就在 <http://localhost:8000>：

```bash
curl -s localhost:8000/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3-0.6B","messages":[{"role":"user","content":"hi"}],"max_tokens":16}'
```

### Slinky

`slurmd` 启动时只检测一次 GPU，而 `slurmctld` 会保留节点首次的 GPU-to-CPU 绑定。因此重装 nvml-mock 或切换 `--gpu-profile` 之后，要先删掉节点记录（`scontrol delete nodename=...`），再删 `slurmd` Pod。从 controller 提交作业：

```bash
kubectl -n slurm exec slurm-controller-0 -c slurmctld -- srun -N1 --gres=gpu:2 nvidia-smi -L
```

[Slinky 指南](../docs/guides/slinky/README.md) 用已发布的制品走了一遍同样的搭建过程。
