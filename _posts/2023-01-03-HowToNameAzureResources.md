---
layout: post
title:  "Azure资源命名规范"
date:   2023-01-03 21:42:55 +0800--
categories: [Azure]
tags: [Rules, 资源命名规范]  
---

### 1. 前言

当我们在Azure创建大量资源的时候, 如果没有一个好的命名规范, 那么寻找具体资源的时候就会是个头疼的事情了。

### 2. 命名规范

管理云资源需要从一个规范的名字开始, 最好保持一致并使用:

- 全部小写
- 使用"-"作为分隔符
- 包含资源类型缩写（以便于在脚本中找到资源）
- 包括资源适用于哪个环境，即开发、测试、生产等。
- 如果适用，请在名称中包含资源的预期用途，例如，应用服务可能具有后缀 API
- 在资源名中保留产品的名称，这样在Azure更容易检索, 建议遵循"product-name-environment"命名约定，最重要的是:**保持一致!**
- Azure 定义了[命名和标记资源的一些最佳做法](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/naming-and-tagging)

![好的例子](https://ssw.com.au/rules/static/4444b8e83ff8dd1e95618bd96bfbb4a9/2bef9/better-example.png)

跨项目的资源名称不一致会产生各种痛苦

- 开发人员将很难找到项目的资源并确定这些资源的用途
- 开发人员不知道该如何称呼他们需要创建的新资源。
- 创建重复资源的风险...因为开发人员不知道另一个开发人员在 6 个月前在不同的资源组中以不同的名称创建了相同的内容

#### 2.1 特殊情况处理

有些资源不能使用上述的命名约定(例如，存储帐户不能使用"-"分隔)，需要特别为这些特定资源制定规范。

#### 2.2 自动化部署Azure资源

因为人会犯错误，所以有时通过手动的方式创建资源时, 可能会无法保证会根据命名规范来命名资源, 所以最好使用ARM、Bicep、terrraform和Pulumi等工具，通过基础设施即代码(IaC)以编程方式, 部署Azure资源。使用IaC，你可以在代码中加入命名约定，同时，你可以跟踪你的标准的任何变化，因为你的代码是有版本管理的。

更多关于自动化部署的内容, 请参照: [如何创建Azure资源](https://jerryluo.com/2022/12/17/HowToCreateAzureResource/)


