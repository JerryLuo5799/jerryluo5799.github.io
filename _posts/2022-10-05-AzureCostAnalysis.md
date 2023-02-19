---
layout: post
title:  "使用Azure Cost Analysis来减少成本"
date:   2022-10-15 21:42:55 +0800--
categories: [Azure]
tags: [Cost Analysis, Azure成本分析]  
---

### 1. 前言

Azure 成本分析提供任何 Azure 支出来源的详细细分。它通过以下维度来分析成本：

- 范围区域，例如订阅
- 资源组，例如 Northwind.Website
- 位置，例如澳大利亚东部
- 服务类型，例如 Azure App Service

```Text
注意：您还可以“过滤”这些内容中的任何一个，以缩小视图范围
```

### 2. 分析出最贵的成本项

若要优化支出，请分析每个类别中的主要成本。一般来说，关注成本支出的前3名是个好主意 - 超出此范围的优化通常不值得付出努力。

要问的关键问题：

- 您需要该资源吗？
- 你能缩小规模吗？
- 是否可以重构应用程序以减少消耗？
- 您可以更改服务类型或消费模式吗？

### 3. 统计选定区域内的累计成本

通过选定区域, 计算一定时间段内的累计成本，例如去年的订阅成本，可识别成本的峰值。

![累计成本](https://ssw.com.au/rules/static/123f1deacbed52cc8365c6a285ba1e2f/72e01/area-chart.jpg)

### 4. 资源组维度

通过选定区域,按资源组维度计算成本，可以查看最昂贵的资源组并尝试减少它。忽略微小的。

![资源组维度](https://ssw.com.au/rules/static/b6885959d213bc724bd243b73d1e02ba/18815/resource-groups.jpg)

### 5. 位置维度

通过选定区域,按资源所在位置维度计算成本，可以帮助确定其中一个位置的成本是否高于其他位置。

![位置维度](https://ssw.com.au/rules/static/8716991f073415dd8a6ee0472cf7778e/b6e7a/locations.jpg)

### 6. 服务类型维度

使用的每项服务的成本，例如Azure App Service。

如果特定服务花费大量资金，请考虑是否有可能更适合的服务，或者该服务是否可以调整其消费模型以更好地适应使用级别。

![服务类型维度](https://ssw.com.au/rules/static/1da6bb5e2aadda638e676dcd28836658/a2d2e/services.jpg)

### 7. 特定资源维度

Azure 成本分析工具还允许选择不同的视图。如果您认为特定资源导致了问题，请选择“CostByResource”视图，然后您可以查看正在运行的资源的各个方面。

![特定资源维度](https://ssw.com.au/rules/static/f94756119620b1b4322fcaf0d65ba379/0f98f/service-breakdown.jpg )