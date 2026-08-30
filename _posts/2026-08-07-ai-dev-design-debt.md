---
layout: post
title:  "AI 辅助开发（九）：设计系统与设计债"
date:   2026-08-07 09:30:00 +0800--
categories: [AI]
tags: [AI辅助开发, Github Copilot, 设计系统]
---

## 前言

AI 写前端有个很讨厌的特点：**它单次的产出质量很高，长期的产出质量很低。**

你让它做一个报销单提交页，十分钟，出来的东西能跑、样式不难看、还带了 loading 状态。你很满意。

三个月后，ExpenseFlow 的前端有九个页面，每个页面的"提交"按钮都不一样。有的圆角 6px 有的全圆，有的 `#2563eb` 有的 `#3b82f6`，有的是实心有的是描边。产品经理说想把主色调整一下，你发现要改十几个文件，而且改完没人敢说改全了。

**关键在于，这九个按钮当初每一个都通过了 review。**单看都没毛病。

## 一、为什么 UI 是方差最大的地方

后端代码有三层东西在收敛 AI 的产出：编译器、类型系统、测试。写错了立刻红。

前端有什么？`<div style="color: #3b82f6">` 完全合法，编译通过，页面能跑，截图看着也挺好。**没有任何机制告诉它"这个蓝色不是我们的蓝色"。**

再加上两个放大器：

**第一，前端的"能跑"门槛特别低。**后端一个接口写错了，测试会红；前端一个按钮样式不统一，什么都不会发生。反馈信号缺失，错误就会持续累积。

**第二，AI 有很强的"重新发明"倾向。**你让它加一个筛选面板，它不会先去看现有的 `<Card>` 组件长什么样，它会直接写一个新的 div 加 border 加 shadow——因为这样更快，而且它确实不知道你有 `<Card>`。

![设计债的积累过程](/assets/imgs/ai-dev-design-debt.svg)

## 二、根治的办法是减少选择

处理设计债的常见做法是写规范文档："请使用统一的设计语言"。这基本没用——它太模糊，模型没法执行，你也没法验证。

有效的做法反过来：**不是告诉它该选什么，是让它没得选。**

第一步，把所有视觉决策收进一份 token 文件。`src/ExpenseFlow.Web/src/styles/tokens.css`：

```css
:root {
  /* 颜色：只有这些，没有别的 */
  --color-primary: #2563eb;
  --color-primary-hover: #1d4ed8;
  --color-danger: #dc2626;
  --color-success: #059669;
  --color-warning: #ca8a04;

  --color-text: #334155;
  --color-text-muted: #64748b;
  --color-text-subtle: #94a3b8;

  --color-surface: #ffffff;
  --color-surface-sunken: #f8fafc;
  --color-border: #e2e8f0;

  /* 间距：4 的倍数，只有这六档 */
  --space-1: 4px;
  --space-2: 8px;
  --space-3: 12px;
  --space-4: 16px;
  --space-6: 24px;
  --space-8: 32px;

  /* 圆角：三档 */
  --radius-sm: 4px;
  --radius-md: 6px;
  --radius-lg: 10px;

  /* 字号：五档 */
  --text-xs: 11px;
  --text-sm: 12.5px;
  --text-base: 14px;
  --text-lg: 17px;
  --text-xl: 22px;
}
```

**"只有这六档间距"这句话是整份文件的重点。**不是"推荐使用这些间距"，是"除了这些没有别的"。当 AI 需要一个 20px 的间距时，它必须在 16 和 24 之间选一个，而不是随手写个 20px。

第二步，把组件收口。有了 `<Button>` 组件，规则才有落点：

```tsx
// src/ExpenseFlow.Web/src/components/Button.tsx
import styles from './Button.module.css';

type Variant = 'primary' | 'secondary' | 'danger';

interface Props extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: Variant;
  pending?: boolean;
}

export function Button({ variant = 'primary', pending, children, ...rest }: Props) {
  return (
    <button
      className={`${styles.base} ${styles[variant]}`}
      disabled={pending || rest.disabled}
      {...rest}
    >
      {pending ? '处理中…' : children}
    </button>
  );
}
```

```css
/* Button.module.css —— 注意：没有一个字面量 */
.base {
  padding: var(--space-2) var(--space-4);
  border-radius: var(--radius-md);
  font-size: var(--text-base);
  border: 1px solid transparent;
  cursor: pointer;
}
.base:disabled { opacity: 0.6; cursor: not-allowed; }

.primary   { background: var(--color-primary); color: #fff; }
.primary:hover:not(:disabled) { background: var(--color-primary-hover); }
.secondary { background: transparent; color: var(--color-primary); border-color: var(--color-primary); }
.danger    { background: var(--color-danger); color: #fff; }
```

注意 `pending` 那个 prop。**把"提交中要禁用按钮"这件事做进组件，比在规则文件里写十遍"记得防重复提交"有效得多**——因为它变成了默认行为，AI 想不做都难。

第三步，把这些写进 AI 能读到的地方。`.github/instructions/web.instructions.md`：

```markdown
---
applyTo: "src/ExpenseFlow.Web/**"
---

# 前端规则

## 硬性禁止

- **禁止颜色字面量。**不允许 `#hex`、`rgb()`、`hsl()`，
  一律用 `var(--color-*)`。找不到合适的 token 就来问我，不要自己造。
- **禁止间距和字号字面量。**用 `var(--space-*)` 和 `var(--text-*)`。
  需要的值不在档位里，就选最接近的那一档。
- **禁止行内 `style` 属性。**动态样式用 CSS 变量或 class 切换。
- **禁止新建按钮、输入框、卡片、弹窗。**用 `src/components/` 下已有的。
  确实缺一个新的通用组件时，先告诉我，我们一起定 API。

## 必须

- 表单提交按钮必须传 `pending`，禁止自己实现 loading 状态。
- 金额展示一律 `formatCurrency(amount)`，不要自己 `toFixed(2)`。
- 所有可交互元素必须能键盘操作，按钮必须有可访问的名称。

## 开工前

先 `ls src/components/` 看有什么可用的。**不要凭印象假设组件不存在。**
```

最后那句是有的放矢。**AI 重新发明组件的最大原因是它没去看**，而不是它看了觉得不好用。一句明确的"先 ls 一下"，能挡掉大半。

## 三、把设计约束变成会变红的东西

规则文件仍然是软约束，后端那套经验在这里同样适用：**把约定变成测试。**

前端这块用 [Stylelint](https://stylelint.io/) 最直接。`.stylelintrc.json`：

```json
{
  "rules": {
    "color-no-hex": true,
    "declaration-property-value-disallowed-list": {
      "/^(padding|margin|gap)/": ["/^\\d+px$/"],
      "border-radius": ["/^\\d+px$/"],
      "font-size": ["/^\\d+(\\.\\d+)?px$/"]
    }
  },
  "ignoreFiles": ["**/tokens.css"]
}
```

`ignoreFiles` 里放行 `tokens.css`，因为那是唯一允许出现字面量的地方——**所有真实的值都必须在那一个文件里**。

再补一个脚本，抓 Stylelint 管不到的 TSX 行内样式和 className 里的字面量。`scripts/check-design-tokens.mjs`：

```javascript
import { readFileSync } from 'node:fs';
import { execSync } from 'node:child_process';

const files = execSync('git ls-files "src/ExpenseFlow.Web/src/**/*.tsx"')
  .toString().trim().split('\n').filter(Boolean);

const patterns = [
  { name: '行内 style 属性',   re: /\bstyle=\{\{/ },
  { name: '颜色字面量',        re: /#[0-9a-fA-F]{3,8}\b/ },
  { name: '像素字面量',        re: /:\s*['"`]?\d+px/ },
  { name: '自己写的 toFixed',  re: /\.toFixed\(2\)/ },
];

let failed = false;

for (const file of files) {
  const lines = readFileSync(file, 'utf8').split('\n');
  lines.forEach((line, i) => {
    for (const { name, re } of patterns) {
      if (re.test(line)) {
        console.error(`❌ ${file}:${i + 1}  ${name}`);
        console.error(`   ${line.trim()}`);
        failed = true;
      }
    }
  });
}

if (failed) {
  console.error('\n设计 token 检查未通过。所有视觉值必须来自 tokens.css。');
  process.exit(1);
}
console.log('✅ 设计 token 检查通过');
```

挂进 `package.json` 和 CI：

```json
{
  "scripts": {
    "lint:css": "stylelint \"src/**/*.css\"",
    "lint:tokens": "node scripts/check-design-tokens.mjs",
    "verify": "npm run lint:css && npm run lint:tokens && npx playwright test"
  }
}
```

然后在 `AGENTS.md` 的验证一节里加上：

```markdown
改动 `src/ExpenseFlow.Web/` 下的文件后，除了 `dotnet test`，还要跑：

    npm run verify

全绿才算完成。
```

**这一步做完，设计债的产生速率会直接掉一个数量级。**不是因为 AI 变听话了，是因为它每次违反都会立刻收到红色反馈，然后自己改回来。

第三层是视觉回归。Playwright 自带：

```typescript
import { test, expect } from '@playwright/test';

test('提交页视觉快照', async ({ page }) => {
  await page.goto('/expenses/new');
  await expect(page).toHaveScreenshot('expense-new.png', {
    maxDiffPixelRatio: 0.01,
  });
});
```

视觉回归有个前提：**只有在设计已经收敛之后才值得开。**九种按钮的时候开它，每次 PR 都是一片红，团队三天后就会把它关掉。先收口，再开快照。

## 四、AI 擅长和不擅长的前端活

前端不是"AI 不适合做"，是它的能力分布很不均匀：

| 任务 | 靠谱程度 | 说明 |
|---|---|---|
| 按已有组件拼页面 | ⭐⭐⭐⭐⭐ | 有 token 和组件约束时，质量很稳 |
| 表单校验、状态管理的样板 | ⭐⭐⭐⭐⭐ | 重复且模式清晰，最省人力的一块 |
| 无障碍属性补全 | ⭐⭐⭐⭐ | 它比大多数人记得全 |
| 响应式断点适配 | ⭐⭐⭐⭐ | 常规布局没问题，复杂网格要盯 |
| 从设计稿还原 | ⭐⭐⭐ | 结构对，间距和字重经常差一档 |
| 定义组件 API | ⭐⭐ | 它会做出过度灵活的设计，八个可选 prop |
| 从零做视觉风格 | ⭐⭐ | 出来的东西"像所有 AI 做的网站" |
| 动效与微交互 | ⭐⭐ | 时长和缓动几乎总是要调 |

分界线很清楚：**约束越明确，它越强；需要审美判断的地方，它越弱。**

所以合理的分工是——**人定义系统，AI 使用系统**。token 谁定？人。组件 API 怎么设计？人（或者人主导，AI 提草案）。九个页面按这套系统拼出来？AI，而且它做得比人快十倍还不会手抖。

那行"定义组件 API ⭐⭐"值得展开一句。让 AI 设计一个 `<DataTable>`，它会给你一个有 `sortable`、`filterable`、`selectable`、`expandable`、`virtualized`、`stickyHeader`、`resizable`、`draggable` 八个 prop 的东西，每个都"可能有用"。**这是过度设计，而且是最难还的一种债**——因为它一开始看起来非常专业。

## 五、把设计决策也留档

后端有 `docs/DESIGN.md` 记架构决策，前端同样需要一份，否则半年后没人记得为什么表单是那样布局的。

`docs/DESIGN-UI.md`：

```markdown
# ExpenseFlow 前端设计说明

## 为什么只有三种按钮

primary / secondary / danger。没有 tertiary，没有 ghost，没有 link 样式。

三种覆盖了目前所有场景。**新增一种之前，先确认现有三种真的表达不了**，
而不是因为"这里看着有点重"。按钮种类是设计债最主要的入口。

## 为什么金额一律走 formatCurrency

前端不做金额的舍入判断。后端返回什么，`formatCurrency` 就格式化什么。

历史上出过一次事故：列表页用 `toFixed(2)` 做了四舍五入，
详情页用了 `Math.floor`，同一张单子在两个页面显示差 1 分钱，财务对不上账。

## 为什么阈值提示要请求后端

新建页在用户输入金额时会提示「这张单需要二级审批」。这个判断**必须调后端**
`POST /expenses/preview`，不允许前端复制一份阈值表。

前端复制业务规则，等于同一个规则有两个实现，迟早不一致。
多一次网络请求，换规则的唯一真相来源。

## 非目标

- 不做暗色主题。（token 结构预留了，但现在不做）
- 不做移动端适配。报销在 PC 上填，手机只看审批通知。
- 不做富文本说明。说明字段是纯文本，避免 XSS 和 PII 检测的复杂度。
```

和后端那份的用法一样：**写清"为什么"和"什么情况下这个决定可以推翻"**，agent 才知道什么能动什么不能动。

## 六、已经欠了债怎么还

上面都是防新债。存量呢？

好消息是，**还设计债恰好是 AI 最擅长的活之一**——机械、重复、有明确的验收标准。

分三步。

**第一步，扫出来。**先搞清楚欠了多少：

```bash
# 代码里到底用了多少种颜色
git ls-files 'src/ExpenseFlow.Web/src/**/*.{css,tsx}' \
  | xargs grep -ohE '#[0-9a-fA-F]{3,8}\b' \
  | sort | uniq -c | sort -rn

# 多少种间距
git ls-files 'src/ExpenseFlow.Web/src/**/*.css' \
  | xargs grep -ohE '(padding|margin|gap)[^:]*:\s*[^;]+' \
  | grep -oE '\b\d+px' | sort | uniq -c | sort -rn
```

数字通常很难看。见过一个中等规模的项目扫出 43 种灰色。

**第二步，让 AI 做归类，人做决策。**

```
> 上面是当前代码里用到的所有颜色和出现次数。把它们按视觉相近程度分组，
  每组给出建议保留的那一个，以及这一组总共出现多少次。
  只出报告，不要改代码。
```

它做聚类做得很好。**但哪一组保留哪个值，必须人来定**——这是审美判断，而且一旦定错，后面几百处替换全要重来。

**第三步，分批替换，一次一类。**

```
> 把 src/ExpenseFlow.Web/ 下所有 #3b82f6、#1d4ed8、#2563eb 替换成
  var(--color-primary)。
  只改颜色，不要动布局、不要动组件结构、不要顺手重构。
  改完跑 npm run lint:css，然后 npx playwright test --update-snapshots=none，
  把视觉 diff 报告给我看。
```

三个要点：

- **一次只还一类债**（颜色一批、间距一批、组件收口一批）。混着改，视觉回归的 diff 就没法看了。
- **"不要顺手重构"必须写。**否则你会得到一个既换了颜色又重排了 DOM 结构的 diff。
- **视觉 diff 是这里的验收标准。**颜色替换后如果某个页面的截图变了，说明那处原本就用错了颜色——这正是你要找的东西，人看一眼确认。

按这个节奏，几百处的设计债一两天能还完，而且过程可控。**这大概是 AI 辅助开发里投入产出比最高的一类任务**：纯机械、量大、人做会疯、验收标准还特别明确。

## 总结

前端这一环，五件事：

**第一，UI 是方差最大的地方，因为没有编译器帮你收敛。**再加上 AI 有强烈的"重新发明"倾向——它不会先去看你有没有 `<Card>`。

**第二，根治靠减少选择，不靠写规范。**六档间距、三档圆角、一份颜色 token，"除了这些没有别的"。组件层面把 `pending` 这类行为做成默认，比在规则里写十遍有效。

**第三，把设计约束变成会变红的东西。**Stylelint 禁 hex 和 px 字面量，加一个脚本抓 TSX 里的漏网之鱼，挂进 `npm run verify`。视觉回归要等设计收敛之后再开。

**第四，人定义系统，AI 使用系统。**它按现成的组件拼页面很强，定义组件 API 很弱——会给你八个可选 prop 的过度设计。

**第五，还存量债正是 AI 最擅长的活。**扫描 → AI 归类人决策 → 一次只还一类，用视觉 diff 验收。

---

六个环节到这里全部走完了：工具与环境、上下文与记忆、节奏与护栏、能力扩展、验证闸门、团队规模化。

回头看，会发现这套东西有个共同的形状：**每一环都在做同一件事——把人脑子里的隐性知识，变成 agent 能读到、并且违反了会变红的东西。**

`AGENTS.md` 是把规矩写下来，`ConventionTests` 是让规矩会变红。`DESIGN.md` 是把决策写下来，PR 红线检查是让违反决策会变红。tokens.css 是把视觉决策写下来，Stylelint 是让它会变红。

工具会换，模型会更新，但这个形状不会变。哪天 Copilot 换成了别的东西，你的 `AGENTS.md`、你的约定测试、你的 token 文件，一行都不用改。

不妨现在回去重做一遍那份 18 题的自测表，看看分数动了多少。
