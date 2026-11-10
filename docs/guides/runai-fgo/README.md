# Run:ai Fake GPU Operator

把 Mokka 和 Run:ai 的 [fake-gpu-operator](https://github.com/run-ai/fake-gpu-operator)（FGO）放进同一个集群，各管自己更擅长的那部分节点。

## 为什么要搭配使用

FGO 是在 **Kubernetes API 层面**模拟 GPU：它直接向调度器声明 GPU 资源，完全不碰驱动栈，因此速度极快，还能撑起几百个 KWOK 虚拟节点。

Mokka 则是在**驱动层面**模拟：Pod 里有一个真实的 `libnvidia-ml.so`，所以 `nvidia-smi`、DCGM 和 GPU Operator 的 validator 都能照常工作。

两者搭配，就得到一个**混合集群**：少数节点具备完整的驱动保真度，其余是一大群低成本的虚拟节点。

| | 只用 FGO | 只用 Mokka | 两者搭配 |
|---|---|---|---|
| 向调度器声明 GPU | 是 | 是 | 是 |
| KWOK 虚拟节点 | 是 | 否 | 是 |
| Pod 内有真实 NVML | 否 | 是 | 仅 Mokka 节点 |
| 真实的 `nvidia-smi` 输出 | 有限 | 是 | 仅 Mokka 节点 |
| DCGM 指标 | 3 个合成值 | 真实 | 仅 Mokka 节点上真实 |
| GPU Operator 校验 | 否 | 是 | 仅 Mokka 节点 |
| 扩展到 1000+ 节点 | 是 | 否 | 是 |

!!! warning "这套搭配没有在 CI 中验证过"
    Mokka 这一侧的契约是验过的：ConfigMap 结构有单元测试和 `fgo` 端到端用例做断言。但两个项目从未在 CI 里一起跑过，而且这里既没有 vendor FGO，也没有固定它的版本。依赖本文结论之前，请重新核对 FGO 的加载逻辑。

## 发现契约 <a id="the-discovery-contract"></a>

这是最容易**悄悄出问题**的一环，装之前值得先弄明白。

FGO 不会去监听带某些标签的 profile ConfigMap，也不会按标签筛选。它的加载器直接按名字 `Get`，并且只读一个 data key。下面三项必须分毫不差，任何一项错了都会直接失败：

| 字段 | 必须是 | 写错了会怎样 |
|---|---|---|
| 名称 | `gpu-profile-<profile>` | `NotFound` |
| Data key | `profile.yaml` | 报 “missing key” |
| 命名空间 | FGO 自己 release 所在的命名空间 | `NotFound`——加载器不会跨命名空间回退 |

由于这些名字不带 release 前缀，同一个命名空间里装两个 Mokka release 就会在这 8 个 ConfigMap 上撞车。**每个命名空间只给一个 release 开启该集成。**

!!! warning "先把 FGO 的 `builtinProfiles.enabled=false` 设上"
    FGO 自带一套 profile，用的正是同样七个名字，且归它自己的 Helm release 所有。Helm 3 拒绝接管属于别的 release 的资源，所以后装的那个 chart 一定会报 `invalid ownership metadata`。这两套 profile 是二选一，不能互补叠加。

## 前置条件

- [Docker](https://docs.docker.com/get-started/get-docker/) 和 [Kind](https://kind.sigs.k8s.io/)——本指南会创建自己的集群 `nvml-mock-fgo-demo`，不会动你当前的 context。
- [Helm 3.8 或更高版本](https://helm.sh/docs/intro/install/)——chart 托管在 OCI 仓库。
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)
- 克隆好的本仓库，需要用到里面的 Kind 拓扑文件。

全程约 10 分钟。

## 第 1 步 — 创建集群

拓扑文件会给 1 个 worker 打 `integration` 标签、给 2 个打 `scale` 标签。少了这些标签，Helm 依然会「安装成功」，但对应节点池不会有任何 Pod。

```bash
kind create cluster --name nvml-mock-fgo-demo --config=docs/guides/kind.yaml
```

## 第 2 步 — 在 integration 节点池安装 Mokka

这些 ConfigMap 的 metadata 里带着 FGO 的命名空间，所以那个命名空间必须先存在，Mokka 才能往里写：

```bash
kubectl --context kind-nvml-mock-fgo-demo create namespace gpu-operator

helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --kube-context kind-nvml-mock-fgo-demo \
  --namespace mokka --create-namespace \
  --set integrations.fakeGpuOperator.enabled=true \
  --set integrations.fakeGpuOperator.targetNamespace=gpu-operator \
  --set gpu.profile=h100 \
  --set gpu.count=8 \
  --set "nodeSelector.run\.ai/simulated-gpu-node-pool=integration" \
  --wait --timeout 120s
```

`integrations.fakeGpuOperator.enabled=true` 负责按 FGO 加载器能读懂的格式生成 profile ConfigMap

!!! note "国内镜像拉取"
    本指南会从 `ghcr.io`（nvml-mock、fake-gpu-operator）拉取镜像。若这些镜像源拉取缓慢或超时，可给相关 chart 的镜像加私有仓库前缀（如给 `nvml-mock` 加 `--set global.imageRegistry=harbor.heilansc.com/ghcr.io`）。具体做法与示例见[国内镜像加速](../README.md#domestic-mirror)。；`targetNamespace` 把它们放到 FGO 会去读的位置——不设的话，ConfigMap 会落在 Mokka 自己的 release 命名空间里，而 FGO 根本不看那里。

## 第 3 步 — 在 scale 节点池安装 FGO

```bash
helm upgrade --install gpu-operator \
  oci://ghcr.io/run-ai/fake-gpu-operator/fake-gpu-operator \
  --kube-context kind-nvml-mock-fgo-demo \
  -n gpu-operator \
  --set builtinProfiles.enabled=false \
  --wait --timeout 120s -f - <<EOF
topology:
    nodePools:
      integration:
        backend: mock
        gpuCount: 8
        gpuProfile: h100
      scale:
        backend: fake
        gpuCount: 8
        gpuProfile: h100
EOF
```

`backend: mock` 表示这个节点池交给 Mokka，`backend: fake` 表示继续用 FGO 自带的垫片。如今这 8 个 ConfigMap 名字已归 Mokka 所有，只有加上 `builtinProfiles.enabled=false` 这次安装才装得上，参见[发现契约](#the-discovery-contract)。

## 第 4 步 — 验证

```bash
CTX=kind-nvml-mock-fgo-demo

# Mokka 只跑在 integration worker 上。
kubectl --context $CTX -n mokka get pods -l app.kubernetes.io/name=nvml-mock -o wide

# profile ConfigMap 存在于 FGO 会去读的命名空间里。
kubectl --context $CTX -n gpu-operator get cm -l fake-gpu-operator/gpu-profile=true

# 在一台没有 GPU 的节点上跑真正的 nvidia-smi。
kubectl --context $CTX -n mokka exec ds/nvml-mock -- nvidia-smi

# FGO 各组件的 Pod 跑在它自己的 release 命名空间里。
kubectl --context "$CTX" -n gpu-operator get pods -o wide
```

最终应该得到这样的结果：

| 节点 | 节点池 | Backend | GPU 来自 | Mokka DaemonSet |
|---|---|---|---|---|
| worker | `integration` | `mock` | Mokka | 有 |
| worker | `scale` | `fake` | FGO | 无 |

## FGO 没读到 profile 时

先查**名字和命名空间**，别去查标签——标签不在 FGO 的加载路径上：

```bash
kubectl get cm gpu-profile-h100 -n gpu-operator -o name
```

返回 `NotFound` 无非三种原因：ConfigMap 还留在 Mokka 的命名空间里（补上 `targetNamespace`）、名字写错了，或者安装时撞上了前面说的所有权错误。如果 ConfigMap 存在，再确认内容确实放在 `profile.yaml` 这个 key 下：

```bash
kubectl get cm gpu-profile-h100 -n gpu-operator -o jsonpath='{.data.profile\.yaml}' | head -5
```

修好任意一项后，重启 FGO 的 controller：

```bash
kubectl rollout restart deployment fake-gpu-operator -n gpu-operator
```

## 给 ConfigMap 加自定义标签

`profileLabels` 只能**新增**标签，删不掉契约要求的那些，所以在这里做覆盖不会破坏发现流程：

```yaml
integrations:
  fakeGpuOperator:
    enabled: true
    profileLabels:
      my-org/custom-label: "gpu-sim"
```

## 清理

```bash
kind delete cluster --name nvml-mock-fgo-demo
```

## 延伸阅读

| 想了解 | 参见 |
|---|---|
| 全部 chart 配置项 | [安装](../../helm-chart.md) |
| 一个配置档定义了什么 | [配置](../../configuration.md) |
| 单独使用 Mokka | [快速入门](../../quickstart.md) |
