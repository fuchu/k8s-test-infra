# NVIDIA DRA Driver

通过动态资源分配（DRA）暴露模拟 GPU，调度一个申请 GPU 的 Pod，并在其中运行 `nvidia-smi`。

DRA 是 Kubernetes 超越 `nvidia.com/gpu` 计数器的方向：driver 发布描述各节点设备的 **ResourceSlice**，工作负载通过 **ResourceClaim** 请求所需内容。Mokka 让你无需硬件就能演练这条路径。

## 前置条件

- [Docker](https://docs.docker.com/get-started/get-docker/) 和 [Kind](https://kind.sigs.k8s.io/)
- [Helm 3.8 或更新版本](https://helm.sh/docs/intro/install/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)
- 验证步骤需要 [jq](https://jqlang.github.io/jq/)

约需 10 分钟。

## 第 1 步——创建启用 DRA 的集群

DRA 需要一个特性门控和一个非默认 API 版本，集群配置会打开它们。申请 GPU 的 Pod 还需要 CDI 和 containerd 的节点资源接口（NRI），Mokka 的 NRI 插件用它把模拟驱动放进 Pod。Kind 节点的 containerd 2.x 默认同时开启两者：

```bash
cat > kind-dra.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
featureGates:
  DynamicResourceAllocation: true
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: ClusterConfiguration
        apiServer:
          extraArgs:
            runtime-config: "resource.k8s.io/v1beta1=true"
EOF

kind create cluster --name mokka-dra --config kind-dra.yaml
```

## 第 2 步——安装 Mokka

```bash
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  --wait --timeout 120s
```

Mokka 的 [NRI 插件](../components/nri-plugin.md)默认开启，它会给持有 GPU 分配的容器提供模拟驱动，因此第 5 步的 Pod 无需 Mokka 专用 Pod spec 就能运行 `nvidia-smi`。

## 第 3 步——安装 DRA driver

kubelet 插件的节点亲和性要求一个 GPU 存在标签。Node Feature Discovery 会从 Mokka 写入的 feature 文件派生一个；集群中没有 NFD 时，手工设置：

```bash
kubectl label node --all nvidia.com/gpu.present=true

helm repo add nvidia https://helm.ngc.nvidia.com/nvidia && helm repo update

helm install nvidia-dra-driver nvidia/dra-driver-nvidia-gpu \
  --version 0.5.0 \
  --namespace nvidia --create-namespace \
  --set nvidiaDriverRoot=/var/lib/nvml-mock/driver \
  --set gpuResourcesEnabledOverride=true \
  --set resources.computeDomains.enabled=false \
  --wait --timeout 180s
```

| 设置 | 原因 |
|---|---|
| `nvidiaDriverRoot` | 把 driver 指向 Mokka 部署好的目录树，而不是真实驱动根目录 |
| `gpuResourcesEnabledOverride` | 否则 driver 会用模拟节点不满足的检查来决定是否发布 GPU 资源 |
| `resources.computeDomains.enabled=false` | ComputeDomain 支持需要 NVLink fabric——那条路径参见 [ComputeDomain 指南](compute-domain/README.md) |

## 第 4 步——检查 ResourceSlice

```bash
kubectl -n nvidia wait --for=condition=ready pod --all --timeout=120s

kubectl get resourceslices -o json | jq '[.items[].spec.devices // [] | length] | add // 0'
# 4 —— gb300 默认 profile 有四台设备
```

driver 发布的每台设备都来自 Mokka 在该节点上对 NVML 的应答。

## 第 5 步——调度一个申请 GPU 的 Pod

```bash
kubectl apply -f - <<'EOF'
apiVersion: resource.k8s.io/v1beta1
kind: ResourceClaimTemplate
metadata:
  name: gpu-claim
spec:
  spec:
    devices:
      requests:
        - name: gpu
          deviceClassName: gpu.nvidia.com
---
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test-pod
spec:
  restartPolicy: Never
  containers:
    - name: app
      image: debian:bookworm-slim
      command: ["sleep", "infinity"]
      resources:
        claims:
          - name: gpu
  resourceClaims:
    - name: gpu
      resourceClaimTemplateName: gpu-claim
EOF

kubectl wait --for=condition=ready pod/gpu-test-pod --timeout=120s
```

Pod 进入 `Running` 意味着调度器把 claim 匹配到了一个已发布的设备，*并且* kubelet 插件的 `NodePrepareResources` 成功了——这一步原本会触碰真实硬件。镜像选用 debian，因为注入的 `nvidia-smi` 是 glibc 二进制，`busybox` 无法运行。Pod 会一直 sleep 到你删除它：一旦退出，为它生成的 claim 也会随之删除，第 6 步就无从读取。

## 第 6 步——检查 Pod 只看到它申请的那块 GPU

先向 claim 询问调度器分配了哪台设备，再到 ResourceSlice 中按 UUID 查找：

```bash
claim=$(kubectl get pod gpu-test-pod -o jsonpath='{.status.resourceClaimStatuses[0].resourceClaimName}')
device=$(kubectl get resourceclaim "$claim" -o jsonpath='{.status.allocation.devices.results[0].device}')
pool=$(kubectl get resourceclaim "$claim" -o jsonpath='{.status.allocation.devices.results[0].pool}')

kubectl get resourceslices -o json | jq -r --arg d "$device" --arg p "$pool" \
  '.items[] | select(.spec.pool.name == $p) | .spec.devices[]
   | select(.name == $d) | (.basic.attributes // .attributes).uuid.string'
# GPU-…
```

然后询问 Pod：

```bash
kubectl exec gpu-test-pod -- nvidia-smi -L
# GPU 0: NVIDIA GB300 NVL (UUID: GPU-…)
```

`nvidia-smi` 只列出一块 GPU，其 UUID 正是分配给该 claim 的，而不是节点上所有 GPU：NRI 插件把容器严格限制在分配给它的设备上。

## 排障

**没有 ResourceSlice 出现。** driver 的 kubelet 插件未能调度，最常见的原因是节点上缺少 `nvidia.com/gpu.present=true`。

**Pod 一直处于 `Pending`。** 用 describe 查看——无法调度的 claim 会指出它无法满足的 device class。检查 `deviceClassName` 是否与 driver 注册的一致：

```bash
kubectl get deviceclasses
```

**Pod 中找不到 `nvidia-smi`。** NRI 插件没有注入它。检查 nvml-mock Pod 是否运行 `nvml-mock-nri` 容器并处于 Ready，以及 containerd 是否启用了 NRI。参见[启用了 NRI 但 Pod 得不到模拟 GPU](../troubleshooting.md#a-pod-gets-no-mock-gpus-even-though-nri-is-enabled)。插件只在容器创建时注入，修复后请删除并重建 Pod。

**API 拒绝清单。** `resource.k8s.io/v1beta1` 未提供服务。集群配置中的 `runtime-config` 行才是启用它的地方，且只在集群创建时生效。

## 清理

```bash
kind delete cluster --name mokka-dra
```

## 相关链接

| 想了解 | 参见 |
|---|---|
| 所有 chart 取值 | [安装](../helm-chart.md) |
| NRI 插件注入哪些容器 | [NRI 插件](../components/nri-plugin.md#which-containers-are-injected) |
| 用 operator 技术栈替代 DRA | [NVIDIA GPU Operator](gpu-operator.md) |
| ComputeDomain 的 NVLink fabric 身份 | [ComputeDomain](compute-domain/README.md) |
| 在 CI 中驱动这一切 | [在 CI/CD 中使用](ci-cd.md) |
