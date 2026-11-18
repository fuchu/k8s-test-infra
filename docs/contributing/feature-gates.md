# 为新行为加门控

某项行为若要在成为 Mokka 默认契约之前先发布，就把它放到特性门控之后。Mokka 通过 `internal/features` 使用 Kubernetes 的 [`component-base` 特性门控](https://pkg.go.dev/k8s.io/component-base/featuregate)，与 Kubernetes 组件、Cluster API 和 Kueue 的做法一致。运维人员如何设置门控以及各阶段的含义，参见[特性门控](../feature-gates.md)。

## 门控还是 chart 取值？

特性门控是运行中服务内部某项行为的临时逃生舱，行为稳定后即移除。永久性选择应改用 chart 取值，例如 chart 是否部署某个组件（`nri.enabled`），或组件以何种模式运行（`nri.deviceInjectionMode`）：由 chart 决定渲染什么，而二进制内部的门控做不到这一点。

## 添加门控

在 `internal/features/features.go` 中声明门控及其初始阶段。名称使用 PascalCase，描述行为本身而非其实现：

```go
const (
    // owner: @github-handle
    // issue: https://github.com/NVIDIA/k8s-test-infra/issues/1234
    //
    // Selects the alternate placement planner.
    AlternatePlanner featuregate.Feature = "AlternatePlanner"
)

var versionedSpecs = map[featuregate.Feature]featuregate.VersionedSpecs{
    AlternatePlanner: {
        {Version: version.MajorMinor(0, 5), Default: false, PreRelease: featuregate.Alpha},
    },
}
```

`Version` 是引入该阶段的 Mokka 次版本。

在同一个 pull request 中：

- 把门控以 `{ "type": "boolean" }` 的形式加入 chart 的 `values.schema.json` 中的 `featureGates.properties`。在 schema 与已注册门控一致之前，`TestChartSchemaListsRegisteredGates` 会失败。
- 把门控列入[可用门控](../feature-gates.md#available-gates)。
- 添加 `Added` 类型的[变更日志片段](pull-requests.md#changelog-entries)，点出门控名称。

## 检查门控

只有 `node-agent`、`nri-plugin` 和 `control-plane` 会应用门控。`nvml-mock-ctl`（同时也作为 chart 的 `allocation-watcher` sidecar 运行）不会应用；加载进工作负载的模拟 NVML 和 CUDA 库也不会：`features.Enabled` 在那里总是返回门控的默认值，且不报错。`pkg/gpu/` 下的包，以及它们导入的 `internal/gpuarch` 等内部包，也会在这些进程中运行，因此不得调用 `features.Enabled`。

在拥有该行为的服务中读取门控，并把结果作为值向下传递，例如作为服务在启动时构建的配置中的一个字段：

```go
cfg.Planner = planner.Current
if features.Enabled(features.AlternatePlanner) {
    cfg.Planner = planner.Alternate
}
```

模拟库内部的行为需要由某个服务以同样的方式转交决定，例如通过库读取的配置。只在某个服务内部运行的包，如 `internal/agent`、`internal/nri` 或 `internal/controlplane`，可以直接在新旧行为分叉处检查门控。要检查某个包，确保 `go list -deps ./cmd/nvml-mock-ctl ./pkg/gpu/mocknvml/bridge` 不列出它。

不要为门控读取 flag、环境变量或 Helm 取值；这些由 `internal/features` 统一拥有，三个服务以相同方式应用它们。

## 测试两种状态

因为大多数代码以值的形式收到决定，大多数测试根本不需要门控。在必须翻转进程级门控的测试中，使用 `features.SetFeatureGateDuringTest`，它会在测试结束时恢复之前的值：

```go
func TestAlternatePlanner(t *testing.T) {
    features.SetFeatureGateDuringTest(t, features.AlternatePlanner, true)
    // ...
}
```

设置门控的测试不得调用 `t.Parallel()`。切勿在测试中把门控拒绝的值传给 `features.ConfigureFromCLI`。进程级门控会保留被拒绝的条目，此后在同一测试二进制中对它的每次更改都会失败。拒绝行为要放在一个全新的门控上测试，`internal/features` 就是这么做的。

## 改变门控阶段

为改变阶段的版本追加一条 spec，保留更早的记录作为历史：

```go
AlternatePlanner: {
    {Version: version.MajorMinor(0, 5), Default: false, PreRelease: featuregate.Alpha},
    {Version: version.MajorMinor(0, 6), Default: true, PreRelease: featuregate.Beta},
    {Version: version.MajorMinor(0, 7), Default: true, PreRelease: featuregate.GA, LockToDefault: true},
},
```

- GA（正式可用）门控设置 `LockToDefault: true`，于是该行为再也无法关闭。
- 已弃用门控默认为 `false`。当行为再也无法重新开启时，设置 `LockToDefault: true`。
- 已锁定的门控保持锁定，因此 GA 门控是被移除，而不是被弃用。
- 门控锁定时，把它的 schema 条目改为 `{ "const": <default> }`，使 Helm 在滚动发布前拒绝覆盖。
- 更新可用门控下的条目，并添加变更日志片段：晋升用 `Changed`，门控被弃用用 `Deprecated`。

如果门控用于淘汰一个从未有过门控的行为，则以一条未锁定的 `version.MajorMinor(0, 0)` 条目记录该行为作为起点：

```go
LegacyPlanner: {
    {Version: version.MajorMinor(0, 0), Default: true, PreRelease: featuregate.GA},
    {Version: version.MajorMinor(0, 6), Default: false, PreRelease: featuregate.Deprecated},
},
```

没有它，component-base 会拒绝在 0.6 这样的版本上首条 spec 即为 deprecated 的情况；其错误要求提供一个 "1.0 entry"，对 Mokka 的 0.x 版本来说就是 0.0。

## 移除门控

当[移除策略](../feature-gates.md#stages)允许时，移除 GA 或已弃用的门控；alpha 门控可在任何版本中移除。目前还没有东西检查该策略，首个达到 GA 的门控应添加一个测试，把它的阶段版本与 chart 的 `appVersion` 比较。移除时要删除常量、它的 spec、schema 条目、不再可能执行的分支、针对另一状态的测试，以及可用门控下的条目。此外添加 `Removed` 变更日志片段，让运维人员在拒绝该门控的升级之前把它从取值中去掉。

## 相关链接

| 想了解 | 参见 |
|---|---|
| 设置门控及各阶段含义 | [特性门控](../feature-gates.md) |
| 运行测试 | [测试](testing.md) |
| 编写变更日志片段 | [Pull Request](pull-requests.md#changelog-entries) |
