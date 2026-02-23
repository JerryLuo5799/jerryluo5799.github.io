---
layout: post
title:  "打破.NET依赖注入的“死锁”：循环依赖的4种解决方案"
date:   2025-05-01 12:30:00 +0800--
categories: [.NET]
tags: [DI]
---

## 前言

在进行复杂的业务系统开发时，你迟早会遇到那个让程序崩溃的报错：`AggregateException: A circular dependency was detected...`。

这意味着你的代码陷入了“鸡生蛋，蛋生鸡”的怪圈：`ServiceA` 构造函数里需要 `ServiceB`，而 `ServiceB` 的构造函数又在等 `ServiceA`。DI 容器在初始化时就像被关进了无限循环的迷宫，最终只能选择“罢工”。

这通常是系统架构向你发出的**求救信号**。今天，我们不仅要学会如何“通过代码”修好它，更要学会如何“通过设计”规避它。

## 核心原理：为什么 DI 容器会“崩溃”？

.NET 默认的依赖注入容器（Microsoft.Extensions.DependencyInjection）采用的是**构造函数注入（Constructor Injection）**。

当容器尝试创建 `ServiceA` 时，它会检查构造函数参数：

1. 发现需要 `ServiceB`。
2. 于是转身去创建 `ServiceB`。
3. 发现 `ServiceB` 的构造函数又需要 `ServiceA`。
4. 容器意识到 `ServiceA` 还没创建完呢！

为了防止由于无限递归导致的内存溢出（StackOverflow），容器会主动抛出异常。

## 实战演示：破解循环依赖的四种招式

我们以“订单系统（OrderService）”与“库存系统（StockService）”互相调用为例，拆解不同的解决方案。

### 招式一：逻辑重构——提取“第三者”（金牌方案）

**原理**：循环依赖通常说明两个类的职责划分模糊。通过将共同依赖的逻辑提取到 **Service C** 中，将“环形依赖”解耦为“树形依赖”。

* **场景**：`OrderService` 创建订单时要预扣库存，`StockService` 库存不足时要通知订单系统取消订单。
* **重构**：提取一个 `InventoryCoordinator`（库存协调器）专门处理这两者的交互逻辑。

```csharp
// 新提取的中间层，只负责协调
public class InventoryCoordinator(IOrderService orderService, IStockService stockService) 
{
    public async Task HandleStockShortage(int productId) 
    {
        // 协调逻辑...
    }
}

```

| 优点 | 缺点 |
| --- | --- |
| 符合单一职责原则（SRP），代码结构最健康。 | 需要较大的重构工作量。 |

### 招式二：`Lazy<T>` 延迟加载（快速补救）

**原理**：告诉 DI 容器：“现在先别急着创建对方，等我真正用到它的时候，你再通过工厂方法帮我生成。”

```csharp
// 使用 C# 12 核心构造函数语法
public class OrderService(Lazy<IStockService> lazyStockService) : IOrderService
{
    public void CreateOrder()
    {
        // 只有在访问 .Value 时，StockService 才会真正被实例化
        lazyStockService.Value.ReserveStock();
    }
}

// 在 Program.cs 中注册 Lazy 解析支持
builder.Services.AddTransient(typeof(Lazy<>), typeof(LazyService<>));

public class LazyService<T>(IServiceProvider sp) : Lazy<T>(sp.GetRequiredService<T>) where T : class;

```

| 优点 | 缺点 |
| --- | --- |
| 改动极小，能快速解决报错。 | 掩盖了设计问题；如果在构造阶段就访问 .Value，依然会报错。 |

### 招式三：使用委托/函数注入（轻量化方案）

**原理**：与其注入整个服务，不如只注入一个返回该服务的**委托（Delegate）**。这在不需要使用对方全部功能，只需要调用某个特定方法时非常有效。

```csharp
public class OrderService(Func<IStockService> stockServiceFactory) : IOrderService
{
    public void Process()
    {
        var stockService = stockServiceFactory(); // 按需获取
        stockService.Check();
    }
}

// 注册
builder.Services.AddScoped<IStockService, StockService>();
builder.Services.AddScoped<Func<IStockService>>(sp => sp.GetRequiredService<IStockService>);

```

| 优点 | 缺点 |
| --- | --- |
| 比 Lazy 更显式，且不需要额外的辅助类。 | 注册代码稍显繁琐。 |

### 招式四：MediatR 中介者模式（终极解耦）

**原理**：**“不要互相说话，通过中台传话。”** Service A 发出一个“信号（Event/Command）”，Service B 监听并处理。两者在代码层面完全不认识对方。

```csharp
public class OrderService(IMediator mediator) 
{
    public async Task PlaceOrder() 
    {
        // 业务逻辑...
        await mediator.Publish(new StockReduceEvent(ProductId: 1)); // 只发消息，不关心谁处理
    }
}

// StockService 只需要实现处理接口，无需注入 OrderService
public class StockHandler : INotificationHandler<StockReduceEvent> { ... }

```

| 优点 | 缺点 |
| --- | --- |
| 极致解耦，非常适合复杂的微服务或大型项目。 | 增加了理解成本，无法直接通过代码跳转找到调用者。 |

## 深度思考：如何选择最优方案？

作为一名追求卓越的开发者，选择方案时可以参考这个优先级：

1. **逻辑重构（方案一）**：如果是新项目或核心业务，优先考虑提取中间层。
2. **中介者模式（方案四）**：如果系统规模庞大，且模块间交互频繁，直接引入 MediatR。
3. **委托注入（方案三）**：如果只是两个服务间偶尔的互相调用，且不想引入额外的库。
4. **Lazy/IServiceProvider**：仅用于修复无法大规模重构的老旧代码（Legacy Code）。

> **避坑指南**：有些开发者喜欢直接注入 `IServiceProvider` 然后在方法内部 `GetRequiredService`。虽然能解决循环依赖，但这是典型的 **Service Locator（服务定位器）反模式**，会让单元测试变得极其困难，请谨慎使用。

## 总结

循环依赖不是系统的“ Bug”，它是架构在提示你：**这两者的逻辑太粘稠了，该分家了。**

* **重构**是治本，**延迟加载**是治标，**中介者**是改变沟通方式。

**官方参考链接：**

* [.NET 依赖注入设计原则与规约 - Microsoft.com](https://learn.microsoft.com/zh-cn/dotnet/core/extensions/dependency-injection/guidelines?wt.mc_id=MVP_324329)
* [在 .NET 中使用第三方容器解决循环依赖 - Github.com](https://github.com/autofac/Autofac?wt.mc_id=MVP_324329)
