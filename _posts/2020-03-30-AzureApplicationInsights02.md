---
layout: post
title:  "使用Azure Application Insights监测你的应用 (二) - 主要功能服务介绍"
date:   2020-03-30 21:42:55 +0800--
categories: [Azure]
tags: [Application Insights, 监控平台, 日志系统]  
---

### 前言
在所有的监测系统中， 如何记录日志其实都是大同小异的, Azure Application Insights提供了各种强大的功能服务来帮助用户监测应用和定位问题, 而这些功能服务,才是其最有价值的部分。

### Application Map
![Application Map](https://docs.azure.cn/zh-cn/azure-monitor/app/media/app-map/app-map-001.png)

如图所示, Application Map可以根据所有启用Application Insights的应用, 自动生成应用程序拓扑, 根据这个, 技术人员可以:
1. 清晰直观的看到所有应用的依赖情况
2. 快速定位定位问题所在的应用
3. 方便的进行性能问题的排查, 直观的看到性能问题是由于网络原因还是因为接口响应

### 用户使用情况分析
#### 用户统计
“用户和会话”报告按页面或自定义事件筛选数据，并按位置、环境和页面等属性将数据分段。 也可以添加自己的筛选器。

![Application Map](https://docs.azure.cn/zh-cn/azure-monitor/app/media/usage-overview/users.png)

#### 用户回头率

![Application Map](https://docs.azure.cn/zh-cn/azure-monitor/app/media/usage-overview/retention.png)

#### 用户导航
该工具可视化显示用户在网站的页面和功能之间导航的方式。 它非常适合解释以下问题，例如：
1. 用户如何从网站上的某个页面进行导航？
2. 用户在网站页面上单击了哪些内容？
3. 网站中用户流失最多的地方在哪里？
4. 是否存在用户反复重复同一操作的位置？

![Application Map](https://docs.azure.cn/zh-cn/azure-monitor/app/media/usage-flows/00001-flows.png)