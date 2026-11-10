# 可观测性（Prometheus + Grafana）

在没有 GPU 的集群上，Prometheus 抓取的是真实、未经修改的 NVIDIA `dcgm-exporter`，而后者读取的是 Mokka 的模拟 `libnvidia-ml.so`；Grafana 再把结果渲染出来。随后有两条手动触发器注入 GPU 故障，并**断言故障确实到达了 Prometheus**，否则报错——也就是说，整条抓取路径是被断言验证的，而不是靠肉眼看。

`tilt up -- --observability` 会在 `grafana` 资源的健康检查通过后，把 Grafana 服务提供在 <http://localhost:3000/d/mokka-gpu>（账号 `admin` / 密码 `mokka`）。这套栈为 Kind 做了精简（无 Alertmanager、保留 2 小时）；原因写在了 `kube-prometheus-stack.values.yaml`。

## 两个故障触发器

两者都在 Tilt UI 中归在 `observability-tests` 标签下，且都是手动的：对着运行中的栈执行，同时盯着看板。

**`inject-thermal`** 把 GPU 0 固定在 90 °C，并断言 Prometheus 恰好上报这个值，而其兄弟 GPU 仍保持各自的读数。在 *GPU temperature* 面板里，会有一条线阶跃上升并持平。

**`inject-xid`** 在 GPU 0 上触发一个不可纠正 ECC 的 Xid，并断言该错误码到达 `DCGM_FI_DEV_XID_ERRORS`。*Last Xid code reported* 显示的是被锁存的 Xid **错误码**，而不是计数。

*Component health* 面板显示 `nvml-mock` 与 `dcgm-exporter` DaemonSet 的就绪数（ready vs desired）以及各 Pod 的重启次数，因此能直接指出是哪个组件出了问题。

## 看起来像 bug、其实不是的现象

**Xid 面板在故障触发前是空的。** 健康集群上 `DCGM_FI_DEV_XID_ERRORS` 没有序列数据：模拟层通过 NVML 事件集传递 Xid，而 `dcgm-exporter` 在该字段没有值时不会输出它。

**注入的 Xid 错误码在各次运行间交替变化**（79、48、79……）。DCGM 会按设备锁存最后一次 Xid，所以重复同一个错误码反而什么都证明不了。

**注入的温度被截断**到该 GPU 配置档的 `shutdown_threshold_c`。默认的 90 °C 是**唯一**对每个 `--gpu-profile` 都有效的区间：它高于模拟器的 73 °C 上限，因此兄弟 GPU 不会"误闯"进注入值、伪造出作用域泄漏；同时又处于（或低于）任一配置档的最低阈值（a100/h100 为 92 °C）。更高的 `HOT_TEMP_C` 会在最开始就被拒绝，而不是在模拟层永远不报那个值的情形上超时。

## 静默失败

有三种情况会让这套栈起来后，GPU 面板全空、或者没有看板、也没有报错。每一种都在其所在位置用注释标了出来：

- **`ServiceMonitor` 的 `release` 标签**（`gpu-operator.values.yaml`）。Prometheus 只发现打了 `release: kube-prometheus-stack` 标签的 ServiceMonitor。
- **看板 sidecar 标签**（`kube-prometheus-stack.values.yaml`、`dashboards/mokka-gpu-dashboard.yaml`）。ConfigMap 必须带 `grafana_dashboard: "1"`。
- **`ServiceMonitor` 的抓取间隔**（`gpu-operator.values.yaml`）。若其值小于抓取超时，Prometheus Operator 会跳过该 ServiceMonitor，且只会在自己的日志里留一条告警。

## 排查

所有 GPU 面板都空了？检查 Prometheus 是否发现了 exporter，这能覆盖上面第一类和第三类失败：

```bash
kubectl get --raw '/api/v1/namespaces/monitoring/services/kube-prometheus-stack-prometheus:9090/proxy/api/v1/targets?state=active' \
  | jq -r '.data.activeTargets[] | select(.labels.job|test("dcgm")) | "\(.scrapePool) \(.health)"'
```

期望每个 GPU worker 各有一条 `up`，位于 `serviceMonitor/...` 抓取池。如果标签无误却一条都没有？去问 Prometheus Operator 为什么跳过了该 ServiceMonitor：

```bash
kubectl -n monitoring logs deploy/kube-prometheus-stack-operator | grep 'skipping object'
```

Grafana 里看不到看板？直接问 Grafana，别只看 ConfigMap：

```bash
kubectl -n monitoring exec deploy/kube-prometheus-stack-grafana -c grafana -- \
  curl -sS -u admin:mokka 'http://localhost:3000/api/search?query=Mokka'
```
