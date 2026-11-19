# NVIDIA Device Plugin

把模拟 GPU 作为 `nvidia.com/gpu` 可分配资源通告出去，然后调度一个请求 GPU 的工作负载。

这是让 Kubernetes 自身相信节点有 GPU 的最简单方式。真实的 NVIDIA device plugin 未经修改地运行——它通过 NVML 发现设备，而应答它的是 Mokka。

## 前置条件

- [Docker](https://docs.docker.com/get-started/get-docker/) 和 [Kind](https://kind.sigs.k8s.io/)
- [Helm 3.8 或更新版本](https://helm.sh/docs/intro/install/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)

约需 5 分钟。

## 第 1 步——创建集群并安装 Mokka

```bash
kind create cluster --name mokka-device-plugin

helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  --wait --timeout 120s
```

!!! note "国内镜像拉取"
    国内环境可给 Mokka chart 加 `--set global.imageRegistry=shharbor.heilansc.com/ghcr.io`；第 3 步清单里的 device plugin 镜像也应改成 `shharbor.heilansc.com/nvcr.io/nvidia/k8s-device-plugin:v0.18.2`。镜像需预先同步到私有仓库的对应路径，完整做法见[国内镜像加速](README.md#domestic-mirror)。

## 第 2 步——给节点打标签

下面的 device plugin 清单针对模拟 GPU 节点池，因此节点必须带有该标签：

```bash
kubectl label node --all mokka.nvidia.com/type=sgpu
```

## 第 3 步——部署 device plugin

```bash
kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-mock
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: nvidia-device-plugin-mock
  template:
    metadata:
      labels:
        name: nvidia-device-plugin-mock
    spec:
      nodeSelector:
        mokka.nvidia.com/type: sgpu
      tolerations:
        - operator: Exists
      containers:
        - name: nvidia-device-plugin
          image: nvcr.io/nvidia/k8s-device-plugin:v0.18.2
          args:
            - "--nvidia-driver-root=/var/lib/nvml-mock/driver"
            - "--driver-root-ctr-path=/var/lib/nvml-mock/driver"
            - "--device-discovery-strategy=nvml"
            - "--pass-device-specs=true"
          securityContext:
            privileged: true
          volumeMounts:
            - name: mock-root
              mountPath: /var/lib/nvml-mock
              readOnly: true
            - name: device-plugins
              mountPath: /var/lib/kubelet/device-plugins
      volumes:
        - name: mock-root
          hostPath:
            path: /var/lib/nvml-mock
        - name: device-plugins
          hostPath:
            path: /var/lib/kubelet/device-plugins
EOF

kubectl -n kube-system wait --for=condition=ready \
  pod -l name=nvidia-device-plugin-mock --timeout=120s
```

| 参数 | 原因 |
|---|---|
| `--nvidia-driver-root`、`--driver-root-ctr-path` | 把插件指向 Mokka 部署好的目录树，而不是真实驱动根目录 |
| `--device-discovery-strategy=nvml` | 通过 NVML 发现，这正是 Mokka 实现的接口 |
| `--pass-device-specs=true` | 把设备节点交付进容器。Mokka 的 [NRI 插件](../components/nri-plugin.md#recognising-a-gpu-allocation)需要它才能识别分配 |

## 第 4 步——验证可分配 GPU

```bash
kubectl wait \
  --for=jsonpath='{.status.allocatable.nvidia\.com/gpu}'=4 \
  node --all --timeout=120s

kubectl get nodes -o custom-columns='NODE:.metadata.name,GPUS:.status.allocatable.nvidia\.com/gpu'
# NODE                                GPUS
# mokka-device-plugin-control-plane   4
```

Pod 就绪只确认 device plugin 进程在运行。kubelet 注册和节点状态更新是异步的，因此调度 GPU 工作负载前请等待预期资源数量出现。

该数量来自 profile——`gb300` 是默认值，带四台设备。

## 第 5 步——调度工作负载

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  restartPolicy: Never
  containers:
    - name: app
      image: busybox:1.36
      command: ["sleep", "300"]
      resources:
        limits:
          nvidia.com/gpu: 1
EOF

kubectl wait --for=condition=ready pod/gpu-pod --timeout=120s
kubectl get nodes -o custom-columns='NODE:.metadata.name,GPUS:.status.allocatable.nvidia\.com/gpu'
```

结果是 Pod 成功调度：尽管节点上根本没有 GPU，kubelet 仍接受了 GPU 请求，因为插件分配了 Mokka 的某块 GPU。

## 不同节点使用不同 GPU 型号

一个 release 覆盖一个节点池，因此异构机群就是几个互不重叠的 release。从一个有多个 worker 的集群开始：

```bash
cat > kind-fleet.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
    labels:
      nvml-mock/profile: a100
  - role: worker
    labels:
      nvml-mock/profile: t4
EOF

kind create cluster --name mokka-fleet --config kind-fleet.yaml
kubectl label node --all mokka.nvidia.com/type=sgpu
```

每个池安装一个 release，各自选择自己的节点：

```bash
helm install nvml-mock-a100 oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  --set gpu.profile=a100 --set gpu.count=4 \
  --set "nodeSelector.nvml-mock/profile=a100" --wait --timeout 120s

helm install nvml-mock-t4 oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka \
  --set gpu.profile=t4 --set gpu.count=2 \
  --set "nodeSelector.nvml-mock/profile=t4" --wait --timeout 120s
```

像第 3 步那样部署 device plugin，每个 worker 都会报告自己的数量：

```bash
kubectl wait \
  --for=jsonpath='{.status.allocatable.nvidia\.com/gpu}'=4 \
  node -l nvml-mock/profile=a100 --timeout=120s
kubectl wait \
  --for=jsonpath='{.status.allocatable.nvidia\.com/gpu}'=2 \
  node -l nvml-mock/profile=t4 --timeout=120s

kubectl get nodes -l nvml-mock/profile \
  -o custom-columns='NODE:.metadata.name,GPUS:.status.allocatable.nvidia\.com/gpu'
```

!!! warning "`nodeSelector` 才是让各池彼此隔离的关键"

    chart 的 hostPath 挂载——`/var/lib/nvml-mock`、`/var/run/cdi`、`/run/nvidia` 和 NFD features 目录——对每个 release 都相同，而且 DaemonSet 容忍所有 taint。去掉 `nodeSelector` 后，两个 release 都会落到两个 worker 上，并覆盖彼此的逐节点状态。一个节点只能属于一个池。

## 节点标签

device plugin 通告资源，但不给节点打标签。`nvidia.com/` 下的标签来自 Node Feature Discovery 和 GPU Feature Discovery，与真实硬件完全一样——参见 [GPU Operator 指南](gpu-operator.md)，它会部署这两者。

## 排障

**可分配数量始终为零。** 插件在运行但没发现设备。检查其日志中的 NVML 错误，并确认 Mokka 已部署驱动根目录：

```bash
kubectl -n kube-system logs -l name=nvidia-device-plugin-mock --tail=30
```

**插件 Pod 处于 `Pending`。** 节点缺少 `mokka.nvidia.com/type=sgpu` 标签。

**请求一块 GPU 的 Pod 却看到了所有 GPU。** NRI 插件也在注入。`--pass-device-specs=true` 让 Mokka 的插件能检测到分配并让位——参见 [NRI 插件](../components/nri-plugin.md)。

## 清理

```bash
kind delete cluster --name mokka-device-plugin
```

## 相关链接

| 想了解 | 参见 |
|---|---|
| 所有 chart 取值 | [安装](../helm-chart.md) |
| 节点标签与完整 operand 技术栈 | [NVIDIA GPU Operator](gpu-operator.md) |
| 用 claim 代替计数器进行分配 | [NVIDIA DRA Driver](dra.md) |
| 无资源请求也能获得 GPU | [`devices` 注解](../components/nri-plugin.md#which-containers-are-injected) |
