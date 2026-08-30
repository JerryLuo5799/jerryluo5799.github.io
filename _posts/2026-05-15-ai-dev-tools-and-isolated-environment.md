---
layout: post
title:  "AI 辅助开发（二）：Copilot CLI、容器隔离与 worktree 并行"
date:   2026-05-15 09:30:00 +0800--
categories: [AI]
tags: [AI辅助开发, Github Copilot, DevContainer]
---

## 前言

有个问题值得先想清楚：为什么同一个模型，装在编辑器里和装在终端里，干活能力差这么多？

答案不在模型，在**它能碰到什么**。编辑器里的补全看得见你光标附近的几百行；终端里的 agent 能 `ls` 整个仓库、能跑 `dotnet test`、能读到编译器报的错然后回头改代码。前者是个会写字的助手，后者是个能自己闭环的工人。

工具形态决定了能力上限。这一环没选对，后面所有关于上下文、护栏、验证的功夫都使不上劲——你没法给一个只会吐文本的东西架"跑测试才算完成"的闸门。

## 一、四种形态，四种活

![四种 AI 开发工具形态](/assets/imgs/ai-dev-tool-forms.svg)

这四种不是升级关系，是并存关系。一天里你会在它们之间切好几次：

**内联补全**用来干那些"我知道要写什么，只是懒得敲"的活。给 `ApprovalPolicyTests` 补第七个 `[InlineData]`，或者写 DTO 的一堆属性——你按 Tab 的速度就是它的价值。

**IDE 的 Agent 模式**用来干需要你在场的活。改一个功能、修一个具体的 bug，你想边看它改边判断方向对不对。改动直接落在你的工作区，好处是所见即所得，坏处也是所见即所得——它把你没保存的东西搅了，你得自己收拾。

**终端 Agent** 用来干你不想盯着看的活。"把 `ApprovalPolicy` 的阈值改成从配置读，所有引用点跟着改，测试补齐"——这种任务它可能要跑二三十步，中间试错好几次，你盯着看纯属浪费时间。

**云端托管 Agent** 用来干边角料。依赖升级、改个文案、补个缺失的 XML 注释。你在 Issue 里描述清楚，它在别人的机器上跑完，给你一个 PR。你电脑关机也不影响。

一个粗暴但好用的判断标准：**这个任务如果交给一个刚入职的实习生，你会站在他背后看吗？** 会，就用 IDE Agent 模式；不会但会 review 他的 PR，就用 CLI 或云端。

## 二、Copilot CLI 上手

先说清楚一件容易混的事：`gh copilot` 和 `copilot` 是两个东西。

前者是 GitHub CLI 的一个扩展，用来问命令怎么写：

```bash
gh extension install github/gh-copilot

gh copilot suggest "找出仓库里所有超过 300 行的 .cs 文件"
gh copilot explain "git worktree add -b feat/x ../x main"
```

它只会给你建议，不动你的文件。有用，但它不是 agent。

真正能改代码的是独立的 [Copilot CLI](https://docs.github.com/copilot?wt.mc_id=MVP_324329)：

```bash
npm install -g @github/copilot
copilot
```

首次运行会让你登录。进去之后是一个交互式会话，你描述任务，它读文件、改文件、跑命令。在 ExpenseFlow 里试一个真任务：

```
> 把 ApprovalPolicy 里硬编码的阈值表改成从 appsettings.json 的
  "Approval:Thresholds" 节读取。保留 DefaultThreshold 的兜底行为，
  现有测试必须全绿。
```

它大致会走这么一条路：读 `ApprovalPolicy.cs` → 读 `Program.cs` 找注册位置 → 改成 `IOptions<ApprovalOptions>` → 改 `appsettings.json` → 跑 `dotnet test` → 发现 `ApprovalPolicyTests` 因为静态方法变实例方法编译不过 → 回头改测试 → 再跑一次。

**最后那两步是关键。**它自己发现了编译错误、自己回头改，这个循环就是终端 agent 和聊天框的分水岭。而这个循环之所以能转起来，是因为 [ExpenseFlow 的骨架](/2026/05/08/ai-dev-where-are-you-stuck/) 里那句 `dotnet test` 从第一天起就是绿的。

非交互模式适合写进脚本：

```bash
copilot -p "给 Expense 加一个 SoftDeleted 字段，更新 GET /expenses 过滤掉已删除的，补集成测试"
```

还有一个参数你迟早会遇到：`--allow-all-tools`。加上它，agent 执行命令前不再逐条问你。效率高很多，风险也高很多——它可以 `rm -rf`，可以 `git push --force`，可以把你的 `~/.aws/credentials` 读出来发到某个 API。

所以下一节是必修课。

## 三、把 agent 关进容器里

不是因为不信任模型，是因为**授权范围应该匹配任务范围**。一个"改阈值配置"的任务，没有任何理由能访问你的 SSH 私钥。

用 [dev container](https://containers.dev/) 是最省事的做法。`.devcontainer/devcontainer.json`：

```json
{
  "name": "ExpenseFlow AI Sandbox",
  "image": "mcr.microsoft.com/devcontainers/dotnet:1-9.0",

  "features": {
    "ghcr.io/devcontainers/features/node:1": { "version": "22" },
    "ghcr.io/devcontainers/features/github-cli:1": {}
  },

  "postCreateCommand": "npm install -g @github/copilot && dotnet restore",

  "runArgs": [
    "--cap-drop=ALL",
    "--security-opt=no-new-privileges"
  ],

  "remoteUser": "vscode",

  "customizations": {
    "vscode": {
      "extensions": [
        "GitHub.copilot",
        "GitHub.copilot-chat",
        "ms-dotnettools.csdevkit"
      ]
    }
  }
}
```

三个细节值得说：

- **`--cap-drop=ALL`** 把 Linux capability 全部丢掉。容器里的进程连改系统时间都做不到，更别说挂载设备。
- **没有 `mounts` 配置。** 默认只挂载工作区目录，你的 `~/.ssh`、`~/.aws`、`~/.kube` 一个都进不去。这是白名单思维——想让它访问什么，你显式加进来。
- **`remoteUser` 不是 root。** 万一有东西逃出进程边界，也只是个普通用户。

如果你不用 VS Code，只想在终端跑 headless 的 agent，一个独立 Dockerfile 就够，`.devcontainer/agent.Dockerfile`：

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:9.0

RUN apt-get update \
 && apt-get install -y --no-install-recommends git curl ca-certificates \
 && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
 && apt-get install -y nodejs \
 && rm -rf /var/lib/apt/lists/*

RUN npm install -g @github/copilot

RUN useradd -m agent
USER agent
WORKDIR /work
```

跑起来：

```bash
docker build -t ef-agent -f .devcontainer/agent.Dockerfile .

docker run --rm -it \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  -v "$PWD":/work \
  -e GH_TOKEN \
  ef-agent \
  copilot --allow-all-tools -p "把 ApprovalPolicy 的阈值改成从 appsettings.json 读取，并补测试"
```

`-v "$PWD":/work` 是唯一的挂载点，`-e GH_TOKEN` 是唯一透传的凭据。在这个边界里，`--allow-all-tools` 就是可接受的了——最坏情况是它把当前仓库搞乱，`git checkout .` 就能回来。

**这个组合才是重点：容器提供边界，`--allow-all-tools` 提供效率。**只用后者不用前者，是在拿整台机器赌；只用前者不用后者，你会被无穷无尽的确认弹窗磨到放弃。

## 四、并行：一个仓库，多个工作区

当 agent 能自己跑二十分钟不用你管的时候，一个新问题冒出来了：这二十分钟你干嘛？

答案是同时开第二个、第三个任务。但它们不能在同一个目录里——两个 agent 同时改 `Program.cs`，结果只能是灾难。

`git worktree` 正好解决这个：

![用 worktree 并行跑多个 agent](/assets/imgs/ai-dev-worktree-parallel.svg)

写成脚本，`scripts/new-agent-task.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

task="${1:?用法: ./scripts/new-agent-task.sh <task-slug>}"
branch="feat/${task}"
dir="../ef-${task}"

if [ -d "$dir" ]; then
  echo "工作区 $dir 已存在" >&2
  exit 1
fi

git worktree add -b "$branch" "$dir" main

# 本地配置不进 git，但每个工作区都需要
[ -f .env.local ] && cp .env.local "$dir/"

# 先确认基线是绿的，再交给 agent
( cd "$dir" && dotnet restore && dotnet test )

echo "✅ 工作区就绪：$dir （分支 $branch）"
echo "   下一步：cd $dir && copilot"
```

用起来：

```bash
chmod +x scripts/new-agent-task.sh

./scripts/new-agent-task.sh receipt-ocr    # → ../ef-receipt-ocr
./scripts/new-agent-task.sh multi-currency # → ../ef-multi-currency
```

收工时清理：

```bash
git worktree remove ../ef-receipt-ocr
git branch -D feat/receipt-ocr
git worktree prune
```

脚本里那句 `dotnet test` 不是多余的。**把一个已经是红的基线交给 agent，它会花二十分钟去修不是它造成的问题。**先确认绿，再派活。

有个坑要提醒：`bin/` 和 `obj/` 在每个 worktree 里是独立的，三个工作区就是三份编译产物。SSD 上无所谓，但如果你的仓库带大量 NuGet 缓存或者前端 `node_modules`，注意磁盘。

## 五、本地模型：什么时候真的值得

"数据不出内网"是个很有说服力的理由，但它不能自动推出"所以我们要自建"。先看清楚代价：

| 用途 | 本地模型是否合适 | 原因 |
|---|---|---|
| 代码补全 | 合适 | 上下文短、延迟敏感，14B 级别的模型够用 |
| 敏感数据预处理 | 合适 | 脱敏、分类这类任务简单且高频，本地跑最划算 |
| 单文件重构 | 勉强 | 能做，质量比顶级云端模型明显差一档 |
| 多步 agent 任务 | 不合适 | 需要强推理和长上下文，本地模型的工具调用稳定性还不够 |
| 跨仓库分析 | 不合适 | 上下文窗口不够 |

用 [Ollama](https://ollama.com) 跑一个本地补全模型很容易：

```bash
ollama pull qwen2.5-coder:14b
ollama serve
```

验证一下它真的能用：

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5-coder:14b",
  "prompt": "用 C# 写一个把 decimal 金额格式化成人民币字符串的方法",
  "stream": false
}'
```

务实的结论是**混着用**：日常补全和敏感数据处理走本地，需要长链推理的 agent 任务走云端。如果你的合规要求真的一刀切禁止代码出网，那也不是不能做，只是要接受"能力档位降一级"，并且在上下文工程上花更多功夫——本地模型对好的 `AGENTS.md` 更敏感，因为它自己猜不出来。

## 六、账单看得见

这一节短，但漏掉的话，月底会有惊喜。

组织级的席位和用量，用 GitHub CLI 直接查：

```bash
# 席位分配情况：买了多少、用了多少
gh api /orgs/YOUR_ORG/copilot/billing --jq '{
  seats_total: .seat_breakdown.total,
  active_this_cycle: .seat_breakdown.active_this_cycle,
  inactive_this_cycle: .seat_breakdown.inactive_this_cycle
}'
```

`inactive_this_cycle` 是最值钱的一个数字——买了席位但整个计费周期一次没用的人。这笔钱可以直接省下来。

用量趋势：

```bash
# 最近 28 天的每日指标
gh api /orgs/YOUR_ORG/copilot/metrics --jq '.[] | {
  date: .date,
  active_users: .total_active_users,
  engaged_users: .total_engaged_users
}'
```

`total_active_users`（打开过）和 `total_engaged_users`（真的采纳了建议）之间的差值，就是你团队里"装了但没用起来"的人数。这个数字如果一直很大，问题通常不在工具，在没人教他们怎么用。

把它塞进一个定时任务，每周往群里发一次，比季度末看财务报表有用得多。

## 总结

工具形态这一环，四件事：

**第一，四种形态并存，按"你会不会站在他背后看"来选。**内联补全省敲键盘的时间，IDE Agent 干需要你在场的活，CLI Agent 干你不想盯着的活，云端 Agent 干边角料。

**第二，`--allow-all-tools` 必须配容器。**这两个是一对，缺一个都会出事：只有前者是拿整台机器赌，只有后者是被弹窗磨死。挂载点用白名单思维——想让它碰什么，你显式加进来。

**第三，worktree 让并行成为可能。**当 agent 能自己跑二十分钟，你就该同时开第二个任务了。派活前先确认基线是绿的。

**第四，账单和用量要能一行命令查出来。**`inactive_this_cycle` 和 active/engaged 的差值，是最容易变现的两个数字。

环境搭好之后，下一个瓶颈立刻就来了：agent 有手有脚了，但它不知道你们项目的规矩——阈值判定该用 `>` 还是 `>=`、金额该用 `decimal` 还是 `double`。这些东西得写下来给它看，而且不能写成一个八百行的大文件。
