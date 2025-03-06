---
layout: post
title:  "Semantic Kernel 入门系列之一: 什么是Semantic Kernel "
date:   2024-10-26 21:42:55 +0800--
categories: [.NET]
tags: [Semantic Kernel]  
---

<iframe src="//player.bilibili.com/player.html?isOutside=true&aid=113366936851935&bvid=BV1XxySY5Ek5&cid=26454265629&p=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" class="bilibili"></iframe>

链接地址: [https://www.bilibili.com/video/BV1XxySY5Ek5/](https://www.bilibili.com/video/BV1XxySY5Ek5/)

### 1. 前言

随着大语言模型的迅速发展与普及，自然语言处理的门槛不断降低，越来越多的开发者希望在他们的系统中集成AI功能以提升用户体验。今天，我们将介绍一款名为Semantic Kernel（简称SK）的轻量级开发工具，它能够帮助初级开发者轻松地在项目中实现AI集成。

### 2. Semantic Kernel快速体验

在深入了解之前，我们先通过一个简单的示例来感受SK的魅力。假设我们想要实现一个智能问答服务，使用SK只需几步即可达成：

1. **创建SK实例**：基于Azure OpenAI或其他支持的AI服务创建一个SK实例。
2. **定义聊天记录存储**：为了保持对话的上下文关联，我们需要定义一个聊天记录存储对象。
3. **定义提示词**：通过提示词为AI设定一个交互模板。
4. **调用AI服务**：通过SK提供的接口直接调用大模型进行问答。

**安装NuGet包:**

```
dotnet add package Microsoft.SemanticKernel
```

**引用命名空间:**

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
```

**实现AI服务调用:**

```csharp
// Create a kernel with Azure OpenAI chat completion
var builder = Kernel.CreateBuilder().AddAzureOpenAIChatCompletion(Config.DEPLOYMENT_NAME, Config.ENDPOINT, Config.API_KEY);

// Build the kernel
Kernel kernel = builder.Build();
var chatCompletionService = kernel.GetRequiredService<IChatCompletionService>();

// Create a history store the conversation
var history = new ChatHistory("""
    You are an AI assistant that helps people find information,  you will provide all the detailed information.
    You like to speak Chinese, you don't output results in Markdown format.
    """);

// Initiate a back-and-forth chat
string? userInput;
do
{
    // Collect user input
    Console.ForegroundColor = ConsoleColor.Red;
    Console.Write("User > ");
    userInput = Console.ReadLine();

    // Add user input
    history.AddUserMessage(userInput);

    // Get the response from the AI
    var result = await chatCompletionService.GetChatMessageContentAsync(history, null, kernel);

    // Print the results
    Console.ForegroundColor = ConsoleColor.White;

    Console.WriteLine("Azure OpenAI > " + result);
    Console.ForegroundColor = ConsoleColor.Red;

    // Add the message from the agent to the chat history
    history.AddMessage(result.Role, result.Content ?? string.Empty);
} while (userInput is not null);
```

**运行结果:**

![alt text](/assets/imgs/SK01-QA.png)

### 3. 什么是Semantic Kernel？

![What's Semantic Kernel](/assets/imgs/SK01-WhatIsSK.png)

Semantic Kernel，翻译为语义内核，是一款由微软开源的轻量级SDK。它专为简化AI服务的集成而设计，支持多种编程语言如C#、Java、Python等。微软自家也在多款Copilot产品中使用了SK，证明了其稳定性和实用性。

### 4. 为什么使用Semantic Kernel？

![Why Semantic Kernel](/assets/imgs/SK01-WhySK.png)

#### 4.1 简化AI服务对接

在对接大模型时，SK提供了一组连接器，这些连接器简化了模型与存储的连接过程。SK默认支持OpenAI、Azure OpenAI、Hugging Face等主流模型，使得开发者能够轻松地在业务代码中调用这些AI服务。

#### 4.2 多样的存储选项

SK还支持与十多种向量数据库的连接，这些数据库用于存储和检索信息。这对于构建具有上下文感知能力的AI应用至关重要。

#### 4.3 模块化的插件架构

SK的插件架构是其核心优势之一。开发者可以创建智能插件（Skills）来增强模型的能力。这些插件由提示函数（Prompt Functions）和原生函数（Native Functions）组成。SK作为中间件，将提示函数的请求转换为对本机函数的调用，并处理结果。例如，要实现一句话创建日程的功能，只需编写一个解析日期、会议主题、参与人等元素的提示函数，并实现一个日程创建的本机函数即可。

#### 4.4 易于集成与扩展

由于SK的轻量级设计和模块化架构，它非常易于集成到现有的开发环境中。同时，其插件架构使得系统可以根据需求进行灵活扩展，满足不断变化的业务需求。

### 5.结语

Semantic Kernel为初级开发者提供了一个简单而强大的工具，用于在他们的项目中集成AI功能。通过简化AI服务的对接、提供多样的存储选项、以及模块化的插件架构，SK使得构建智能应用变得更加容易和高效。如果你正在寻找一种方法来提升你的项目的智能化水平，那么Semantic Kernel无疑是一个值得尝试的选择。

更多资料可以参考: [https://learn.microsoft.com/en-us/semantic-kernel/overview/](https://learn.microsoft.com/en-us/semantic-kernel/overview/?wt.mc_id=MVP_324329)

Demo地址: [https://github.com/JerryLuo5799/SemanticKernelDemo](https://github.com/JerryLuo5799/SemanticKernelDemo?wt.mc_id=MVP_324329)