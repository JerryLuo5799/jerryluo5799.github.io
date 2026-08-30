---
layout: post
title:  "AI 辅助开发（五）：审计划、小步走与硬护栏"
date:   2026-06-05 09:30:00 +0800--
categories: [AI]
tags: [AI辅助开发, Github Copilot]
---

## 前言

地基打好之后，agent 已经能自己跑很长一段路了。新问题随之而来：它跑得太远。

你说"帮我实现发票附件功能"，二十分钟后它交回来一个改了 23 个文件、加了 1180 行的 diff，附一句"已完成，测试通过"。

你打开 diff，看了三分钟，关掉。因为你根本 review 不动。

于是你有两个选择：合了它，赌它是对的；或者全撤，二十分钟白干。**这两个选择都很糟，而糟的根源不在 agent，在任务的给法。**

## 一、大任务为什么必然失败

三个原因，而且是叠乘关系。

**错误会累积。**假设 agent 每一步做对的概率是 95%——已经相当高了。连续 20 步全对的概率是 0.95 的 20 次方，约等于 36%。也就是说，一个需要 20 步的任务，有接近三分之二的可能在某处埋了个错。

麻烦的是它不知道自己错了，会继续在错误的基础上往下建。第 8 步选错了存储抽象，第 9 到 20 步全部是围绕这个错误抽象写的。

**Review 成本是超线性的。**50 行的 diff 你能逐行看；500 行你会开始跳着看；1180 行你只会看文件名列表然后说"看起来还行"。而 review 是这套流程里唯一的人类把关点。

**回滚粒度太粗。**23 个文件里可能有 18 个是对的，但错的那 5 个和对的那 18 个混在一次提交里。你没法只撤销错的部分。

![两种任务切法的对比](/assets/imgs/ai-dev-task-slicing.svg)

切开之后，第 4 步用错了库，前三步已经在 main 里了，损失只有一次任务的时间。

## 二、先出计划，再动手

在 agent 写第一行代码之前拦一道，是投入产出比最高的一个习惯。成本是三十秒，收益是拦掉大部分方向性错误。

做成一个 prompt 文件，`.github/prompts/plan.prompt.md`：

```markdown
---
mode: agent
description: 只出方案，不改代码
---

针对我描述的需求，产出一份实现方案。**这一轮不要修改任何文件。**

方案必须包含：

1. **改动清单**：要新增/修改哪些文件，每个文件一句话说明改什么。
2. **拆分建议**：如果这个需求超过 5 个文件或涉及多个关注点，
   拆成若干个可独立合并的步骤，标出顺序和依赖。
3. **验收标准**：每一步怎么算做完？具体到能跑的命令或能观察的行为。
4. **风险点**：哪些地方你不确定、需要我拍板？现有代码里有没有
   会被你的改动影响到但我可能没想到的地方？
5. **不做什么**：为了控制范围，这次明确不碰的东西。

先读 `docs/DESIGN.md` 和 `.agent/journal/` 下最近三份摘要，
确认方案不与既有决策冲突。如果冲突，直接指出来。
```

用起来：

```bash
copilot
> /plan 报销单要支持上传发票附件，审批时能看到
```

**审计划要审什么？**给一份清单，五条：

| 审查项 | 具体在看什么 |
|---|---|
| 文件数 | 超过 8 个就该拆。这个数字比任何"复杂度"描述都可靠 |
| 有没有新依赖 | 它有没有为了省事引入一个新 NuGet 包？值不值得？ |
| 验收标准可执行吗 | "确保功能正常"不算，"`dotnet test` 全绿且新增 3 个用例"才算 |
| 风险点是不是空的 | 空的通常意味着它没认真想，让它重出 |
| 有没有碰不该碰的 | 需求只说附件，方案里为什么有 `ApprovalPolicy` 的改动？ |

最后一条最值钱。**agent 顺手改掉它认为"不合理"的东西，是最常见的一类事故。**你让它加附件，它觉得阈值判定用严格大于"有 bug"，顺手改成了大于等于——测试它也一起改了，所以是绿的。

计划阶段就能看见这种越界。

## 三、切多小才合适

一把粗糙但好用的尺子：**一个任务的验收标准，你能不能用一句话说清楚？**

- ✅ "POST 一个超过 10MB 的文件，返回 413" —— 一句话
- ✅ "`IReceiptStore` 有本地文件实现，两个单元测试覆盖存取" —— 一句话
- ❌ "发票附件功能可用" —— 这不是验收标准，是需求标题

另外两个信号：

**改动集中在一个关注点。**存储抽象是一个关注点，HTTP 端点是另一个，前端展示是第三个。一次任务跨越三个关注点，review 时你的脑子要切三次上下文。

**能独立合并。**第 3 步（大小校验）不依赖第 4 步（缩略图）就能合进 main 并发布。如果两步必须一起合，那它们其实是一步。

具体到前面那个需求，五步的派活是这样的：

```bash
# 第 1 步
copilot -p "在 src/ExpenseFlow.Api/Storage/ 下定义 IReceiptStore 接口
（Save/Get/Delete）和 LocalFileReceiptStore 实现，文件落在
.data/receipts/ 下。加两个单元测试：存进去能读出来、读不存在的返回 null。
不要改动任何现有文件，Program.cs 的注册留到下一步。"

# 确认、review、合并，然后第 2 步
copilot -p "把 IReceiptStore 注册到 DI，加端点
POST /expenses/{id}/receipt 接收 multipart 文件，存储后把返回的
key 写到 Expense.ReceiptKey。加集成测试覆盖上传成功和单据不存在两种情况。
暂时不做大小和类型校验。"
```

注意每一条最后那句 **"不要改动 X"、"暂时不做 Y"**。这是在给任务划边界。没有这句话，agent 会很热心地把后面几步一起做了——它觉得这样是帮你，实际上是把你切好的任务又粘回去了。

## 四、护栏：让环境替你说不

规则文件里写"金额一律用 decimal"是一种护栏，但它是**软的**——模型可能读到了、也可能读到了但在第 15 步的时候忘了。

硬护栏是那种违反了就会变红的东西。最好用的一类，是把约定写成测试。

`tests/ExpenseFlow.Tests/ConventionTests.cs`：

```csharp
using System.Text.RegularExpressions;

public class ConventionTests
{
    private static readonly string SrcRoot = FindSrcRoot();

    private static IEnumerable<(string Path, string Text)> SourceFiles() =>
        Directory.EnumerateFiles(SrcRoot, "*.cs", SearchOption.AllDirectories)
                 .Where(p => !p.Contains($"{Path.DirectorySeparatorChar}obj{Path.DirectorySeparatorChar}")
                          && !p.Contains($"{Path.DirectorySeparatorChar}bin{Path.DirectorySeparatorChar}"))
                 .Select(p => (p, File.ReadAllText(p)));

    [Fact]
    public void 金额不允许使用_double_或_float()
    {
        var offenders = SourceFiles()
            .Where(f => Regex.IsMatch(f.Text, @"\b(double|float)\s+\w*(Amount|Price|Total|Threshold)"))
            .Select(f => Path.GetFileName(f.Path))
            .ToList();

        Assert.True(offenders.Count == 0,
            $"金额字段必须用 decimal，违规文件：{string.Join(", ", offenders)}");
    }

    [Fact]
    public void 不允许使用_DateTime_Now()
    {
        var offenders = SourceFiles()
            .Where(f => f.Text.Contains("DateTime.Now") || f.Text.Contains("DateTime.UtcNow"))
            .Select(f => Path.GetFileName(f.Path))
            .ToList();

        Assert.True(offenders.Count == 0,
            $"时间一律用 DateTimeOffset.UtcNow，违规文件：{string.Join(", ", offenders)}");
    }

    [Fact]
    public void 阈值字面量只允许出现在_ApprovalPolicy_里()
    {
        var offenders = SourceFiles()
            .Where(f => !f.Path.EndsWith("ApprovalPolicy.cs"))
            .Where(f => Regex.IsMatch(f.Text, @"\b\d{3,}m\b"))   // 300m、1500m 这类
            .Select(f => Path.GetFileName(f.Path))
            .ToList();

        Assert.True(offenders.Count == 0,
            $"金额字面量只能写在 ApprovalPolicy 里，违规文件：{string.Join(", ", offenders)}");
    }

    private static string FindSrcRoot()
    {
        var dir = new DirectoryInfo(AppContext.BaseDirectory);
        while (dir is not null && !Directory.Exists(Path.Combine(dir.FullName, "src")))
            dir = dir.Parent;
        return Path.Combine(dir!.FullName, "src");
    }
}
```

这几个测试很粗糙——正则会误伤，`double` 出现在注释里也会中招。但**粗糙的硬护栏胜过精致的软约定**。agent 违反了，`dotnet test` 立刻红，它自己就会去改，根本不用你介入。

再加一道 commit 层面的闸，`.githooks/pre-commit`：

```bash
#!/usr/bin/env bash
set -euo pipefail

# 防止一次提交过大：agent 一口气改太多时，先停下来问人
CHANGED=$(git diff --cached --name-only | grep -cE '^(src|tests)/' || true)
LIMIT=${MAX_STAGED_FILES:-12}

if [ "$CHANGED" -gt "$LIMIT" ]; then
  echo "⛔ 本次提交涉及 $CHANGED 个源文件，超过上限 $LIMIT。" >&2
  echo "   把它拆成多次提交；确认无误要放行，用："                >&2
  echo "   MAX_STAGED_FILES=99 git commit ..."                  >&2
  exit 1
fi

# 不允许把测试标记为 Skip
if git diff --cached -U0 -- 'tests/**' | grep -qE '^\+.*Skip\s*='; then
  echo "⛔ 检测到新增的 [Fact(Skip=...)]。测试不该跑就删掉，不要装绿。" >&2
  exit 1
fi
```

挂上去：

```bash
chmod +x .githooks/pre-commit
git config core.hooksPath .githooks
```

`core.hooksPath` 让这个 hook 跟着仓库走，团队里每个人（和每个容器里的 agent）都会生效，不用各自手动装。

## 五、Back pressure：跑偏了怎么拉回来

护栏是防患于未然，back pressure 是它已经跑偏了、你要把它拽回来。

**最没用的做法是"再试一次"。**你说"不对，重来"，它会换一个写法再撞一次同样的墙，因为你没告诉它墙在哪。

有效的手法有三种，按力度递增：

**第一种，给具体的反例。**不说"这样不对"，说清楚哪里不对、正确的长什么样：

```
> 不对。你把 RequiresSecondApproval 改成了计算属性。
  docs/DESIGN.md 里写了它必须是持久化字段，因为财务要求单据提交后
  审批要求就锁定。改回持久化字段，并且在提交时算好。
```

**第二种，收缩它的活动范围。**跑偏往往是因为它有太多自由度：

```
> 停。这一轮只允许修改 src/ExpenseFlow.Api/Storage/ 下的文件。
  Program.cs、ApprovalPolicy.cs、任何测试文件都不要动。
  如果你认为必须改这些文件才能完成，先告诉我原因，别自己动手。
```

**第三种，让环境把它顶回去。**这是最省事的——你什么都不用说，让红色的测试和失败的 hook 去教育它：

```
> 你说改完了。跑一下 dotnet test，把完整输出贴给我。
```

它跑完会发现 `ConventionTests` 红了，然后自己回头改。**你从"争论者"变成了"裁判"**，这个角色转换是关键——争论要消耗你的注意力，裁判只需要看红绿。

顺带说一个反模式：**不要在同一个会话里连续纠正超过三次。**三次还没回到正轨，说明上下文里已经堆了太多矛盾信息（它既记得原方案，又记得你的三次否定），继续纠正的边际收益是负的。正确做法是让它写份摘要，关掉，带着摘要重开。

## 六、给新手的三十分钟

如果你要带一个完全没用过 agent 的同事上手，别从"配置 MCP"开始。按这个顺序：

**第 0 到 5 分钟：装环境，跑通一个只读任务。**

```bash
cd ExpenseFlow
copilot
> 这个项目是干什么的？审批阈值定义在哪里？
```

只让它读、让它答。目的是建立"它真的看得见我的代码"这个直觉。

**第 5 到 15 分钟：一个能一句话验收的小改动。**

```bash
> 给 ApprovalPolicy 加一个 Training 类别，阈值 2000。
  按 AGENTS.md 的要求补测试，改完跑 dotnet test。
```

看着它改，看着它跑测试。目的是建立"它能闭环"的直觉。

**第 15 到 30 分钟：故意让它撞一次护栏。**

```bash
> 把 Expense.Amount 改成 double 类型。
```

`ConventionTests` 会红，它会告诉你改不了。**这一步是整个上手过程里最重要的**——新手最大的恐惧是"它会不会把我代码搞坏"，亲眼看见护栏挡住它，这个恐惧就没了。

三十分钟之后再讲 `/plan`、讲任务切分、讲 back pressure。**先建立信任，再教方法。**顺序反了，人会因为不敢用而根本不给方法练手的机会。

## 总结

节奏和护栏这一环，五件事：

**第一，动手前先出计划。**成本三十秒，能拦掉大部分方向性错误。审计划重点看文件数、新依赖、验收标准是否可执行、以及有没有碰不该碰的东西。

**第二，一个任务只做一个关注点，验收标准一句话说得清。**派活时明确写"不要改 X"、"暂时不做 Y"，否则它会把你切好的任务粘回去。

**第三，把约定写成测试。**粗糙的硬护栏胜过精致的软约定。违反了 `dotnet test` 直接红，agent 自己就会改。

**第四，跑偏时给具体反例、收缩范围、或者让测试顶回去。**别说"重来"。连续纠正超过三次就该重开会话。

**第五，带新人先建立信任再教方法。**让他亲眼看见护栏挡住一次危险操作。

节奏对了之后，下一个瓶颈是能力边界：agent 需要查数据库的真实表结构、需要调内部 API、需要跑浏览器确认页面对不对——这些都超出了"读写文件加跑命令"的范围。补这些能力的手段有四种，名字都很像，但解决的问题完全不同。
