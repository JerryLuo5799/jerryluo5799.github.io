---
layout: post
title:  "AI 辅助开发（四）：AI 的四类记忆与任务摘要"
date:   2026-05-29 09:30:00 +0800--
categories: [AI]
tags: [AI辅助开发, Github Copilot, DESIGN.md]
---

## 前言

有个场景大概你也遇到过。

周二你和 agent 折腾了一下午，把 ExpenseFlow 的审批流从"单级审批"改成"按金额分级"。中间试过三种设计，前两种都因为状态机会出现不可达状态而废掉，最后选了第三种。过程很痛苦，但结果不错。

周五你让它加个"审批超时自动升级"的功能。它张口就来：**"建议在 Expense 上加一个 `ApproverLevel` 字段……"**——正是周二第一个被废掉的方案。

它不是忘了，是**它从来就不知道**。周二那场讨论存在于一个已经关掉的会话里，仓库里一个字都没留下。

## 一、四类记忆，只有一类是白送的

把"AI 记不住东西"拆开看，其实是四种不同的东西：

![AI 的四类记忆](/assets/imgs/ai-dev-memory-types.svg)

第一类**工作记忆**是模型自带的，你什么都不用做，但它关掉就没了。

第二类**语义记忆**是 [`AGENTS.md` 那一套](/2026/05/22/ai-dev-agents-md-context-engineering/)——项目里那些稳定的、月度级别才变一次的事实。

第三类**情景记忆**才是真正的痛点：做过什么、为什么这么选、哪条路走不通。它每天都在产生，也每天都在丢失。

第四类**程序记忆**是把重复流程固化下来——「加一个报销类别要走七步」这种做法，存成文件用到时才读。

大部分团队在第二类上花了功夫，然后就停了。但真正让 agent 显得"没长进"的，是第三类的缺失——**它每周都在重新发现同一批坑。**

## 二、工作记忆：会话为什么越聊越笨

在讲怎么留下记忆之前，先说清楚为什么不能靠"一直聊下去"解决。

一个跑了两小时的会话，上下文里堆着几十个文件的内容、十几次失败的命令输出、还有你中途改主意的三次澄清。这时候模型的表现会明显变差，具体症状是：

- 它开始重复已经做过的事（因为早期的动作被挤出了有效注意范围）
- 它回到你已经否决过的方案
- 它对新指令的遵守度下降，反而更像在延续之前的惯性

一个实用的判断：**当你开始需要提醒它"我们刚才不是说了不这么做吗"，就该重开会话了。**

重开的正确姿势不是"从零开始"，而是**先让它把状态写下来，再开新会话读进去**：

```bash
# 在旧会话里
> 把我们这次做的事总结成一份任务摘要，写到 .agent/journal/，
  格式按 .agent/journal/TEMPLATE.md。重点写：最终方案、
  放弃了哪些方案及原因、遗留问题。

# 然后开新会话
copilot -p "读 .agent/journal/2026-05-26-tiered-approval.md，
            在这个基础上继续做审批超时自动升级"
```

这个动作把易失的工作记忆转成了持久的情景记忆。它是整个记忆体系里投入产出比最高的一步，也是最容易被跳过的一步——因为任务做完了，人只想赶紧收工。

所以别指望自觉，把它写进规则。

## 三、情景记忆：任务摘要

先定模板，`.agent/journal/TEMPLATE.md`：

```markdown
# <一句话说清做了什么>

- 日期：YYYY-MM-DD
- 分支 / PR：
- 触发原因：（是什么需求或什么 bug 引出的）

## 最终方案

（两三句话，不要贴代码，说清楚结构上做了什么改动）

## 放弃的方案

| 方案 | 为什么放弃 |
|---|---|
|  |  |

## 踩到的坑

（下次遇到能少走弯路的具体信息：报错、版本问题、隐藏约束）

## 遗留问题

- [ ] 
```

**"放弃的方案"是这份模板里最值钱的一节。**代码本身能告诉你"最后做成了什么"，`git log` 能告诉你"什么时候做的"，只有这一节能告诉你"为什么不是别的样子"。而这恰恰是 agent 最需要、也最猜不到的信息。

填出来长这样，`.agent/journal/2026-05-26-tiered-approval.md`：

```markdown
# 审批流从单级改为按金额分级

- 日期：2026-05-26
- 分支 / PR：feat/tiered-approval #142
- 触发原因：财务反馈 5000 元以上的设备采购需要总监签字，
  但当前只有一级审批

## 最终方案

在 `Expense` 上加 `RequiresSecondApproval`（bool），由 `ApprovalPolicy`
在提交时一次性算出并持久化。审批端点检查这个标志决定需要几次 Approve。
状态机不变，仍然是 Draft → Submitted → Approved / Rejected → Reimbursed。

## 放弃的方案

| 方案 | 为什么放弃 |
|---|---|
| 在 `Expense` 上加 `ApproverLevel`（int），每次审批 +1 | 状态机会出现不可达状态：一级审批完但金额被改小时，`ApproverLevel=1` 却已经不需要二级了。清理逻辑很脏 |
| 加一张 `ApprovalStep` 表记录每一步 | 当前只有两级，一张表纯属过度设计。等真要做多级会签时再说 |
| 每次读取时实时计算是否需要二级审批 | 阈值配置改了以后，历史单据的审批要求会跟着变，财务不接受 |

## 踩到的坑

- `RequiresSecondApproval` 必须在**提交时**算好并存下来，不能实时算。
  这是财务的硬要求：单据一旦提交，审批要求就固定了。
- EF Core 的 `HasPrecision(18, 2)` 要显式配，SQLite 默认会把 decimal
  存成 REAL，边界值测试（正好等于 300）会随机失败。

## 遗留问题

- [ ] 三级审批（超过 20000）还没设计，等财务确认阈值
- [ ] 审批超时自动升级没做
```

看到没——周五那次对话如果它读过这份文件，就不会再提 `ApproverLevel` 了。**"为什么放弃"这一栏，是在给未来的 agent 立路障。**

接下来是让这件事自动发生。在 `AGENTS.md` 里加一节：

```markdown
## 任务收尾

完成一个跨越多个文件的任务后，主动做两件事：

1. 在 `.agent/journal/` 下按 `YYYY-MM-DD-<slug>.md` 写一份任务摘要，
   格式见 `.agent/journal/TEMPLATE.md`。
2. 如果这次改动影响了架构决策或领域规则，同步更新 `docs/DESIGN.md`。

摘要里必须包含「放弃的方案」一节。如果这次没有放弃任何方案，
写「无」，不要省略这一节。
```

最后那句是有意的。省略一节比写"无"容易得多，而一旦允许省略，这一节就会消失。

Copilot 里可以把这个流程做成一个可复用的 prompt 文件，`.github/prompts/wrap-up.prompt.md`：

```markdown
---
mode: agent
description: 任务收尾：写任务摘要并同步设计文档
---

回顾本次会话，完成收尾：

1. 读 `.agent/journal/TEMPLATE.md` 拿到格式。
2. 用 `git diff main...HEAD --stat` 确认这次实际改了哪些文件，
   不要凭印象写。
3. 在 `.agent/journal/` 下新建摘要文件，文件名用今天日期加任务 slug。
4. 「放弃的方案」一节必须填，回顾本次会话里我否决过的思路。
5. 检查 `docs/DESIGN.md` 里是否有内容因为这次改动而过期，有就更新。
6. 最后把摘要文件路径打出来给我。
```

之后在 VS Code 的 Copilot Chat 里输入 `/wrap-up` 就能触发，不用每次重打一遍要求。

第 2 步"不要凭印象写"是必要的。让模型回忆自己改了什么，它会漏，也会把"打算改但最后没改"的东西写进去。让它去查 `git diff`，摘要的准确度会高一大截。

## 四、DESIGN.md：和 AGENTS.md 分工

情景记忆里有一部分不该躺在流水账里——那些还在生效的架构决策。它们需要一个更显眼的位置。

`AGENTS.md` 和 `DESIGN.md` 的分工是这样的：

| | `AGENTS.md` | `docs/DESIGN.md` |
|---|---|---|
| 回答的问题 | 怎么写代码 | 为什么是这个设计 |
| 内容形态 | 祈使句，可执行的规则 | 陈述句，决策与权衡 |
| 谁是主要读者 | agent，每次都读 | 人和 agent，需要时读 |
| 违反的后果 | 代码被 review 打回 | 做出和现有架构冲突的设计 |
| 长度 | 越短越好，五十行封顶 | 可以长，但要有结构 |

一份给 ExpenseFlow 的 `docs/DESIGN.md`：

```markdown
# ExpenseFlow 设计说明

## 领域模型

一张 `Expense` 表承载单据全生命周期。审批过程不单独建表——
当前只有两级审批，`RequiresSecondApproval` 一个布尔字段就够。
**如果将来要做任意级数的会签，这个决定需要推翻**，届时引入
`ApprovalStep` 表，参见 `.agent/journal/2026-05-26-tiered-approval.md`。

## 阈值为什么在代码里而不在数据库里

阈值变更是需要 code review 的业务规则变更，不是配置。
放进数据库意味着任何有写权限的人都能改财务规则，且没有变更记录。
`ApprovalPolicy` 里的字典配合 Git 历史，天然就是审计日志。

代价是改阈值要发版。财务能接受——他们一年改不了两次。

## 金额为什么一律 decimal

`double` 在 0.1 + 0.2 这类运算上有精度问题，报销场景不可接受。
数据库列显式配 `HasPrecision(18, 2)`；SQLite 没有原生 decimal，
不显式配会退化成 REAL，边界值测试会随机失败。

## 审批标志为什么在提交时固化

财务硬要求：单据一旦提交，审批要求就锁定，后续改阈值不影响存量单据。
所以 `RequiresSecondApproval` 是持久化字段，不是计算属性。
**不要把它改成 computed property**，虽然看起来更"干净"。

## 边界与非目标

- 不做多币种。所有金额视为人民币。
- 不做发票 OCR。收据以 URL 形式挂在单据上，识别是另一个系统的事。
- 不做审批人的组织架构建模。审批人是谁由外部 HR 系统决定。
```

最后那节"非目标"经常被忽略，但它对 agent 特别有用。**没有边界说明的时候，agent 会热心地帮你实现你根本不想要的功能。**你让它加个"外币报销"的输入框，它可能顺手给你把整套汇率转换、多币种汇总都做了。写明"不做多币种"，它至少会先问你一句。

那两处加粗的 **"不要改成 computed property"**、**"这个决定需要推翻"** 也是刻意的。设计文档如果只写"我们选了 A"，agent 会觉得改成 B 也行；写清楚"选 A 是因为 X，如果 X 变了就该改成 B"，它才知道什么时候可以动、什么时候不能动。

## 五、程序记忆：把重复流程存起来

第四类记忆是把重复流程固化下来。

比如 ExpenseFlow 里"加一个报销类别"这件事，每次都是同样七步：改 `Thresholds` 字典 → 补 `ApprovalPolicyTests` 的三个边界用例 → 更新 `appsettings.json` 的类别枚举 → 前端下拉框加选项 → 加中文显示名 → 更新 `docs/DESIGN.md` → 跑测试。

你可以每次都打一遍这七步，也可以写一次存起来：

```markdown
---
mode: agent
description: 新增一个报销类别
---

新增报销类别，我会告诉你类别的英文标识、中文名和阈值。

步骤：
1. 在 `ApprovalPolicy.Thresholds` 里加一项。
2. 在 `ApprovalPolicyTests` 里加三个 `[InlineData]`：
   阈值-1、阈值、阈值+1，预期分别是 false / false / true。
3. 更新 `src/ExpenseFlow.Web/src/constants/categories.ts` 的显示名映射。
4. 跑 `dotnet test`，全绿才算完成。
5. 如果这个类别的阈值明显偏离现有量级，在 `docs/DESIGN.md` 里说明原因。

不要改动 `ApprovalPolicy` 的其他逻辑。
```

存成 `.github/prompts/add-category.prompt.md`，之后 `/add-category` 一句话就完事。

prompt 文件、Chat Mode、Skill、自定义 agent 各自的边界在哪，是另一个话题。这里只需要记住：**当你发现自己第三次打同一段要求时，它就该变成文件了。**

## 六、接起来看一遍

四类记忆配齐之后，一次完整的任务是这样跑的：

```bash
# 1. 开工。语义记忆自动加载（AGENTS.md + 命中的 instructions）
cd ../ef-timeout-escalation
copilot

# 2. 先喂情景记忆——上次相关的任务摘要
> 读 .agent/journal/2026-05-26-tiered-approval.md 和 docs/DESIGN.md，
  然后给我一个「审批超时自动升级」的实现方案。先只给方案，别动手。

# 3. 它这次不会再提 ApproverLevel 了，因为摘要里写了为什么放弃

# 4. 确认方案后动手，中间该重开会话就重开

# 5. 收尾
> /wrap-up
```

再加一道 CI 检查，防止收尾这一步被跳过。`.github/workflows/journal-check.yml`：

```yaml
name: journal-check

on:
  pull_request:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: 大改动必须附任务摘要
        run: |
          CHANGED=$(git diff --name-only origin/main...HEAD -- 'src/**' 'tests/**' | wc -l)
          echo "改动文件数：$CHANGED"

          if [ "$CHANGED" -lt 5 ]; then
            echo "小改动，跳过检查"
            exit 0
          fi

          if git diff --name-only origin/main...HEAD | grep -q '^\.agent/journal/'; then
            echo "✅ 找到任务摘要"
            exit 0
          fi

          echo "❌ 本次改动涉及 $CHANGED 个文件，但没有新增任务摘要" >&2
          echo "   在 agent 会话里跑 /wrap-up，或手动写一份到 .agent/journal/" >&2
          exit 1
```

阈值定成 5 个文件是经验值——低于这个数通常是改字符串、调样式一类的活，不值得写摘要。太严格会让人开始糊弄，糊弄出来的摘要比没有更糟，因为它会污染后续 agent 的判断。

等一下，前面说过 `.agent/` 是 gitignore 的。所以 `.gitignore` 要改成这样：

```gitignore
# agent 的临时工作区：不进版本库
.agent/*
!.agent/.gitkeep

# 但任务摘要要进版本库，它是团队资产
!.agent/journal/
```

**草稿是垃圾，摘要是资产**，两者都在 `.agent/` 下但待遇不同。

## 总结

记忆这一环，四件事：

**第一，会话越长越笨，该重开就重开。**判断信号是你开始说"我们刚才不是说了"。重开前先让它把状态写成摘要。

**第二，任务摘要的价值在「放弃的方案」那一节。**代码告诉你做成了什么，只有它能告诉你为什么不是别的样子。强制这一节不许省略。

**第三，`DESIGN.md` 装决策和权衡，`AGENTS.md` 装规则。**决策要写清楚"因为 X 所以选 A，X 变了就该改"，还要写"非目标"，否则 agent 会热心地实现你不想要的东西。

**第四，收尾靠制度不靠自觉。**做成 `/wrap-up` 这样的 prompt 文件，再加一道 CI 检查兜底。

到这里地基层就打完了：agent [有手脚](/2026/05/15/ai-dev-tools-and-isolated-environment/)、[知道规矩](/2026/05/22/ai-dev-agents-md-context-engineering/)、记得住事。接下来的问题变成——**它一次干多少活合适？**给它一个大需求，它会一口气改二十个文件然后告诉你"完成了"，而你根本没法 review。这就轮到节奏和护栏了。
