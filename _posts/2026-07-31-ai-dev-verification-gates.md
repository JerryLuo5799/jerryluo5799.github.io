---
layout: post
title:  "AI 辅助开发（八）：四道验证闸门与 Playwright"
date:   2026-07-31 09:30:00 +0800--
categories: [AI]
tags: [AI辅助开发, Github Copilot, Playwright]
---

## 前言

产能上去之后，瓶颈会准时换个位置。

一个人挂三个 worktree、两条巡检流水线，一天能产出七八个 PR。然后这七八个 PR 堆在那里，因为 review 它们需要三个小时，而你一天只有八个小时。

于是团队开始出现两种反应。一种是"先合了吧，测试是绿的"——三周后线上出事，回溯发现问题在两周前那个没人细看的 PR 里。另一种是"这些 PR 先放着"——放到分支冲突，最后全部关掉重来。

**两种反应的根源是同一个：所有验证压力都集中在人类 review 这一个点上。**

## 一、把压力分散到四道门

![AI 产出的四道验证闸门](/assets/imgs/ai-dev-verification-gates.svg)

原则很简单：**每一类问题，应该在能发现它的最便宜的那道门被拦住。**

"金额用了 `double`"这种问题，机器闸一秒钟就能拦，不该出现在人的 review 意见里。"按钮点了没反应"，行为闸能拦。"一万条数据时页面卡死"，环境闸能拦。

人类闸留给前三道拦不住的东西——**意图对不对、该做的做了没、有没有埋坑**。这三件事机器目前确实做不了，而它们恰恰是最重要的。

一个自检信号：**翻一下你最近十条 review 意见，有几条是前三道门本该拦住的？**超过一半，说明门没架好，而不是你 review 得不够仔细。

## 二、第一道：机器闸

前面已经攒了一部分——`dotnet build`、`dotnet test`、`ConventionTests`、pre-commit hook。把它们接到 CI 上，再加一道 AI 评审。

`.github/workflows/ci.yml`：

```yaml
name: ci

on:
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'

      - run: dotnet restore
      - run: dotnet build --no-restore -warnaserror
      - run: dotnet test --no-build --logger "trx;LogFileName=results.trx"

      - name: 上传测试结果
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: '**/results.trx'
```

`-warnaserror` 那个参数值得单独说。agent 对警告的敏感度接近于零——它会留下一堆 "possible null reference" 然后告诉你构建成功。把警告变成错误，这类问题就自动进了机器闸。

代价是你得先把存量警告清干净，否则第一天就红一片。**这件事本身就很适合派给 agent**：改动机械、验收标准明确（`dotnet build -warnaserror` 能过）、风险低。

至于 AI 评审，GitHub 自带的 Copilot code review 在 PR 上点一下就能用，仓库设置里也能配成自动触发。但它给的是通用意见——"考虑处理这个异常"这种。**它不知道你的红线。**

想让它按你的红线来，用一个 agentic workflow，`.github/workflows/ai-review.md`：

```markdown
---
on:
  pull_request:
    types: [opened, ready_for_review, synchronize]

permissions:
  contents: read
  pull-requests: write

engine: copilot

tools:
  github:
    allowed: [get_pull_request, get_pull_request_diff, create_pull_request_review_comment, add_comment]
  bash: ["dotnet build", "dotnet test"]

timeout_minutes: 15
---

# ExpenseFlow PR 红线检查

拿到本次 PR 的 diff，只检查以下红线。**不要做通用的代码质量评审**，
那部分由人负责。

## 红线清单

1. 金额相关的变量、字段、参数用了 `double` 或 `float`
2. 金额字面量（如 `300m`、`1500m`）出现在 `ApprovalPolicy.cs` 之外
3. 使用了 `DateTime.Now` 或 `DateTime.UtcNow`
4. `Expense.Status` 的赋值跳过了中间状态，或从 `Reimbursed` / `Rejected` 回退
5. 新增了 `[Fact(Skip = ...)]` 或 `[Theory(Skip = ...)]`
6. **修改了现有测试的断言**——这一条重点查，
   区分「因为需求变了所以断言也变了」和「因为实现不对所以把断言改松了」
7. `RequiresSecondApproval` 被改成了计算属性
8. 实现了 `docs/DESIGN.md` 「非目标」一节里列出的东西

## 输出

每发现一条，用 `create_pull_request_review_comment` 评论到具体行上，
格式：`🚫 红线 N：<问题> → <怎么改>`

一条都没发现时，用 `add_comment` 回复一句
`✅ 红线检查通过（8 项）`，不要写别的。

**不要评论代码风格、命名、注释、性能。**
```

第 6 条是这个 workflow 最值钱的部分，也是通用 AI 评审绝对做不到的。**"改实现让测试绿"和"改测试让实现绿"在 diff 里长得很像，但后者是灾难。**明确让它区分这两种，命中率会高很多。

## 三、第二道：行为闸

单元测试和集成测试验证的是代码的行为，不是**用户看到的行为**。`WebApplicationFactory` 能确认 `POST /expenses` 返回 201，但确认不了提交按钮点下去有没有反应。

[Playwright](https://playwright.dev/?wt.mc_id=MVP_324329) 补的就是这一段。关键不是"写 UI 测试"这件事本身——那个大家都会——而是**让 agent 自己能开浏览器看**。

装 Playwright 的 MCP 服务器，加到 `.vscode/mcp.json`：

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
    },
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest", "--headless"]
    }
  }
}
```

接上之后，agent 改完前端可以自己验：

```
> 你刚才改了报销提交表单。用 playwright 打开 http://localhost:5173/expenses/new，
  填一张 880 元的餐费单提交，确认：
  1. 提交后跳到详情页
  2. 详情页显示「需要二级审批」
  3. 金额显示为 ¥880.00，不是 880 或 ¥880.0000001
  截图给我。
```

**这个循环的价值在于它是闭合的。**没有 Playwright 的时候，agent 改完前端只能说"应该没问题"；有了之后它能自己发现"点了提交没反应，因为 onClick 绑错了"，然后自己改。

验证过的行为要沉淀成测试，否则下次还得重来一遍。让它顺手写：

```
> 把刚才验证的这三点写成 Playwright 测试，放 e2e/expense-submit.spec.ts。
```

`e2e/expense-submit.spec.ts`：

```typescript
import { test, expect } from '@playwright/test';

test.describe('提交报销单', () => {
  test('超过阈值的单据应标记为需要二级审批', async ({ page }) => {
    await page.goto('/expenses/new');

    await page.getByLabel('类别').selectOption('Meal');
    await page.getByLabel('金额').fill('880');
    await page.getByLabel('说明').fill('团队聚餐');
    await page.getByRole('button', { name: '提交' }).click();

    await expect(page).toHaveURL(/\/expenses\/\d+/);
    await expect(page.getByTestId('second-approval-badge')).toBeVisible();
    await expect(page.getByTestId('amount')).toHaveText('¥880.00');
  });

  test('正好等于阈值的单据不需要二级审批', async ({ page }) => {
    await page.goto('/expenses/new');

    await page.getByLabel('类别').selectOption('Meal');
    await page.getByLabel('金额').fill('300');
    await page.getByLabel('说明').fill('工作餐');
    await page.getByRole('button', { name: '提交' }).click();

    await expect(page.getByTestId('second-approval-badge')).toBeHidden();
  });

  test('提交中禁止重复点击', async ({ page }) => {
    await page.goto('/expenses/new');

    await page.getByLabel('类别').selectOption('Taxi');
    await page.getByLabel('金额').fill('120');
    await page.getByLabel('说明').fill('打车');

    const submit = page.getByRole('button', { name: '提交' });
    await submit.click();
    await expect(submit).toBeDisabled();
  });
});
```

第二个用例——**阈值边界在 UI 上也要测一遍**。后端 `ApprovalPolicyTests` 已经覆盖了 300 这个边界，但前端可能自己又算了一遍（很多前端为了即时提示会重复实现规则），两边不一致是很常见的 bug。

挂到 CI：

```yaml
  e2e:
    runs-on: ubuntu-latest
    needs: build-and-test
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'
      - uses: actions/setup-node@v4
        with:
          node-version: '22'

      - run: npm ci
      - run: npx playwright install --with-deps chromium

      - name: 起后端
        run: |
          dotnet run --project src/ExpenseFlow.Api &
          npx wait-on http://localhost:5000/health --timeout 60000

      - run: npx playwright test

      - if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
```

失败时上传报告这一步别省。**Playwright 的失败报告带截图和 DOM 快照，agent 读得懂**——你可以直接把 artifact 链接甩给它让它自己修。

## 四、第三道：环境闸

前两道门都在 CI 的干净环境里跑。有一类问题只在"接近真实"的环境里才暴露：迁移脚本在有存量数据的库上跑不过、配置项在本地有默认值但生产没配、一万条数据时列表页卡死。

做法是**给每个 PR 一个能点的环境**。

对 Web 应用，用 Azure Static Web Apps 或类似服务的 PR 预览最省事，PR 一开就自动部署，关闭时自动销毁。

对没有托管预览的情况，退一步：**把可运行的产物挂到 PR 上**。

{% raw %}
```yaml
  preview-artifact:
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.0.x'

      - name: 打自包含包
        run: |
          dotnet publish src/ExpenseFlow.Api \
            -c Release -r win-x64 --self-contained \
            -p:PublishSingleFile=true \
            -o ./publish

      - name: 塞一份带存量数据的库进去
        run: cp fixtures/expenses-seeded.db ./publish/expenses.db

      - uses: actions/upload-artifact@v4
        with:
          name: preview-pr-${{ github.event.number }}
          path: ./publish
          retention-days: 7

      - name: 在 PR 上留链接
        uses: actions/github-script@v7
        with:
          script: |
            const url = `${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId}`;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `📦 预览包已就绪：[下载](${url})\n\n解压后直接运行 \`ExpenseFlow.Api.exe\`，内含 5000 条测试数据。`
            });
```
{% endraw %}

那句 `cp fixtures/expenses-seeded.db` 是重点。**空库测不出来的问题，占了环境闸能拦住的问题的一多半**——分页在 5000 条数据下的表现、迁移脚本在有数据时的行为、列表接口有没有 N+1 查询。

准备一份有代表性数据量的 fixture 库，一次投入长期收益。

## 五、第四道：人类闸

前三道门过了，PR 到你手里。现在你该看什么？

**不看什么比看什么更重要。**语法、格式、空指针、命名、有没有 await——这些前面拦过了，你再看一遍是浪费。

四个问题，按顺序问：

**第一，它做的是我要的吗？**打开 PR 描述和原始需求对照。AI 经常做出一个"技术上正确但解决了另一个问题"的东西。这是最贵的一类错误，因为前三道门全部拦不住——测试是绿的，UI 是能点的，只是做错了东西。

**第二，它有没有顺手改不该改的？**看文件列表。需求只说附件，为什么 `ApprovalPolicy.cs` 在列表里？这个检查一分钟就能做完，收益极高。

**第三，它绕过了什么？**具体看三处：有没有加 `#pragma warning disable`、有没有把某个校验改成可选、有没有在测试里加特判。**agent 遇到阻碍时的第一反应是绕过，不是解决**——这不是它坏，是"让红色变绿"这个目标本身就有两条路。

**第四，三个月后这里会变成什么样？**唯一需要你真正动脑的一条。它加的那个抽象，是会长成一棵好树，还是会变成没人敢碰的一坨？

把这四条写进 PR 模板，`.github/pull_request_template.md`：

```markdown
## 这个 PR 做了什么

<!-- 一句话。如果一句话说不清，考虑拆开 -->

## 关联

- Issue：
- 任务摘要：`.agent/journal/`

## AI 参与情况

- [ ] 主要由 AI 生成
- [ ] AI 辅助，人工大幅修改
- [ ] 纯人工

## 人类 review 检查项

- [ ] 做的确实是需求要的（不只是技术上正确）
- [ ] 文件列表里没有意外的文件
- [ ] 没有 `#pragma warning disable`、没有放松校验、测试里没有特判
- [ ] 引入的抽象三个月后不会变成负担
```

## 六、归属：让 AI 的痕迹留下来

`git blame` 出来一行代码，你想知道当时是怎么想的。如果是人写的，你去问他；如果是 AI 写的，你得知道这一点——因为**没有人能回答"当时怎么想的"，你只能重新判断这段逻辑对不对**。

所以 AI 参与的提交要标出来。用标准的 co-author trailer：

```bash
git commit -m "$(cat <<'EOF'
给 ApprovalPolicy 加 Training 类别

阈值 2000，按 AGENTS.md 补了三个边界用例。

Co-authored-by: Copilot <copilot@users.noreply.github.com>
EOF
)"
```

让它自动发生，写进 `AGENTS.md`：

```markdown
## 提交规范

你生成或大幅修改的代码，提交时必须在 message 末尾加：

    Co-authored-by: Copilot <copilot@users.noreply.github.com>

trailer 前留一个空行。人工从零写的提交不要加。
```

加一道 CI 检查确认没漏：

{% raw %}
```yaml
  attribution:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: PR 标了「主要由 AI 生成」就必须有 co-author
        env:
          BODY: ${{ github.event.pull_request.body }}
        run: |
          if ! echo "$BODY" | grep -q '\[x\] 主要由 AI 生成'; then
            echo "未勾选 AI 生成，跳过"
            exit 0
          fi

          if git log origin/main..HEAD --format='%(trailers:key=Co-authored-by)' | grep -qi copilot; then
            echo "✅ 归属信息完整"
          else
            echo "❌ PR 勾选了「主要由 AI 生成」，但没有任何提交带 Co-authored-by trailer" >&2
            exit 1
          fi
```
{% endraw %}

这件事的收益是滞后的，半年后才看得出来：你能统计出 AI 参与的代码在缺陷率、返工率上和人工代码的差异，能在出事时快速判断"这块要不要整体重看"。**没有归属数据，这些判断只能靠感觉。**

## 七、数据边界：最后一道，也是最容易漏的

前面配的数据库 MCP 有三层防线：只读连接、SQL 白名单、字段脱敏。那个 `Mask` 方法按列名做了个粗暴的字符替换，够用来演示，不够用在真实项目。

问题出在自由文本上。报销单的 `Description` 字段里会有什么？

```
"和张伟、李娜在xx餐厅招待客户，客户联系人王总 13800138000，发票抬头：某某科技有限公司"
```

列名是 `Description`，按列脱敏一个字都盖不住。而这段文本会原样进入模型的上下文。

正确的做法是按**实体类型**而不是按列名识别——姓名、手机号、身份证、公司名，各自识别各自脱敏。[什么是 Presidio](/2026/07/03/what-is-presidio/) 和 [用 .NET 调用 Presidio](/2026/07/10/call-presidio-from-dotnet/) 讲了这套东西怎么搭，[中文 PII 识别](/2026/07/17/presidio-chinese-pii/) 补了中文姓名和身份证的识别器，[做成 LLM 的 PII 网关](/2026/07/24/presidio-llm-pii-gateway/) 则正好是这个场景——把脱敏放在模型调用的必经之路上。

放到 MCP 工具里，就是把 `Mask` 换掉：

```csharp
// 原来：按列名粗暴替换
private static string Mask(string column, string value) => ...

// 换成：按实体类型识别后脱敏
private static async Task<string> Redact(string value) =>
    await _analyzer.AnonymizeAsync(value, new[]
    {
        "PERSON", "PHONE_NUMBER", "ID_CARD", "ORGANIZATION", "EMAIL_ADDRESS"
    });
```

除了脱敏，给 agent 开数据访问还有三条底线：

**永远给独立的只读账号，不要复用应用的连接串。**应用的账号有写权限，`Mode=ReadOnly` 只是客户端行为，换个客户端就绕过去了。真正的边界应该在数据库的权限系统里。

**生产库默认不开。**agent 需要的绝大多数信息，在一个数据形态相似的预发库里就能拿到。真需要生产数据时，走一次性的、有审计的临时授权。

**查询要留痕。**MCP 服务器里记一条日志，谁在什么时候查了什么。出事的时候你需要这个；没出事的时候，看一眼它平时都在查什么，也能发现不少配置问题。

## 总结

四道闸门，五件事：

**第一，每类问题在最便宜的那道门被拦住。**自检信号：翻你最近十条 review 意见，有几条是前三道门本该拦住的。

**第二，机器闸开 `-warnaserror`，AI 评审只查你的红线。**通用 AI 评审给的是通用意见，价值有限；专查红线的评审能发现"改测试让实现绿"这种通用评审看不出的事。

**第三，行为闸的价值是让 agent 自己能看见。**接 Playwright MCP，让它改完前端自己开浏览器验证，验证过的行为顺手沉淀成测试。阈值边界在 UI 上要再测一遍。

**第四，环境闸要有存量数据。**空库测不出来的问题占了一多半。准备一份有代表性数据量的 fixture 库。

**第五，人类闸只看四件事**：做的是不是我要的、有没有顺手改不该改的、绕过了什么、三个月后会变成什么样。其余的都该由前三道门负责。

**外加两条底线**：AI 参与的提交打 co-author，半年后你会需要这份数据；给 agent 开数据访问时，脱敏按实体类型做，只读权限落在数据库而不是连接串上。

后端这条线到这里就闭环了。还剩一块没碰：前端。UI 是 AI 产出质量方差最大的地方——它能在十分钟里给你一个能跑的页面，也能在三个月里给你堆出一坨谁都不敢动的样式。
