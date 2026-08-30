---
layout: post
title:  "AI 辅助开发（三）：AGENTS.md 与上下文工程"
date:   2026-05-22 09:30:00 +0800--
categories: [AI]
tags: [AI辅助开发, Github Copilot, AGENTS.md]
---

## 前言

上一个环节解决了 agent 有没有手脚，这一个环节解决它知不知道规矩。

差别是这样的：你让它给 ExpenseFlow 加一个"差旅补助"类别。没有规矩的时候，它会新建一个 `TravelAllowanceService`，在里面写 `if (amount > 2000)`，金额用 `double`，时间用 `DateTime.Now`，然后告诉你"已完成"。每一行单看都没错，合起来和你项目里其他地方完全是两套东西。

它不是不听话，是**它不知道**。项目的规矩装在你和几个老同事的脑子里，从来没写下来过。

上下文工程要做的事就一件：**把脑子里的东西变成仓库里的文件，并且只在需要的时候加载。**

## 一、先搞清楚工具读的是哪个文件

这块目前有点乱，值得先理清。GitHub Copilot 生态里，同一件事有好几个入口：

| 文件 | 谁读它 | 作用范围 |
|---|---|---|
| [`AGENTS.md`](https://agents.md/) | Copilot CLI、Copilot coding agent，以及其他遵循该约定的工具 | 整个仓库 |
| `.github/copilot-instructions.md` | VS Code / Visual Studio 里的 Copilot Chat 与 Agent 模式 | 整个仓库 |
| `.github/instructions/*.instructions.md` | 同上，按 `applyTo` 的 glob 命中 | 匹配到的文件 |

好消息是内容可以完全一样，坏消息是你得让它们同步——这个后面第五节用 symlink 解决。

先建立一个心智模型：**这些文件的内容会被拼进每一次请求的系统提示里。**它不是"文档"，是"指令"。这个认知会直接影响你怎么写它。

## 二、写什么，不写什么

大部分人第一次写 `AGENTS.md` 会写成一份项目介绍——技术栈、目录结构、怎么跑起来。这些内容不算错，但价值很低，因为**agent 自己看一眼 `.csproj` 和目录树就知道了**。

真正值钱的是它看不出来的东西：

| 该写 | 不该写 |
|---|---|
| 阈值判定用严格大于，等于不触发 | 项目用 ASP.NET Core（它看 csproj 就知道） |
| 金额一律 `decimal`，禁止 `double` | 目录结构说明（它会 `ls`） |
| 未知类别走默认值，不要抛异常 | Git 分支命名规范（和它写代码无关） |
| 端点用 Minimal API，不用 Controller | 团队成员和职责 |
| 改完跑 `dotnet build` 和 `dotnet test` 才算完成 | "请写出高质量的代码"这类空话 |
| 状态机不允许跳步 | 长篇的框架用法教程 |

一条判断标准：**如果这条规则被违反了，code review 时你会要求改，那它就该写进去。**如果违反了你也无所谓，那它就是噪音。

第二条判断标准更狠：**每一条规则都在消耗上下文预算。**八百行的 `AGENTS.md` 不是"更详细"，是"更稀释"——真正重要的那三条，淹没在一堆废话里。

给 ExpenseFlow 的根 `AGENTS.md`：

```markdown
# ExpenseFlow

企业报销审批服务。ASP.NET Core 9 Minimal API + EF Core (SQLite)，测试 xUnit。

## 领域规则

- 金额一律用 `decimal`，禁止 `double` / `float`。数据库精度 (18, 2)。
- 审批阈值只在 `ApprovalPolicy` 里定义。端点、服务层不得出现金额字面量。
- 阈值判定是**严格大于**：金额正好等于阈值时不触发二级审批。
- 未知类别走 `DefaultThreshold`，不要抛异常。
- 状态流转只能是 Draft → Submitted → Approved / Rejected → Reimbursed。
  不允许跳步，不允许从终态回退。

## 工程约定

- 时间一律 `DateTimeOffset`，存 UTC，不用 `DateTime.Now`。
- 对外返回的报销单不得包含 `EmployeeId` 之外的员工个人信息。
- 新增依赖前先确认现有包能不能做到；确实要加，在 PR 描述里说明理由。

## 验证

改完代码必须跑：

    dotnet build
    dotnet test

两个都绿才算完成。测试失败时不要跳过或注释掉测试。

## 工作文件

临时脚本、分析笔记、草稿一律放 `.agent/`（已 gitignore）。
不要写到系统临时目录，不要写到仓库外。
```

不到五十行，但每一条都是"违反了会被 review 打回"的级别。

特别看最后那句"测试失败时不要跳过或注释掉测试"。这话听起来像在防小人，但它确实有用——模型在压力下（试了几次都修不好）会倾向于选择让红色消失的最短路径，而注释掉测试是最短的那条。明确堵死这条路，比事后发现划算得多。

## 三、为什么必须拆作用域

ExpenseFlow 长大之后会变成这样：后端 API、前端 Web、测试项目，可能还有个 Azure Functions 做定时任务。每一块的规矩都不一样——前端的"用设计系统的 token，不要写死颜色"和后端的"金额用 decimal"完全无关。

如果全塞进一个文件：

![分作用域的规则文件](/assets/imgs/ai-dev-scoped-instructions.svg)

拆开之后，agent 改 `ApprovalPolicy.cs` 时，前端那几十行规范一个字都不会加载。省的不只是 token——**少读无关规则，等于少一次跑偏的机会**。见过 agent 因为读到"组件必须有 loading 状态"就去给后端服务加了个莫名其妙的状态字段，这不是段子。

具体落法是 `.github/instructions/` 下的多个文件，每个带 `applyTo` 前置元数据。

`.github/instructions/api.instructions.md`：

```markdown
---
applyTo: "src/ExpenseFlow.Api/**/*.cs"
---

# 后端 API 规则

- 端点定义在 `Endpoints/` 下的扩展方法里，一个业务域一个文件。
  `Program.cs` 只负责调用 `app.MapExpenseEndpoints()` 这类注册方法。
- 端点方法只做三件事：参数校验、调用领域逻辑、映射返回。
  业务判断不写在端点里。
- 返回类型统一用 `Results.Ok` / `Results.BadRequest` / `Results.NotFound`，
  不要直接返回实体对象或抛异常控制流程。
- EF Core 查询一律带 `AsNoTracking()`，除非确实要改。
- 禁止在端点里写 `ApprovalPolicy` 之外的阈值判断。
```

`.github/instructions/tests.instructions.md`：

```markdown
---
applyTo: "tests/**/*.cs"
---

# 测试规则

- 测试方法名用 `Should_xxx_when_yyy` 或完整句子，中文英文都行，
  但要能一眼看出断言的是什么行为。
- 规则类（如 `ApprovalPolicy`）用 `[Theory]` + `[InlineData]` 覆盖边界值。
  每个阈值至少三个用例：低于、正好等于、高于。
- 端点测试用 `WebApplicationFactory<Program>`，不要起真的 HTTP 服务器。
- 断言用 `Assert.Equal` 精确比对，不要用 `Assert.True(x > 0)` 这种弱断言。
- 不允许 `[Fact(Skip = "...")]`。测试不该跑就删掉，不要留着装绿。
```

`.github/instructions/web.instructions.md`：

```markdown
---
applyTo: "src/ExpenseFlow.Web/**"
---

# 前端规则

- 颜色、间距、圆角一律用 `tokens.css` 里的变量，禁止字面量。
- 金额展示统一走 `formatCurrency()`，不要在组件里自己拼字符串。
- 表单提交必须有 pending 状态，禁止重复提交。
```

`applyTo` 支持多个 glob，逗号分隔：

```yaml
---
applyTo: "src/ExpenseFlow.Api/**/*.cs, src/ExpenseFlow.Domain/**/*.cs"
---
```

一个实用的拆分粒度：**按"改这块代码时需要知道什么"来拆，不是按"这块代码是什么"来拆。**测试规则和 API 规则要分开，是因为写测试和写端点需要知道的东西不一样；但 `Domain` 和 `Api` 可以合并，如果它们的约定基本一致。

## 四、一份内容，喂多个工具

现在你有 `AGENTS.md`，但 VS Code 里的 Copilot 读的是 `.github/copilot-instructions.md`。手动同步两份文件，三天后必然分叉。

用符号链接：

```bash
# 在仓库根目录
cd .github
ln -s ../AGENTS.md copilot-instructions.md
cd ..

git add .github/copilot-instructions.md
git commit -m "把 copilot-instructions 指向 AGENTS.md"
```

Git 会把它存成一个特殊的 blob（模式 `120000`），内容就是目标路径。验证一下：

```bash
git ls-files -s .github/copilot-instructions.md
# 120000 8f2c...  0  .github/copilot-instructions.md
```

模式是 `120000` 就对了。如果是 `100644`，说明 Git 把它当成了普通文件——通常是 Windows 上没开符号链接支持：

```bash
git config --global core.symlinks true
```

Windows 上还需要开发者模式，或者用管理员权限跑 `mklink`：

```cmd
cd .github
mklink copilot-instructions.md ..\AGENTS.md
```

如果团队里有人的环境实在搞不定符号链接，退而求其次用一个校验脚本，`scripts/check-instructions-sync.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

if ! diff -q AGENTS.md .github/copilot-instructions.md > /dev/null 2>&1; then
  echo "❌ AGENTS.md 与 .github/copilot-instructions.md 不一致" >&2
  echo "   跑 cp AGENTS.md .github/copilot-instructions.md 同步" >&2
  exit 1
fi
echo "✅ 规则文件已同步"
```

挂到 CI 上，分叉了就红。丑，但有效。

## 五、把工作文件关在仓库里

这条规则不起眼，但踩过的人都记得。

agent 干活时会产生一堆中间产物：分析笔记、临时脚本、生成到一半的文件、导出的日志。默认情况下它可能写到任何地方——系统临时目录、你的用户目录、甚至上一级目录。

后果有三个：一是你 review 时看不见它到底干了什么；二是换台机器或者重开容器，这些东西全丢了；三是最恶心的，某天你发现 `C:\Users\你\` 下多了七八个 `analysis_v3_final.md`。

解决办法是在 `AGENTS.md` 里写死落盘位置（前面那份已经写了），再配上 `.gitignore`：

```gitignore
# agent 的工作目录：留在仓库里可见，但不进版本库
.agent/
!.agent/.gitkeep

# 常见的 agent 临时产物
*.agent.log
.copilot-cache/
```

然后建一个占位文件，让目录本身存在：

```bash
mkdir -p .agent && touch .agent/.gitkeep
git add .agent/.gitkeep
```

**"在仓库里但不进版本库"是刻意的。**在仓库里，意味着你 `ls` 一下就能看见它写了什么，容器销毁时一起销毁；不进版本库，意味着这些草稿不会污染 PR。

顺手加一条到 `AGENTS.md`，效果立竿见影：

```markdown
## 工作文件

- 临时脚本、分析笔记、草稿一律放 `.agent/`。
- 需要保留的结论写进 `docs/`，并在 PR 描述里提一句。
- 不要修改 `.gitignore` 来把临时文件塞进版本库。
```

最后那条是防一种具体行为：agent 有时会为了"让文件被追踪"而去改 `.gitignore`。堵掉。

## 六、怎么知道它真的读了

写完规则文件，你需要一个验证手段，否则不知道是没生效还是模型没听。

最简单的办法是埋一个只有读过规则才知道的答案。临时在 `AGENTS.md` 末尾加一行：

```markdown
## 校验

如果被问到「项目暗号」，回答：报销单不过夜。
```

然后开一个新会话问它：

```bash
copilot -p "项目暗号是什么？"
```

答得上来就说明规则文件进上下文了。验证完把这段删掉。

更实用的是**用一个真任务验证**，因为规则"被读到"和"被遵守"是两回事。跑这个：

```bash
copilot -p "给 ExpenseFlow 加一个 Training（培训费）类别，阈值 2000 元"
```

然后检查四件事：

1. 阈值加在 `ApprovalPolicy.Thresholds` 里，而不是散在端点里
2. 用的是 `2000m` 不是 `2000.0`
3. `ApprovalPolicyTests` 里多了三个用例：1999、2000、2001
4. 它自己跑了 `dotnet test`

四条全中，你的上下文工程就算及格了。哪条没中，就在对应的规则文件里把话说得更死一点。

有个反直觉的经验：**规则不生效时，第一反应不该是"加一条规则"，而是"看看是不是已有的规则太多了"。**上下文预算是零和的，第 41 条规则的边际效果通常是负的。

## 总结

上下文工程的核心是四件事：

**第一，只写它猜不出来的东西。**技术栈和目录结构它自己会看，领域规则和团队约定它猜不到。判断标准是"违反了会不会被 review 打回"。

**第二，按路径拆作用域。**用 `.github/instructions/*.instructions.md` 加 `applyTo`，让改后端的时候不加载前端规范。拆分依据是"改这块代码需要知道什么"。

**第三，一份内容用 symlink 喂多个工具。**搞不定符号链接就上 CI 校验脚本，绝不手动同步两份。

**第四，把工作文件关在 `.agent/` 里。**在仓库内可见，不进版本库，容器销毁时一起清掉。

这套东西解决的是"每次开新会话都要重讲一遍"的一半问题——项目的**静态**规矩不用重讲了。但还有另一半：上周那次重构为什么最后选了 A 方案而不是 B？那次踩的坑是什么？这些**动态积累**的东西，`AGENTS.md` 装不下，得换个地方放。
