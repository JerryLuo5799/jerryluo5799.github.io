---
layout: post
title:  "AI 辅助开发（六）：Chat Mode、MCP、Skill 怎么选"
date:   2026-06-12 09:30:00 +0800--
categories: [AI]
tags: [AI辅助开发, Github Copilot, MCP]
---

## 前言

这四个词是目前 AI 开发工具里最容易搞混的一组。它们都能"让 AI 更懂你的项目"，都是往仓库里放几个 Markdown 或 JSON 文件，文档里的示例看起来也差不多。

于是常见的结果是：一个团队把四种都配上了，`.github` 下堆了十几个文件，用起来还是不顺——因为**每一个都在解决不同的缺口，配错了地方等于没配**。

分清它们只需要问一个问题：**我缺的到底是什么？**

## 一、四个缺口，四种补法

![四种能力扩展手段的选择](/assets/imgs/ai-dev-capability-decision.svg)

用一句话区分：

| | 补的缺口 | 一句话 |
|---|---|---|
| **Prompt 文件** | 你的打字量 | 把一段反复输入的要求存成文件 |
| **Chat Mode** | 它的行为姿势 | 限定它在某类工作里的角色和可用工具 |
| **MCP 服务器** | 它的能力边界 | 让它够得着原本够不着的东西 |
| **Skill** | 它的流程知识 | 一套多步做法，用到才加载 |

关键区别在这里：**Prompt 文件和 Skill 给的是"知识"，MCP 给的是"能力"，Chat Mode 给的是"约束"。**

一个 agent 读完再多文档也不可能知道生产库里 `Expense` 表实际有多少行——这是能力缺口，只能用 MCP 补。反过来，一个接了数据库 MCP 的 agent 也不会自动知道你们"加一个报销类别要走七步"——这是知识缺口，MCP 补不了。

## 二、Prompt 文件：省你的打字

最简单的一种，本质就是把常用要求存成文件。

两个已经登场过的例子：`/plan`（只出方案不动手）和 `/wrap-up`（任务收尾写摘要）。格式是 `.github/prompts/<名字>.prompt.md`：

```markdown
---
mode: agent
description: 给现有端点补集成测试
---

给我指定的端点补集成测试。

要求：
- 用 `WebApplicationFactory<Program>`，放在 `tests/ExpenseFlow.Tests/` 下。
- 至少覆盖三种情况：正常路径、参数非法、目标不存在。
- 涉及金额的断言必须用 `decimal` 字面量（`300m` 而不是 `300`）。
- 不要修改被测端点的实现。如果你认为端点有 bug，写在回复里告诉我，
  不要顺手改掉。
- 写完跑 `dotnet test`，把结果贴给我。
```

存成 `.github/prompts/add-tests.prompt.md`，之后在 Copilot Chat 里 `/add-tests`，再补一句"给 POST /expenses/{id}/receipt"就行。

**判断标准很直白：同一段要求你打到第三遍，它就该变成文件。**

那句"如果你认为端点有 bug，写在回复里告诉我，不要顺手改掉"值得单独说。补测试的任务里，agent 发现实现有问题时的默认行为是把实现改了——然后测试当然全绿。**这会让"补测试"这个动作失去意义**，因为你不知道它是让测试适配了实现，还是让实现适配了测试。

## 三、Chat Mode：管住它的手

Prompt 文件是一次性的：你 `/add-tests` 之后，这轮对话结束，约束就没了。

有些约束需要**持续**存在。典型场景是架构评审——你想让它通读一遍代码给意见，但绝对不要它顺手改。这时候你需要的不是一段 prompt，是一个模式。

`.github/chatmodes/review.chatmode.md`：

```markdown
---
description: 只读架构评审
tools: ['codebase', 'search', 'usages', 'problems', 'findTestFiles']
---

你现在是架构评审员。**你没有任何写文件的工具，也不要请求获得。**

评审时按这个顺序看：

1. 这次改动和 `docs/DESIGN.md` 里的既有决策有没有冲突？
2. 领域规则有没有被绕过？特别看：金额类型、阈值来源、状态流转、
   时间类型。
3. 测试是不是真的在验证行为？有没有出现「改实现让测试变绿」的痕迹？
4. 有没有引入新依赖？值不值得？

输出格式：

- **阻断项**：必须改，不改不能合。每条给出文件、行号和理由。
- **建议项**：可以改，说明收益。
- **问题**：你不确定、需要作者解释的地方。

不要夸奖，不要总结代码做了什么（作者知道）。只说问题。
```

关键在 frontmatter 的 `tools` 那一行——**你把写文件的工具从工具箱里拿掉了**。这不是靠"请你不要改代码"这句提示词约束的，是它压根没有这个能力。软约定和硬约束的区别，这里体现得最明显。

VS Code 的 Copilot Chat 里，模式在输入框上方的下拉里切换。切到 `review` 之后，整个会话都在这个约束下。

另一个常用的模式是"只写测试"：

```markdown
---
description: 只补测试，不碰实现
tools: ['codebase', 'search', 'findTestFiles', 'editFiles', 'runTests']
---

你只负责 `tests/` 目录下的文件。`src/` 下的任何文件都不要修改。

发现实现有问题时，在回复里说明，不要动手改。
```

这个模式可以做到 prompt 文件做不到的事：即使你在对话中途改主意说"顺便把那个 bug 修了吧"，它也做不到——工具不在。

**Prompt 文件和 Chat Mode 的分界线：一次性的要求用前者，需要贯穿整个会话的约束用后者。**

## 四、MCP：给它一双新的手

前两种都是在改"怎么说"。[MCP](https://modelcontextprotocol.io/) 改的是"能做什么"。

[MCP 协议实战指南](/2025/06/13/what-is-mcp/) 里已经讲过协议本身和怎么用 .NET 写一个 server。这里只讲增量：**给 ExpenseFlow 接一个数据库 MCP，并且把安全边界做对。**

为什么值得做？因为 agent 经常需要事实而不是猜测。"生产环境有多少单据卡在 Submitted 超过 7 天"这种问题，它没法从代码里读出来，只能查库。没有 MCP 的时候，流程是你手动查、复制、粘贴给它——每次都是这样，很烦。

先建项目：

```bash
dotnet new console -n ExpenseFlow.Mcp -o tools/ExpenseFlow.Mcp
dotnet sln add tools/ExpenseFlow.Mcp/ExpenseFlow.Mcp.csproj

cd tools/ExpenseFlow.Mcp
dotnet add package ModelContextProtocol --prerelease
dotnet add package Microsoft.Extensions.Hosting
dotnet add package Microsoft.Data.Sqlite
```

`tools/ExpenseFlow.Mcp/Program.cs`：

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

var builder = Host.CreateApplicationBuilder(args);

// 关键：stdio 是 MCP 的通信管道，日志必须走 stderr，
// 打到 stdout 会污染协议消息
builder.Logging.AddConsole(o => o.LogToStandardErrorThreshold = LogLevel.Trace);

builder.Services
    .AddMcpServer()
    .WithStdioServerTransport()
    .WithToolsFromAssembly();

await builder.Build().RunAsync();
```

工具定义，`tools/ExpenseFlow.Mcp/ExpenseQueryTools.cs`：

```csharp
using System.ComponentModel;
using System.Text;
using System.Text.RegularExpressions;
using Microsoft.Data.Sqlite;
using ModelContextProtocol.Server;

[McpServerToolType]
public static class ExpenseQueryTools
{
    private static readonly string ConnectionString =
        Environment.GetEnvironmentVariable("EXPENSEFLOW_READONLY_CONN")
        ?? "Data Source=expenses.db;Mode=ReadOnly";

    private const int MaxRows = 200;

    [McpServerTool]
    [Description("""
        对报销库执行一条只读 SELECT 查询，返回前 200 行。
        只允许单条 SELECT 语句，禁止 INSERT/UPDATE/DELETE/DDL 和多语句。
        EmployeeId 会被脱敏，不要指望拿到完整员工编号。
        表结构见 describe_schema 工具。
        """)]
    public static string Query(
        [Description("一条 SELECT 语句，不要带分号结尾的多语句")] string sql)
    {
        var guard = Reject(sql);
        if (guard is not null) return $"❌ 拒绝执行：{guard}";

        using var conn = new SqliteConnection(ConnectionString);
        conn.Open();

        using var cmd = conn.CreateCommand();
        cmd.CommandText = $"SELECT * FROM ({sql.TrimEnd(';', ' ')}) LIMIT {MaxRows}";

        using var reader = cmd.ExecuteReader();
        var sb = new StringBuilder();

        var cols = Enumerable.Range(0, reader.FieldCount)
                             .Select(reader.GetName).ToArray();
        sb.AppendLine(string.Join(" | ", cols));
        sb.AppendLine(string.Join(" | ", cols.Select(_ => "---")));

        var rows = 0;
        while (reader.Read())
        {
            var values = Enumerable.Range(0, reader.FieldCount)
                .Select(i => Mask(cols[i], reader.IsDBNull(i) ? "" : reader.GetValue(i)?.ToString() ?? ""));
            sb.AppendLine(string.Join(" | ", values));
            rows++;
        }

        sb.AppendLine();
        sb.AppendLine(rows == MaxRows ? $"（已截断到 {MaxRows} 行）" : $"（共 {rows} 行）");
        return sb.ToString();
    }

    [McpServerTool]
    [Description("返回报销库的表结构，包含每个表的列名和类型。")]
    public static string DescribeSchema()
    {
        using var conn = new SqliteConnection(ConnectionString);
        conn.Open();
        using var cmd = conn.CreateCommand();
        cmd.CommandText =
            "SELECT name, sql FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%'";

        using var reader = cmd.ExecuteReader();
        var sb = new StringBuilder();
        while (reader.Read())
        {
            sb.AppendLine($"## {reader.GetString(0)}");
            sb.AppendLine("```sql");
            sb.AppendLine(reader.GetString(1));
            sb.AppendLine("```");
            sb.AppendLine();
        }
        return sb.ToString();
    }

    private static string? Reject(string sql)
    {
        var s = sql.Trim();

        if (!s.StartsWith("SELECT", StringComparison.OrdinalIgnoreCase)
            && !s.StartsWith("WITH", StringComparison.OrdinalIgnoreCase))
            return "只允许 SELECT 或 WITH 开头的查询";

        if (s.TrimEnd(';', ' ').Contains(';'))
            return "不允许多条语句";

        var banned = new[] { "insert", "update", "delete", "drop", "alter",
                             "create", "attach", "pragma", "vacuum" };
        var lowered = s.ToLowerInvariant();
        foreach (var kw in banned)
            if (Regex.IsMatch(lowered, $@"\b{kw}\b"))
                return $"包含被禁止的关键字：{kw}";

        return null;
    }

    // 员工编号脱敏：E1024 → E1***
    private static string Mask(string column, string value) =>
        column.Equals("EmployeeId", StringComparison.OrdinalIgnoreCase) && value.Length > 2
            ? value[..2] + new string('*', value.Length - 2)
            : value;
}
```

注册到 VS Code，`.vscode/mcp.json`：

```json
{
  "servers": {
    "expenseflow-db": {
      "type": "stdio",
      "command": "dotnet",
      "args": ["run", "--project", "tools/ExpenseFlow.Mcp"],
      "env": {
        "EXPENSEFLOW_READONLY_CONN": "Data Source=expenses.db;Mode=ReadOnly"
      }
    }
  }
}
```

接上之后就能这么问了：

```
> 用 expenseflow-db 查一下：Submitted 状态且提交超过 7 天的单据有多少，
  按类别分组。
```

**这段代码里真正重要的不是查询逻辑，是那三层防线：**

1. **连接串里的 `Mode=ReadOnly`。**这是数据库层面的强制，即使前面两层都被绕过，SQLite 也会拒绝写操作。
2. **`Reject()` 的语句白名单。**只放行 `SELECT`/`WITH` 开头的单语句，拦掉分号拼接和 DDL 关键字。
3. **`Mask()` 的字段脱敏。**这一层是给"数据本身"设的边界——即使查询合法，员工编号也不该原样进入模型上下文。

第三层在真实项目里会复杂得多。这里只按列名做了个粗暴的字符替换，但报销单的 `Description` 字段里可能有姓名、手机号、发票抬头——这些是自由文本，列名帮不了你。真要认真做，`Mask` 这个位置应该换成专门的 PII 识别组件，按实体类型而不是按列名脱敏。

**给 agent 开数据库权限时最常见的错误，是只做第 2 层。**只在应用层校验 SQL，是在跟一个比你更会写 SQL 的东西比拼正则表达式——这场比赛你赢不了。真正兜底的是第 1 层，那个只读连接。

## 五、Skill：一套做法，用到才读

前面三种都有一个共同的局限：**它们要么常驻上下文（instructions），要么需要你主动触发（prompt 文件、chat mode）。**

有些知识两头不靠。比如"ExpenseFlow 的数据库迁移怎么做"——完整流程有七八步，还有分支判断（有没有破坏性变更、要不要停机、怎么回滚）。写进 `AGENTS.md` 太长，做成 prompt 文件又需要你记得它存在。

Skill 解决的就是这个：**一份带描述的流程手册，agent 自己判断什么时候该读。**

结构是一个目录加一个 `SKILL.md`：

```
.github/skills/
└── db-migration/
    ├── SKILL.md
    ├── checklist.md
    └── rollback.md
```

`SKILL.md`：

```markdown
---
name: db-migration
description: 当需要修改 ExpenseFlow 的数据库结构时使用——新增/删除列、
  改类型、加索引、数据回填。涉及 EF Core Migration 的任何操作都适用。
---

# ExpenseFlow 数据库迁移

## 先判断变更类型

| 类型 | 例子 | 走哪条路 |
|---|---|---|
| 加列（可空） | 加 `ReceiptKey` | 直接迁移，一步到位 |
| 加列（非空） | 加 `SubmittedBy` | 三步走：加可空 → 回填 → 改非空 |
| 删列 | 删 `LegacyCode` | 两阶段：先停用再删，间隔一个发布周期 |
| 改类型 | `int` → `decimal` | 新增列 + 回填 + 切换 + 删旧列 |

## 标准流程

1. 改 `Expense` 实体和 `ExpenseDbContext.OnModelCreating`。
2. `dotnet ef migrations add <名字> -p src/ExpenseFlow.Api`
3. **打开生成的迁移文件人工检查**——EF 对 SQLite 的删列/改类型
   会生成「建新表、拷数据、删旧表」，务必确认拷贝语句没有漏列。
4. 破坏性变更参照 `checklist.md` 逐条过。
5. `dotnet ef database update` 在本地验证。
6. 跑 `dotnet test`，特别关注 `ConventionTests`。
7. 回滚方案写进 PR 描述，模板见 `rollback.md`。

## 硬性禁止

- 不允许手写 SQL 迁移脚本绕过 EF Migration。
- 不允许在迁移里写业务逻辑（回填用单独的一次性脚本）。
- 生成的迁移文件不允许事后手改后再提交同名迁移。
```

关键在 frontmatter 的 `description`。**它不是给人看的，是给 agent 判断"现在要不要读这份手册"用的。**所以要写得具体：写清楚触发场景（"新增/删除列、改类型、加索引"），不要写"数据库相关的最佳实践"这种模糊的东西。

`SKILL.md` 本身要短，详细内容放到旁边的文件里，正文引用它们。这样 agent 先读一页纸判断相关性，确认相关了才去读 `checklist.md`。上下文预算是零和的，能不加载就不加载。

## 六、四种错配，四种症状

实际用下来，配错的情况比配不上更常见。对照表：

| 症状 | 你以为缺的 | 实际缺的 |
|---|---|---|
| 写了一大段 instructions 讲数据库表结构，它还是猜错字段 | 上下文不够 | **MCP**——让它自己去查，别喂 |
| 配了数据库 MCP，它还是不知道加类别要走七步 | MCP 没配好 | **Skill 或 prompt 文件**——这是流程知识 |
| 每次都要提醒"只看不要改" | 提示词不够狠 | **Chat Mode**——把写工具拿掉 |
| Skill 写了但从来不被触发 | Skill 没用 | **`description` 太模糊**——改成具体的触发场景 |
| `AGENTS.md` 涨到三百行，效果反而变差 | 规则不够细 | **该拆的拆到 instructions，该沉的沉到 Skill** |

最后一行是最普遍的。**`AGENTS.md` 变长是个警报，不是成就。**超过五十行就该问：这些内容里，哪些只在特定目录下才需要（→ `instructions`），哪些只在特定任务里才需要（→ Skill 或 prompt 文件）。

## 七、串起来跑一遍

一个真实任务，四种手段各就各位：

```bash
# 1. 用只读模式先摸清现状（Chat Mode 保证它不会手贱）
#    切到 review 模式
> 现在的附件存储是怎么做的？和 docs/DESIGN.md 说的一致吗？

# 2. 用 MCP 查真实数据，而不是猜
> 用 expenseflow-db 查：有多少单据的 ReceiptKey 是空的？按月份分组。

# 3. 用 prompt 文件出方案（切回 agent 模式）
> /plan 给附件加过期清理：超过 3 年的报销单，附件转冷存储

# 4. 方案里涉及加一列 ArchivedAt —— agent 自己识别出该读迁移 Skill，
#    于是它按三步走（加可空列 → 回填 → 加索引），而不是一把梭

# 5. 收尾
> /wrap-up
```

第 4 步是 Skill 存在的意义：**你没提醒它，它自己知道该翻哪本手册。**

## 总结

四种手段，四个缺口：

**Prompt 文件补打字量。**同一段要求打到第三遍就该存成文件。

**Chat Mode 补行为约束。**需要贯穿整个会话的限制用它，靠的是把工具从工具箱里拿掉，不是靠提示词求它。

**MCP 补能力边界。**让它够得着数据库、内部 API、浏览器。开数据库权限时三层防线一层都不能少，其中只读连接串是唯一真正兜底的那层。

**Skill 补流程知识。**多步且带判断的做法，`description` 写清触发场景，正文保持一页纸，细节放旁边的文件。

**以及一条通用判断：`AGENTS.md` 变长是警报。**该拆的拆，该沉的沉。

配齐这四样，单次任务的能力就基本到顶了。再往上就是另一个维度的问题——怎么让多个任务自动串起来跑，让 agent 不只是被你召唤，而是被事件触发。
