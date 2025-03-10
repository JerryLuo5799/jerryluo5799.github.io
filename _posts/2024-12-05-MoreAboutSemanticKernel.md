---
layout: post
title:  "Semantic Kernel 入门系列之二: 深入理解Semantic Kernel相关概念"
date:   2024-12-05 21:42:55 +0800--
categories: [.NET]
tags: [Semantic Kernel]  
---

<iframe src="//player.bilibili.com/player.html?isOutside=true&aid=113599452288090&bvid=BV1uviRYNEdf&cid=27191545044&p=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" class="bilibili"></iframe>

链接地址: [https://www.bilibili.com/video/BV1uviRYNEdf/](https://www.bilibili.com/video/BV1uviRYNEdf/)

### 1. 前言  

在上一篇文章中，我们简单介绍了 Semantic Kernel（简称 SK）的基本概念。作为微软推出的 AI 集成框架，SK 通过协调大模型（如 GPT）与自定义功能，帮助开发者快速构建智能应用。本文将进一步深入解析 SK 的四个核心概念：**内核（Kernel）**、**插件（Plugins）**、**函数调用（Function Calling）** 和 **规划（Planning）**，并通过实例演示其应用场景。  

### 2. 内核（Kernel）：AI 应用的“大脑”  

内核是 SK 的核心组件，扮演着 **依赖注入容器** 的角色。它的核心职责是统一管理 AI 应用所需的所有服务和插件，并协调它们的生命周期。  

#### 2.1 内核的工作原理  

当用户向应用发送请求时，内核会完成以下关键步骤：  

1. **选择 AI 服务**：根据配置自动选择合适的大模型（如 OpenAI、Azure AI 等）。  
2. **构建提示词**：结合用户输入和预设模板生成提示词。  
3. **调用 AI 服务**：将提示词发送给大模型并接收响应。  
4. **解析结果**：将大模型的返回结果解析为结构化数据。  
5. **返回应用**：将最终结果传递给用户端，形成闭环。  

#### 2.2 内核的优势  

- **中间件支持**：开发者可以插入日志、监控等中间件，记录每一步执行过程。  
- **生命周期管理**：统一管理服务和插件的初始化、调用和销毁。  

**示例**：  
在开发中，创建内核通常只需一行代码（C#）：  

```csharp  
var kernel = Kernel.CreateBuilder().Build();  
```  

### 3. 插件（Plugins）：扩展 AI 的能力边界  

插件是 SK 的“功能模块”，通过封装函数（Functions）向大模型提供增强能力。例如，一个“天气插件”可以包含查询城市编号和获取天气的函数。  

#### 3.1 插件的核心作用  

- **功能解耦**：将业务逻辑封装为独立模块，便于复用和维护。  
- **自动化调用**：大模型基于用户请求，自动规划并调用插件中的函数。  

**示例场景**：  
假设开发一个写作插件（`WriterPlugin`），包含 `ShortPoem`（生成短诗）和 `StoryGen`（生成故事）函数。当用户请求“生成一首短诗并改编成故事”时，SK 会自动调用这两个函数，并将结果拼接后返回。  

#### 3.2 插件的实现要点  

- **标记函数**：使用 `[KernelFunction]` 特性声明可被调用的方法。  
- **描述清晰**：通过注释或元数据说明函数的作用和参数，帮助大模型理解其用途。  

### 4. 规划（Planning）：智能调度的指挥官  

规划是 SK 的“决策引擎”，负责协调多个插件和函数，解决复杂问题。  

#### 4.1 规划的应用场景

假设用户提问：“杭州的天气怎么样？” 若直接调用天气查询函数，但该函数需要城市编号而非中文名，规划机制会自动执行以下步骤：  

1. 调用“根据城市名获取编号”函数，将“杭州”转换为编号。  
2. 使用编号调用“查询天气”函数，返回结果。  

#### 4.2 规划的优势  

- **自动化流程**：无需手动编写调用顺序，SK 自动迭代解决问题。  
- **灵活性**：支持多步骤、多插件的复杂逻辑组合。  

### 5. 函数调用（Function Calling）：自动化的执行引擎  

函数调用是 SK 实现插件与大模型协作的关键机制，其流程如下：  

1. **序列化函数**：将内核中所有可用函数及其参数转换为 JSON 结构。  
2. **发送请求**：将用户输入和序列化后的函数列表发送给大模型。  
3. **模型处理**：大模型决定返回聊天消息还是调用函数。  
4. **执行函数**：若需调用函数，SK 提取函数名和参数并执行。  
5. **迭代处理**：将函数结果返回给大模型，继续生成最终响应。  

**示例**：  
当用户问“杭州的天气”时，SK 自动调用 `GetCityCode("杭州")` 和 `GetWeather(12345)`，最终返回结果。  

### 6. 实战演示：开发一个天气插件

以下是一个简化的实现步骤（以 C# 为例）：  

1. **定义插件类**

```csharp  
public class WeatherPlugin  
{  
    [KernelFunction]  
    [Description("根据城市名称获取城市编号")]  
    public int GetCityCode(string cityName) => cityName == "杭州" ? 12345 : 0;  

    [KernelFunction]  
    [Description("根据城市编号获取天气")]  
    public string GetWeather(int cityCode) => cityCode == 12345 ? "晴，25℃" : "未知";  
}  
```

2. **注册插件到内核**

```csharp  
var kernel = Kernel.CreateBuilder()  
    .AddPlugin(new WeatherPlugin(), "WeatherPlugin")  
    .Build();  
```  

3. **用户提问测试**

```
用户输入：杭州的天气怎么样？  
输出：杭州的天气是晴，25℃。  
```  

### 7. 总结与建议

#### 7.1 核心价值  

- **降低开发门槛**：通过自动化规划与函数调用，开发者无需手动编排复杂逻辑。  
- **灵活扩展**：插件机制允许快速集成新功能，适应多样化需求。  

#### 7.2 给初学者的建议

1. **从简单插件入手**：例如实现一个“时间查询”或“单位转换”插件。  
2. **重视函数描述**：清晰的描述能帮助大模型更准确地调用函数。  
3. **善用中间件**：通过日志中间件调试流程，快速定位问题。  

更多资料可以参考: [https://learn.microsoft.com/en-us/semantic-kernel/overview/](https://learn.microsoft.com/en-us/semantic-kernel/overview/?wt.mc_id=MVP_324329)

Demo地址: [https://github.com/JerryLuo5799/SemanticKernelDemo](https://github.com/JerryLuo5799/SemanticKernelDemo?wt.mc_id=MVP_324329)