---
layout: post
title:  "Azure中的冗余存储选项和策略"
date:   2023-02-03 21:42:55 +0800--
categories: [Azure]
tags: [Rules, Redundant Storage]  
---

### 1. 前言

Azure 存储始终会存储数据的多个副本，以防范各种计划内和计划外的事件，例如暂时性的硬件故障、网络中断或断电、自然灾害等。 冗余可确保即使遇到故障，存储帐户也能达到其可用性和持久性目标。

我们需要在安全性和价格上找到最大的平衡。

![Redundant](https://ssw.com.au/rules/static/b8e687da6193dae51bf6ebd0a3067301/d7854/azure-graphic.jpg)

### 2. 本地冗余存储 (Local-redundant storage, LRS)

- 维护数据的三个副本。
- 在单个区域的单个设施内复制三次。
- 保护您的数据免受正常硬件故障的影响，但不能防止单个设施出现故障。
- 比GRS便宜
  
在以下情况下可以考虑使用：

- 数据的重要性较低 – 例如，对于测试网站或测试虚拟机
- 可以轻松重建数据
- 数据是非关键的
- 数据治理要求将数据限制在单个区域

### 3. 异地冗余存储 (Geo-redundant storage, GRS)

- 创建存储帐户时的默认值。
- 维护数据的六个副本。
- 数据在主要区域中复制三次，在距离主要区域数百英里的次要区域中也复制三次
- 如果主要区域发生故障，Azure 存储将故障转移到次要区域。
- 确保数据在两个单独的区域中持久存在。

在以下情况下使用：

- 数据丢失后无法恢复

### 4. 读取访问异地冗余存储 (Read Access Geo-redundant storage, LRS)

- 将数据复制到辅助地理位置（与 GRS 相同）
- 提供对辅助位置中数据的读取访问权限
- 允许您在某个位置不可用时从主位置或辅助位置访问数据。
  
在以下情况下使用：

- 数据至关重要，需要访问主要区域和次要区域

更多信息可以参照: [Azure 存储冗余](https://learn.microsoft.com/zh-cn/azure/storage/common/storage-redundancy)