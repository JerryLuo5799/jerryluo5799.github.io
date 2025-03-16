---
layout: post
title:  ".NET9详解系列之二: C#13的新特性 "
date:   2024-11-29 21:42:55 +0800--
categories: [.NET]
tags: [.NET9]  
---

## 前言

EF Core 9作为.NET 9生态系统的重要组成部分，为开发人员带来了许多令人兴奋的新特性和改进。这些新特性不仅提高了性能，还简化了数据操作，为开发人员提供了更多的灵活性。本文将详细介绍EF Core 9的新特性，并通过实际的代码示例来帮助你理解如何在项目中应用这些新特性。

## 一、JSON列支持

### 1. 什么是JSON列支持

在现代应用程序开发中，JSON数据格式被广泛使用。EF Core 9增强了对关系数据库中JSON列的支持，使开发人员能够更高效地查询、映射和操作JSON数据。这一特性尤其适用于处理半结构化数据的场景。

### 2. 如何使用JSON列支持

首先，你需要在实体类中定义一个属性来映射JSON列。然后，在DbContext的OnModelCreating方法中，使用 `modelBuilder.Entity<T>().Property(e => e.JsonProperty).HasJsonConversion()` 来配置该属性为JSON列。

```csharp
public class Blog
{
    public int BlogId { get; set; }
    public string? Url { get; set; }
    public Dictionary<string, string> MetaData { get; set; } = new Dictionary<string, string>();
}

public class BloggingContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Blog>()
            .Property(e => e.MetaData)
            .HasJsonConversion();
    }
}
```

### 3. 查询JSON数据

EF Core 9允许你在LINQ查询中直接使用JSON属性进行筛选和排序等操作。

```csharp
using var context = new BloggingContext();
var blogs = context.Blogs
    .Where(b => b.MetaData.ContainsKey("Author"))
    .OrderBy(b => b.MetaData["Author"])
    .ToList();
```

## 二、LINQ查询优化

### 1. 优化的SQL生成

EF Core 9在将LINQ查询转换为SQL时进行了优化，去除了不必要的SQL元素，使生成的SQL更加简洁和高效。

### 2. 表裁剪和投影裁剪

在处理继承映射或复杂查询时，EF Core 9会自动裁剪掉不必要的表连接和投影，提高查询性能。

```csharp
public class Order
{
    public int Id { get; set; }
    public Customer Customer { get; set; }
}

public class DiscountedOrder : Order
{
    public double Discount { get; set; }
}

public class Customer
{
    public int Id { get; set; }
    public List<Order> Orders { get; set; }
}

public class BlogContext : DbContext
{
    public DbSet<Customer> Customers { get; set; }
    public DbSet<Order> Orders { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Order>().UseTptMappingStrategy();
    }
}

// 查询所有有订单的客户
using var context = new BlogContext();
var customers = context.Customers
    .Where(c => c.Orders.Any())
    .ToList();
```

在EF Core 8中，生成的SQL可能包含不必要的表连接，而在EF Core 9中，这些多余的连接将被裁剪掉，使查询更加高效。

## 三、迁移改进

### 1. 并发迁移保护

EF Core 9引入了一种锁定机制，以防止多个迁移操作同时执行，从而避免数据库处于不一致的状态。

### 2. 迁移操作警告

当检测到迁移中包含无法在事务中运行的操作时，EF Core 9会发出警告，提醒开发人员注意潜在的风险。

```csharp
public class MyDbContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer("YourConnectionString");
    }
}

// 应用迁移
using var context = new MyDbContext();
context.Database.Migrate();
```

## 四、模型构建改进

### 1. 自动编译模型

编译模型可以提高大型应用程序的启动时间。EF Core 9现在可以自动检测并使用编译后的模型，无需手动配置。

```csharp
public class BloggingContext : DbContext
{
    public DbSet<Blog> Blogs { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer("YourConnectionString");
    }
}
```

## 五、SQL Server HierarchyId支持

### 1. HierarchyId路径生成

EF Core 9为SQL Server的HierarchyId类型添加了糖方法，使创建新子节点更加容易。

```csharp
public class Halfling
{
    public int Id { get; set; }
    public HierarchyId PathFromPatriarch { get; set; }
    public string Name { get; set; }
}

public class HalflingContext : DbContext
{
    public DbSet<Halfling> Halflings { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        optionsBuilder.UseSqlServer("YourConnectionString");
    }
}

// 创建子节点
using var context = new HalflingContext();
var daisy = await context.Halflings.SingleAsync(e => e.Name == "Daisy");
var child1 = new Halfling { PathFromPatriarch = HierarchyId.Parse(daisy.PathFromPatriarch.ToString() + "/1"), Name = "Toast" };
var child2 = new Halfling { PathFromPatriarch = HierarchyId.Parse(daisy.PathFromPatriarch.ToString() + "/2"), Name = "Wills" };
context.Halflings.Add(child1);
context.Halflings.Add(child2);
await context.SaveChangesAsync();
```

## 六、工具改进

### 1. MSBuild集成

EF Core 9现在支持MSBuild集成，可以在构建项目时自动更新编译后的模型。

```xml
<PropertyGroup>
    <EFOptimizeContext>true</EFOptimizeContext>
    <EFScaffoldModelStage>build</EFScaffoldModelStage>
</PropertyGroup>
```

## 总结

EF Core 9的新特性为开发人员提供了更强大、更高效的工具来处理数据访问层的开发。无论是对JSON数据的支持，还是LINQ查询的优化，亦或是迁移过程的改进，这些特性都将帮助你构建更高质量、更易于维护的应用程序。希望本文的介绍和示例能够帮助你更好地理解和应用EF Core 9的新特性。

更多信息: [C#13 文档](https://learn.microsoft.com/en-us/ef/core/what-is-new/ef-core-9.0/whatsnew?wt.mc_id=MVP_324329)