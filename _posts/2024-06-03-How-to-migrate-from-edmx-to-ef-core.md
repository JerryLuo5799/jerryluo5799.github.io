---
layout: post
title:  ".NET Framework升级.NET8之七: 如何将edmx迁移到EF Core"
date:   2024-06-20 21:42:55 +0800--
categories: [.NET]
tags: [NET Framework升级.NET8]  
---

### 前言

某些较旧的.NET Framework 项目会使用 EDMX，而不是现代的 DbContext（DbContext 最初是在 [Entity Framework 4.1](https://devblogs.microsoft.com/cesardelatorre/entity-framework-4-1-just-released/?wt.mc_id=MVP_324329) 中引入的），该版本在 2012 年首次引入了 DbContext 和 Code-First 方法，取代了 EDMX 所使用的 ObjectContext（用于 Database-First 方法）。

在本文中，我们将 ObjectContext 和 Entities 互换使用。ObjectContext 是生成类所使用的基类，通常以 Entities 结尾（例如 DataEntities）。

### 1. 策略

关于从完整重写到更原地迁移的策略，取决于项目的规模和复杂性。本规则将描述一种平衡我们需要重写的代码量和现代化的方法。

重点是尽量减少因迁移而导致的不部署时间。

本规则中的策略将包括以下步骤：

1. 使用自定义 `IDbContext` 接口（例如 `ITenantDbContext`）抽象现有的 `ObjectContext/Entities` 类。
2. 构建数据库。
   1. [EF Core Power Tools](https://marketplace.visualstudio.com/items?itemName=ErikEJ.EFCorePowerTools?wt.mc_id=MVP_324329)
      1. 如果工具失败，使用 EF Core 3.1 或 EF Core 8+ CLI 进行构建。EF Core 3.1 在处理较旧的数据库架构方面比 EF Core 8+ 更胜一筹。
3. 实现第 1 步中的接口并重构实体。
   1. 审查实体，调整生成的代码并更新 `DbContext.OnConfiguring`。
   2. 将 `ObjectSet<T>` 替换为 `DbSet<T>`。
   3. 进行其他必要的重构。
      1. 可能会遇到不同的空值处理问题。
      2. 某些属性的类型可能不同，需要修复映射。
      3. 延迟加载可能会出现问题，可以通过急切加载来解决。
      4. 当升级到 EF Core 3.1 时，不支持分组和某些其他功能。
         1. 使用 `.AsEnumerable()`，使用原始 SQL 或更改查询的工作方式。
         2. 添加 **技术债务** 注释和 PBI
4. 更新命名空间（对于实体、EF Core 命名空间以及移除旧版命名空间）。
   1. 在使用 EF Core 3.1 的所有文件中移除 `System.Data.Entity` 命名空间（否则，您将遇到奇怪的 Linq 异常）。
   2. 添加 `Microsoft.EntityFrameworkCore` 命名空间。
5. 更新依赖注入。
   1. 使用现代的 `.AddDbContext()` 或 `.AddDbContextPool()`。
6. 更新迁移策略（从数据库优先到代码优先）。
   1. 使用 EF Core CLI 而不是 DbUp。
7. 完全移除 EDMX（如果一次性完成迁移，而不是分步骤进行，可以更早完成此步骤）。
8. 可选：升级到 .NET 8+（如果当前使用的是 .NET Framework 或 .NET Core 3.1）。
9. 可选：升级到 EF Core 8+（如果需要 EF Core 3.1 的路径）。
10. 测试，测试，再测试...
    1. 从 EDMX 迁移到 EF Core 3.1 或更高版本是一项重大的现代化工作，涉及许多底层变化。
    2. 常见问题包括：
       1. 延迟加载。
       2. 分组（如果在 EF Core 3.1 中）。
       3. 不支持的查询（代码实际上在 .NET 端运行而不是在 SQL Server 上）。
       4. 因复杂查询导致的性能问题。
       5. EF Core 查询返回的结果不正确。

步骤 6 和 7 是从 .NET Framework 升级到 .NET 8 时必需的，如果解决方案过于复杂，无法一次性完成迁移，则需要分步骤进行。对于简单项目，如果 EDMX 是唯一的主要阻碍问题，应直接升级到 .NET 8 和 EF Core 8。

```text
注意：通过一些智能的抽象策略，可以在仍有一个可运行的应用程序的同时完成步骤 3 - 5。这仅适用于对架构和 EF 运行方式有经验的开发人员，以避免因运行两个 EF 跟踪系统而产生的错误。这将影响 EF 的内部缓存和保存更改。
```

在本规则中，我们仅涵盖使用自定义 `IDbContext` 接口抽象访问 `ObjectContext` 以及如何构建数据库。其余步骤需要深入审查代码，可能因项目而异。

### 2. 抽象访问 ObjectContext/Entities

在开始之前，重要的是要注意 ObjectContext 和 EDMX 不再受支持，我们需要对数据层进行完全重写。可以使用一个看起来像现代 DbContext 的接口来包装 ObjectContext，因为大多数常用方法是相同的。

下面的包装器不仅允许我们以更清洁的方式使用 ObjectContext，还可以在不重构业务逻辑的情况下更好地管理 ObjectContext 和 DbContext 之间的差异。

```csharp
using System.Data.Entity.Core.Objects;

public interface ITenantDbContext
{
    ObjectSet<Client> Clients { get; }

    int SaveChanges();
    Task<int> SaveChangesAsync(CancellationToken ct = default);
}

/// <summary>
/// 将 DbContext 实现为内部类，以便外部库无法直接访问它。
/// 而是通过接口暴露功能。
/// </summary>
internal class TenantDbContext : ITenantDbContext
{
    private readonly DataEntities _entities;

    public TenantDbContext(DataEntities entities)
    {
        _entities = entities;
    }

    public ObjectSet<Client> Clients => _entities.Clients;

    public int SaveChanges() => _entities.SaveChanges();
    public Task<int> SaveChangesAsync(CancellationToken ct = default) => _entities.SaveChangesAsync(ct);
}
```

```text
注意：本节中所做的更改仍与 .NET Framework 兼容，允许我们在进行上述更改的同时为客户交付价值。
```

#### 3. 构建数据库

现在我们已经抽象了数据访问，是时候构建数据库了。最简单的方法是使用 [EF Core Power Tools](https://marketplace.visualstudio.com/items?itemName=ErikEJ.EFCorePowerTools?wt.mc_id=MVP_324329)。

1. 右键单击项目 | **EF Core Power Tools** | **反向工程**

![图：选择反向工程工具](/assets/imgs/project-reverse-engineer-tool-1.png)

2. 选择数据连接和 EF Core 版本

![图：数据连接](/assets/imgs/project-reverse-engineer-tool-2.png)

3. 选择数据库对象（表、视图、存储过程等）

![图：数据库对象](/assets/imgs/project-reverse-engineer-tool-3.png)

4. 选择项目的设置
   1. 推荐：**使用 DataAnnotation 特性配置模型** 以减少 DbContext 中的代码行数。
   2. 可选：如果尚未完成，**在项目中安装 EF Core 提供程序包**。
   3. 可选：如果现有代码依赖于数据库中的命名方案，**直接使用表和列名**。

![图：项目设置](/assets/imgs/project-reverse-engineer-tool-4.png)

5. 代码将生成在我们决定的路径下（**实体类型路径**）。在这个例子中，它是 `Persistence` 文件夹。

![图：项目设置](/assets/imgs/project-reverse-engineer-tool-5.png)

6. EF Core Power Tools 将自动生成一个 `DbContext` 类。

![图：项目设置](/assets/imgs/project-reverse-engineer-tool-6.png)

### 4. 资源

- 迁移到 EF Core 3.1 的视频 - [https://learn.microsoft.com/en-us/shows/on-net/migrating-edmx-projects-to-entity-framework-core](https://learn.microsoft.com/en-us/shows/on-net/migrating-edmx-projects-to-entity-framework-core#time=08m10s?wt.mc_id=MVP_324329)
- 官方迁移到 EF Core 3.1 的文档 - [https://learn.microsoft.com/en-us/ef/efcore-and-ef6/porting/port-edmx](https://learn.microsoft.com/en-us/ef/efcore-and-ef6/porting/port-edmx?wt.mc_id=MVP_324329)
