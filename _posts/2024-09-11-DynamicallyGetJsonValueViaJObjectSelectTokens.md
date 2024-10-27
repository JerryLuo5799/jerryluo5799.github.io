---
layout: post
title:  "基于JObject.SelectTokens和JSONPath动态解析JSON"
date:   2024-09-11 21:42:55 +0800--
categories: [.NET]
tags: [JObject, JSONPath]  
---

### 1. 前言

在处理复杂的 JSON 数据时，经常需要从不同层级中提取特定的字段值。对于初级技术人员来说，理解并掌握如何使用 `JObject.SelectTokens` 方法结合 JSONPath 表达式进行这种动态提取是一项非常有用的技能。本文将指导你如何使用这些工具来获取所有 `popupData` 对象下的 `frontendId` 字段的值。

### 2. 什么是 JSONPath？

JSONPath 是一种查询语言，用于在 JSON 文档中提取数据。它类似于用于 XML 的 XPath。JSONPath 表达式基于路径来选择 JSON 对象中的数据，使其成为一种强大的工具，用于在复杂的 JSON 结构中导航和提取信息。

### 3. 示例代码

下面是一个 C# 示例，展示了如何使用 `JObject.SelectTokens` 方法结合 JSONPath 表达式来提取 JSON 对象中所有 `frontendId` 字段的值。

```csharp
using System;
using System.Collections.Generic;
using Newtonsoft.Json.Linq;
public class Program
{
    public static void Main()
    {
        string jsonString = @"
        {
          ""level1"": {
            ""popupData"": {
              ""frontendId"": ""id1"",
              ""otherField"": ""value1""
            },
            ""level2"": {
              ""popupData"": {
                ""frontendId"": ""id2"",
                ""otherField"": ""value2""
              },
              ""level3"": {
                ""popupData"": {
                  ""frontendId"": ""id3"",
                  ""otherField"": ""value3""
                }
              }
            }
          },
          ""anotherLevel1"": {
            ""popupData"": {
              ""frontendId"": ""id4"",
              ""otherField"": ""value4""
            }
          }
        }";
        List<string> frontendIds = GetFrontendIds(jsonString);
        foreach (var id in frontendIds)
        {
            Console.WriteLine(id);
        }
    }
    public static List<string> GetFrontendIds(string jsonString)
    {
        List<string> frontendIds = new List<string>();
        JObject jsonObject = JObject.Parse(jsonString);
        // 使用 JSONPath 表达式提取所有 frontendId 字段的值
        var tokens = jsonObject.SelectTokens("$..popupData.frontendId");
        foreach (var token in tokens)
        {
            frontendIds.Add(token.ToString());
        }
        return frontendIds;
    }
}
```

#### 3.1 步骤分解

1. **解析 JSON 字符串**：首先，我们需要将 JSON 字符串解析为 `JObject`。
2. **使用 SelectTokens 和 JSONPath 表达式**：然后，使用 `SelectTokens` 方法和 JSONPath 表达式来提取所有 `frontendId` 字段的值。
3. **JSONPath 表达式**：在这个例子中，我们使用了表达式 `"$..popupData.frontendId"`。这个表达式意味着：从根节点开始，选择所有路径中包含 `popupData` 并且有 `frontendId` 字段的节点。
4. **提取并存储值**：最后，我们将所有提取的 `frontendId` 值存储在一个 `List<string>` 中并返回。

#### 3.2 运行结果

当你运行这个程序时，它将输出所有找到的 `frontendId` 值：

```csharp
id1
id2
id3
id4
```

### 4. 结论

通过理解 `JObject.SelectTokens` 和 JSONPath 的使用，你可以轻松地从复杂的 JSON 结构中提取所需的字段值。这种技术对于处理多层嵌套的 JSON 数据非常有用，能够帮助初级技术人员提高工作效率和代码质量。
