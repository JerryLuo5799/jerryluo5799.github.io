---
layout: post
title:  "深度解析.NET映射新秀Facet"
date:   2025-08-15 12:30:00 +0800--
categories: [.NET]
tags: [Facet]
---

## 前言

在 .NET 开发中，对象映射（Mapping）几乎是每个项目的刚需。无论是将数据库实体（Entity）转换为 DTO，还是在微服务间传递数据，我们都在不断编写 `a.Name = b.Name` 这种重复代码。

长期以来，**AutoMapper** 是社区的默认选择。但随着 .NET 8/9 拥抱 **Native AOT** 以及对冷启动性能的追求，基于运行时反射（Reflection）的映射器逐渐显露弊态：性能瓶颈、调试困难、以及 AOT 不兼容。今天，我们要深度拆解一个全新的“源生成”映射神器：**Facet**。它通过 **C# Source Generators** 在编译时就把代码写好，带你进入“零反射”映射时代。

## 核心概念：什么是源生成映射？

传统的映射器像是一个“黑盒”，在程序运行时通过反射去寻找匹配的字段。而 **Facet** 利用了 **Source Generators** 技术，在编译器工作时就为你生成了对应的赋值逻辑。

* **零反射开销**：映射速度接近手写代码。
* **调试透明**：你可以直接 **F11** 步入生成的代码，清楚看到每一行赋值，不再有“消失的异常”。
* **AOT 友好**：天然支持 Native AOT，没有任何运行时动态生成的代码。

## 实战演示：三步玩转 Facet

### 第一步：定义 DTO 并开启映射

使用 `[Facet]` 特性标记 DTO。在实际开发中，我们经常需要排除多个敏感字段（如密码、盐值）。Facet 支持通过数组实现**多字段排除**。

> **注意**：DTO 类必须标记为 **`partial`**，否则源生成器无法将生成的代码“缝合”进去。

```csharp
using Facet;

[Facet(typeof(UserEntity), Exclude = [
    nameof(UserEntity.PasswordHash), 
    nameof(UserEntity.SecurityStamp),
    nameof(UserEntity.InternalNotes)
])]
public partial class UserDto
{
    // 源生成器会自动根据 UserEntity 补全除上述三个字段外的所有属性
}

```

### 第二步：执行映射

Facet 自动生成了构造函数和流式扩展方法，使用起来非常直观。

```csharp
var user = new UserEntity { Id = 1, Name = "Gemini", Points = 500 };

// 方式 A：直接使用生成的构造函数
var dto = new UserDto(user);

// 方式 B：使用 ToFacet 扩展方法
var dto2 = user.ToFacet<UserEntity, UserDto>();

```

### 第三步：EF Core 数据库投影

这是 Facet 的“杀手锏”：它生成的 `Projection` 表达式能让 EF Core 在 SQL 层面只查询 DTO 需要的列，极大优化查询性能。

```csharp
var users = await dbContext.Users
    .Select(UserDto.Projection) // 自动生成 Select 语句，仅查询非排除字段
    .ToListAsync();

```

## 进阶技巧：属性微操与复杂逻辑

### 1. 声明式属性配置

如果你只是想改个名字或忽略某个字段，可以使用内置的特性，这比写配置类更轻量。

```csharp
[Facet(typeof(UserEntity))]
public partial class UserDto
{
    // 别名映射：将数据库的 usr_auth_id 映射到 AuthorizationId
    [FacetProperty(MapFrom = "usr_auth_id")]
    public string AuthorizationId { get; set; } = default!;

    // 显式忽略：即使源对象有匹配字段也不映射
    [FacetIgnore]
    public string SessionToken { get; set; } = default!;
}

```

### 2. 自定义映射逻辑（如加密/解密）

当涉及复杂逻辑（如：**数据库存的是 Base64 密文，DTO 需要明文**）时，我们需要实现 `IFacetMappingConfiguration` 接口。

```csharp
public class UserSecurityConfig : IFacetMappingConfiguration<UserEntity, UserDto>
{
    public void Apply(UserEntity source, UserDto target)
    {
        // 1. 字段合并
        target.FullDisplayName = $"{source.FirstName} {source.LastName}";

        // 2. 解密逻辑：将 DB 密文转为明文
        if (!string.IsNullOrEmpty(source.EncryptedEmail))
        {
            var bytes = Convert.FromBase64String(source.EncryptedEmail);
            target.Email = System.Text.Encoding.UTF8.GetString(bytes);
        }
    }
}

// 绑定配置
[Facet(typeof(UserEntity), Configuration = typeof(UserSecurityConfig))]
public partial class UserDto { /* ... */ }

```

## 深度思考

### 1. 性能数据的真相

在 .NET 8 环境下映射 100 万个对象的参考数据：

| 映射工具 | 耗时 (1M Ops) | 内存分配 | 调试友好度 |
| --- | --- | --- | --- |
| **Manual Mapping (手写)** | ~2.5 ms | 0 B | 极高 |
| **Facet (源生成)** | **~70.5 ms** | **极低** | **极高 (可步入)** |
| **AutoMapper (反射)** | ~185.0 ms | 较高 | 差 (黑盒) |

### 2. 特性 (Attribute) vs 配置类 (Config)

* **特性**：适合“元数据”描述，如改名、忽略。它简单、直观，是声明式的。
* **配置类**：适合“行为”逻辑，如解密、格式化。它易于单元测试，符合关注点分离原则。

### 3. 微服务中的 DTO 应该放在哪？

在微服务场景下，DTO 往往散落在多个项目中。

* **不推荐集中管理**：强行建立一个 `Common.DTO` 会导致服务间的**强耦合**，形成“分布式单体”。
* **推荐按需定义**：在每个微服务内部定义自己的 Facet DTO。虽然会有少量代码重复，但换取了服务独立演变的能力。Facet 名字的本意正是——每个调用方都有看数据的独特视角（切面）。

## 总结

Facet 代表了 .NET 开发的新范式：**能交给编译器的，绝不留给运行时**。它通过源生成技术，既保留了手写代码的透明度和高性能，又获得了框架级的开发效率。

### 引用与资源

* [Facet GitHub 官方仓库](https://github.com/Tim-Maes/Facet?wt.mc_id=MVP_324329)
