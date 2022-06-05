---
layout: post
title:  "ASP.NET Core 的新特性 - 限流中间件"
date:   2022-05-27 21:42:55 +0800--
categories: [.NET Core]
tags: [.NET Core, Rate Limit, 限流]  
---

### 1. 什么是限流
限流就是限制系统的输入和输出流量以达到保护系统的目的。是系统高可用的三大利器之一:
- 限流
- 熔断
- 降级  

![差异](/assets/imgs/limit01.png)

### 2. 限流的使用场景

- 控制系统服务请求的速率
  - 防爬虫
- 特定的业务场景
  - 秒杀
  - 云服务根据并发计费
- 突发的高并发请求
  - DDoS攻击

### 3.限流常见算法

- 计数器
  - 固定窗口（Fixed Window）
  - 滑动窗口（Sliding Window）
- 令牌桶算法 (Token Bucket)
- 漏桶算法 (Leakey Bucket)

#### 3.1 计数器-固定窗口（Fixed Window）
**场景: 假设我们限制每秒只能请求2次**
![固定窗口](/assets/imgs/limit02.png)

虽然看第一个**1秒时间窗口**和第二个**1秒时间窗口**都只是访问了两次, 但是如果把第一个1秒的后500毫秒和第二个1秒的前500毫秒连起来看, 实际的请求次数是4次, 超过了我们的限制，限流失败。

#### 3.2 计数器-滑动窗口（Sliding Window）
![滑动窗口](/assets/imgs/limit03.png)

当第一个**1秒时间窗口**的前500毫秒执行完毕后, 会滑动整个计数窗口, 会把第一个1秒的后500毫秒和第二个1秒的前500毫秒联起来变成**一个1秒窗口**, 所以根据限制, 会抛弃后两次请求, 限流成功了。

#### 3.3 固定窗口 VS 滑动窗口
- 滑动窗口算法部分解决了固定窗口算法d的 “时间临界点” 问题。
- 只要有时间窗口的存在，还是有可能发生时间临界点的问题。
- 更进一步的方案是, 实现多层次限流, 设置多条限流规则。比如 1 分钟 30 个，100ms 2 个。

#### 3.4 令牌桶算法 (Token Bucket)
![令牌桶算法](/assets/imgs/limit04.png)

#### 3.5 漏桶算法 (Leakey Bucket)
![漏桶算法](/assets/imgs/limit05.png)


#### 3.6 令牌桶算法 VS 漏桶算法
**假设有一个接口的QPS为10, 为了保护系统, 我们将接口的每秒请求数限制为10, 然后在1秒内发起11次请求**
- 令牌桶算法
  - 前10次请求成功, 第11请求失败
- 漏桶算法 (Leakey Bucket)
  - 前10次请求成功,第11次请求缓存, 到第二秒的时候, 第11次执行成功
- 总结
  - 漏桶的本质是总量控制，令牌桶的本质是速率控制
  - 令牌桶算法的瓶颈在接口的QPS, 漏桶算法的瓶颈在”桶”(缓存)的大小
  - 漏桶算法对于业务场景突发流量(如:整点打卡签到), 漏桶算法更加适合

### 4. .NET下如何实现限流
- 伴随着.Net 7 Preview 4, 微软发布了限流中间件, 可以实现对http请求的限流
- 之前实现限流通常使用第三方组件,如:
  - AspNetCoreRateLimit
  - FireflySoft.RateLimit.AspNetCore

#### 4.1 如何使用限流中间件
- 安装.NET 7.0 SDK（v7.0.100-preview.4）
- 通过nuget包安装Microsoft.AspNetCore.RateLimiting
- 在Startup.cs文件中实现
  
``` CSharp
app.UseRateLimiter(new RateLimiterOptions
{
    Limiter = PartitionedRateLimiter.Create<HttpContext, string>(resource =>
    {
        return RateLimitPartition.CreateConcurrencyLimiter("MyLimiter",
            _ => new ConcurrencyLimiterOptions(1, QueueProcessingOrder.NewestFirst, 1));
    }),
    DefaultRejectionStatusCode=429
});
```

执行结果为:
![执行结果](/assets/imgs/limit07.png)

再次执行:
![执行结果](/assets/imgs/limit08.png)

### 5. 总结
- 没有实现分布式统一计数
- 没有默认实现对各种限流策略的支持(如:IP, 客户端等)
- 目前更多的是实现了一个限流框架, 用户可以根据不同的需求, 实现不同的Limiter

