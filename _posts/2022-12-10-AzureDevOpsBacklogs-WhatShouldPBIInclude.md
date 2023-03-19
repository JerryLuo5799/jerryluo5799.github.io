---
layout: post
title:  "Azure DevOps探索系列之二: 一个好的PBI标题需要包含哪些元素 (一)"
date:   2022-12-10 21:42:55 +0800--
categories: [Azure DevOps]
tags: [Azure DevOps探索系列]  
---

### 1. 前言

在[Azure DevOps探索系列之一: 使用Backlogs规划和管理需求](/2022/11/10/AzureDevOpsBacklogs/)中, 我们简单了解Backlogs列表的使用, 在实际工作过程中, Backlogs是项目运行流畅的基石。团队成员在这里可以跟踪功能、错误、任务等。所以， 当团队成员查看Backlogs的时候，标题是否一目了解是非常重要的，他们需要可以通过PBI的标题就能够对PBI有一个很好的理解。

### 2. 什么是好的标题

如果PBI的标题没有实际意义, 只是一个非常宽泛的描述的话, 团队成员就不得不打开详情页面才能了解具体的内容, 这样会花费更多的时间, 更不好的一点是, 团队成员下次再进入Backlogs的时候,很可能会忘记之前的内容, 那就需要重新一条条打开才行。

- ❌不好的示例

```text
修复菜单显示异常
```

无法清晰的得知, 具体是一个什么问题, 在什么情况下会出现？这有多重要？


- ✅好的示例

```text
🐛首页 | 菜单在Safari下不显示🔥 
```

🐛表示这是一个bug, 🔥表示非常紧急, 同时非常清晰的描述了问题

#### 2.1 需要避免的问题

- ❌不要取一些非常宽泛的标题, 例如: 修改网站bug
- ❌不要用一些难以理解的黑话, 或者不要写的太过啰嗦

#### 2.2 需要包含的元素

- ✅固定格式并包含更具体的信息, 如: [模块名称] \| [需求描述]
- ✅对于紧急需求,需要加上标识,如: 🔥
- ✅标识Bug, 可以加🐛或Bug前缀, 一般来说Bug需要优先解决
- ✅在标题上加emojis, 例如:
  
```
💄 UI
📃 文档
🐛 Bug
✨ 新的功能
🔍 调研
📈 报告
⚒️ DevOps
♻️ 重构
```
更多emojis可以参考: [https://gitmoji.dev](https://gitmoji.dev)

### 3. 一些标题示例

可以使用以下结构: [emojis] [业务、模块等] \| [简短的描述]

Bugs:

```

🐛 Newsletter form | returns HTTP 500

```

新功能:

```

✨ Newsletter form | Validate email address

```

UI样式:

```

💄 Header | Update site header with new logo

```

DevOps基础设施:

```

👷‍♂️ DevOps | Add ephemeral deployment slots for PRs

```

紧急任务:

```

👷‍♂️ SysAdmin | Northwind app inaccessible through company VPN🔥

```

其他示例:

```

🐛 Invoices | Invoice totals are rounded incorrectly

⚒️ Infrastructure | Implement staging deployment pipeline

✨ Clients | Add create/edit client page 

```
