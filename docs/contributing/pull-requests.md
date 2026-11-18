# Pull Request

## 提交

Mokka 使用 [Conventional Commits](https://www.conventionalcommits.org/)：

```text
type(scope): imperative summary

Why the change is needed, and what it does about it. Wrap at 72 columns.
```

使用的类型有：`feat`、`fix`、`docs`、`refactor`、`test`、`chore`、`ci`。scope 是你改动的组件——`chart`、`mocknvml`、`node-agent`、`nri`、`e2e`、`demo`、`helm`。破坏性变更在末尾追加 `!`。

```text
feat(nri): mock IMEX channel injection over NRI
fix(chart): correct GPU Operator flags in NOTES.txt
docs(e2e): correct the harness lifecycle
chore!: remove vendor/ and resolve dependencies through the proxy
```

写正文时，设想一年后有人在没有 pull request 可看的情况下阅读 `git log`。解释原因，而不是复述 diff。

### 每个提交都要签名

项目使用开发者来源证书（Developer Certificate of Origin）：

```bash
git commit -s
```

## 打开 PR 之前

```bash
make lint-fix
make test
make helm-tests     # 如果改到了 chart
```

完整的关卡列表及各关卡检查什么，参见[测试](testing.md)。

还要确认：

- **新文件带有 SPDX 头。** 与周边文件保持一致。
- **已添加变更日志片段**（如果变更面向用户）。参见[变更日志条目](#changelog-entries)。
- **文档与它所描述的行为一并更新**，而不是留到后续再补。

## 变更日志条目

每条记录都是一个 [changie](https://changie.dev) 片段：`.changes/unreleased/` 下的一个小 YAML 文件，每次变更一个。不要编辑 `CHANGELOG.md`。它只包含已发布版本，由维护者在发版时撰写，因此两个 pull request 不会在它上面冲突。

```bash
make changelog                                    # 逐项提示输入
make changelog KIND=Fixed BODY="..." ISSUE=637    # 非交互方式
```

`KIND` 是 [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) 的一个小节：`Added`、`Changed`、`Deprecated`、`Removed`、`Fixed` 或 `Security`。`ISSUE` 可选，渲染为指向该 issue 或 pull request 的末尾链接。正文按它应出现在 `CHANGELOG.md` 中的样子写，不要带开头的 `- `。条目较长时，打开生成的文件，把 `body:` 块按已发布条目的方式换行，续行相对首行缩进两个空格。查看条目的渲染效果：

```bash
make changelog-preview RELEASE_VERSION=0.0.0
```

重构、测试、CI 和纯文档变更不需要条目。维护者会为它们打上 `skip-changelog` 标签；既没有片段也没有标签的 pull request 会被 CI 判失败。Dependabot 的 pull request 通过 `dependencies` 标签豁免。

## 打开 PR

说明变更做了什么以及为什么。链接它关闭的 issue。如果变更大到需要先做设计，那它应该是一个 [MEP](enhancements.md)，而不是一个巨大的 pull request。

保持小粒度。这个仓库只有一位评审者，三个聚焦的 pull request 比一个改动一切的 PR 落地更快。

!!! note "E2E 在镜像分支上运行"
    单元、lint 和 chart 检查在 pull request 本身上运行。E2E 套件从 copy-pr-bot 创建的 `pull-request/N` 分支运行，因此出现在一次单独的运行中，而不是 PR 事件上。没有立刻看到它是正常的。

## 评审

评审者关注的是 linter 无法检查的同一类问题：变更是否如其说明所述，行为回归时测试是否会失败，以及注释是否解释意图而非复述代码。

还有一位自动评审者会在 pull request 上留言。把它的发现当作建议——它被配置为不涉足 linter 已覆盖的领域，但它并不权威。

## 仅在冲突时 rebase

pull request 不需要与基分支保持最新就能合并。合并 agent 会合并落后于 `main` 的已批准 pull request，因此其他 pull request 先合并并不是更新你 PR 的理由，这种情况下 **Update branch** 按钮也不再出现。

仅当 pull request 带有 `needs-rebase` 标签时才 rebase。当 GitHub 报告与基分支存在合并冲突时，自动化会添加该标签，冲突消失后移除；参见[仓库自动化](../maintainers/repository-automation.md)。

每次推送，包括 rebase，都会驳回现有批准并移除 `lgtm` 证据，因此 pull request 需要重新评审。一次并不需要的更新只会消耗一轮评审，毫无收益。

## 相关链接

| 想了解 | 参见 |
|---|---|
| 运行检查 | [测试](testing.md) |
| 先提出设计 | [增强提案](enhancements.md) |
| 使用维护者命令与合并策略 | [仓库自动化](../maintainers/repository-automation.md) |
| 启动集群 | [本地开发](local-development.md) |
