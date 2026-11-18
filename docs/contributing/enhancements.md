# 增强提案

Mokka 增强提案（Mokka Enhancement Proposal，MEP）是针对重大变更的设计文档。该流程改编自 Kubernetes 的 [KEP](https://github.com/kubernetes/enhancements/tree/master/keps) 流程。

大多数变更不需要 MEP。错误修复和常规特性直接进入 pull request。

## 何时需要写

变更符合以下任一情况时，请写 MEP：

- 重构现有组件的组合方式；
- 增加或移除顶层组件——chart、CLI、服务；
- 改变用户可见的契约，例如配置或 Helm 取值；
- 引入新的依赖、平台目标或发布产物。

拿不准时，先开一个 discussion，维护者会告诉你需要与否。

## 怎么写

1. **复制模板。** 把 `enhancements/template.md` 复制为 `enhancements/meps/NNNN-short-title/README.md`，其中 `NNNN` 是下一个未使用的四位数编号。
2. **先填 Summary、Motivation 和 Goals。** 内容要足以让评审者在你就 Design Details 投入大量精力之前，先对方向展开讨论。在这个阶段得知整个思路是错的，代价要小得多。
3. **打开 pull request**，打上 `kind/mep` 标签。讨论在 PR 上进行。
4. **迭代。** 随着共识形成，扩充 Design Details、Risks 和 Alternatives。
5. **合并**——维护者批准后即可。

已合并的 MEP 记录的是意图，并不承诺代码已经与之相符。实现落在后续 pull request 中，并通过编号（`MEP-NNNN`）引用它。

## 保持诚实

MEP 描述的是它被接受时的设计。有些已完整实现，有些只实现了一部分，还有些已被另一种方式取代——而读者仅凭文档无法分辨。

如果你实现、修改或放弃了提案的某部分，请在同一变更中修订 MEP。被取代的提案保留在 `meps/` 中，而不是删除：历史要保留，而不是重写。

## 阅读现有提案

| MEP | 提议内容 |
|---|---|
| [MEP-0001](https://github.com/NVIDIA/k8s-test-infra/tree/main/enhancements/meps/0001-mokka-control-plane) | 一个负责集群级模拟 GPU 资产清单的控制面 |
| [MEP-0002](https://github.com/NVIDIA/k8s-test-infra/tree/main/enhancements/meps/0002-device-plugin-nri-composition) | NRI 插件如何与 NVIDIA device plugin 组合 |
| [MEP-0003](https://github.com/NVIDIA/k8s-test-infra/tree/main/enhancements/meps/0003-node-agent) | 节点守护进程：用一个协调器替代十几个 CLI |
| [MEP-0006](https://github.com/NVIDIA/k8s-test-infra/tree/main/enhancements/meps/0006-automatic-cdi-setup) | 从节点 Pod 自动完成节点的 CDI 配置 |
| [MEP-0007](https://github.com/NVIDIA/k8s-test-infra/tree/main/enhancements/meps/0007-vgpu-host-guest) | 根据 profile 模拟 vGPU 宿主机与客户机 |

MEP-0003 还包含本项目关于哪些契约接口面已模拟、哪些未模拟的最完整记录。

## 相关链接

| 想了解 | 参见 |
|---|---|
| 提交实现 | [Pull Request](pull-requests.md) |
| 组件当前如何组合 | [架构](../architecture.md) |
