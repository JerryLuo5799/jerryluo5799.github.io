---
layout: post
title:  "AI 辅助开发（七）：专职 Agent 与 Agentic Workflows"
date:   2026-06-19 09:30:00 +0800--
categories: [AI]
tags: [AI辅助开发, Github Copilot, Agentic Workflows]
---

## 前言

到目前为止，所有事情都有一个共同前提：**你在场**。你打开终端，你描述任务，你 review 结果。agent 再能干，也是个需要被召唤的东西。

这个前提有个天花板。有些活它的价值恰恰在于"不需要人记得"——每天扫一遍卡在审批里超过七天的单据、每周检查一次有没有新的安全公告、每次有人提 PR 就先过一遍规则。这些活交给人，人会忘；交给传统的定时脚本，脚本不会判断。

跨过这个天花板，需要两样东西：**给一类任务配一个专职的 agent**，以及**让事件而不是人来触发它**。

## 一、先判断值不值得

自动化最常见的浪费，是给一件半年发生一次的事建了一条流水线。

两个维度就够判断：

![自动化手段的选择象限](/assets/imgs/ai-dev-automation-quadrant.svg)

**左下角**——低频且需要判断的活，老老实实开个会话聊。给它建流水线的维护成本，永远高于你直接做。

**右下角**——低频但确定的活，存成 prompt 文件或 Skill。数据库迁移半年做一次，但每次七步都一样，值得写下来，不值得自动跑。

**左上角**——高频但需要判断的活，配一个专职 agent。代码评审每天都在发生，但每次要看什么、什么算问题，需要真实的判断力。

**右上角**——高频且确定的活，做成 workflow，事件触发，人只看结果。

有个反直觉的点：**"高频"的门槛比你想的低。**一周三次就已经值得做成专职 agent 了——不是为了省那几分钟，是为了消除每次都要重新设定约束带来的方差。

## 二、专职 Agent：配一次，用一年

"自定义 agent"听起来很唬人，实际就是三件事的组合：**一段角色定义 + 一套受限的工具集 + 一份专属的上下文**。

评审这类活最典型。[只读评审模式](/2026/06/12/ai-dev-chat-mode-mcp-skill/) 那个 `review.chatmode.md` 已经是雏形，把它做完整：

`.github/chatmodes/pr-review.chatmode.md`：

```markdown
---
description: ExpenseFlow PR 评审员
tools: ['codebase', 'search', 'usages', 'problems', 'changes', 'findTestFiles']
---

你是 ExpenseFlow 的 PR 评审员。**你没有写文件的工具，不要请求获得。**

## 开始前必读

1. `docs/DESIGN.md` —— 既有架构决策
2. `.agent/journal/` 下最近三份摘要 —— 最近放弃过哪些方案
3. 本次改动：用 `changes` 工具拿 diff，不要凭文件名猜

## 评审顺序

**第一遍，看有没有踩红线**（发现任何一条直接列为阻断项）：

- 金额用了 `double` / `float`
- 阈值字面量出现在 `ApprovalPolicy` 之外
- 用了 `DateTime.Now` 而不是 `DateTimeOffset.UtcNow`
- 状态流转跳步，或从终态回退
- 新增了 `[Fact(Skip = ...)]`
- 修改了测试断言使其适配实现（而不是相反）

**第二遍，看和既有决策冲不冲突**：

- 有没有把 `RequiresSecondApproval` 改成计算属性？（DESIGN.md 明确禁止）
- 有没有实现「非目标」里列的东西？（多币种、OCR、组织架构）
- 有没有引入新的 NuGet 包？PR 描述里说明理由了吗？

**第三遍，看测试**：

- 新增的阈值有没有三个边界用例（低于/等于/高于）？
- 断言是精确比对还是 `Assert.True(x > 0)` 这种弱断言？

## 输出

- **阻断项**：文件、行号、理由、怎么改。
- **建议项**：收益说清楚，作者可以不采纳。
- **问题**：你不确定的地方，用提问的方式写。

不要夸奖。不要复述代码做了什么。不要给出「整体不错」这类结论。
如果三遍下来一个问题都没有，就只回复「未发现阻断项」加上你检查过的清单。
```

这份文件比一段临时 prompt 强在哪？**它把"评审"这件事的隐性知识写死了。**红线清单不是靠评审员当天记得多少，是固定的六条；检查顺序是固定的三遍；输出格式是固定的三类。

最后那句"如果一个问题都没有，就只回复未发现阻断项"是必要的。模型有强烈的倾向要说点什么，不加这句，你会收到大量"建议考虑添加更多注释"这类噪音，然后你会开始不看它的输出——那这个 agent 就废了。

如果你用的是 GitHub 的云端 coding agent，它跑在托管环境里，需要提前把依赖装好，否则它每次都得现装。`.github/workflows/copilot-setup-steps.yml`：

```yaml
name: Copilot Setup Steps

on: workflow_dispatch

jobs:
  # 这个 job 名字是固定的，coding agent 会找它
  copilot-setup-steps:
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'

      - name: 还原依赖并预热构建
        run: |
          dotnet restore
          dotnet build --no-restore -c Debug

      - name: 准备本地数据库
        run: |
          dotnet run --project src/ExpenseFlow.Api --no-build &
          sleep 5
          kill %1 || true
```

最后那段有点脏，但目的很实在：让 `EnsureCreated()` 跑一遍，把 `expenses.db` 建出来。否则 agent 接到任务的第一件事是发现测试连不上数据库，然后花五分钟自己摸索。**给它一个开箱即用的环境，比给它一段"如何初始化数据库"的说明有效得多。**

## 三、事件触发：一个真实的巡检工作流

[用 Markdown 编写 GitHub Agentic Workflows](/2026/02/27/github-agentic-workflows/) 里讲过这套东西的原理和 `gh aw` 的用法。这里直接上一个 ExpenseFlow 会真的用的工作流。

需求：每天早上扫一遍卡住的单据，发现异常就开 Issue。

`.github/workflows/stale-expense-scan.md`：

```markdown
---
on:
  schedule:
    - cron: "0 1 * * 1-5"   # 工作日 UTC 01:00 = 北京时间 09:00
  workflow_dispatch:

permissions:
  contents: read
  issues: write

engine: copilot

tools:
  github:
    allowed: [create_issue, list_issues, update_issue]
  bash: ["dotnet run --project tools/ExpenseFlow.Mcp*"]

timeout_minutes: 10
---

# 超期单据巡检

用 `expenseflow-db` 查询以下两类异常：

1. 状态为 `Submitted` 且 `SubmittedAt` 早于 7 天前的单据，按类别分组计数。
2. `RequiresSecondApproval = 1` 且 `SubmittedAt` 早于 3 天前的单据数量。

## 判断规则

- 两类都为 0：**什么都不要做**，直接结束。不要开 Issue 说「一切正常」。
- 第 1 类超过 10 单，或第 2 类超过 3 单：开一个 Issue。

## 开 Issue 时

先用 `list_issues` 查有没有标题以 `[巡检] 超期单据` 开头且状态为 open 的 Issue。

- 有：用 `update_issue` 追加一条评论更新数字，不要新开。
- 没有：新建，标题 `[巡检] 超期单据 YYYY-MM-DD`，打标签 `ops`。

正文包含：各类别的超期数量表格、环比昨天的变化、以及最久的那单卡了多少天。
**不要给出处理建议**，你不知道业务背景。只报数字。
```

编译并提交：

```bash
gh extension install githubnext/gh-aw
gh aw compile
git add .github/workflows/
git commit -m "加超期单据巡检工作流"
```

三个细节值得说：

**`tools` 是白名单。**这个 workflow 只能开 Issue 和读 Issue，它连 `create_pull_request` 都调不到。权限最小化在自动化场景里比交互场景重要得多——交互时你在旁边看着，自动化时没人看着。

**"两类都为 0 就什么都不做"是必须写死的。**不写这句，你会每天收到一个"今日巡检正常"的 Issue，一周后你就把这个仓库的通知关了。**自动化的第一杀手是噪音**，比出错还致命，因为出错你会去修，噪音你会去无视。

**"不要给出处理建议"也是有意的。**模型很乐意告诉你"建议督促相关审批人尽快处理"，但它不知道那个审批人正在休产假。让它报事实，判断留给人。

## 四、串联：什么时候该，什么时候不该

单个 workflow 跑通之后，很自然会想：能不能让第一个的产出触发第二个？

能，而且很简单——GitHub 的事件系统本来就是这么工作的。

第二个工作流，`.github/workflows/auto-fix-labeled.md`：

```markdown
---
on:
  issues:
    types: [labeled]

if: contains(github.event.label.name, 'agent-fixable')

permissions:
  contents: write
  pull-requests: write
  issues: read

engine: copilot

tools:
  github:
    allowed: [get_issue, create_pull_request]
  bash: ["dotnet build", "dotnet test"]

timeout_minutes: 20
---

# 自动修复被标记的 Issue

读取触发本次运行的 Issue 内容，实现修复。

## 硬性要求

- 严格遵守 `AGENTS.md` 和 `.github/instructions/` 下命中的规则。
- 改动不得超过 5 个文件。**超过就不要做**，在 Issue 下回复
  「改动范围超出自动修复上限，需要人工拆分」然后结束。
- 必须跑 `dotnet build` 和 `dotnet test`，两个都绿才能开 PR。
  测试红就不要开 PR，把失败输出贴到 Issue 里。
- PR 必须标记为 draft，标题以 `[auto]` 开头。
- PR 描述里必须写清楚：改了什么、为什么这么改、你不确定的地方。

## 禁止

- 不允许修改 `ApprovalPolicy` 的阈值数值。
- 不允许修改或删除任何现有测试。
- 不允许改 `.github/` 下的任何文件。
```

链条就成了：巡检发现问题 → 开 Issue → 人看一眼，觉得能自动修就打 `agent-fixable` 标签 → 修复 workflow 接手 → 出一个 draft PR → 人 review。

**注意那个标签动作是人做的。**这不是偷懒，是有意的设计。

自动串联的边界在哪？三条判断：

| 情况 | 该串吗 |
|---|---|
| 上一步的输出是结构化的、下一步只是执行 | ✅ 串 |
| 中间需要一次"这个值不值得做"的判断 | ❌ 留一个人类闸门 |
| 下一步的失败代价高（改代码、发消息给客户、动生产数据） | ❌ 至少让产出是 draft |
| 纯粹为了少点一次鼠标 | ❌ 不值得，链条越长越难排查 |

**最容易出事的是第二类。**巡检 → 自动修复 → 自动合并，听起来很酷，实际上你造了一个没人看得懂的黑箱。某天它开始批量修改错误的东西，你要花一整天才能搞清楚是链条的哪一环出了问题。

链条每多一环，排查成本翻一倍。**留一个人类闸门，通常是留在"要不要做"而不是"做得对不对"上**——前者一秒钟就能判断，后者需要 review 代码。

## 五、让它变成全队的，而不是你一个人的

到这里你的仓库里已经有一堆东西了：

```
ExpenseFlow/
├─ AGENTS.md
├─ .github/
│  ├─ copilot-instructions.md      → symlink 到 AGENTS.md
│  ├─ instructions/                 分作用域规则
│  ├─ prompts/                      /plan、/wrap-up、/add-tests
│  ├─ chatmodes/                    pr-review、tests-only
│  ├─ skills/                       db-migration
│  └─ workflows/                    巡检、自动修复、CI
├─ .vscode/mcp.json                 数据库 MCP
├─ .githooks/pre-commit             提交闸
└─ .agent/journal/                  任务摘要
```

**这些全部在 git 里，这就是"团队化"的全部秘密。**新人 clone 下来，`git config core.hooksPath .githooks` 一句话，剩下的自动生效——他的 Copilot 立刻知道金额要用 `decimal`，`/plan` 立刻可用，PR 评审员立刻可用。

对比一下没做这件事的团队：老王的 prompt 写得特别好，但它在老王的 VS Code 用户设置里。老王请假，团队的 AI 能力就退回原始水平。

跨仓库复用有两条路。

**第一条，组织级的 `.github` 仓库。**在组织下建一个叫 `.github` 的仓库，里面的 workflow 和 instructions 会被组织内其他仓库继承。适合放那些全公司通用的东西——安全扫描、commit 规范、通用的评审红线。

**第二条，把 workflow 做成可复用的。**用 `workflow_call`：

```yaml
# 在 your-org/.github 仓库里：.github/workflows/dotnet-ai-review.yml
name: dotnet-ai-review
on:
  workflow_call:
    inputs:
      chatmode:
        required: false
        type: string
        default: 'pr-review'
```

各个仓库引用它：

```yaml
# ExpenseFlow/.github/workflows/pr.yml
jobs:
  ai-review:
    uses: your-org/.github/.github/workflows/dotnet-ai-review.yml@v1
    with:
      chatmode: 'pr-review'
```

注意那个 `@v1`。**给共享的 workflow 打 tag，不要用 `@main`。**否则你在组织仓库里改一行，二十个项目的 CI 同时变了行为，而且没人知道发生了什么。

一个务实的建议：**先在一个仓库里跑三个月，再考虑抽到组织级。**过早抽象出来的"通用规则"，通常既不适合 A 项目也不适合 B 项目。

## 六、别让它烧钱和失控

自动化跑起来之后，两个新风险：

**成本。**每个 workflow 运行都在消耗额度。一个每 15 分钟跑一次的巡检，一个月就是三千次。三条防线：

```yaml
timeout_minutes: 10        # 单次运行的上限

concurrency:               # 同一个工作流不并发
  group: stale-scan
  cancel-in-progress: true
```

再加上 cron 频率本身——**问一句"这件事真的需要每 15 分钟做一次吗"，通常答案是不需要。**巡检类的东西每天一次足够。

**失控。**agent workflow 有写权限时，最坏情况是它开始批量改东西。除了前面说的 `tools` 白名单和 draft PR，再加一道停止开关：

```yaml
if: |
  vars.AI_WORKFLOWS_ENABLED == 'true'
```

然后在仓库的 Variables 里设一个 `AI_WORKFLOWS_ENABLED`。出事的时候，把它改成 `false`，所有 AI workflow 立刻停摆，不用一个个去禁用，也不用等谁有权限改代码。

**这个开关一定要在你需要它之前就装好。**真出事的时候没人有心情去写 YAML。

## 总结

从单兵到流水线，五件事：

**第一，先判断值不值得。**低频的活别建流水线；高频但需要判断的配专职 agent；高频且确定的才做成 workflow。"高频"的门槛比你想的低，一周三次就够了。

**第二，专职 agent 就是角色 + 受限工具集 + 专属上下文。**把评审的红线清单、检查顺序、输出格式全部写死，并且明确规定"没问题时不要说废话"。

**第三，自动化的第一杀手是噪音。**"什么都没发现就什么都不做"必须写死，否则你会关掉通知，然后这条流水线就死了。

**第四，串联要留人类闸门，闸门放在"要不要做"上。**链条每多一环，排查成本翻一倍。自动产出一律 draft。

**第五，配置进 git 就是团队化。**跨仓库复用打 tag 不用 `@main`；抽到组织级之前先在一个仓库跑三个月。以及——停止开关要提前装好。

到这里，产出的**数量**已经不是问题了：一个人可以同时跑三个 worktree、挂着两条巡检流水线。真正的瓶颈换了个位置——**这些产出怎么进主干？**十个 draft PR 堆在那里没人敢合，和没有它们是一样的。
