---
layout: post
title:  "使用资源组来管理Azure资源"
date:   2023-01-15 21:42:55 +0800--
categories: [Azure]
tags: [Rules, Azure Resource Groups]  
---

### 1. 前言

资源组是 Azure 平台的基本元素。 资源组是部署在Azure上的资源的逻辑容器。所有Azure资源必须在资源组中，并且Azure资源只能是一个资源组的成员。

### 2. 命名规范

资源组的名称在整个业务中需要保持一致性，开发人员或系统管理员可以看到给定环境(dev/test/prod)中给定产品使用的所有资源，并让它们准确地识别其中包含的内容。

资源组的命名应该遵循 [产品].[环境], 加入我们有个产品叫**Northwind**:

- Northwind.Dev
- Northwind.Staging
- Northwind.Production

创建资源组是完全免费的，所以需要尽情的使用它, 最重要的是:**在命名约定上保持一致**。

### 3. 将资源和资源组正确的匹配

您应该将产品的所有资源保持在同一个资源组中。然后，开发人员可以快速、轻松地找到所有相关资源，并有助于将创建重复资源的风险降至最低。资源组是帮助我们管理和区分开发环境和生产环境使用资源的最佳方式。

❌不好的示例 - 一个开发资源在生产环境资源组中
![不好的示例](https://ssw.com.au/rules/static/a8857602e962239da011399aacefd1d3/0a867/rogue-resource.png)

### 4. 不要将不同环境的资源放到同一个资源组中

没有什么比打开一个资源组并找到相同资源的几个实例更糟糕的了，而不知道dev/staging/production中的资源是什么。类似地，如果发现Notification Hub的单个实例，如何知道它是在测试环境中构建的，还是生产环境中需要的遗留资源?

❌不好的示例 - staging和production的资源在在同一个资源组中
![不好的示例](https://ssw.com.au/rules/static/3b02ff43ae778730220bf39eaa9b22b2/2bef9/bad-azure-environments.png)

### 5. 不要根据不同的资源类型划分资源组

将相同类型的资源分组在一起不会节省成本。例如，没有理由将所有数据库放在一个资源组下。最好将数据库提供在与使用它的应用相同的资源组中。

❌不好的示例 - SSW.SQL资源组包含了不同应用程序的数据库
![不好的示例](https://ssw.com.au/rules/static/8846e5bed2cd15fa8b7362e23babacd3/0470c/arrange-azure-resources-bad.jpg)

### 6. 给资源组打标 (Tags)

Azure有标签功能，允许您将不同的标签名称和值应用到资源和资源组:

![Tags](https://ssw.com.au/rules/static/276b5903e47b1a142c2f8054d8b72d2f/a4262/tags-in-resources-group.png)

您可以利用这个特性更加细致的划分资源，而不仅仅依赖于名称。

- 所有者标记:您可以指定谁拥有该资源
- 环境标记:您可以指定资源所在的环境

你可以使用[微软推荐的命名和标记指南](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/naming-and-tagging?wt.mc_id=MVP_324329)。

