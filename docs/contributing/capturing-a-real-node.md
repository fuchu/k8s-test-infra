# 采集真实节点

profile 的质量取决于它对照校验的对象。本指南带你走完整个流程：从拥有真实 GPU 节点的 Kubernetes 集群出发，得到一份可提交的脱敏采集数据，再检查 Mokka profile 能在多大程度上复现真实节点。添加 GPU 型号、发布新驱动分支，或想知道 Mokka 已能模拟某平台的多少内容时，都可以使用这份指南。

每一步都用同一个工具 [`mokka-capture`](../tools/mokka-capture.md)，在你自己的机器上运行。流程产出三样东西：

- 每个节点及集群的原始输出目录（`mokka-capture collect`）；
- 同一次运行中，驱动容器里每个 getter 应答的 NVML 录制（`mokka-capture dump`）；
- 两者的脱敏副本（`mokka-capture scrub`），可安全提交到 `tests/hardware-captures/` 下。

## 采集覆盖什么

Mokka 模拟真实消费者读取的若干层。每行列出一个采集的接口面，以及 Mokka 目前在哪里生成它。

| 接口面 | 采集形式 | Mock 侧 |
|---|---|---|
| NVML 库 | `nvml-dump.json`、`nvml-symbols.txt` | `pkg/gpu/mocknvml/bridge`、`engine`、profile YAML |
| `nvidia-smi` | `smi-*.txt`、`smi-q.xml` | 同一引擎，通过随附的二进制 |
| 驱动 `/proc` 文件 | `proc-*.txt` | `internal/agent/gpudriver`、`internal/agent/migcaps` |
| 设备文件与主设备号 | `host-dev-nvidia.txt`、`host-proc-devices.txt` | 节点 agent、CDI 与 NRI 规格 |
| PCI sysfs 与 DMI | `host-lspci-*.txt`、`host-pci-*.txt`、`pci-*.txt`、`host-dmi.txt` | profile `pcie_topology`、`libmockfs` |
| NUMA 与 CPU | `host-numa*.txt`、`host-lscpu.txt`、`host-pagesize.txt` | `pcie_topology.numa_node` |
| NVLink 与 fabric | `smi-nvlink-*.txt`、`smi-q-d-FABRIC.txt`、`proc-capabilities.txt` | profile `nvlink`、IMEX 部署 |
| RDMA 设备与端口 | `rdma-*.txt`、`host-ib*.txt`、`host-rdma-link.txt` | profile `infiniband`、`libibmock` |
| 网络接口 | `net-*.txt` | 节点 agent |
| 内核模块与日志 | `sys-module-*.txt`、`kernel-log-nic-gpu.txt` | `libmockfs` |
| MIG | `smi-mig-*.txt`、`proc-mig-minors.txt` | `MOCK_MIG_PROFILES_CONFIG` |
| CDI 规格 | `cdi-*.txt` | 节点 agent |
| Kubernetes 视图 | `cluster/*` | Helm chart、e2e 断言 |

有些接口面无法从节点读取：fabric 的交换机侧拓扑（哪块 NIC 在哪条 rail 或 leaf 上），以及未安装 DRA driver 时它本应发布的属性。这些数据要从 fabric 的所有者处采集，或在 driver 安装后采集。

## 开始之前

你需要：

- 一个可检查其 GPU 节点的集群 kubeconfig，具有在 GPU Operator 命名空间中执行 `exec` 和创建特权 Pod 的权限。host Pod 会挂载节点根文件系统并以 root 运行。它的镜像（`-host-pod-image`）按摘要固定，每次退出都会删除该 Pod，但按摘要固定并不限制它安装工具：Pod 会从镜像发行版的软件源用 `apt-get install` 安装 `pciutils`、`rdma-core` 等。在不允许这样做的集群上，把 `-host-pod-image` 设为已包含这些工具的镜像；
- 运行 GPU Operator driver 容器的节点。驱动和 `nvidia-smi` 位于该容器中，而非宿主机上。如果驱动实际装在宿主机上，请在宿主机上直接运行 [`internal/capture/collect`](https://github.com/NVIDIA/k8s-test-infra/blob/main/internal/capture/collect/surfaces.go) 中的 `nvidia-smi` 和 `/proc` 命令；
- `kubectl`、用于构建 `mokka-capture` 的 Go，以及用于为 driver 容器构建其 Linux 副本的 Docker。

至少采集同一型号的两个节点。两个节点能显示哪些字段逐台变化、哪些描述型号。它们之间不同的值（托盘索引、槽位号、次设备号）应作为逐节点字段写进 profile。

采集时让节点处于空闲。随负载变化的读数，如利用率、温度和功耗，不参与比对。

## 第 1 步——构建 `mokka-capture`

先为本机构建工具，再为 driver 容器构建一个 Linux 副本。NVML dump 会加载驱动自身的 `libnvidia-ml.so.1`，因此它必须在 driver 容器内运行，并按节点架构构建。go-nvml 在运行时加载库，因此二进制是动态链接的：要在节点架构的主机上构建。

```bash
export OUT=$PWD/capture
go build -o bin/mokka-capture ./cmd/mokka-capture
bin/mokka-capture collect -out "$OUT" build-dump
```

这会写入 `$OUT/bin/mokka-capture-linux-<arch>`。如果 Docker 拉取镜像时卡住，先用匿名配置指向它：`export DOCKER_CONFIG=$(mktemp -d)`。

## 第 2 步——采集

列出节点，然后对每个 GPU 节点运行采集。`all` 会进行 driver 容器采集、宿主机采集，并进行一次集群采集：

```bash
kubectl get nodes
bin/mokka-capture collect -out "$OUT" -dump-binary "$OUT/bin/mokka-capture-linux-<arch>" \
  all <node-a> <node-b>
```

每条命令都在集群中运行，每个输出都写到你的机器上，因此每个输出都是 `$OUT/<node>/` 下的一个文件，无需从 Pod 中复制出来。命令有意拆分，方便单独重跑某一部分：

| 命令 | 运行位置 | 采集内容 |
|---|---|---|
| `driver <node>...` | driver 容器中 | `nvidia-smi`、NVML dump、`/proc/driver/nvidia` |
| `host <node>...` | 临时特权 Pod 中 | PCI、NUMA、设备文件、NIC、CDI 规格 |
| `network <node>...` | 同一 Pod 中 | 仅 NIC、接口和 RDMA 接口面，加上集群网络对象 |
| `cluster` | 你的机器上 | 节点标签、ResourceSlice、`ClusterPolicy`、DCGM 指标 |

host 命令会在 `default` 命名空间创建一个名为 `capture-<node>-<pid>` 的 Pod（每次运行一个）并在结束时删除。其工具（`pciutils`、`numactl`、RDMA 工具）来自 `apt`。如果集群无法访问 Ubuntu 软件源，sysfs 采集仍可用，其余会失败；若默认命名空间禁止特权 Pod，请设置 `-host-pod-namespace`。其他 flag 由 `mokka-capture collect -h` 列出。

## 第 3 步——检查采集结果

`collect` 最后会列出空文件。空文件几乎总是意味着连接中断，而不是答案为空；重跑产生它的那条命令即可。失败的命令还会在输出旁的 `errors.log` 中留下一行。而失败本身就是数据：某个 `nvidia-smi` 选项不受支持，或某个 getter 应答 `NOT_SUPPORTED`，这些正是 mock 需要复现的东西。

```bash
find "$OUT" -type f -size 0 ! -name errors.log
cat "$OUT"/*/errors.log
```

至少检查：

- `smi-L.txt` 列出了你期望的所有 GPU；
- 每个节点都有 `nvml-dump.json`，在 Vera Rubin 托盘上每块 GPU 约 1,550 次调用（`grep -o '"ret"' nvml-dump.json | wc -l` 可计数；该文件只有一行，因此 `grep -c` 总是打印 1）；
- 如果 operator 生成 CDI 规格，`cdi-specs-host.txt` 中包含这些规格文件；
- 有 NIC 的节点上存在 `host-ib*.txt` 和 `rdma-*.txt`。

## 第 4 步——脱敏

原始采集中带有来源机器的名字：GPU 和 fabric UUID、序列号、GUID、MAC 地址、主机名。**不要提交或发布它。** `mokka-capture scrub` 会写一份副本，把每个这样的值替换为占位符。整个运行中，每个不同的值对应一个占位符，因此某块 GPU 在每个文件中都保留同一个占位符：

```bash
bin/mokka-capture scrub -out scrubbed \
  -replace <node-a-name>=node-a -replace <node-b-name>=node-b \
  -replace <cluster-name>=tenant \
  -seed "$OUT/<node-a>/smi-L.txt" -seed "$OUT/<node-b>/smi-L.txt" \
  -report /tmp/substitutions.tsv \
  "$OUT/<node-a>" "$OUT/<node-b>" "$OUT/cluster"
```

`-replace` 覆盖工具无法按形状识别的名字：节点、集群和网络名。`-seed` 按索引顺序给 GPU 编号。二进制文件和 `errors.log` 会被跳过。NVML 录制是按形状脱敏的一个例外。NVML 以字节数组返回序列号、GUID 和 fabric 集群 UUID，以整数返回 PDI，没有文本规则能匹配它们。因此，dump 在写入时按字段名遮盖这些值，`scrub` 对交给它的录制（明文或 `.json.gz`）做同样处理。

然后检查结果。脱敏是尽力而为，不是保证：

1. 确认没有原值残留。`-report` 表列出了工具找到的每个原值；确认没有一个出现在输出中。它无法列出工具未识别的值，因此还要搜索你从别处知道的标识符（机架标签、DMI 中的机箱序列号，或节点通过 `nvidia-smi -L` 得到的 GPU UUID），并留意它们在采集中的形式：文本、十六进制、十进制数字和字节值列表（`nvml-dump.json`）。

   ```bash
   cut -f2 /tmp/substitutions.tsv | sort -u | while read -r v; do
     [ ${#v} -ge 8 ] && grep -rqF -- "$v" scrubbed && echo "LEAK: $v"
   done
   ```

   报告中包含原值，因此绝不要提交它。
2. 搜索模式无法知道的内容：项目、租户、站点和用户名、内部域名和镜像仓库。

   ```bash
   grep -rniE '<tenant>|<site>|<user>|\.corp|\.internal|token|secret|passw' scrubbed
   ```

3. 阅读集群文件。省略任何带有平台不需要的环境细节的文件，例如带注解的完整 Node 对象、operator Pod 和 ConfigMap 列表，以及 operator 日志。
4. 不要提交专有二进制。驱动的 `libnvidia-ml.so` 属于 NVIDIA；只保留符号列表。

脱敏后的 `smi-q.xml` 同时也是 `tests/e2e/go/assertions/nvidiasmi/testdata/hardware/` 的硬件 fixture。`mokka-capture scrub -check <file>` 验证它只包含 fixture 所需的占位符；`TestHardwareCapturesAreScrubbed` 在 CI 中强制这一点。

最后，当采集数据放到 `tests/hardware-captures/` 下后运行一次 `go test ./internal/capture/scrub -run TestCommittedCapturesAreScrubbed`；CI 会在每次变更时运行它。它只检查形状和字段名，因此上面的阅读仍然重要。

## 第 5 步——存储

按惯例，脱敏采集放在 `tests/hardware-captures/<platform>/` 下：节点一个目录、一个 `cluster/` 目录，以及一个记录以下内容的 `README.md`：

- 平台和被采集的节点（型号、架构、操作系统、内核和页大小）；
- 驱动和 CUDA 版本，以及驱动的安装方式；
- 采集日期；
- 采集不包含什么，以及原因。

保持提交内容精简。仓库被克隆的次数远多于采集被阅读的次数，而 git 会永久保留文件的每个版本。因此只提交 profile 或测试所依赖的内容，其余省略：

- **gzip NVML dump**（`gzip -9 -n nvml-dump.json`）；`mokka-capture diff nvml` 可直接读取 `.json.gz`。
- **每种型号完整提交一个节点。** 同型号的第二个节点可以让你了解哪些字段逐台变化（托盘索引、槽位号、设备次设备号），但其文件几乎与第一个节点相同。与其把它也提交，不如在 `README.md` 中记录差异以及两者的一致程度（例如匹配的 NVML 返回码占比）。
- **省略已被更小文件总结的批量内容**：`lspci -vv` 列表（`host-lspci-*`）、NVLink 错误计数器（`smi-nvlink-e.txt`）和支持时钟表（`smi-q-d-SUPPORTED_CLOCKS.txt`，`smi-q.xml` 也包含它）。在 `README.md` 中写明省略了什么。

除必要之处外，不要把失败留在提交树中：`README` 注释胜过一份点名机器的 `errors.log`。

## 第 6 步——与 mock 比对

有了采集数据，就可以衡量某个 profile 距离真实节点有多远。每项比对都是一条命令，接受真实采集和从 mock 取到的同一接口面。命令中 `REAL` 是真实节点的目录（原始采集或已提交的采集），`MOCK` 是保存 mock 文件的目录。

用同样的命令获取 mock 的文件，指向运行着待测 profile 的 Mokka 集群。`-mock` 会在一个替代 Pod 中运行 driver 命令，并选择让 Pod 接受注入——原因见 [Mokka 节点](../tools/mokka-capture.md#a-mokka-node)——`-dump-binary` 则是为该集群架构构建的：

```bash
bin/mokka-capture collect -mock -dump-binary <linux binary> -out mock all <node>
MOCK=mock/<node>
```

1. **符号。** 桥接实现了真实库中的哪些函数？`generate-bridge -symbols` 读取真实库导出的符号，说明桥接实现了多少个、其中多少个是生成的桩（在真实 GPU 有应答的地方，这些桩返回 `NOT_SUPPORTED`），以及哪些完全缺失。把构建出的 mock 库的符号（`nm -D --defined-only libnvidia-ml.so.1 | awk '{print $3}' | sort`）作为 `-mock-symbols` 传入。不传时，报告会读取桥接源码，而源码中看不到 C shim 导出的旧版无版本号名称，于是把它们算作缺失。`collect -mock` 会让 `$MOCK/nvml-symbols.txt` 为空，因此要从你构建的库中获取：

   ```bash
   go run ./cmd/generate-bridge -symbols "$REAL/nvml-symbols.txt" \
     -mock-symbols mock-nvml-symbols.txt
   ```

2. **数值。** 比对 NVML dump，它给出返回码和板卡数值的匹配率，并列出每处差异：

   ```bash
   bin/mokka-capture diff nvml "$REAL/nvml-dump.json.gz" "$MOCK/nvml-dump.json"
   ```

   字段 ID 在不同驱动分支间不同；dump 会探测每个 ID 并记录逐字段返回码，因此布局变化会直接显现。
3. **`nvidia-smi`。** 逐元素比对两份 `-q -x` 文档：

   ```bash
   bin/mokka-capture diff smi "$REAL/smi-q.xml" "$MOCK/smi-q.xml"
   ```

   报告说明 mock 拥有真实文档中的多少元素、其中多少值相同，并列出缺失元素和不同值。不属于板卡属性的值不参与比对；报告会逐一列出它们，[`diff smi`](../tools/mokka-capture.md#diff-smi) 说明它们是哪些。
4. **接口面。** 对 `host-*`、`proc-*`、`rdma-*` 和 `net-*` 文件没有工具可用。把它们与节点 agent 为该 profile 渲染的内容逐一对照，判断差异是否重要。真实节点与 mock 在编号和计数器上必然不同，因此这一步没有通过或失败。

按接口面报告覆盖率，例如"NVML 返回码：588 中匹配 472"。这个数字说明了某个 profile 能在多大程度上复现节点。每次 diff 最后都会列出它没有比对的值；请把它们与分数并列说明，因为分数对这些值不作任何陈述。

## 从采集到 profile

1. 从已提交的采集生成 profile，用同型号的一个现有 profile 作为 `-base`，提供采集无法给出的内容（标识符、仅 mock 才有的块）；对于尚无 profile 的 GPU，省略 `-base`：

   ```bash
   bin/mokka-capture profile \
     -base deployments/nvml-mock/helm/nvml-mock/profiles/<name>.yaml \
     -o pkg/gpu/mocknvml/configs/mock-nvml-config-<name>.yaml \
     tests/hardware-captures/<platform>/<node>
   ```

   每个值的注释都指明它在采集中的来源，`-explain` 列出每块 GPU 对每个键的应答；参见 [`profile`](../tools/mokka-capture.md#profile)。如果值有错，要修正规则，而不是修改生成的文件。dump 只记录逐设备的 NVLink 带宽模式 getter，因此模式为系统级的板卡（Hopper）要从 `-base` 取 `nvlink.bw_mode`。
2. 把结果及其旁边的 `.mig.yaml` 复制到 `deployments/nvml-mock/helm/nvml-mock/profiles/<name>.yaml` 和 `profiles/mig/<name>.yaml`；chart 中的副本是事实来源，`pkg/gpu/mocknvml/configs/` 下的那份用于在 Kubernetes 之外运行库。[配置参考](../configuration.md)记录了每个字段。
3. 把该采集加入 `internal/capture/profile/golden_test.go` 中的 `goldens`，运行 `go test ./internal/capture/profile -update`，这样后续对规则的修改会显示为该 profile 的一处 diff。
4. 部署并查看：`tilt up -- --gpu-profile <name>`，然后检查 mock 上的 `nvidia-smi` 是否报告采集所示内容。
5. 运行防漂移守卫：`go test -tags=e2e ./tests/e2e/go/profile/... ./tests/e2e/go/assertions/...` 会对所有 profile 断言派生期望，因此不一致的 profile 会在这里失败，而不是等到集群上才失败。
6. 把 `nvidia-smi -q -x` fixture 加入 `tests/e2e/go/assertions/nvidiasmi/testdata/hardware/`，并在该目录的 README 中添加一行。`hardware_test.go` 断言的是读数而非解析成功，因为元素改名会被解码为缺失，而不是错误。

租用或共享的机器在保留其采集前还需一项检查：确认它没有 MIG 分区、没有 vGPU 模式、没有残留进程。

## 排障

| 症状 | 原因与修复 |
|---|---|
| `errors.log` 中出现 `TLS handshake timeout` 或 `connection refused` | API 连接中断。恢复后重跑失败的部分；`collect` 会在每个节点前检查连接。 |
| 某个选项报 `nvidia-smi: ... not recognized` | 驱动的 `nvidia-smi` 没有该选项。失败本身就是答案；保留它。 |
| `-q -d <section>` 报 `Failed to parse` | 该 section 名在此驱动上无效。各 section 按文件分别采集，因此一个无效名字只损失一个文件。 |
| 运行 dump 时出现 `GLIBC` 错误 | driver 容器的 libc 比构建镜像的旧。在基础更旧的镜像上构建 Linux 版 `mokka-capture`。 |
| 找不到 `nvidia-ctk` | 它不在 driver 容器中。采集从宿主机读取 CDI 规格，并从 toolkit 容器运行 `nvidia-ctk`。 |
| host Pod 一直处于 `Pending` | Pod 安全策略禁止在该命名空间运行特权 Pod。把 `-host-pod-namespace` 设为允许它们的命名空间。 |
| 从 Pod 看 `/var/run/cdi` 为空 | `/var/run` 是指向 `/run` 的符号链接，在 Pod 内解析。请读取 `/host/run/cdi`。 |
