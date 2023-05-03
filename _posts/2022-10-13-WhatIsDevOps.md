---
layout: post
title:  "Azure DevOps探索系列前言: 什么是DevOps"
date:   2022-10-13 21:42:55 +0800--
categories: [Azure DevOps]
tags: [Azure DevOps探索系列]  
---

<iframe src="//player.bilibili.com/player.html?aid=817694408&bvid=BV1oG4y1x7kN&cid=893387629&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" class="bilibili"> </iframe>

链接地址: [https://www.bilibili.com/video/BV1oG4y1x7kN/](https://www.bilibili.com/video/BV1oG4y1x7kN/)


### 1. 背景

我们知道，一个软件项目从零开始到最终交付，大体包括了规划、编码、构建、测试、部署和维护这几个阶段。

早期软件团队一般采用瀑布模型作为的软件交付模型, 瀑布模型的核心定义是, 软件开发是一个阶段化的精确的过程， 因此， 在瀑布模型中， 各团队会按部就班的工作, 一个阶段所有工作完成之后，再进入下一个阶段。

### 2. 瀑布模型

#### 2.1 瀑布模型

![瀑布模型](/assets/imgs/waterfall01.png)

- 阶段间有明显的界线
- 按照顺序依次进行
- 以文档为驱动

#### 2.2 瀑布V模型

![瀑布V模型](/assets/imgs/waterfall02.png)

- 单元测试：是否满足详细设计的要求
- 集成测试：验证已测试过的部分是否可以很好地结合在一起
- 系统测试：检验系统功能、性能是否达到系统的要求。

#### 2.3 瀑布VV模型

![瀑布V模型](/assets/imgs/waterfall03.png)

- 测试与开发并行

#### 2.4 瀑布模型的缺点

- 单一流程，不可逆
  - 后续流程会不断放大需求分析阶段的偏差
- 难以响应变化
  - 每个阶段都强依赖于上个阶段的结果
- 交付周期过长
  - 用户到最后才能确定产品是否符合要求

### 3. 敏捷开发 (Agile)

随着软件的规模不断的变大, 软件的复杂度也在不断的攀升。用户的需求在不断增加和变更的同时，软件团队的开发时间却越来越少。于是，软件行业引入了一个新的概念，那就是现在非常流行的敏捷开发。

#### 3.1 什么是敏捷开发 (Agile)

所谓的敏捷开发， 简而言之就是将大项目拆分成N个小项目，把项目总体时间切割成固定长度的时间段， 不断的迭代交付小项目，这样的好处是大幅提高了开发团队的工作效率，让版本的更新速度变得更快，也可以更快的响应用户的需求的变更。

#### 3.2 敏捷模型

敏捷开发本质上是一种思想, 具体的实践需要根据实际情况选取不同的模型, 例如:

- Scrum
- 极限编程 (XP)
- LeSS（Large Scale Scrum）

#### 3.3 敏捷开发 (Agile) VS 瀑布模型

![敏捷开发 (Agile) VS 瀑布模型](/assets/imgs/DevOps01.png)

#### 3.4 敏捷开发带来的问题

大家知道, 开发团队的核心目标是不断的根据要求, 上线新的需求变更, 这样就造成了应用的不断变化。而对于运维团队来说，保持应用稳定的运行， 所以本能上会相对的抗拒变更, 因为变更就意味着不稳定的风险，在敏捷开发快速迭代的背景下，团队之前的冲突越显激烈，在此基础上， DevOps应运而生。

### 4. DevOps

#### 4.1 什么是DevOps

![什么是DevOps](/assets/imgs/DevOps02.png)
DevOps其实就是Development和Operations两个单词的组合。DevOps是一组过程、方法与系统的统称，用于促进开发应用程序或软件工程、技术运营和质量保障QA部门之间的沟通、协作与整合。

#### 4.2 DevOps核心目标和实现方式

DevOps核心目标是: **可持续的又快又好的交付价值**

- “可持续性”是比较稳定的节奏长期的持续进行交付
- “快”是指研发效能提升，具备按时可预测交付的能力
- “好”是指研发质量提升，减少缺陷和返工，降低成本，提升用户体验。
- “交付价值”是指通过持续的频繁的迭代交付可用的产品

DevOps的实现方式是:

- 流程标准化
- 工作自动化

#### 4.3 Scrum + DevOps 工作流程

![Scrum + DevOps](/assets/imgs/DevOps03.png)

#### 4.4 DevOps工具集

![Scrum + DevOps](/assets/imgs/DevOps04.png)

