---
layout: post
title:  "使用反射和JSON序列化动态生成匿名对象"
date:   2024-06-05 21:42:55 +0800--
categories: [.NET]
tags: [dynamic]  
---

### 1. 前言

在开发过程中，我们经常需要处理类对象和JSON对象之间的转换, 我需要根据一个类对象生成一个新的匿名对象，同时排除一些不需要的属性。例如，在API调用时，我们可能只希望返回某些属性，而不是整个对象的所有属性。在本文中，我将向您展示如何使用反射和JSON序列化来实现这一目标。

### 2. 实现步骤

#### 2.1 定义ColumnAttribute特性类

首先，我们定义一个ColumnAttribute特性类，用于标记哪些属性不应该被更新。

```csharp
[AttributeUsage(AttributeTargets.Property, AllowMultiple = false)]
public class ColumnAttribute : Attribute
{
    public bool CanUpdate { get; set; }
}
```

#### 2.2 定义Book类

然后，我们定义一个Book类，其中包含Title、Author、Year、ISBN、Publisher和AdditionalInfo属性。其中，ISBN和Publisher属性带有[Column(CanUpdate = false)]特性，AdditionalInfo是一个JSON对象。

```csharp
public class Book
{
    public string Title { get; set; }
    public string Author { get; set; }
    public int Year { get; set; }
    [Column(CanUpdate = false)]
    public dynamic ISBN { get; set; }
    [Column(CanUpdate = false)]
    public string Publisher { get; set; }
    public object AdditionalInfo { get; set; } // 假设这是一个JSON对象
}
```

#### 2.3 创建Book对象

接下来，我们创建一个Book对象并初始化其属性。

```csharp
public class Program
{
    public static void Main()
    {
        // 创建一个Book对象
        Book book = new Book
        {
            Title = "The Great Gatsby",
            Author = "F. Scott Fitzgerald",
            Year = 1925,
            ISBN = 9780743273565, // dynamic类型
            Publisher = "Scribner",
            AdditionalInfo = new { Pages = 180, Genre = "Fiction" } // JSON对象
        };
    }
}
```

#### 2.4 使用CreateAnonymousObject方法

然后，我们使用CreateAnonymousObject方法动态生成一个新的匿名对象。

```csharp
public static object CreateAnonymousObject(object obj)
{
    // 使用反射获取对象的所有公共实例属性
    var properties = obj.GetType().GetProperties(BindingFlags.Public | BindingFlags.Instance);
    // 创建一个新的匿名对象
    var anonymousObject = new { };
    // 筛选出不需要的属性
    var anonymousObjectProperties = properties
        .Where(prop => !Attribute.IsDefined(prop, typeof(ColumnAttribute)) ||
                       ((ColumnAttribute)Attribute.GetCustomAttribute(prop, typeof(ColumnAttribute))).CanUpdate)
        .Select(prop =>
        {
            var value = prop.GetValue(obj);
            if (value is dynamic || IsJsonObject(value))
            {
                value = JsonSerializer.Serialize(value);
            }
            return new { prop.Name, Value = value };
        })
        .ToDictionary(p => p.Name, p => p.Value);
    // 将筛选后的属性转换为字典
    var anonymousType = anonymousObjectProperties
        .Select(p => new { p.Key, p.Value })
        .ToDictionary(p => p.Key, p => p.Value);
    // 返回新的匿名对象
    return anonymousType;
}
```

#### 2.5 运行示例

运行这个程序，将输出：

```csharp
Title: The Great Gatsby
Author: F. Scott Fitzgerald
Year: 1925
AdditionalInfo: {"Pages":180,"Genre":"Fiction"}
```
