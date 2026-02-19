---
layout: post
title:  "C# 15 预览：集合表达式补全最后一块拼图，性能与优雅兼得"
date:   2026-02-19 15:30:00 +0800--
categories: [.NET]
tags: [C#15]  
---

### 前言

在 C# 12 中，**集合表达式 (Collection Expressions)** 的引入彻底改变了我们编写集合代码的方式。简单的 `[]` 语法不仅让代码更加清爽，还统一了不同集合类型的初始化。然而，对于追求性能和灵活性的初级开发者来说，它一直存在一个令人遗憾的局限：**无法直接向集合的构造函数传递参数**。

今天，我们将深入探讨 C# 15 正在酝酿的首个重磅特性——**集合表达式参数 (Collection Expression Arguments)**，看它如何解决高性能场景下的初始化痛点。

### 核心概念

在目前的 .NET 8/9 环境下，当你写下 `List<int> list = [1, 2, 3];` 时，编译器会自动处理集合的创建。但在生产级开发中，我们往往需要更精细的控制：

1. **Capacity (容量预分配)**：为了避免动态扩容导致的频繁内存申请与数据拷贝，我们需要提前指定 `Capacity`。
2. **Behavior (行为定制)**：例如 `HashSet` 或 `Dictionary` 需要指定 `IEqualityComparer` 来决定去重或匹配逻辑（如忽略大小写的字符串比较）。

在 C# 15 之前，如果你需要这些控制，就必须放弃 `[]` 语法回归传统的 `new List<int>(100)`。C# 15 的新特性正是为了打破这种局限。

### 实战演示

#### 1. 预设容量以优化性能

假设我们需要合并多个集合。为了性能最优，我们希望一次性分配好所有空间。

**C# 15 预览语法：**
通过新增的 `args` 关键字（暂定语法），我们可以直接在括号内传递构造参数。

```csharp
var numbers = new List<int> { 1, 2, 3 };
var squares = new List<int> { 1, 4, 9 };

// 使用 args 指定构造函数参数 capacity
// 即使使用集合表达式，也能避免底层数组的多次 Resize
List<int> allItems = [args: capacity: 10, .. numbers, .. squares]; 

#### 2. 定制集合行为 (以 HashSet 为例)

在处理字符串集合时，忽略大小写的去重是一个高频需求。

**C# 15 预览语法：**

```csharp
// 直接在集合表达式中注入 StringComparer
HashSet<string> uniqueNames = [
    args: comparer: StringComparer.OrdinalIgnoreCase, 
    "Alice", 
    "ALICE", 
    "Bob"
];

// 结果：集合中仅包含 "Alice" 和 "Bob"

```

### 深度思考

从工程实战和底层原理来看，这个特性的引入具有重要的进阶意义：

* **减少 GC 压力 (Reduce GC Pressure)**：
.NET 集合的动态扩容机制（通常是容量翻倍）在处理大数据量时会产生大量临时对象。通过 `args: capacity` 显式预分配，可以**显著减少内存分配次数和垃圾回收频率**，这在高性能 API 开发中至关重要。
* **代码声明式与性能的统一**：
现代 .NET 提倡声明式编程。过去，我们常在“代码优雅”和“极致性能”之间纠结。新特性让初级开发者也能在保持代码简洁的同时，遵循**“预分配原则”**这一进阶开发准则。
* **避坑指南**：
虽然 `args` 很方便，但切忌**过度优化**。如果集合元素数量是确定的且很小（如 10 个以内），手动指定 `capacity` 带来的收益微乎其微，反而增加了代码维护成本。建议仅在**大数据量合并**或**性能敏感的热点路径**上使用。

### 结语

C# 15 的集合表达式参数特性，补全了 `[]` 语法的最后一块拼图。它告诉我们，C# 语言的进化始终围绕着“降低门槛”与“保留上限”展开：让代码既能写得简单，又能跑得飞快。

参考链接: [https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-15?wt.mc_id=MVP_324329)

**思考题：**
除了 `List` 和 `HashSet`，你觉得在 `Dictionary` 的集合表达式中，这个 `args` 参数最应该用来传递什么？

