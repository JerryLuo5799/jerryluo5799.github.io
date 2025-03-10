---
layout: post
title:  "Semantic Kernel 入门系列之三: 深入了解插件的概念"
date:   2025-02-14 21:42:55 +0800--
categories: [.NET]
tags: [Semantic Kernel]  
---

<iframe src="//player.bilibili.com/player.html?isOutside=true&aid=114001015020261&bvid=BV1v5KPerEMK&cid=28431224206&p=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"  class="bilibili"></iframe>

链接地址: [https://www.bilibili.com/video/BV1v5KPerEMK/](https://www.bilibili.com/video/BV1v5KPerEMK/)


### 1. 什么是插件（Plugins）？  
插件是SK的核心功能之一，它们允许开发者扩展和定制AI模型的能力。简单来说，插件就像是“工具包”，可以告诉模型如何完成特定任务，例如翻译文本、调用API或处理文件。SK主要支持两种插件类型：**提示词插件**和**原生插件**，同时还能通过OpenAPI规范导入插件，并提供了丰富的预定义插件库。

### 2. 插件类型详解  

#### 2.1 提示词插件（Prompt Plugin）  
**功能特点**：  
- 通过文本模板（提示词）指导模型生成特定输出，减少结果的不确定性。  
- 典型应用场景包括翻译、内容摘要、格式转换等。  
- 由两个核心文件组成：  
  - `skprompt.txt`：包含用于生成输出的文本模板。  
  - `config.json`：配置模型的生成参数，例如输出长度、随机性等。  

**关键配置参数**：  
- `max_tokens`：控制输出的最大长度（以token为单位）。  
- `temperature`：调节输出的随机性（0为最稳定，1为最随机）。  
- `top_p`：限定模型选择下一个词的候选范围（例如0.9表示仅考虑概率前90%的词）。  
- `presence_penalty` 和 `frequency_penalty`：减少重复内容，提升多样性。  

**适用场景**：  
当需要精准控制模型输出时，例如生成固定格式的翻译结果或标准化回复。  

#### 2.2 原生插件（Native Plugin）  
**功能特点**：  
- 基于代码实现（支持Python、C#、Java等语言），可直接调用本地方法或服务。  
- 例如，通过原生插件可以调用天气API获取实时数据，或操作本地文件系统。  
- 开发方式：定义一个类，其中包含标记为`KernelFunction`的方法，供模型调用。  

**适用场景**：  
当需要扩展模型能力以执行复杂任务（如调用外部API、访问数据库）时，原生插件是最佳选择。  

#### 2.3 通过OpenAPI规范导入插件  
OpenAPI规范（原Swagger）是一种接口描述语言（YAML/JSON格式），用于定义API的结构。SK支持直接导入OpenAPI文档，将其转换为原生插件。  
**优势**：  
- 快速集成现有API，无需重复开发。  
- 保证代码简洁性和可维护性。  

**使用步骤**：  
1. 提供OpenAPI文档的URL或本地路径。  
2. SK自动将API接口转换为可调用的插件函数。  

#### 2.4 预定义插件库  
SK提供了开箱即用的预定义插件，安装`Microsoft.SemanticKernel.Plugins.Core`包即可使用（目前为预发行版）。以下是核心插件示例：  

| 插件名称                | 功能说明                          |  
|-------------------------|----------------------------------|  
| `ConversationSummaryPlugin` | 对话总结、提取行动项和主题识别。 |  
| `FileIOPlugin`           | 基础文件读写操作。               |  
| `HttpPlugin`             | 支持GET/POST/PUT/DELETE请求。    |  
| `MathPlugin`             | 基本数学运算。                   |  
| `TextPlugin`             | 字符串处理（如大小写转换）。     |  
| `TimePlugin`             | 获取当前时间、日期信息。         |  
| `WaitPlugin`             | 在执行操作前暂停指定时间。       |  

### 3. 总结 
通过本文，我们了解了SK插件的核心类型及其应用场景：  
- **提示词插件**：适合需要精确控制模型输出的场景。  
- **原生插件**：适合扩展模型能力以执行复杂任务。  
- **OpenAPI插件**：快速集成现有API。  
- **预定义插件**：开箱即用，提升开发效率。  
- 
更多资料可以参考: [https://learn.microsoft.com/en-us/semantic-kernel/concepts/plugins/?pivots=programming-language-csharp](https://learn.microsoft.com/en-us/semantic-kernel/concepts/plugins/?pivots=programming-language-csharp?wt.mc_id=MVP_324329)

Demo地址: [https://github.com/JerryLuo5799/SemanticKernelDemo](https://github.com/JerryLuo5799/SemanticKernelDemo?wt.mc_id=MVP_324329)