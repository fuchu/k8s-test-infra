# 排障

在 Kubernetes 集群上运行 Mokka 时可能遇到的症状，以及每种症状通常意味着什么。

## 首先，让 mock 告诉你它在做什么

```bash
MOCK_NVML_DEBUG=1 nvidia-smi
```

mock 应答的每次 NVML 调用都会留下跟踪记录，包括它加载了哪份配置以及构建了多少设备。下面的大多数问题，答案都在这段输出的前五行里。

## Kubernetes 安装问题

### Pod 中的 `nvidia-smi` 报告没有 GPU

先检查节点上是否真的有部署好的目录树，再检查 Pod 能否看到它：

```bash
kubectl get pods -n mokka -l app.kubernetes.io/name=nvml-mock -o wide
kubectl logs -n mokka -l app.kubernetes.io/name=nvml-mock --tail=50
```

如果 DaemonSet 没有在该节点上运行，通常是节点选择器的问题：chart 固定到带有 `mokka.nvidia.com/type=sgpu` 标签的节点。

### device plugin 报告 0 块 GPU

两种不同原因，按值得检查的顺序：

1. **mock 文件不在节点上。** 确认部署好的目录树存在——参见[手工检查节点](helm-chart.md#checking-a-node-by-hand)。
2. **GPU Operator 试图管理真实驱动。** 必须用 `driver.enabled=false` 和 `toolkit.enabled=false` 安装它；这里没有真实驱动可供它管理。

```bash
kubectl -n gpu-operator logs -l app=nvidia-device-plugin-daemonset | head -20
```

### GPU Operator validator 不断 crash-loop

validator 是静态链接的 Go 二进制，没有 shell，它会探测驱动根目录中应有的文件。部署树中少一个文件，看起来就像 crash loop，而不是错误信息。

```bash
NODE=$(docker ps --filter name=control-plane -q)
docker exec "$NODE" ls -la /run/nvidia/driver/usr/lib64/libnvidia-ml.so*
docker exec "$NODE" cat /var/lib/nvml-mock/driver/config/config.yaml
```

### CDI 规格从未出现在 `/var/run/cdi`

```bash
kubectl logs -n mokka -l app.kubernetes.io/name=nvml-mock | grep -i cdi
```

### 节点 Pod 处于 NotReady，`nvml-mock-nri` 不断 crash-loop

```text
nri-plugin: nri stub: failed to connect to NRI service: dial unix /var/run/nri/nri.sock: connect: no such file or directory
```

该节点上的 containerd 关闭了 NRI——这是 containerd 1.7 的默认状态。节点守护进程会在其[容器运行时设置](components/node-daemon.md#container-runtime-setup)中打开 NRI，因此原因可能是：设置被关闭、设置失败、设置没有覆盖该节点，或是在 `restartMode: none` 下等待 containerd 重启；节点 agent 的日志会说明是哪一种。节点 agent 继续运行，工作负载仍会启动，但不带 mock，`helm install --wait` 会超时。

如果有待处理的重启，请重启 containerd。否则，设置 `nri.enabled=false`，或在 containerd 配置中打开 NRI 并重启 containerd：

```toml
[plugins."io.containerd.nri.v1.nri"]
  disable = false
```

节点上的 `containerd config dump | grep -A1 'io.containerd.nri.v1.nri'` 显示 containerd 实际运行所用的设置。

### Pod 得不到模拟驱动，或节点 Pod 不就绪

节点守护进程会把 `nvidia` 运行时 handler 设为 containerd 默认值，设置失败、或它决定不改动节点运行时时，其日志会说明原因：

```bash
POD=$(kubectl -n mokka get pods -l app.kubernetes.io/name=nvml-mock \
  --field-selector spec.nodeName=<node> -o jsonpath='{.items[0].metadata.name}')
kubectl -n mokka logs "$POD" -c node-agent | grep 'container runtime\|containerd'
```

`container runtime setup failed` 这一行会指出失败的步骤，例如 `nvidia-ctk` 写配置失败，或 containerd 带着配置重启失败。在节点上，`crictl info` 显示运行中的 containerd 是否已把该 handler 设为默认，`journalctl -u containerd` 显示重启失败的原因。在设置成功之前，容器会在节点自身的默认 handler 下启动且不带模拟驱动，设置 `runtimeClassName: nvidia` 的 Pod 无法启动。设置需要什么、如何手工移除以及如何关闭，参见[容器运行时设置](components/node-daemon.md#container-runtime-setup)。

### 启用了 NRI 但 Pod 得不到模拟 GPU

先检查这个 Pod 是否本应接收 GPU。插件只把模拟 GPU 给两类容器：持有来自 device plugin 或 NVIDIA DRA driver 的 GPU 分配的容器，或其 Pod 带有 `nvml-mock.nvidia.com/devices: "true"` 注解的容器。`nvml-mock.nvidia.com/infiniband` 和 `nvml-mock.nvidia.com/imex-channels` 注解只添加各自的接口面，绝不添加 GPU。没有请求 GPU 也没有注解的 Pod，按设计不会改动。device plugin 若没有 `--pass-device-specs=true` 或 `cdi-cri` 策略，就不会留下插件可识别的分配，因此它的 Pod 也得不到任何东西。

NRI 插件采用 fail-open：无法注入时，容器会在不带 mock 的情况下创建，而不是失败。这就可能出现无声的窗口，因此要检查插件的探针，而不是进程是否还活着——containerd 注销的插件会继续运行，只是不再注入。

```bash
kubectl get pods -n mokka -l app.kubernetes.io/name=nvml-mock
kubectl describe pod -n mokka -l app.kubernetes.io/name=nvml-mock | grep -A3 Readiness
```

Pod 显式退出或所在命名空间被排除时，注入也会按设计跳过。参见 [NRI 插件失败模式](helm-chart.md#nri-plugin-failure-modes)和 [NRI 插件](components/nri-plugin.md)页面。

### 运行时修改没有生效

`nvml-mock-ctl` 写入一个覆盖文件，每个已加载的库副本按 TTL 重新读取它——默认约一秒。等待一个间隔再检查。

如果始终不生效，确认你针对的是正确的节点：覆盖是逐节点文件，在一个节点上修改状态对运行在另一个节点上的消费者没有影响。参见[运行时控制](nvml-mock-ctl.md)。

### 启用 NRI 的节点上挂载不断堆积

在 0.4.1 及之前，注入的容器若同时带有 `Bidirectional` 卷，例如 DRA kubelet 插件或 GPU Operator 的 validator，就会把覆盖层中可写的配置绑定复制到节点上。每这样一个容器都会让 `/var/lib/nvml-mock/driver/config` 下的副本翻倍，并且 Pod 消失后副本仍在。大量副本会拖慢容器启动以及其他读取节点挂载表的操作。参见[挂载传播](components/nri-plugin.md#mount-propagation)。

在 Kind 集群上，可在每个节点容器中统计数量。Kubernetes 节点名可能与容器名不同，因此用 `kind` 列出容器。健康节点打印 0：

```bash
for NODE in $(kind get nodes --name mokka); do
  echo "${NODE}: $(docker exec "${NODE}" awk '$5 == "/var/lib/nvml-mock/driver/config" { n++ } END { print n + 0 }' /proc/self/mountinfo)"
done
```

要停留在 0.4.x 又不让副本增长，请让插件避开那些 Pod 所在的命名空间，例如 GPU Operator 用 `--set 'nri.excludedNamespaces={gpu-operator}'`。排除后，这些命名空间中的 Pod 得不到覆盖层，因此只排除那些 Pod 不需要节点 GPU profile 的命名空间。

升级会停止增长，但不会清除已有副本。重启节点会清掉它们；在 Kind 上则重建集群。要就地清除，先确认升级前注入的容器都不再运行在该节点上：卸载副本会带走其可写配置目录，之后的写入会因只读而失败。然后反复卸载直到没有剩余。没有副本的节点保持原样，卸载失败的节点会被点名：

```bash
for NODE in $(kind get nodes --name mokka); do
  docker exec "${NODE}" sh -c 'while mountpoint -q /var/lib/nvml-mock/driver/config; do umount /var/lib/nvml-mock/driver/config || exit 1; done' \
    || echo "${NODE}: unmount failed"
done
```

### `kind load docker-image` 报 "content digest not found"

```text
ctr: content digest sha256:...: not found
```

`ghcr.io/nvidia/nvml-mock` 是多架构镜像。Docker Desktop 的 containerd 镜像仓库保留整个 manifest 列表，而 `kind load docker-image` 交给节点的归档缺少对应平台的层。错误信息点出的是摘要，而不是原因。这是 macOS 上 Docker Desktop 的默认配置。

改为只保存单一平台并加载该归档：

```bash
ARCH=$(uname -m | sed 's/x86_64/amd64/; s/aarch64/arm64/')
docker save --platform "linux/${ARCH}" ghcr.io/nvidia/nvml-mock:latest -o nvml-mock.tar
kind load image-archive nvml-mock.tar --name <cluster>
```

平台必须与 kind 节点的架构一致。本地构建的镜像已经是单架构，可直接加载。

### 某个 profile 取值没有出现在 `nvidia-smi` 中

先确认 profile 到底有没有加载——`MOCK_NVML_DEBUG=1 nvidia-smi` 的前几行会点出配置和设备数量。如果加载了，说明该值设在另一个层级：逐设备设置覆盖 `device_defaults`，而运行时覆盖又高于两者。[配置参考](configuration.md)记录了优先级。

如果 profile 本身没加载，通常是 ConfigMap 的问题：

```bash
kubectl get configmap -n mokka -l app.kubernetes.io/name=nvml-mock
kubectl logs -n mokka -l app.kubernetes.io/name=nvml-mock | grep -i config
```

### 字段显示 `N/A` 或返回 `NOT_SUPPORTED`

mock 根据配置应答，因此 profile 未设置的字段确实是不支持的——与较旧的真实驱动给出同样的答案。把值加入 profile 并 `helm upgrade`。

有少数 NVML 函数是生成的桩，总是返回 `NOT_SUPPORTED`。[库与 Shims](components/libraries-and-shims.md)解释哪些以及为什么；[贡献指南](contributing/index.md)介绍如何实现一个。

### `nvidia-smi` 段错误或符号缺失

两者都意味着镜像比调用的那个函数更旧。确认节点运行的是你以为的镜像：

```bash
kubectl get daemonset -n mokka -o jsonpath='{.items[*].spec.template.spec.containers[*].image}'
```

提交 bug 前，用 `MOCK_NVML_DEBUG=1` 运行以找出涉及哪个调用。

## 在 Kubernetes 之外运行库

构建 `libnvidia-ml.so` 并用 `LD_LIBRARY_PATH` 针对它运行二进制，是贡献者的工作流——构建方法见[贡献指南](contributing/index.md)，Tilt 循环见[本地开发](contributing/local-development.md)。

有一条失败是这条路径特有的，值得了解：如果真实 NVML 在 `LD_LIBRARY_PATH` 存在的情况下仍然加载，说明搜索顺序中有别的东西胜出。

```bash
LD_DEBUG=libs nvidia-smi 2>&1 | grep nvml   # 实际解析到了什么
grep nvml /etc/ld.so.conf.d/*               # 某个 ld.so 条目可能优先于你的路径
```

绝对路径比相对路径可靠。

## 仍然卡住

选择与问题相符的渠道——tracker 禁用空白 issue，因此期望以下之一：

| 问题 | 去哪里 |
|---|---|
| Mokka 行为不正确 | [Bug 报告](https://github.com/NVIDIA/k8s-test-infra/issues/new?template=bug_report.yml) |
| 本站某处有误或缺失 | [文档修正](https://github.com/NVIDIA/k8s-test-infra/issues/new?template=documentation.yml) |
| 你希望 Mokka 做它还不会做的事 | [特性请求](https://github.com/NVIDIA/k8s-test-infra/issues/new?template=feature_request.yml) |
| "我该怎么做……？" | [Discussions](https://github.com/NVIDIA/k8s-test-infra/discussions)——tracker 用于跟踪工作项 |

!!! warning "绝不要为安全漏洞开公开 issue"
    请私下报告。参见[安全策略](https://github.com/NVIDIA/k8s-test-infra/blob/main/SECURITY.md)。

### bug 报告应包含什么

你运行了什么、期望什么、发生了什么，以及：

```bash
helm list -n mokka                      # chart 版本和所用 profile
kubectl get pods -n mokka -o wide
kubectl logs -n mokka -l app.kubernetes.io/name=nvml-mock --tail=100
kubectl get nodes --show-labels
```

再加上受影响的 Pod 内部执行的 `MOCK_NVML_DEBUG=1 nvidia-smi 2>&1 | head -40`。
