# 配置 NRI 注入

Mokka 的 [NRI 插件](../components/nri-plugin.md)让按常规方式获得 GPU 的工作负载无需修改 Pod spec 以适配 Mokka，就能运行 `nvidia-smi` 并加载模拟 NVML 库。chart 默认运行它。本页介绍 device plugin 需要哪些条件、如何检查插件是否注入，以及如何让某些工作负载排除在外。

## 插件做什么

插件作为 `nvml-mock-nri` 容器运行在节点 DaemonSet 中。每个容器创建时，它把模拟驱动加到应该拥有它的容器上：

- 持有来自 NVIDIA device plugin 或 NVIDIA DRA driver 的 GPU 分配的容器，会获得模拟驱动，并且只保留分配给它的 GPU。
- 带有 `nvml-mock.nvidia.com/devices: "true"` 注解的 Pod（例如节点级监控 agent），无需 GPU 请求即可获得节点上所有模拟 GPU。
- InfiniBand 和 IMEX 通道分别通过各自的注解单独选择。
- 其他容器完全保持作者原本的写法。

完整规则见[哪些容器会被注入](../components/nri-plugin.md#which-containers-are-injected)，所有注解见[注解](../components/nri-plugin.md#annotations)。

## 前置条件

插件本身对节点没有额外要求：节点守护进程会在其[容器运行时设置](../components/node-daemon.md#container-runtime-setup)中打开 containerd 的 NRI。

NVIDIA device plugin 需要 `--pass-device-specs=true` 或 `cdi-cri` 设备列表策略——GPU Operator 在启用 CDI 时使用的就是后者。两者都没有时，分配不会在容器中留下插件可识别的内容；参见[识别 GPU 分配](../components/nri-plugin.md#recognising-a-gpu-allocation)。

## 安装 Mokka

把 Mokka 安装到自己的命名空间。插件从不注入自己的 release 命名空间或 `kube-system`，因此安装到共享命名空间会让该命名空间的工作负载得不到模拟驱动。

```bash
helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
  --namespace mokka --create-namespace \
  --wait --timeout 180s
```

已有的 release 会在下次 `helm upgrade` 时开启 NRI。从 0.4.x chart 升级时不要使用 `--reuse-values`，它会渲染该 release 的取值，包括 `nri.enabled=false` 及其镜像 tag。`--reset-then-reuse-values`（Helm 3.14+）则会保留你自己的覆盖值。

每个节点 Pod 都会在节点 agent 旁运行 NRI 插件，且只有插件向 containerd 注册后才变为 Ready：

```bash
kubectl -n mokka get pods -l app.kubernetes.io/name=nvml-mock \
  -o custom-columns='POD:.metadata.name,NODE:.spec.nodeName,READY:.status.conditions[?(@.type=="Ready")].status,INIT:.spec.initContainers[*].name,CONTAINERS:.spec.containers[*].name'
```

在 Kubernetes 1.29 及以后，节点 agent 作为可重启的 init 容器列在 `INIT` 下，`nvml-mock-nri` 列在 `CONTAINERS` 下；参见 [NRI Pod 生命周期](../helm-chart.md#nri-pod-lifecycle)。在更旧的集群上，或设置 `nri.nativeSidecar=false` 时，两者都列在 `CONTAINERS` 下。

节点 Pod 一直 NotReady、`nvml-mock-nri` 不断 crash-loop，说明该节点上的 containerd 禁用了 NRI；参见[排障](../troubleshooting.md#the-node-pod-is-notready-and-nvml-mock-nri-crash-loops)。

## 验证注入

运行一个通过 `devices` 注解显式加入的 Pod，这样检查就不需要 GPU 消费者：

```bash
kubectl -n default apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: nri-check
  annotations:
    nvml-mock.nvidia.com/devices: "true"
spec:
  restartPolicy: Never
  containers:
    - name: app
      image: debian:bookworm-slim
      command: ["sleep", "300"]
EOF

kubectl -n default wait --for=condition=ready pod/nri-check --timeout=120s
kubectl -n default exec nri-check -- nvidia-smi -L
```

镜像选用 debian，因为注入的 `nvidia-smi` 是 glibc 二进制：musl 镜像（如 `busybox` 或 `alpine`）无法运行它。`nvidia-smi -L` 会列出节点上所有模拟 GPU。没有注解也没有 GPU 请求的 Pod 看不到任何 GPU，与真实 GPU 节点一样。

要检查分配路径，请在基于 debian 的镜像上通过 [device plugin](device-plugin.md) 调度一个 GPU 请求，或通过 [DRA driver](dra.md) 创建一个 ResourceClaim：该 Pod 中的 `nvidia-smi -L` 只列出分配给它的 GPU。

```bash
kubectl -n default delete pod nri-check
```

## 让工作负载排除在外

| 目标 | 做法 |
|---|---|
| 跳过某个 Pod | 给它加注解 `nvml-mock.nvidia.com/inject: "false"` |
| 跳过某个命名空间 | 把它加入 `nri.excludedNamespaces` |
| 关闭插件 | `helm upgrade nvml-mock ... -n mokka --reuse-values --set nri.enabled=false` |

关闭插件只影响之后创建的容器。运行中的容器会保留已有内容，直到重启。

## 何时不注入

插件采用 fail-open：如果无法注入，容器会不带模拟驱动启动，而不是失败。排查请从[启用了 NRI 但 Pod 得不到模拟 GPU](../troubleshooting.md#a-pod-gets-no-mock-gpus-even-though-nri-is-enabled)开始，然后是 [NRI 插件失败模式](../helm-chart.md#nri-plugin-failure-modes)。
