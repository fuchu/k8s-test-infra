# 测试与 Lint

两条命令几乎覆盖一切：

```bash
make lint-fix    # 修复可修复的问题，报告其余问题
make test        # 带竞态检测器的单元测试
```

## `make lint` 实际做了什么

它不只是 `golangci-lint`。这是最常让人意外的关卡，因为其中几步会在你没有编辑的文件上失败：

| 步骤 | 失败条件 |
|---|---|
| `gen-check` | 生成代码已过期——运行 `make gen` |
| `go mod tidy` diff | `go.mod` 或 `go.sum` 不同步 |
| Vendor 检查 | 重新出现了 `vendor/` 目录；依赖应通过 Go proxy 解析 |
| Go proxy 摘要 | 内嵌的 `setup-dgxc-goproxy` action 被就地修改 |
| 文档链接 | 某个 Markdown 链接在仓库中无处可解析 |
| 组件页面同步 | `cmd/` 下的某个二进制没有参考页面，或某个页面没有对应二进制——在 `docs/tools/` 下添加或删除页面 |
| `go vet`、`golangci-lint`、`govulncheck` | 常规问题 |

`make lint-fix` 运行同样的检查，并自动修复能修复的部分。

## 测试分层

| 层 | 命令 | 需要集群 |
|---|---|---|
| 单元测试 | `make test` | 否 |
| Helm chart | `make helm-tests`、`make helm-crds-tests` | 否 |
| Shim 集成 | `make test-mockfs`、`make test-mocknvml-bridge` | 否 |
| E2E 框架单元测试 | `make test-e2e-framework` | 否 |
| 端到端 | `make e2e` 及相关目标 | **是** |

除最后一项外，其余都可以在没有集群的笔记本上运行。

### 端到端

E2E 套件会连接到你已创建的集群；它既不构建镜像，也不创建集群。先用 [Tilt](local-development.md) 准备好两者，然后：

```bash
make e2e                  # 独立场景，默认 profile
make e2e-dra              # DRA driver
make e2e-compute-domain   # 通过 DRA driver 运行 ComputeDomain（双机架集群）
make e2e-gpu-operator     # GPU Operator
make e2e-multi-node       # 异构 a100/t4 机群
make e2e-nri              # 节点级 NRI 注入
make e2e-nfd              # NFD 标签来源
make e2e-mig              # 通过 device plugin 进行 MIG 分区
make e2e-vgpu             # vGPU 宿主机与客户机
```

!!! warning "`make e2e` 不会运行全部"
    它默认套用一个标签过滤器，排除上面的各个场景，因此单独运行 `make e2e` 只会跑独立场景。其余场景请使用各场景专属的 make 目标。

套件自身的手册——场景、Ginkgo 标签、环境变量和诊断——位于 [`tests/e2e/README.md`](https://github.com/NVIDIA/k8s-test-infra/blob/main/tests/e2e/README.md)。实际演练了哪些上游消费者版本，记录在 [`tests/e2e/VERSION-MATRIX.md`](https://github.com/NVIDIA/k8s-test-infra/blob/main/tests/e2e/VERSION-MATRIX.md)。

## 编写测试

断言使用 `testify/require`，测试允许时使用 `t.Parallel()`。添加那些断言值得保护的行为的测试，而不是复述实现的测试。

变更涉及 Helm chart 时，运行 `make helm-tests`——chart 有自己的单元测试套件和快照测试，能捕获 Go 测试发现不了的模板回归。

## CI 运行什么

这些就是各道关卡。在本地运行它们可精确复现 CI：

```bash
make lint
make test
make test-mockfs
make test-mocknvml-bridge
make test-e2e-framework
make helm-tests          # 仅 chart 路径
make helm-crds-tests     # 仅 chart 路径
make verify-greptile
```

!!! note "为什么 E2E 没有在你的 pull request 上运行"
    单元、lint 和 chart 检查在 pull request 本身上运行。E2E 套件则从 copy-pr-bot 创建的镜像分支 `pull-request/N` 运行，因此出现得稍晚，且位于一次单独的运行中——不在 PR 事件上。

## 相关链接

| 想了解 | 参见 |
|---|---|
| 启动供测试的集群 | [本地开发](local-development.md) |
| 提交变更 | [Pull Request](pull-requests.md) |
