---
layout: post
title:  "Azure DevOps探索系列之三: 一个好的PBI需要包含哪些元素"
date:   2023-03-02 21:42:55 +0800--
categories: [Azure DevOps]
tags: [Azure DevOps探索系列]  
---

<iframe src="//player.bilibili.com/player.html?aid=311396064&bvid=BV1yP411o7kV&cid=1066573911&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" class="bilibili"> </iframe>

链接地址: [https://www.bilibili.com/video/BV1yP411o7kV/](https://www.bilibili.com/video/BV1yP411o7kV/)

### 1. 前言

在[Azure DevOps探索系列之一: 一个好的PBI标题需要包含哪些元素](/2022/12/10/AzureDevOpsBacklogs-WhatShouldPBIInclude/)中, 我们介绍了如何编写一个好的PBI标题, 那么一个好的PBI需要包含哪些内容呢?

![PBI](/assets/imgs/AzureDevOpsPBI01.png)

### 2. 描述

描述是PBI的重要组成部分, 目的是明确需求的目标和期望达成的效果, 所以：

- 需要尽可能具体和完整的列出PBI涉及的功能和特性, 以减少团队成员的理解偏差
- 也需要提供与PBI相关的附加信息, 如参考资料、示例 、原型等, 以帮助团队更好的理解需求
- 最后，对于一些特定类型的PBI，还需要提供非功能性需求描述，例如：性能指标、相关约束条件等

以当前的PBI为例，在描述中，我们包含了Bug复现方式，URL地址、页面截图等信息, 团队成员可以非常直观的了解PBI所描述的内容, 也可以直接点击链接, 便捷的复现问题, 这样就可以很好的减少团队间的沟通成本。

### 3. 验收条件

验收条件可以理解为“完成的定义”（Definition of Done）。很多时候，我们会因为一个开发工程师完成编码后没有后续跟进系统的运行情况，而判断他没有责任心，不是一个好的开发。但其实这只是因为团队所有成员对工作完成的定义不一样而已。所以在编写PBI的时候，我们必须要明确什么情况下，这个PBI才算是真正的完成。

验收标准没有一个统一的定义标准，在我们实际工作过程中, 会存在多种可能，比如：

- PBI是编写文档，验收条件可以是 “文档编写完成”
- PBI与用户需求有关，验收条件可以是 “发布到正式环境并且客户确认通过”
- 非功能性的PBI，验收条件可以是“运行一段时间后的监控结果”

### 4. 相关工作

相关工作指的是当前PBI和其他PBI的依赖关系，并不是每个PBI都会存在依赖关系，因为在很多情况下，PBI都是比较独立的, 并不存在相关工作。

我们点击相关工作, 可以看到, 在Azure DevOps中,我们可以比较自由的设置PBI的前置任务, 后置任务， 父级，子级等关联类型, 大家可以根据实际情况进行设置。

### 5. 其他

除了以上四点, 需求还可以包含更多的信息, 比如优先级和标签(标记)。 在我们团队中, 通常使用PBI的排序代替优先级的作用, 另外, 我也建议大家给PBI加上对应的标签(标记), 方便后续的检索。
![PBI](/assets/imgs/AzureDevOpsPBI02.png)