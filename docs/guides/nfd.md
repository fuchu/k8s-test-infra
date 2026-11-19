<!-- SPDX-License-Identifier: Apache-2.0 -->
<!-- SPDX-FileCopyrightText: Copyright 2026 NVIDIA CORPORATION -->

# Node Feature Discovery

在 Mokka 旁边安装 [Node Feature Discovery（NFD）](https://kubernetes-sigs.github.io/node-feature-discovery/)，验证 NFD 会把 Mokka 的 feature 文件转换为真实节点会暴露的那种 GPU 存在标签。

Mokka 不写 Kubernetes 节点标签。它的节点 agent 会向 NFD local-source 目录写入 `pci-10de.present=true`，最终产生的 `feature.node.kubernetes.io/pci-10de.present=true` 标签则由 NFD 拥有。这让模拟路径与真实部署保持一致。

## 前置条件

- [Docker](https://docs.docker.com/get-started/get-docker/) 和 [Kind](https://kind.sigs.k8s.io/)
- [Helm 3.8 或更新版本](https://helm.sh/docs/intro/install/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)

本指南约需 10 分钟，不需要 GPU。

## 第 1 步——创建集群并安装 Mokka

Kind 默认集群只包含控制面节点。该节点可能带有 `NoSchedule` taint，因此 NFD worker 永远不会在那里运行。为本指南显式创建一个 worker：

```console
$ cat > mokka-nfd-kind.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
EOF
$ kind create cluster --name mokka-nfd --config mokka-nfd-kind.yaml
$ helm install nvml-mock oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
    --namespace mokka --create-namespace \
    --wait --timeout 120s
$ kubectl label node --all mokka.nvidia.com/type=sgpu
$ rm mokka-nfd-kind.yaml
```

chart 默认的 `nodeLabels.featuresDir` 是 `/etc/kubernetes/node-feature-discovery/features.d`。该目录从每个节点挂载，节点 agent 写入文件后 NFD 即可读取。

## 第 2 步——安装 NFD

仓库中的 NFD 端到端场景固定使用 `0.19.0` 版本。这里使用相同版本以保证 local-source 行为可复现：

```console
$ helm repo add nfd https://kubernetes-sigs.github.io/node-feature-discovery/charts
$ helm repo update
$ helm upgrade --install nfd nfd/node-feature-discovery \
    --namespace node-feature-discovery --create-namespace \
    --version 0.19.0 \
    --wait --timeout 120s
```

!!! note "国内镜像拉取"
    国内环境可给 Mokka chart 加 `--set global.imageRegistry=shharbor.heilansc.com/ghcr.io`；给 NFD 加 `--set image.repository=shharbor.heilansc.com/registry.k8s.io/nfd/node-feature-discovery --set image.tag=v0.19.0`。镜像需预先同步到私有仓库的对应路径，完整做法见[国内镜像加速](README.md#domestic-mirror)。

检查标签前先等待 NFD 就绪：

```console
$ kubectl -n node-feature-discovery wait \
    --for=condition=ready pod \
    -l app.kubernetes.io/name=node-feature-discovery \
    --timeout=120s
```

## 第 3 步——验证标签及其来源

NFD 会在其 worker 运行的每个节点上轮询 local-source 目录，并在 feature 文件存在的地方加上带命名空间的标签：

```console
$ kubectl get nodes \
    -o custom-columns='NODE:.metadata.name,NFD_GPU:.metadata.labels.feature\.node\.kubernetes\.io/pci-10de\.present'
```

worker 报告 `true`：

```text
NODE                      NFD_GPU
mokka-nfd-control-plane   <none>
mokka-nfd-worker          true
```

控制面保持无标签。Mokka 也会在那里写 feature 文件，但 NFD worker 不容忍控制面 taint，因此该节点上的文件无人读取。

输入是 feature 文件，而不是 Mokka 写的标签。可以从 node-agent 容器中查看它：

```console
$ kubectl -n mokka exec daemonset/nvml-mock -c node-agent -- \
    cat /host/etc/kubernetes/node-feature-discovery/features.d/nvml-mock.features
```

预期输出：

```text
pci-10de.present=true
```

NFD 把自己拥有的标签记录在 `nfd.node.kubernetes.io/feature-labels` 节点注解中。当工作负载或准入规则需要区分 NFD 拥有的标签与其他组件写入的标签时，请检查该注解：

```console
$ kubectl get node mokka-nfd-worker \
    -o jsonpath='{.metadata.annotations.nfd\.node\.kubernetes\.io/feature-labels}{"\n"}'
```

注解中包含不带 `feature.node.kubernetes.io/` 前缀的 `pci-10de.present`。

## 在工作负载中使用该标签

像选择真实 GPU 节点一样，用 NFD 标签选择模拟节点，然后确认 Kubernetes 把 Pod 调度到了带标签的节点上：

```console
$ kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: nfd-labelled-workload
spec:
  nodeSelector:
    feature.node.kubernetes.io/pci-10de.present: "true"
  containers:
    - name: shell
      image: busybox:1.36
      command: ["sleep", "300"]
EOF
$ kubectl get pod nfd-labelled-workload -o wide
```

## 排障

**标签缺失。** 检查 Mokka DaemonSet 是否就绪，以及 feature 文件是否存在于 node-agent 容器中。NFD chart 与 Mokka 必须使用相同的 `featureFilesDir` 和 `nodeLabels.featuresDir` 路径。

**NFD Pod 就绪但没有标签。** 查看 NFD worker 日志：

```console
$ kubectl -n node-feature-discovery logs -l app.kubernetes.io/name=node-feature-discovery
```

**节点有标签但工作负载处于 Pending。** 确认 Pod 的 `nodeSelector` 使用了完整的带命名空间键和字符串值 `"true"`。

## 清理

```console
$ kind delete cluster --name mokka-nfd
```

## 相关链接

| 想了解 | 参见 |
|---|---|
| Mokka chart 取值 | [安装](../helm-chart.md) |
| 通过 NVIDIA device plugin 分配 GPU | [NVIDIA Device Plugin](device-plugin.md) |
| chart 中的 NFD 标签行为 | [Helm chart 参考](../helm-chart.md#node-labels) |
