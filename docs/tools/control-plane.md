# control-plane

运行 [Mokka 控制面](../control-plane.md)并提供 HTTP 健康探针的二进制文件。它通过 Kubernetes API 协调声明式模拟 GPU（sGPU）资产清单。架构页面介绍部署位置、Node 投影、leader 选举和 [Stage 1 排除项](../control-plane.md#stage-1-exclusions)；本页描述命令行和探针。

它默认关闭：只有当 `controlPlane.enabled` 为 `true` 时，chart 才会渲染 Deployment。

## Kubernetes 前置条件

该控制器需要 Kubernetes API 访问权限；没有仅健康检查模式或离线模式。通过 chart 安装需要 Kubernetes 1.30+ 以获得稳定的 `ValidatingAdmissionPolicy`。启动控制器前请先安装 Mokka 自定义资源定义（CRD）；参见[控制器安装](../control-plane.md#install)。

在集群中，二进制使用 in-cluster 配置和挂载的 ServiceAccount token。chart 会为控制器准备 ServiceAccount 和基于角色的访问控制（RBAC）权限。在集群外，传入 `--kubeconfig` 或把 `MOKKA_CP_KUBECONFIG` 设为 kubeconfig 文件路径。该路径留空时使用 in-cluster 配置，而不是 `KUBECONFIG` 或默认的 `~/.kube/config`。

对于针对 chart 所管理安装运行的本地进程，kubeconfig 必须以 chart 的控制器 ServiceAccount 身份认证。其准入策略把 `SGPURack` 的创建和更新限制为该身份，仅有 RBAC 权限并不够。请使用与已部署副本相同的 leader 选举命名空间和 Lease 名称，以便一次只有一个进程进行协调。Lease 命名空间必须存在，且该身份必须有在该命名空间 get、create 和 update Lease 的权限。chart 的 [RBAC 规则](https://github.com/NVIDIA/k8s-test-infra/blob/main/deployments/nvml-mock/helm/nvml-mock/templates/controlplane/rbac.yaml)定义了所需的 Node 和 Mokka 资源权限。

## Flags

下面列出了环境变量回退的 flag；两者都设置时以 flag 为准。短横线表示该 flag 没有环境变量绑定。

| Flag | 环境变量 | 默认值 | 说明 |
|------|----------------------|---------|-------------|
| `--feature-gates` | `MOKKA_FEATURE_GATES` | 空 | 逗号分隔的 `Name=true\|false` 对。参见[特性门控](../feature-gates.md) |
| `--listen-addr` | `MOKKA_CP_LISTEN_ADDR` | `:8080` | HTTP 探针服务监听地址 |
| `--log-level` | `MOKKA_LOG_LEVEL` | `info` | `debug`、`info`、`warn` 或 `error`。`warning` 是 `warn` 的别名；空值回退到 `info` |
| `--log-format` | `MOKKA_LOG_FORMAT` | `json` | `json` 或 `plain`；空值回退到 `json` |
| `--shutdown-timeout` | `MOKKA_CP_SHUTDOWN_TIMEOUT` | `5s` | SIGINT/SIGTERM 时等待进行中 HTTP 请求排空的时间预算 |
| `--kubeconfig` | `MOKKA_CP_KUBECONFIG` | 空 | kubeconfig 路径；空则使用 in-cluster 配置 |
| `--leader-election-namespace` | `MOKKA_CP_LEADER_ELECTION_NAMESPACE` | `POD_NAMESPACE`，为空则 `default` | 存放 leader 选举 Lease 的命名空间 |
| `--leader-election-name` | `MOKKA_CP_LEADER_ELECTION_NAME` | `control-plane.mokka.nvidia.com` | leader 选举 Lease 的名称 |
| `--leader-election-lease-duration` | — | `15s` | 其他副本可以接管的 Lease 有效期 |
| `--leader-election-renew-deadline` | — | `10s` | leader 在放弃领导权前重试续约 Lease 的时长 |
| `--leader-election-retry-period` | — | `2s` | leader 选举尝试之间的间隔 |
| `--workers` | — | `2` | 每个控制器队列的 worker 数 |
| `--status-debounce` | — | `100ms` | 聚合状态更新前的静默期 |
| `--status-progress-interval` | — | `1s` | 持续变化时聚合状态的最大陈旧时间；零表示取 `1s` 与去抖间隔中较大者 |
| `--live-node-get-timeout` | `MOKKA_CP_LIVE_NODE_GET_TIMEOUT` | `2s` | Node 离开过滤缓存后执行精确 GET 的超时 |
| `--kube-api-qps` | — | `50` | 客户端 Kubernetes API 请求速率限制 |
| `--kube-api-burst` | — | `100` | 客户端 Kubernetes API 请求突发限制 |

无法识别的 `--log-level` 或 `--log-format` 会导致启动失败，而不是静默回退。worker 数、实时 Node GET 超时、API QPS 和 burst 必须为正；状态间隔必须非负。非零的状态进度间隔不得短于去抖间隔。Leader 选举时长必须满足 `lease > renew > retry × 1.2` 且为正值。

## HTTP 探针

| 端点 | 响应 |
|----------|----------|
| `GET /healthz` | HTTP 200，正文 `{"ok":true}`，表示 HTTP 服务正在工作；不检查 Kubernetes API 访问 |
| `GET /readyz` | 控制器就绪时 HTTP 200、`{"ok":true}`；否则 HTTP 503、`{"ok":false,"reason":"controller is not ready"}` |

就绪状态跟随[控制器的缓存与 leader 选举状态](../control-plane.md#observe-and-troubleshoot)，而不是资产清单收敛。仅有健康的 HTTP 监听器并不能让副本就绪。

## 用法

要在仓库根目录进行本地开发，先准备满足 [Kubernetes 前置条件](#kubernetes-prerequisites)的 kubeconfig。下面的例子假设所属 Helm release 位于 `mokka` 命名空间，并使用默认 Lease 名称：

```bash
go run ./cmd/control-plane \
  --kubeconfig /path/to/controller.kubeconfig \
  --leader-election-namespace mokka \
  --listen-addr :9090 \
  --log-level debug
```

在另一个终端检查探针（就绪状态最初可能返回 503）：

```bash
curl -i localhost:9090/healthz
curl -i localhost:9090/readyz
```

使用默认值时，chart 会渲染这些参数。Kubernetes 会把 `$(POD_NAMESPACE)` 展开为 Pod 所在命名空间：

```text
/usr/local/bin/control-plane \
  --listen-addr=:8080 \
  --log-level=info \
  --log-format=json \
  --shutdown-timeout=5s \
  --leader-election-name=control-plane.mokka.nvidia.com \
  --leader-election-namespace=$(POD_NAMESPACE) \
  --workers=2 \
  --kube-api-qps=50 \
  --kube-api-burst=100
```

这些值来自 chart 的 `controlPlane.service.port`、`controlPlane.logging.level`、`controlPlane.logging.format`、`controlPlane.shutdownTimeout`、`controlPlane.leaderElection.name`、`controlPlane.workers`、`controlPlane.kubeAPIQPS` 和 `controlPlane.kubeAPIBurst`。其他 flag 保留二进制默认值。

!!! note "在宽限期内为排空留出空间"

    关闭流程先是 10 秒的 preStop 睡眠——在进程停止前让端点退出 Service 轮转——然后是排空。两者都必须落在 `controlPlane.terminationGracePeriodSeconds`（默认 30 秒）之内，因为到这个边界 kubelet 会发送 SIGKILL，无论还有多少请求在进行。`--shutdown-timeout` 只限制 HTTP 排空，不限制控制器清理；移除控制器之前请遵循[安全停用与卸载流程](../control-plane.md#disable-or-uninstall-safely)。

## 部署

镜像基于 `gcr.io/distroless/static-debian12:nonroot`，从 `deployments/control-plane/Dockerfile` 构建，暴露 8080 端口，并使用 `/readyz` 和 `/healthz` 作为就绪与存活探针。

本地开发时，`tilt up -- --control-plane` 会构建镜像，并安装 CRD 和 `controlPlane.enabled=true` 的 chart。

## 另见

- [命令行工具](README.md)
- [Mokka 控制面](../control-plane.md)——架构与生命周期
- [安装](../helm-chart.md)——nvml-mock chart
