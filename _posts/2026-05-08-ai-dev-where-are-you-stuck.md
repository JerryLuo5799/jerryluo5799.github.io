---
layout: post
title:  "AI 辅助开发（一）：先搞清楚你卡在哪一环"
date:   2026-05-08 09:30:00 +0800--
categories: [AI]
tags: [AI辅助开发, Github Copilot]
---

## 前言

同一个团队，同一份 GitHub Copilot 订阅，两个人的产出可以差出一个数量级。

一个人用它写出了完整的功能分支，PR 里三十几个文件，同事 review 完只提了两条意见就合了。另一个人也在用，但用法停留在"补全一行 LINQ"，稍微复杂点的需求让它写，出来的东西命名不对、依赖注入的写法和项目里其他地方完全两个风格，改起来比自己写还费劲。

模型是同一个。差别不在模型。

差别在于，第一个人围绕模型搭了一整套东西——项目规则文件、任务的切分方式、跑之前的计划审查、跑完之后的自动验证——而第二个人只有一个聊天框。

[什么是 Harness Engineering](/2026/04/12/what-is-harness-engineering/) 里聊过这套"模型之外的基础设施"由哪几块组成，[什么是 Agentic Engineering](/2026/04/26/what-is-agentic-engineering/) 聊了站在这套基础设施之上工作方式怎么变。这篇要做的事情更土：给一张地图，加一份体检表，让你能定位自己现在到底卡在哪。

## 一、把「AI 辅助开发」拆成六个环节

![AI 辅助开发的六个环节](/assets/imgs/ai-dev-map.svg)

这六个环节是有依赖顺序的，从下往上：

**地基层**决定 AI 看得见什么、动得了什么。环境和工具决定它有没有手脚——只能在聊天框里吐字，和能在容器里跑 `dotnet test` 然后根据失败结果自己改，是两种生物。上下文与记忆决定它知不知道你们项目的规矩。

**执行层**决定一次任务跑得顺不顺。节奏和护栏决定它跑偏时你能不能及时拉回来；能力扩展决定它遇到超出自身能力的事（查数据库、跑浏览器、调内部 API）时有没有工具可用。

**收口层**决定产出能不能进主干。验证闸门是"不信任但验证"的落地；团队规模化决定这些经验是留在一个人的机器上，还是变成全队的默认配置。

## 二、六种典型症状，对应六个环节

体检的思路是反着来的——你不会主动感知"我上下文工程做得不好"，你只会感知到疼。下面这张表把疼和病因对上：

| 你的感受 | 病灶 | 环节 |
|---|---|---|
| 每开一个新会话，都要把项目背景重讲一遍 | 项目规则没有落成文件 | ② 上下文与记忆 |
| 生成的代码能跑，但一看就不是本项目的风格 | AI 不知道你们的约定 | ② 上下文与记忆 |
| 它改代码得先问我一次，我点同意点到手酸 | 工具形态选错，或权限配置太保守 | ① 环境与工具 |
| 让它做个稍大的需求，一口气改了 20 个文件，出错只能全撤 | 任务没切分，没有中间检查点 | ③ 协作节奏与护栏 |
| 它信誓旦旦说"已修复"，实际上根本没跑过测试 | 它没有验证手段，也没有反馈回路 | ③ + ⑤ |
| 它需要查一下生产库的表结构，但只能靠我复制粘贴 | 缺少 MCP 一类的能力扩展 | ④ 能力扩展 |
| AI 产出的 PR 堆在那里没人敢合 | 没有自动化的第一道 review | ⑤ 验证闸门 |
| 组里就一个人玩得转，他的配置在他自己电脑上 | 配置没有进仓库、没有共享 | ⑥ 团队规模化 |
| 月底账单出来吓一跳，但说不清钱花在哪 | 用量不可见 | ① 环境与工具 |

## 三、一份可以现在就打分的自测表

每题答"是"记 1 分。六个环节各 3 题，满分 18。

**① 环境与工具**

- 你能在终端里让 AI 直接跑命令、读输出、再改代码，而不是只能在聊天框里对话吗？
- 你有办法让 AI 在一个隔离环境（容器或独立工作区）里干活，跑坏了不影响你手上的分支吗？
- 你知道自己团队这个月在 AI 上花了多少钱、花在谁身上吗？

**② 上下文与记忆**

- 你的仓库根目录有一份写给 AI 看的项目说明文件吗？
- 这份说明是分作用域的吗（前端目录一套规则、后端目录另一套），还是一个几百行的大文件？
- 上一次 AI 帮你做完一个复杂重构，那次的决策过程有留下痕迹吗？

**③ 协作节奏与护栏**

- 让 AI 做需求时，你会要求它先给方案、你看过再动手吗？
- 你的任务切分粒度，是"一个 PR 能说清楚"级别的，还是"实现整个模块"级别的？
- 当它跑偏时，你有比"关掉重开"更好的纠正手段吗？

**④ 能力扩展**

- 你给 AI 接过至少一个 MCP 服务器（数据库、内部 API、文档系统）吗？
- 你把常用的重复流程（比如"新建一个 Controller 并配好测试"）固化成可复用的东西了吗？
- 你说得清 Chat Mode、MCP、Skill、Agent 这四个词分别解决什么问题吗？

**⑤ 验证闸门**

- 你的仓库有自动化的第一道 code review（AI 先过一遍，人再看）吗？
- AI 改完 UI 之后，有办法自己打开浏览器确认效果吗？
- 你的 commit 记录能区分哪些是 AI 参与的吗？

**⑥ 团队规模化**

- 你的 AI 配置文件（规则、工作流、工具定义）是进 git 的吗？
- 新人入职拉下仓库，AI 就自动按你们的规矩干活吗？
- 团队里有人改进了 prompt 或工作流，其他人能自动受益吗？

**打分对照：**

| 分数 | 你的位置 | 下一步该补哪里 |
|---|---|---|
| 0–5 | 还停在"高级自动补全" | 先做 ①②，这两块投入产出比最高 |
| 6–10 | 单人用得不错，但很脆 | 补 ③⑤，让产出稳定可交付 |
| 11–14 | 个人层面成熟 | 补 ④⑥，把能力接进来、把经验传出去 |
| 15–18 | 已经是工程实践 | 关注成本、安全边界和长期可维护性 |

## 四、示例项目：一个能跟着跑的仓库

后面所有配置和命令都在同一个项目上演示，你可以现在就把它建起来。这是一个企业报销审批服务，用 ASP.NET Core Minimal API 加 EF Core：

```bash
mkdir ExpenseFlow && cd ExpenseFlow
git init

dotnet new sln -n ExpenseFlow
dotnet new webapi -n ExpenseFlow.Api   -o src/ExpenseFlow.Api --use-minimal-apis
dotnet new xunit  -n ExpenseFlow.Tests -o tests/ExpenseFlow.Tests

dotnet sln add src/ExpenseFlow.Api/ExpenseFlow.Api.csproj
dotnet sln add tests/ExpenseFlow.Tests/ExpenseFlow.Tests.csproj
dotnet add tests/ExpenseFlow.Tests reference src/ExpenseFlow.Api

dotnet add src/ExpenseFlow.Api   package Microsoft.EntityFrameworkCore.Sqlite
dotnet add tests/ExpenseFlow.Tests package Microsoft.AspNetCore.Mvc.Testing
```

核心业务规则只有一条，但它足够典型：**每个报销类别有各自的金额阈值，超过阈值的单子需要二级审批。**

`src/ExpenseFlow.Api/ApprovalPolicy.cs`：

```csharp
namespace ExpenseFlow.Api;

public static class ApprovalPolicy
{
    // 各类别的单笔阈值，超过即需二级审批
    private static readonly Dictionary<string, decimal> Thresholds = new()
    {
        ["Meal"]      = 300m,
        ["Taxi"]      = 500m,
        ["Hotel"]     = 1500m,
        ["Equipment"] = 5000m,
    };

    public const decimal DefaultThreshold = 1000m;

    // 注意是严格大于：正好等于阈值不触发二级审批
    public static bool RequiresSecondApproval(string category, decimal amount) =>
        amount > (Thresholds.TryGetValue(category, out var t) ? t : DefaultThreshold);
}
```

`src/ExpenseFlow.Api/Program.cs`：

```csharp
using ExpenseFlow.Api;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContext<ExpenseDbContext>(o =>
    o.UseSqlite("Data Source=expenses.db"));

var app = builder.Build();

using (var scope = app.Services.CreateScope())
{
    scope.ServiceProvider.GetRequiredService<ExpenseDbContext>().Database.EnsureCreated();
}

app.MapPost("/expenses", async (ExpenseRequest req, ExpenseDbContext db) =>
{
    if (req.Amount <= 0m)
        return Results.BadRequest("金额必须大于 0");

    var expense = new Expense
    {
        EmployeeId  = req.EmployeeId,
        Category    = req.Category,
        Amount      = req.Amount,
        Description = req.Description,
        SubmittedAt = DateTimeOffset.UtcNow,
        Status      = ExpenseStatus.Submitted,
        RequiresSecondApproval = ApprovalPolicy.RequiresSecondApproval(req.Category, req.Amount),
    };

    db.Expenses.Add(expense);
    await db.SaveChangesAsync();
    return Results.Created($"/expenses/{expense.Id}", expense);
});

app.MapGet("/expenses/{id:int}", async (int id, ExpenseDbContext db) =>
    await db.Expenses.FindAsync(id) is { } e ? Results.Ok(e) : Results.NotFound());

app.Run();

public record ExpenseRequest(string EmployeeId, string Category, decimal Amount, string Description);

public enum ExpenseStatus { Draft, Submitted, Approved, Rejected, Reimbursed }

public class Expense
{
    public int Id { get; set; }
    public string EmployeeId { get; set; } = "";
    public string Category { get; set; } = "";
    public decimal Amount { get; set; }
    public string Description { get; set; } = "";
    public DateTimeOffset SubmittedAt { get; set; }
    public ExpenseStatus Status { get; set; }
    public bool RequiresSecondApproval { get; set; }
}

public class ExpenseDbContext(DbContextOptions<ExpenseDbContext> options) : DbContext(options)
{
    public DbSet<Expense> Expenses => Set<Expense>();

    protected override void OnModelCreating(ModelBuilder b) =>
        b.Entity<Expense>().Property(e => e.Amount).HasPrecision(18, 2);
}

public partial class Program;
```

测试有两层。规则层把阈值的边界钉死，`tests/ExpenseFlow.Tests/ApprovalPolicyTests.cs`：

```csharp
using ExpenseFlow.Api;

public class ApprovalPolicyTests
{
    [Theory]
    [InlineData("Meal",      280,  false)]
    [InlineData("Meal",      300,  false)]  // 正好等于阈值，不触发
    [InlineData("Meal",      300.01, true)]
    [InlineData("Hotel",     1500, false)]
    [InlineData("Equipment", 5001, true)]
    [InlineData("Unknown",   1200, true)]   // 未知类别走默认 1000
    public void Second_approval_kicks_in_above_category_threshold(
        string category, decimal amount, bool expected)
        => Assert.Equal(expected, ApprovalPolicy.RequiresSecondApproval(category, amount));
}
```

端到端层确认端点真的把规则用上了，`tests/ExpenseFlow.Tests/ExpensesEndpointTests.cs`：

```csharp
using System.Net;
using System.Net.Http.Json;
using Microsoft.AspNetCore.Mvc.Testing;

public class ExpensesEndpointTests(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task Over_threshold_expense_is_flagged_for_second_approval()
    {
        var client = factory.CreateClient();

        var response = await client.PostAsJsonAsync("/expenses", new
        {
            EmployeeId  = "E1024",
            Category    = "Meal",
            Amount      = 880m,
            Description = "团队聚餐"
        });

        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        var created = await response.Content.ReadFromJsonAsync<Expense>();
        Assert.True(created!.RequiresSecondApproval);
    }

    [Fact]
    public async Task Non_positive_amount_is_rejected()
    {
        var client = factory.CreateClient();
        var response = await client.PostAsJsonAsync("/expenses", new
        {
            EmployeeId = "E1024", Category = "Taxi", Amount = 0m, Description = "空单"
        });

        Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    }
}
```

跑起来：

```bash
dotnet test
```

`dotnet test` 能绿，这个仓库就算准备好了。

选这个场景不是随便选的。阈值判定用 `>` 还是 `>=`、未知类别该不该有兜底、金额该用 `decimal` 还是 `double`——这三类恰好是 AI 最容易悄悄写错、而且看代码看不出来的地方。`[InlineData("Meal", 300, false)]` 这一行的存在，就是为了在它写错时立刻炸。

为什么要强调"测试能跑通"？因为这条命令就是 AI 的眼睛。一个能自己跑 `dotnet test` 并读到失败信息的 agent，和一个只能凭感觉写代码的 agent，是完全不同的两个东西。地基层的第一块砖，就是让它有得验证。

## 五、最省力的起点：三件事

如果你现在分数不高，不用六个环节一起补。按投入产出比排，先做这三件：

**第一件，写一份 `AGENTS.md` 放进仓库根目录。** 哪怕只有十几行，也比没有强。它是 AI 每次开工前默认读的东西：

```markdown
# ExpenseFlow

企业报销审批服务。ASP.NET Core 9 Minimal API + EF Core (SQLite)，测试用 xUnit。

## 领域规则

- 金额一律用 `decimal`，禁止 `double` / `float`。
- 审批阈值只在 `ApprovalPolicy` 里定义，端点里不允许出现魔法数字。
- 阈值判定是**严格大于**：金额正好等于阈值时不触发二级审批。
- 未知类别走 `DefaultThreshold`，不要抛异常。
- 状态流转只能是 Draft → Submitted → Approved / Rejected → Reimbursed，不允许跳步。

## 工程约定

- 端点用 Minimal API 的扩展方法组织，不用 Controller。
- 时间统一 `DateTimeOffset`，存 UTC。
- 每加一个端点，必须在 `tests/ExpenseFlow.Tests` 里补一个集成测试。

## 验证

改完代码后跑 `dotnet build` 和 `dotnet test`，两个都绿才算完成。
```

最后那句"两个都绿才算完成"看着废话，但它是让 AI 从"我觉得写完了"变成"我验证过了"的开关。

**第二件，让 AI 先给计划。** 复杂一点的需求，不要直接说"帮我加个批量审批功能"，改成"先给我一个实现方案，列出要改哪些文件、加哪些测试，我确认后你再动手"。这一步能拦掉大部分方向性错误，成本只有几十秒。

**第三件，把任务切小。** 一个任务的验收标准如果一句话说不清，它就太大了。"给 Expense 加软删除"是合适的粒度，"重构整个审批流"不是。

## 总结

工具年年换，但六个环节的结构是稳的。地基层解决"AI 知道什么、能做什么"，执行层解决"一次任务怎么跑"，收口层解决"产出怎么进主干"。

现在花五分钟做完那份自测表，记住自己的分数和最低的那一环。补课的顺序是固定的：先把工具形态和隔离环境选对，再把 `AGENTS.md` 按作用域拆好，然后才轮到审计划、护栏这些手法；等单次任务跑得稳了，再去接 MCP、做自定义 Agent；最后才是架验证闸门、把配置沉淀成团队默认。

跳过地基层直接上工作流编排，是最常见也最贵的一种走法。
