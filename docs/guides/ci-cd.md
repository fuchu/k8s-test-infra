# 在 CI/CD 中使用

让需要 GPU 的测试跑在普通 CPU runner 上。Mokka 在作业内安装到一个临时 Kubernetes 集群中，你的软件与它交互时就像硬件是真的，作业结束时一并拆除。

这正是 Mokka 存在的用例：GPU runner 稀缺且昂贵，而 GPU 软件在 CI 中所做的大部分事情是*发现*和*调度*硬件，而不是在其上计算。

## 作业的结构

四步，顺序如下：

1. 在 runner 上创建 Kubernetes 集群——通常选 Kind。
2. 安装 Mokka chart：起初用最新版本，测试通过后固定到通过测试的那个版本。
3. 等待 DaemonSet 就绪。
4. 针对集群运行你的测试。

## GitHub Actions

```yaml
name: GPU tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Create a cluster
        uses: helm/kind-action@v1
        with:
          cluster_name: mokka

      - name: Install Mokka
        run: |
          helm install nvml-mock \
            oci://ghcr.io/nvidia/k8s-test-infra/chart/nvml-mock \
            --namespace mokka --create-namespace \
            --set gpu.profile=gb300 \
            --wait --timeout 5m

      - name: Run tests
        run: ./run-my-gpu-tests.sh
```

`--wait` 是可靠性的关键：没有它，下一步会在 DaemonSet 开始服务之前启动，测试就会在一个尚未上报 GPU 的节点上失败。

## 固定 chart 版本

没有 `--version` 时，`helm install` 会取最新的稳定 Mokka 版本。测试通过后，加上 `--version` 并指定通过测试的那个版本。不固定 chart 意味着某天你什么都没改，上游发布却改变了 CI 看到的东西，而且失败看起来不像版本变化。

## 选择 profile

profile 决定你的软件认为发现了什么硬件，因此请选择测试所针对的那一个：

```yaml
--set gpu.profile=a100      # 8 块 GPU，每块 40 GiB
--set gpu.profile=t4        # 4 块 GPU，每块 16 GiB——最小的
--set gpu.profile=gb200     # 4 块带 NVLink fabric 的 GPU
```

要覆盖多个 profile，请按 profile 运行作业矩阵，而不是在一个集群里安装多个 release。

## 测试故障处理

Mokka 可以呈现损坏的硬件，这在其他情况下几乎无法在 CI 中安排。GPU 从总线脱落、不可纠正 ECC 错误，或报告为丢失的设备，都是配置问题：

```yaml
--set gpu.failureInjection.enabled=true \
--set gpu.failureInjection.mode=ecc_uncorrectable
```

要在单个作业*之内*让 GPU 在健康与故障之间切换，请使用[运行时控制](../nvml-mock-ctl.md)而不是重装——它约一秒生效，且无需滚动发布。

## 普通集群能覆盖什么

普通 Kind 集群就足以支持模拟驱动和 `nvidia-smi`、节点标签、NVIDIA device plugin、DRA driver、GPU Operator 和 NRI 注入。Mokka 的节点守护进程会向 containerd 注册 `nvidia` 运行时 handler 并开启 CDI 和 NRI；参见[容器运行时设置](../components/node-daemon.md#container-runtime-setup)。

## 让作业保持快速

大部分耗时是镜像拉取，而不是 Mokka。若 CI 支持，请缓存 runner 的镜像层，并优先选择仍能覆盖测试目标的最小 profile——`t4` 呈现四块 GPU 而非八块。

## 相关链接

| 想了解 | 参见 |
|---|---|
| 所有 chart 取值 | [安装](../helm-chart.md) |
| profile 定义了什么 | [配置参考](../configuration.md) |
| 作业中途修改 GPU 状态 | [运行时控制](../nvml-mock-ctl.md) |
| 故障注入演练 | [故障注入](failure-injection/README.md) |
