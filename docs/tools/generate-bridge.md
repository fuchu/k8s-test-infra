# generate-bridge

保持模拟 NVML 库在 ABI 层面完整。手写桥接代码未实现的每个 NVML 函数都会得到一个自动生成的 cgo 桩，因此该库仍导出完整符号集，调用未实现函数的消费者会得到确定的应答，而不是链接错误。

它从 go-nvml 的 `nvml.go` 读取函数列表，从 `nvml.h` 读取 C 原型，并从桥接目录中读取现有的 `//export` 指令。剩下的部分——`nvml.go` 中命名但没有手写实现的每个函数——会被写入 `stubs_generated.go`。

因此手写实现总会取代它的桩。你永远不需要手工删除桩；添加真正的函数并重新运行生成器即可。

## 运行

```bash
make gen
```

这是受支持的路径：它通过模块缓存解析 go-nvml，并运行 `pkg/gpu/mocknvml/bridge/helpers.go` 中的 `//go:generate` 指令。

只读模式无法通过 `make gen` 到达，因此需要直接调用二进制文件：

```bash
GO_NVML_DIR=$(go list -m -f '{{.Dir}}' github.com/NVIDIA/go-nvml)

go run ./cmd/generate-bridge -stats    -input  "$GO_NVML_DIR/pkg/nvml/nvml.go"
go run ./cmd/generate-bridge -validate -header "$GO_NVML_DIR/pkg/nvml/nvml.h"
```

## 模式

| 模式 | 需要 | 行为 |
|---|---|---|
| generate（默认） | `-input` 和 `-header` | 无条件写入 `-output` |
| `-stats` | `-input` | 打印 NVML 覆盖率表——函数总数、手写实现数、生成的桩数，以及每个文件的 `//export` 计数——然后退出 `0` |
| `-symbols` | 符号文件 | 读取真实 `libnvidia-ml.so` 导出的符号，报告桥接实现了多少个、有多少个是应答 `NOT_SUPPORTED` 的生成桩、哪些缺失，然后退出 `0` |
| `-validate` | `-header` | 把每个手写导出函数的 Go 参数数量与其 `nvml.h` 原型比较。每处不匹配打印一条 `WARNING: <file>:<line>` 后退出 `1`，否则退出 `0` |

`-symbols` 是与真实驱动的对比，而不是与 go-nvml 的对比：从采集数据取符号列表（`nvml-symbols.txt`），并传入 `-mock-symbols`，这样 C shim 提供的别名就不会被报告为缺失。

```bash
go run ./cmd/generate-bridge -symbols real/nvml-symbols.txt -mock-symbols mock/nvml-symbols.txt
```

同时设置两个只读标志时，`-stats` 优先。必需路径缺失或不可读会在做任何工作之前退出 `1`，并给出指向 `make gen` 的提示。

## Flags

| Flag | 默认值 | 说明 |
|------|---------|-------------|
| `-input` | 无 | NVML Go 包装文件（`nvml.go`） |
| `-header` | 无 | NVML C 头文件，用于提取原型 |
| `-bridge` | `pkg/gpu/mocknvml/bridge` | 扫描现有实现的目录 |
| `-output` | `pkg/gpu/mocknvml/bridge/stubs_generated.go` | 生成的桩写入位置 |
| `-stats` | `false` | 打印覆盖率表并退出 |
| `-validate` | `false` | 对照 `nvml.h` 检查导出函数参数数量 |
| `-symbols` | 无 | 真实库导出符号的文件（`nm -D --defined-only` 的输出或每行一个名称）；报告桥接对它们的覆盖率 |
| `-mock-symbols` | 无 | 与 `-symbols` 配合：构建出的模拟库的同样列表，使 C shim 导出的旧版无版本号名称也算作已导出 |

## 为什么要提交输出

`stubs_generated.go` 已签入仓库，`make gen-check` 会重新运行生成，并在桥接目录出现改动时失败。

这道守卫之所以有价值，是因为触发条件不可见：升级 go-nvml 会增加新的 NVML 入口点，但不会重新生成任何东西，因此没有它，模拟库会悄然停止导出新符号，只有调用某个新符号的消费者才会发现。

## 另见

- [命令行工具](README.md)
- [库与 Shims](../components/libraries-and-shims.md)——桥接在模拟 NVML 库下的位置
- [本地开发](../contributing/local-development.md)
