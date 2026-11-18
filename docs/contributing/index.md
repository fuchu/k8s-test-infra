---
title: 贡献指南
---

# 贡献指南

## 前置条件

- 与 `go.mod` 中版本一致的 Go，并启用 CGo
- C 工具链——模拟 NVML 库通过 CGo 构建
- `docker`、`kind`、`tilt`、`helm`、`kubectl`

## 开发循环

```bash
make cluster-create          # 仅一次
tilt up -- --gpu-operator    # 保持运行；保存时自动重新部署

make lint-fix                # 提交前
make test
```

[本地开发](local-development.md)介绍各种集群形态和所有 Tilt 参数。[测试](testing.md)介绍各道关卡。

## 代码都放在哪里

```text
cmd/            二进制文件：node-agent、nri-plugin、nvml-mock-ctl、
                control-plane、check-fabric、generate-bridge
internal/       节点守护进程及其模拟器、NRI 插件、控制面 API
pkg/gpu/        模拟 NVML 库、模拟 CUDA、分配监视器
shims/          LD_PRELOAD 和 execve shims
deployments/    Helm charts、Dockerfile、GPU profile
local/          Tiltfile、Kind 配置、本地 Helm values
tests/          e2e 套件、chart 测试、集成测试
enhancements/   Mokka 增强提案
docs/           本站
```

[架构概览](../architecture.md)解释这些部分在运行时如何组合；每个组件都有自己的页面。

## 常见变更

### 添加 NVML 函数

1. 在 `pkg/gpu/mocknvml/engine/device.go` 中为 `ConfigurableDevice` 实现该方法，当 profile 未配置该值时返回 `ERROR_NOT_SUPPORTED`。
2. 把任何新增配置字段加入 `engine/config_types.go`。
3. 在 `pkg/gpu/mocknvml/bridge/` 下对应的文件中添加 C 导出包装函数。
4. 运行 `make gen`。生成器会扫描 `//export` 指令，并移除它此前为该函数生成的桩。
5. 编写测试。

生成器由 NVML 自身的头文件驱动，因此手写实现总会替换掉它的桩——永远不需要手工删除。各层如何配合，参见[库与 Shims](../components/libraries-and-shims.md)。

### 添加 GPU profile

Profile 是 YAML。已部署的 profile 位于 `deployments/nvml-mock/helm/nvml-mock/profiles/`，是事实来源；`pkg/gpu/mocknvml/configs/` 存放独立版本，用于在 Kubernetes 之外运行该库。

尽可能从真实板卡的采集数据入手：[采集真实节点](capturing-a-real-node.md)列出要记录的内容以及如何把 profile 与采集数据对比。

添加文件后，用 `tilt up -- --gpu-profile <name>` 部署，并检查 `nvidia-smi` 报告是否符合预期。[配置参考](../configuration.md)记录了每个字段。

!!! note "Profile 有防漂移保护"
    `tests/e2e/go/profile` 会对**所有**已知 profile 断言派生期望，无论当前运行的是哪一个，因此格式错误或不一致的 profile 会在单元测试中失败，而不是等到集群运行时才暴露。

### 调试

```bash
MOCK_NVML_DEBUG=1 nvidia-smi     # 跟踪 mock 应答的每一次 NVML 调用
```

## 风格

- 简洁、地道的 Go。高内聚、低耦合。
- 错误要带上下文包装、处理或记录，绝不静默丢弃。如果确实应当忽略，请在注释中说明原因。
- 注释在意图不明显处解释意图：写*为什么*，而不是*是什么*。写给一年后读这个文件的人，而不是写给当前评审者。
- 断言用 `testify/require`；测试允许时用 `t.Parallel()`。

## 相关链接

| 想了解 | 参见 |
|---|---|
| 启动集群 | [本地开发](local-development.md) |
| 运行各道关卡 | [测试](testing.md) |
| 从节点读取 NVLink fabric 状态 | [check-fabric](../tools/check-fabric.md) |
| 重新生成 NVML 桥接桩 | [generate-bridge](../tools/generate-bridge.md) |
| 把行为放到特性门控之后 | [为新行为加门控](feature-gates.md) |
| 提出设计 | [增强提案](enhancements.md) |
| 提交变更 | [Pull Request](pull-requests.md) |
