---
layout: post
title:  "什么是Semantic Kernel"
date:   2024-08-20 21:42:55 +0800--
categories: [.NET]
tags: [Semantic Kernel, SK]  
---

### 1. 前言

作为一名技术开发者，你是否也曾想过将AI技术集成到自己的项目中，却苦于缺乏相关知识和技术门槛？一款来自微软的开源神器——Semantic Kernel（SK），它能帮助你轻松地实现AI功能，让你的应用焕发新的活力！

### 2. 什么是Semantic Kernel?

![What's Semantic Kernel](/assets/imgs/SK01-WhatIsSK.png)

Semantic Kernel（SK）是一款轻量级的SDK，由微软开源，并在其Copilot产品中广泛应用。它支持C#、Java、Python等多种编程语言，可以帮助开发者快速将AI能力集成到自己的系统中。

### 3. 为什么使用Semantic Kernel？

![Why Semantic Kernel](/assets/imgs/SK01-WhySK.png)

连接器： SK提供了一组连接器，方便连接各种AI模型和存储服务。默认支持OpenAI、Azure OpenAI、Hugging Face等模型，以及十多种向量数据库。

插件架构： SK采用模块化的插件架构，允许开发者开发智能插件来增强模型的能力。插件本质是一组插件的本质是一组提示函数（Prompts Functions）和本机函数（Native Functions），SK作为中间件，将提示函数的请求转换成本机函数的调用，并处理结果。

易于集成： SK提供了丰富的API和示例代码，帮助开发者快速上手，轻松集成AI功能到自己的项目中。

### 4. 如何使用Semantic Kernel？

以下是一个使用SK实现智能问答的简单示例：

**安装NuGet包:**
```
dotnet add package Microsoft.SemanticKernel
```

**引用命名空间:**
``` C#
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.ChatCompletion;
```

**接着只需要:**
- 创建Azure OpenAI连接器
- 定义聊天记录存储对象
- 定义聊天提示模板
- 调用智能问答API
``` C#
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

### 5. 总结

Semantic Kernel（SK）是一款功能强大的AI开发工具，可以帮助开发者轻松地将AI能力集成到自己的项目中。无论你是初级开发者还是初级AI人员，都可以通过SK快速学习和掌握AI技术，让你的应用更加智能和高效。

SK仍在不断发展中，未来将支持更多AI模型和功能，并不断完善其插件架构，为开发者提供更多便利。相信SK将成为AI开发领域的重要工具，帮助开发者构建更智能的应用。

### 6. 了解更多

点击访问 [Semantic Kernel官方文档](https://learn.microsoft.com/zh-cn/semantic-kernel/overview/?wt.mc_id=MVP_324329)