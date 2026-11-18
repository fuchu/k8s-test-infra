# 支持策略

本页说明 Mokka 版本的编号方式、哪个版本会获得修复，以及弃用功能会保留多久。

## 版本编号

Mokka 版本号为 `X.Y.Z`：

| 部分 | 何时递增 |
|---|---|
| X（主版本） | 从 1.0 起，破坏 API 的变更：CRD、配置文件、环境变量，或 Mokka 的工作方式。 |
| Y（次版本） | 来自 `main` 的发布，不只含修复，通常是每两周按计划发布的那一版，包含所有已合并内容。 |
| Z（补丁版本） | 只包含安全修复和错误修复的发布。 |

在 1.0 之前，破坏 API 的变更在下一个 Y 版本中发布，并在变更日志条目中以 `**Breaking:**` 标记，说明用户需要采取的行动；从 1.0 起，这类变更会使该版本成为 X 版本。

Mokka 不发布候选版本，只发布稳定的 `vX.Y.Z` 版本。版本发布的镜像和 chart 携带不带 `v` 的版本号：`v0.5.0` 标签发布 `0.5.0`。

## 生命周期

只有最新版本会获得修复：[GitHub releases 页面上的最新版本](https://github.com/NVIDIA/k8s-test-infra/releases/latest)。

修复先落到 `main`，再随下一次计划发布交付。当修复无法等待时，维护者会从 `main` 切一个计划外版本。如果自上次发布以来的所有变更都是修复，则为 Z 版本；否则为下一个 Y 版本；从 1.0 起，若有变更破坏 API，则为下一个 X 版本。

没有发布分支，也没有回溯移植，因此旧版本不会获得修复。要获得修复，请升级到最新版本。

安全修复包括对已公布 CVE（Common Vulnerabilities and Exposures，公共漏洞和暴露）的修复。报告漏洞请遵循[安全策略](https://github.com/NVIDIA/k8s-test-infra/blob/main/SECURITY.md)。

!!! note "自 2026-10-12 起生效"
    本策略于 2026-10-12 生效。`release-X.Y` 分支已退役，v0.4.1 是最后一个从这类分支切出的版本。

## 弃用

弃用会在某个 Y 版本中宣布，记录在其[变更日志](https://github.com/NVIDIA/k8s-test-infra/blob/main/CHANGELOG.md)条目的 `Deprecated` 小节中。弃用的功能最早在三个 Y 版本之后才会移除：在 0.6.0 中宣布的弃用，最早可在 0.9.0 中移除。

## 升级

升级前，请阅读从当前版本到目标版本之间每个版本的[变更日志](https://github.com/NVIDIA/k8s-test-infra/blob/main/CHANGELOG.md)条目。
