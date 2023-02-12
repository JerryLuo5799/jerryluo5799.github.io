---
layout: post
title:  "Azure DevOps探索系列之一: 使用Backlogs规划和管理需求"
date:   2022-11-10 21:42:55 +0800--
categories: [Azure DevOps]
tags: [Azure DevOps探索系列]  
---

### 1. 前言

在之前的博客中, 我们提到了DevOps的产生和敏捷的兴起有直接的关系, 同时, 我们也简单介绍了一个Scrum + DevOps的工作模型, 今天我们结合Azure DevOps的Backlogs,来聊一聊工作模型的第一步, 需求管理 。

` ** 在Azure DevOps中, 需求管理的实质是: 对Backlogs中的工作项以及工作项排序的管理** `

![Scrum + DevOps](/assets/imgs/DevOps03.png)

### 2. Backlogs功能介绍

首先我们先登录进Azure DevOps, 可以看到我们已经创建好了一个项目: **Hello FireUG**

![HomePage](/assets/imgs/ADT01-02.png)

点击进入项目**Hello FireUG**后, 通过左边菜单 **Boards | Backlogs** 可以看到以下界面:

![Backlogs](/assets/imgs/ADT01-01.png)

#### 2.1 什么是Backlogs

Backlogs, 一般翻译为积压工作, 通俗的说就是需求池, 产品负责人(PO) 需要把需要待做的工作拆解成对应的工作项 (Wrok Item), 在Azure DevOps中, 主要有三种类型的工作项:

- Epic (长篇故事)
- Feature (特性)
- Product Backlog Item (简称PBI, 产品积压工作项)

三者直接的结构, 通常来说是属于从属关系, 如 **长篇故事 --> 特性 --> PBI**, 在Azure DevOps中, 我们可以非常自由的设置工作项从属关系, 如 **长篇故事 --> PBI**, **PBI --> PBI** 等。

#### 2.2 什么是Epic (长篇故事)

长篇故事一般指的是远期的目标， 比如需要完成XXX系统的建设， 完成XXX大版本的升级等， 在上图中，FireUG技术社区v1.0，FireUG技术社区v2.0, 2023年度视频周更计划 三项内容都是长篇故事。

#### 2.3 什么是Feature (特性)

特性一般指的是长期故事中的里程碑， 或者某个子系统， 比如上述的FireUG技术社区v1.0，包含了官网建设，线上直播活动等，2023年度的周更计划里面包含的2月和3月的周更内容等。

#### 2.4 什么是Product Backlog Item (产品积压工作项)

PBI一般代表的是需要团队成员完成的需求，主要内容会包含标题、内容描述、验收标准等，由于PBI一般是由客户或PO来写，所以体现的是基于用户本身的痛点和真实的需求，这样避免了最终的结果不是客户想要的尴尬局面。如上图中，明确标明了2023年2月需要周更的视频主题。

### 3. 积压工作列表

通常来说，普通团队成员只需要关注积压工作列表即可，我们可以通过右上角的工作项内容选择，将当前视图切换成积压工作列表。

- 我们可以通过上下拖动工作项, 调整工作项的排序。
- 我们可以通过页面上的 **视图选项 (View Option)**, 隐藏相关联的长篇故事以及特性的显示。

![PBI-List](/assets/imgs/ADT01-03.png)

- 我们可以通过**列选项 (Column Options)**来设置需要显示的项。

![PBI-List](/assets/imgs/ADT01-04.png)

### 4. 白板视图

我们可以点击**列选项 (Column Options)**旁边的**白板视图选项（View as Board）**，切换工作项以白板模式显示， 实际上是切换到了 **Boards | Board** 菜单，在该视图下， 我们可以方便的通过拖动的方式，更改工作项的状态。

![PBI-List](/assets/imgs/ADT01-05.png)

![PBI-List](/assets/imgs/ADT01-06.png)
