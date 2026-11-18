# 特性门控

特性门控用于开启或关闭 Mokka 的单项行为，使某项行为可以在成为 Mokka 默认契约之前先发布出来。它们的工作方式类似 [Kubernetes 特性门控](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)：语法相同，阶段相同。Mokka 在启动时读取一次，因此变更在服务重启后生效。

## 可用门控

| 门控 | 阶段 | 默认值 | 起始版本 | 读取者 | Issue |
|------|-------|---------|-------|---------|-------|
| `IMEXDownload` | Beta | 开启 | 0.6 | 节点 agent | [#821](https://github.com/NVIDIA/k8s-test-infra/issues/821) |

`IMEXDownload` 让多节点 NVLink profile（如 gb200）上的节点 agent 下载 NVIDIA IMEX 归档包。当 `imex.binaries.image` 和节点缓存都没有提供 IMEX 时，节点 agent 会把它固定的归档包下载到 `/var/lib/nvml-mock/cache/imex` 下的节点缓存中。每次尝试受 `imex.binaries.download.timeout` 限制，默认一分钟。每次节点 agent 启动会重试约一分钟，之后节点保持无 IMEX 状态，此时该节点上的 ComputeDomain daemon 会失败，与没有 IMEX 包的真实节点一样。下载需要能 HTTPS 访问 `developer.download.nvidia.com`；在节点不得访问互联网的环境中请关闭该门控。Mokka 发布 IMEX 二进制镜像后会弃用该门控。

## 设置门控

通过 chart 设置门控，chart 会把同一份列表传给节点 agent、NRI 插件和控制面：

```yaml
featureGates:
  AllAlpha: true
```

这些服务各自都接受所有 Mokka 门控，并忽略自己不读取的门控，因此一份列表可供三者使用。更改列表会重启它们所有的 Pod，包括每个节点上的节点 agent，即便只有一个服务读取了发生变化的门控。

在 chart 之外，每个服务都接受相同的值作为 flag，`--feature-gates=AllAlpha=true,AllBeta=false`，或作为 `MOKKA_FEATURE_GATES` 环境变量。值是逗号分隔的 `Name=true` 或 `Name=false` 列表；同一名称出现多次时，最后一个值生效。与 Kubernetes 组件不同，值不会合并：只有最后一个 `--feature-gates` 生效，并且它会完全替换 `MOKKA_FEATURE_GATES`，因此请把所有门控放在同一个值里。

`AllAlpha=true` 和 `AllBeta=false` 会一次性改变每个 [alpha 或 beta](#stages) 门控的默认值；对单个门控的显式条目会覆盖它们。

chart 的 values schema 列出了该 chart 版本接受的每个门控。因此 Helm 会拒绝未知门控、非布尔值，或试图把[锁定](#stages)为默认值的门控改成其他值，并在任何 Pod 变化之前让安装或升级失败。在 chart 之外，服务收到这样的值会拒绝启动并记录原因。服务启动时若注册了门控，会记录每个门控的状态。

## 阶段

| 阶段 | 默认值 | 可更改 | 含义 |
|-------|---------|----------------|---------------|
| Alpha | 关闭 | 是 | 不完整或实验性。任何版本都可能变化或移除 |
| Beta | 开启 | 是 | 完整且经过测试。若引发问题可禁用，并请上报 |
| GA（正式可用） | 开启 | 否 | 永久行为。门控为兼容性保留；设置它会记录一条警告 |
| 已弃用 | 关闭 | 直到锁定 | 正在移除。设置它会记录一条警告 |

GA 或已弃用的门控，最早在弃用或达到 GA 之后三个次版本（Y）才移除，这是[支持策略](support-policy.md#deprecation)设定的下限。门控移除后，仍设置该门控的升级会失败并报出该门控名称，因此升级前请把它从 `featureGates` 中去掉，包括通过 `--reuse-values` 沿用的值。[变更日志](https://github.com/NVIDIA/k8s-test-infra/blob/main/CHANGELOG.md)记录每个门控何时添加、阶段变化或移除。

## 另见

- [安装](helm-chart.md)——所有 chart 取值
- [命令行工具](tools/README.md)——服务 flag
- [为新行为加门控](contributing/feature-gates.md)——把代码放到门控之后
