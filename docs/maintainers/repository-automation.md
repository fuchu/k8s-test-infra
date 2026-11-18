# 仓库自动化

本仓库使用一个打包的 JavaScript action 来处理仓库策略。工作流使用来自默认分支的可信代码和策略，不会以写权限 token 运行来自 pull request 头部的代码。

## 支持的基础功能

该基础提供以下功能：

1. 从 `.github/repo-automation/labels.yml` 进行增量标签同步。
2. Pull request 元数据与评审者协调。
3. 受保护的 `/lgtm`、`/approve`、`/hold`、`/unhold` 和 `/retest` 命令。
4. 评审变更观察。
5. 一个稳定的 `repository-automation/merge-policy` 检查、受保护的 GitHub 原生 SQUASH 自动合并启用，以及不安全自动合并的解除。
6. 为所有未关闭的 pull request 修复冲突标签和元数据标签，包括较旧的 request 和基于其他 feature 分支的 request。
7. 为本 action 之外的合并控制器提供仅标签的策略评估。

该基础不安装 Prow 或 Tide。GitHub 分支保护仍是最终的合并权威。自动化发布其策略检查、为符合条件的 pull request 启用原生 SQUASH 自动合并，并禁用不安全的原生自动合并请求，但不会调用直接合并端点。

## Cherry-pick 与 backport

`/cherry-pick` 和 `/backport` 不是命令。项目只从 `main` 向前修复，不保留发布分支，因此 action 会把它们中的任一个报告为不支持，与其他未知命令一样。

## 指派、评审请求、状态与标题命令

`/assign`、`/unassign`、`/cc`、`/uncc`、`/close`、`/reopen` 和 `/retitle` 属于 Mokka agent，而不属于本 action。对于命令名恰好是其中之一、后跟空格、制表符或行尾的行，无论参数如何，action 都会跳过：不记录命令、不报告诊断、不做任何写入，评估评审正文时如同该行不存在。解析器的行检查先执行，因此这样的行如果超过 4096 个字符，或含有制表符以外的控制字符、格式字符、行或段落分隔符，仍会被报告；和任何诊断一样，这会使同一评审中的 `/lgtm` 不计入。

agent 会在 issue 和 pull request 上读取这些评论，应用 Prow 关于谁可以使用每个命令的规则，并从 `main` 派发 `.github/workflows/issue-commands.yml`。派发时始终带上以下五个字符串输入；命令不使用的输入传空字符串：

| 输入 | 值 |
|---|---|
| `command` | `assign`、`unassign`、`cc`、`uncc`、`close`、`reopen` 或 `retitle` |
| `number` | issue 或 pull request 编号 |
| `users` | 以逗号分隔、不带 `@` 的 GitHub 登录名，最多 10 个。`assign`、`unassign`、`cc` 和 `uncc` 必需；其他命令拒绝 |
| `title` | 新标题，仅限 `retitle`：最多 256 个字符，无控制或格式字符，且不含会关闭 issue 的关键字，如 `fixes #12` |
| `requester` | 评论者的登录名，显示在 run 名称中 |

工作流不决定谁可以使用命令：能派发它的任何人都已有写权限。它会检查输入，然后用 Prow 的 GitHub 客户端所用的调用做出更改：

- `assign` 和 `unassign` 添加或移除指派人。GitHub 会静默跳过它无法指派的用户，因此工作流会比对 GitHub 返回的指派人。
- `cc` 和 `uncc` 请求或移除 pull request 评审。GitHub 会因一个无法请求的评审者而拒绝整个评审请求，因此工作流随后会逐个请求每位评审者。两者在 issue 上都会被拒绝。
- `close` 以完成状态关闭 issue，或关闭 pull request，对已关闭的项会被拒绝。`reopen` 重新打开两者，但已合并的 pull request 除外，对未关闭的项什么也不做。
- `retitle` 设置标题。

`close` 或 `reopen` 成功后，工作流会发布 Prow 生命周期插件所发布的回复，点名请求者，例如 `@alice: Closing this issue.`。其他命令成功时不发布任何内容，与 Prow 一致。

命令可能被上述规则之一拒绝，也可能被 GitHub 以 HTTP 403、404、410、422 拒绝，或因遗漏用户而拒绝。此时，工作流会在 issue 或 pull request 上发布一条评论，点明命令和原因，并让 run 失败。验证失败的输入，只要 `number` 是正整数，也会得到同样的评论：评论点明规则，绝不重复标题，并在代码片段中显示被拒绝的登录名。无效的 `number` 或任何其他错误会让 run 失败但不发布评论。

该 job 仅在 `REPOSITORY_AUTOMATION_ISSUE_COMMANDS_ENABLED` 为 `true` 时运行；在 agent 派发该工作流之前，它什么也不做。每次运行都是各自的并发组，因为 GitHub 每组只保留一个待处理 run，共享组会丢弃被派发的命令。

## 激活顺序

具备写权限的 job 在首次落地时是禁用的。把某个阶段的仓库变量设为精确的小写字符串 `true` 即可启用该阶段。未设置的变量或任何其他值都会使 job 保持禁用。

按以下顺序激活功能：

1. 不带 `apply` 运行**标签同步**。审阅计划。仅在计划正确后，带 `apply` 再运行一次。
2. 设置 `REPOSITORY_AUTOMATION_METADATA_ENABLED=true`。确认 **PR 元数据**只更改受管理的元数据标签和评审者请求。
3. 设置 `REPOSITORY_AUTOMATION_COMMANDS_ENABLED=true`。在一个测试 pull request 上确认命令授权和重复派发行为。
4. 设置 `REPOSITORY_AUTOMATION_REVIEWS_ENABLED=true`。确认**合并评估**只收到预期的完成事件，并保持其 job 禁用。
5. 把 `repository-automation/merge-policy` 添加为必需的分支保护检查。只有到这时才设置 `REPOSITORY_AUTOMATION_MERGE_ENABLED=true`，以发布策略检查、为符合条件的 pull request 启用原生 SQUASH 自动合并、解除不安全的原生自动合并请求。此标志适用于事件、可信派发完成和定时评估器所选的所有未关闭 request，不限于测试 PR。对于严格的仅限 SQUASH 操作，仓库设置必须禁用合并提交和 rebase 合并。评估器会让符合条件的 SQUASH request 保持已启用，并启用那些符合条件的未启用 request。它会解除其观察到的不安全方法，但该方法可能在其最后一次读取之后改变。如果此标志已经启用，安装本 action 也会激活原生启用。

   如果**退役 merge-policy 检查**已运行，打开的头部会带上中性的 `repository-automation/merge-policy` run，而 GitHub 把中性视为必需检查通过。当你从另一个控制器回到此关卡时，要让那个控制器的检查与此检查一起保持必需，直到每个打开头部上最新的 `repository-automation/merge-policy` run 为 `success` 或 `action_required`（评估器从不发布 `neutral`）。以空的 `pr_number` 且关闭 `dry_run` 派发**合并评估**来发布它们，然后才移除另一个控制器的检查。
6. 在 Mokka agent 派发**问题命令**后，设置 `REPOSITORY_AUTOMATION_ISSUE_COMMANDS_ENABLED=true`。在一个测试 issue 或 pull request 上确认一个 `/assign` 和一个 `/retitle`。

验证下一步时，让每一步早先的步骤保持激活。当前一步的验证失败时，不要启用更后面的写入路径。

要把合并交给另一个控制器，请遵循[仅标签的策略评估](#labels-only-policy-evaluation)中的切换步骤。

## 所有未关闭 pull request 的标签

在 `REPOSITORY_AUTOMATION_METADATA_ENABLED=true` 时，**PR 元数据**每小时在第 17 分钟扫描所有未关闭的 pull request。GitHub 可能延迟定时任务。没有创建日期或更新日期过滤。推送到 `main` 也会扫描基于 `main` 的未关闭 request。定时扫描覆盖本仓库中每个有效的基分支，包括堆叠的 pull request 和由机器人创建的 PR。

冲突扫描仅在 GitHub 对当前头部和基分支尖端报告 `CONFLICTING` 时添加 `needs-rebase`。仅当 GitHub 报告 `MERGEABLE` 时移除该标签。未知的合并性、缺失的基分支、身份不一致，或头部或基分支尖端改变，都会推迟该 request 并保留其标签。PR 分支更新（`synchronize`）也会触发此扫描。事件的 `before` 和 `after` SHA 不会取代实时的头部、基分支尖端或合并性检查。

元数据扫描修复 `kind/*`、`size/*`、`area/*` 和 `do-not-merge/work-in-progress`，使用与 PR 元数据相同的分类器。无效的标题会保留现有 kind 标签，但 size、area 和 draft 标签仍可被修复；无效的配置则会保留所有现有标签。即使标签已经匹配，扫描也会刷新可信策略评论。在发出当前头部元数据证据之前，扫描会验证配置、标题、DCO 和 OWNERS；验证失败会移除该证据并记录诊断。扫描不请求评审者，并保留冲突、批准、hold 以及不属于其元数据所有权范围的其他标签。

刷新会保留来自同一可信机器人评论的有效命令状态，包括 hold。重复评论、无效状态或评论被更改都会停止刷新。API 对评论没有原子 compare-and-swap。同一 pull request 的命令和元数据 run 共享一个并发组，因此它们绝不会同时重写其评论。扫描在自己的组中运行；它们与并发命令之间的读后写竞态，仅受最后一次评论读取的限制。

两次扫描都会在第一次变更之前完整读取其候选列表，并拒绝超过 100 个 request 的列表，而不是静默遗漏。每次写入前，它们都会重复实时身份检查；元数据还会检查基分支尖端、派生标签和确切的可信策略修订。job 摘要把每个 request 记录为已应用、未更改、已推迟或失败。每次扫描还会记录同样有界的 JSON，前缀为 `Repository automation conflict-labels: ` 或 `Repository automation metadata-labels: `，可通过 job 日志 API 读取。之后的 API 失败会报告早先的写入，但不会声称它们已被回滚。在宣布一次清扫完成之前，请审阅两份报告和当前未关闭列表，调查每一个被推迟、失败或缺失的 request。

起初正确的元数据标签集仍需要新鲜的证据和输入检查。扫描会验证存储的评论和最终标签，然后重复其实时输入检查。如果下次读取时仍然缺少一次成功的标签更新，扫描会停止并报告部分结果，而不是重复同样的更新。每小时的定时限制了常规 API 使用，且不会假设工作流 token 的实际速率上限。大规模的首次回填可能需要 API 失败后再运行一次。重试前请检查每个 PR 的结果。

批准标签仍是受保护合并评估器的一部分。它们需要其激活关卡以及当前经过验证的人工评审、命令证据，或来自可信 OWNERS 的适用 approver-author 权威。元数据回填不会直接授予批准或启用自动合并。

可信 **PR 元数据**派发的完成会为有界的未关闭 pull request 集合触发受保护的合并评估器。评估器在设置批准标签和 merge-policy 检查之前，会读取当前评审、元数据和源 CI。在合并标志启用时，符合条件的 request 还可能获得原生 SQUASH 自动合并。

评估器只接受带有明确成功验证结果的规范 v2 元数据证据。它拒绝遗留或含糊的证据，并根据签出的策略检查实时标题和 DCO。在每次策略评估前后，确切的可信签出修订必须与实时默认分支匹配。修订变更会阻止合并策略，并在可能时解除同一头部上的原生自动合并。对于已确认的精确头部，若原生合并已完成，则最后一次读取不需要新的策略评估，也不执行进一步的写入。

可信的**评审观察器**完成通常会评估其映射的 pull request。如果 GitHub 返回有效的空 PR 映射，评估器会读取当前未关闭的 PR 列表，限制为 100 个候选。在更改批准标签或 merge-policy 检查之前，它会检查每个候选的当前评审和头部。无效的工作流身份或格式错误的映射不会启动此扫描。

### 原生 SQUASH 自动合并与源 CI

可信评估器 job 启用 GitHub 原生 SQUASH 自动合并。其 token 为此操作拥有 `contents: write` 和 `pull-requests: write` 权限。它只签出可信的默认分支提交，并禁用签出凭据。

评估器从 `.github/repo-automation/policy.yml` 的 `merge.requiredCI` 读取所需的源 CI。每个工作流条目命名一个工作流文件，并可选择性地给出使其成为必需的变更路径 glob；每个检查条目命名一个 check run 以及必须发布它的 GitHub App id。当前列表要求 **Basic checks** 和 **Validate changelog** 的当前头部 run 成功，以及来自 DCO app 的 `DCO` 检查成功。此外，当跟踪的 PR 路径过滤器匹配到变更或重命名路径时，还要求 action CI、Helm、依赖完整性和文档工作流成功。对每个必需工作流，评估器使用最新的 run 编号和当前尝试。run 必须属于本仓库、使用 `pull_request` 事件，并匹配当前头部和预期的工作流路径。非空的 PR 映射必须标识此 PR。空映射可以通过精确的 `@refs/pull/<number>/merge` 后缀标识此 PR。带纯工作流路径的空映射则可以改为匹配实时的源仓库、源分支和头部 SHA；其规范化的 `prNumber` 仍为 `null`。显式映射到另一个 PR 或源 ref 不能使用此回退。

缺失或运行中的证据会阻止成功。失败、取消、跳过或格式错误的必需证据也会阻止成功。不完整或超限的 API 集合会失败关闭。merge-policy 检查和元数据/评审工作流不满足源 CI 关卡。

`files` 模式使用 glob 语法的可移植子集，使 agent 无需 minimatch 实现即可评估同一列表。配置验证会拒绝其他任何内容，包括花括号、方括号、`?`、extglob 组、否定以及段内的 `**`。每个以 `/` 分隔的段要么是 `**`，要么是字母、数字、`.`、`_` 和 `-` 的序列，其中 `*` 匹配该段内的任意字符，包括前导点。`**` 匹配零个或多个完整段，但作为最后一段时匹配一个或多个：`docs/**` 匹配 `docs/a.md` 但不匹配 `docs`，而 `**/OWNERS` 匹配 `OWNERS`。匹配区分大小写，并应用于每个变更路径以及重命名的旧路径。源 CI 只对合并到 `main` 的 pull request 通过；任何其他基分支保持待定。

对于符合条件的 request，评估器首先发布 `action_required` 策略检查，然后再次读取权威、元数据、PR 身份和 CI。如果没有 request 已启用，就用 `expectedHeadOid` 启用原生 SQUASH 自动合并，且不会重试该变更。在发布策略成功之前，评估器再次读取同样的证据，成功之后再读一次。成功之前，失败或撤销的证据会使检查保持阻塞。成功之后，只要能够确认当前 PR 和头部身份，评估器就尝试恢复阻塞检查并解除 request。同一头部的基分支或源身份变更也会触发此修复。读取或写入失败可能阻止该修复，而 GitHub 可能已经完成合并。已有的符合条件的 SQUASH request 会被保留。

这些读取和成功检查是彼此独立的 GitHub 操作，证据可能在它们之间变化。头部守卫不会固定评审、标签、CI 或基分支修订。CI 证据绑定到源头部，在 GitHub 省略 PR 映射时可在 PR 或基分支重定向之间复用，但它不证明当前的基分支尖端或被重定向的基分支分支经过测试。在分支保护 `strict=false` 时，GitHub 可以在基分支不是最新的情况下合并。必需的原生评审和检查仍是最终的合并控制。本 action 中的源 CI 关卡不会使单独必需的原生 CI 检查变得多余。

### 仅标签的策略评估

**合并评估**工作流有第二个 job `policy-labels`，由 `REPOSITORY_AUTOMATION_POLICY_LABELS_ENABLED` 控制。它在与合并 job 相同的事件上运行，从可信默认分支读取相同的评审、命令和 approver-author 权威。该 job 只写入 `lgtm`、`approved`、`do-not-merge/hold` 和 `do-not-merge/needs-approval` 标签；不读取源 CI、分支保护或合并状态，从不发布 merge-policy 检查，也从不启用或禁用原生自动合并。其 token 拥有 `actions: read`、`contents: read`、`issues: write` 和 `pull-requests: write`，并有自己的并发组。

在写入之前，job 会再次读取策略评论、标签和 pull request 头部。如果其中任何内容自其评估读取以来发生变化，它会跳过写入并报告 `inputs-changed`；做出更改的那个 run 会触发下一次评估。如果评估失败，job 什么也不写，run 失败。与合并 job 不同，它不应用失败关闭的标签计划，因为该计划会移除一个正在生效的 `do-not-merge/hold`。

该 job 从不移除 `do-not-merge/hold`。只有 `/unhold` 能清除 hold，应用它的**命令** run 会移除该标签。没有 hold 状态的 hold 标签会保留，例如维护者手工添加的，或某个 `/hold` run 在写入其状态之前写下的。

当另一个控制器基于这些标签合并时，请使用此 job。请遵循以下顺序，因为只要合并 job 启用，它就会在符合条件的 pull request 上重新启用原生自动合并：

1. 设置 `REPOSITORY_AUTOMATION_POLICY_LABELS_ENABLED=true`，确认一次**合并评估** run 从其 `policy-labels` job 应用标签。
2. 设置 `REPOSITORY_AUTOMATION_MERGE_ENABLED=false`。等到没有任何**合并评估** `evaluate` job 处于排队或运行中。
3. 在每一个启用了原生自动合并的未关闭 pull request 上禁用它。以下命令会列出它们：

   ```shell
   gh pr list --repo NVIDIA/k8s-test-infra --state open --limit 200 \
     --json number,baseRefName,autoMergeRequest \
     --jq '.[] | select(.autoMergeRequest != null) | "\(.number) \(.baseRefName)"'
   ```

   对列出的每个 pull request 运行 `gh pr merge <number> --repo NVIDIA/k8s-test-infra --disable-auto`。重复该列表命令直到它不打印任何内容。
4. 在 `main` 的分支保护中，把必需的 `repository-automation/merge-policy` 检查替换为新控制器的关卡。

当两个变量都为 `true` 时，两个 job 会从各自的并发组写入相同的标签。请让这种重叠保持短暂。

### approver 作者的自动批准

在可信基分支 OWNERS 中经过验证的人工 approver，作为 PR 作者时，会在该 approver 权限范围内隐式批准其更改的文件。当这些文件以及任何独立批准覆盖了所有变更文件时，评估器添加 `approved`。OWNERS 别名和生效的嵌套 OWNERS 规则适用；`no_parent_owners` 可以排除根 approver。PR 提出的 OWNERS 变更不能授予此权威。元数据也接受这些作者拥有的路径，而不向作者请求评审。

当前头部仍必须由一位独立的授权人类提供 `/lgtm`。作者不能给自己的 PR 发 LGTM。评估器在成功前再次读取作者身份和可信 OWNERS；被移除的权威会撤销隐式批准。新头部会使旧的 LGTM 证据失效，并要求重新检查覆盖。hold、必需的检查和 GitHub 的原生评审要求仍然适用。`approved` 标签不满足必需的 GitHub 批准评审，也不会启用自动合并。

对话命令使用与合并评估相同的当前原生批准和评审 LGTM 证据。批准覆盖可以跨不同的 OWNERS 范围组合原生评审、`/approve` 命令和 approver-author 权威。`/hold` 命令会保留有效的批准和 LGTM 标签。命令在任何写入前再次读取可信 OWNERS 和已验证的评审证据；证据变化会停止 run。原生评审仍是实时证据，不会被复制进命令状态。

### 排队命令

GitHub 在一个并发组中最多保留一个待处理 run，因此同一 pull request 上较新的**命令**或 **PR 元数据** run 会取消待处理的**命令** run。为此，每个**命令** run 会列出该 pull request 的评论：先按评论 ID 顺序，应用来自早于其自身评论、尚未处理的评论中的 `/hold`、`/unhold` 和 `/retest` 命令，再应用其自身评论中的命令。由不含命令的评论启动的 run 也照此处理。每条评论必须来自人类账户且未被编辑，其作者的实时身份和仓库访问权限也要像对启动 run 的评论那样检查。一条补追的评论只有在改变了某些内容，或其 `/lgtm` 或 `/approve` 本可由其作者给出却被拒绝时，才记录为已处理。因此，其他评论者未改变任何内容的评论无法填充命令历史。已处理的评论 ID 存储在策略评论中，因此重复派发不会改变任何内容。

只有最近 24 小时内创建的评论（按 GitHub 的评论创建时间）才会被补追，因此首次 run 不会重放旧历史。比最新已处理命令更旧的评论不会在以后应用，否则会把它们重新排序到较新命令之后。比 run 自身评论更新的评论，则留给它们自己的 run。来自可给出该证据的评审者或 approver 的补追 `/lgtm` 或 `/approve`，会记录为已处理但不授予证据，因为它可能是在推送头部之前写的；评审者必须再次评论。来自其他任何人（包括 PR 作者）的补追 `/lgtm` 或 `/approve` 会被忽略且不显示。策略评论中的命令摘要会点出每个这样的评论 ID，并要求在当前头部重新发出，最多 20 条外加其余计数。它还会显示最后一条已处理评论的行结果，job 摘要则列出每个已处理的评论 ID。其 run 被取消的评论会保持未应用，直到该 pull request 的下一次**命令** run；如果那次 run 在 24 小时之后才启动，该评论会被丢弃。

### 派发的标签扫描报告

**PR 元数据**工作流还接受在 `main` 上的 `workflow_dispatch` 请求，恰好带两个字符串输入：`request_id`（规范的小写 UUID）和 `workflow_commit_sha`（完整的小写 main 提交 SHA）。元数据标志必须启用。所选工作流提交、run 提交和输入提交必须在 `NVIDIA/k8s-test-infra` 中匹配。工作流在 `control` 中签出那个确切的可信提交；它不接受签出路径或其他调用者输入。

派发会把 `conflict-labels` 和 `metadata-labels` 作为两个独立的 action 调用运行。即使冲突扫描失败，元数据扫描也会运行。两次调用之后，工作流上传 `mokka-label-scan-<request_id>`，其固定成员为 `conflict-labels.json` 和 `metadata-labels.json`，然后如果任一扫描失败就让 job 失败。在获知完整候选列表之前发生的失败，无法产生声称列表为空的报告。

每份版本 1 报告包含请求和仓库身份、工作流提交、模式、dry-run 状态、完整排序的候选编号，以及每个候选的一个结果。结果使用 `applied`、`unchanged`、`deferred` 或 `failed`，并带有固定原因以及输入围栏和受管理标签的 SHA-256 哈希。未处理的 request 是 `failed`，原因为 `not_processed`。报告不含原始标题、节点 ID、分支名或标签名。每份报告限制为 70 KiB 和 100 个候选。

派发成功需要一次新鲜的标签读取，随后是最终的 PR 围栏，包括那些起初标签正确的 request。在下次读取时仍缺失的已确认更新会让扫描失败。未知的合并性、变化的状态或无效策略会使覆盖无法确定，且没有输出标签哈希，因此完整报告可以与之后的实时观察核对。原生 PR、push 和定时 run 保持其现有的摘要和行为，但不创建这些报告文件。在轮询驱动的派发路径通过其实时激活检查之前，保留每小时的恢复定时。

## pull request 评审中的 LGTM

合并评估器接受处于 `APPROVED` 或 `COMMENTED` 评审当前正文中的显式 `/lgtm` 行。评审者必须是经过验证的人类、是变更文件的当前 OWNERS 评审者或 approver，并且不是 PR 作者。评审必须引用当前的 pull request 头部。人类评审可以对机器人创作的 pull request 授予证据。

评审 LGTM 与原生批准覆盖是分开的。没有 `/lgtm` 的批准评审不授予 LGTM。带 `/lgtm` 的 `COMMENTED` 评审不授予批准。评审正文中的其他命令不会被执行。被引用或围栏内的命令不计入，无效的命令语法不授予评审 LGTM。`/lgtm cancel` 不是受支持的命令。

来自每个 actor 的最新提交的评审会取代该 actor 较旧的评审 LGTM。提交时间决定顺序，较高的评审 ID 用于打破平局。待处理的评审不会取代已提交的证据。评估器在标签更改后、成功检查前以及该检查之后，再次读取当前评审正文、状态、作者、提交和提交时间。移除或替换 `/lgtm`、驳回评审或更改 pull request 头部，都会移除该评审证据。评审命令不会作为历史权威存储在策略评论中。存储的问题评论 LGTM 仍然独立，必须通过其现有的实时检查。

机器人创作的 pull request 可以获得元数据标签和人类评审者请求。机器人评审会被验证，然后从人类批准证据中排除。机器人不能提供 OWNERS、评审者、approver 或 LGTM 权威。

当前元数据证据不需要更早的对话命令。可信的元数据评论可以没有命令状态记录。未知、格式错误、重复或上下文错误的命令状态记录会保持阻塞。

## 安全与运维

- 让顶层工作流权限保持为空。按 job 授予权限。
- 让每个外部 action 固定到完整的提交 SHA。
- 让自动化签出与目标签出分开。
- 通过 GitHub 解析默认分支并签出其确切的提交 SHA。
- 不要在自动化签出中启用凭据、子模块或 LFS。
- 把评论和事件 payload 文本当作提示信息。action 在写入前会重新获取实时 GitHub 状态。
- 审阅每次调用的 action job 摘要。
- 保留定时评估器，因为它会修复遗漏或延迟的事件。

要停止仓库写入，请把适用的激活变量设为 `false`。在合并评估被禁用后，维护者必须在每一个未关闭的 pull request 上禁用原生自动合并，并确认没有任何一个仍处于启用状态，因为评估器在运行时会再次启用它。在维护者选择并记录替代关卡之前，不要移除必需的合并检查。
