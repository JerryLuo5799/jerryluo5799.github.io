---
layout: post
title:  "反射、表达式树与源生成器：DTO 到实体映射的性能对比"
date:   2026-09-18 10:00:00 +0800--
categories: [.NET]
tags: [Source Generators, Expression, Reflection, BenchmarkDotNet]
---

### 前言

把前端提交的请求 DTO 转换成数据库实体，是几乎每个 Web API 都在写的代码。项目里可能用的是手写赋值、AutoMapper、Mapster，或者 [Facet](/2025/08/15/using-facet-for-mapping/) 这类新一代映射库。剥开封装，最常见的底层技术有三种：

- **反射**：运行时通过 `PropertyInfo` 读写属性；
- **表达式树**：运行时用 `System.Linq.Expressions` 拼出赋值逻辑，`Compile` 成委托后反复调用；
- **源生成器**：编译时由 Roslyn 生成普通的 C# 赋值代码，运行时没有任何额外机制。

这三种并不是全部。还可以用 `System.Reflection.Emit` 直接生成 IL，表达式树的 `Compile` 在内部做的正是这件事；可以用 `Delegate.CreateDelegate` 把属性的 getter 和 setter 绑定成强类型委托，绕开反射调用的装箱；还有人干脆用 `System.Text.Json` 序列化再反序列化来完成复制。这些做法大多是三条路线的变体或更底层的实现，所以这里只比较最具代表性的三种。

三条路线各自实现一遍，再加上手写赋值作为基线，用 BenchmarkDotNet 在同一台机器上实测，就能看清它们之间到底差多少、差在哪里。所有示例代码都取自同一个可以编译运行的解决方案，测试数据均为实际运行结果，未经修改。

![三种映射方式在什么时候确定赋值逻辑](/assets/imgs/dto-mapping-three-approaches.svg)

### 1. 测试对象

场景是「用户注册」：前端提交 `CreateUserDto`，服务端创建一个新的 `User` 实体。实体比 DTO 多出 `Id` 和 `CreatedAt` 两个由服务端决定的字段，它们不参与映射。属性覆盖了 `string`、可空 `string`、`int`、`bool`、`decimal`、`DateTime`、`Guid` 几种常见类型，其中值类型在反射路径上会触发装箱。

```csharp
/// <summary>前端提交的注册请求</summary>
public sealed class CreateUserDto
{
    public string UserName { get; set; } = "";
    public string Email { get; set; } = "";
    public string DisplayName { get; set; } = "";
    public string? Phone { get; set; }
    public int Age { get; set; }
    public bool IsActive { get; set; }
    public decimal Balance { get; set; }
    public DateTime BirthDate { get; set; }
    public Guid TenantId { get; set; }
    public string Country { get; set; } = "";
}

/// <summary>数据库实体：比 DTO 多出 Id、CreatedAt 两个由服务端决定的字段</summary>
public sealed class User
{
    public long Id { get; set; }
    public string UserName { get; set; } = "";
    public string Email { get; set; } = "";
    public string DisplayName { get; set; } = "";
    public string? Phone { get; set; }
    public int Age { get; set; }
    public bool IsActive { get; set; }
    public decimal Balance { get; set; }
    public DateTime BirthDate { get; set; }
    public Guid TenantId { get; set; }
    public string Country { get; set; } = "";
    public DateTime CreatedAt { get; set; }
}
```

映射规则统一为：**目标属性可写、源属性可读、名称相同且类型相同，才赋值**。手写版本就是这条规则的人肉展开，也是性能的理论下限：

```csharp
public static class HandWrittenMapper
{
    public static User Map(CreateUserDto source) => new()
    {
        UserName = source.UserName,
        Email = source.Email,
        DisplayName = source.DisplayName,
        Phone = source.Phone,
        Age = source.Age,
        IsActive = source.IsActive,
        Balance = source.Balance,
        BirthDate = source.BirthDate,
        TenantId = source.TenantId,
        Country = source.Country,
    };
}
```

手写的问题不在性能，而在维护：DTO 加一个字段而映射忘了加，编译器不会有任何提示。三种自动化方案要解决的正是这件事。

### 2. 反射赋值

最直观的写法是每次调用时遍历目标类型的属性，找到同名源属性就 `GetValue` 再 `SetValue`：

```csharp
using System.Reflection;

public static class ReflectionMapper
{
    private const BindingFlags Flags = BindingFlags.Public | BindingFlags.Instance;

    /// <summary>每次调用都重新查元数据：最朴素的写法</summary>
    public static TTarget MapNoCache<TSource, TTarget>(TSource source) where TTarget : new()
    {
        var target = new TTarget();
        foreach (var tp in typeof(TTarget).GetProperties(Flags))
        {
            if (!tp.CanWrite) continue;

            var sp = typeof(TSource).GetProperty(tp.Name, Flags);
            if (sp is null || !sp.CanRead || sp.PropertyType != tp.PropertyType) continue;

            tp.SetValue(target, sp.GetValue(source));
        }
        return target;
    }
}
```

这段代码每次调用都要做两类事：查元数据（`GetProperties` 会分配一个新数组，`GetProperty` 按名称查找）和通过反射读写值。元数据在程序运行期间不会变，第一类工作完全可以只做一次。利用泛型静态类「每组类型参数各有一份、由运行时保证只初始化一次」的特性，把属性对缓存起来：

```csharp
    /// <summary>元数据只查一次，缓存成 (源属性, 目标属性) 数组</summary>
    public static TTarget MapCached<TSource, TTarget>(TSource source) where TTarget : new()
    {
        var target = new TTarget();
        foreach (var (sp, tp) in PropertyPairs<TSource, TTarget>.Pairs)
        {
            tp.SetValue(target, sp.GetValue(source));
        }
        return target;
    }

    // 泛型静态类：每一对 <TSource, TTarget> 各有一份，由运行时保证只初始化一次
    private static class PropertyPairs<TSource, TTarget>
    {
        public static readonly (PropertyInfo Source, PropertyInfo Target)[] Pairs = Build();

        private static (PropertyInfo, PropertyInfo)[] Build()
        {
            var sourceProps = typeof(TSource).GetProperties(Flags)
                .Where(p => p.CanRead)
                .ToDictionary(p => p.Name);

            return typeof(TTarget).GetProperties(Flags)
                .Where(tp => tp.CanWrite
                             && sourceProps.TryGetValue(tp.Name, out var sp)
                             && sp.PropertyType == tp.PropertyType)
                .Select(tp => (sourceProps[tp.Name], tp))
                .ToArray();
        }
    }
```

缓存之后，剩下的开销集中在 `GetValue` / `SetValue` 本身。它们的签名是 `object`，所以 `int`、`bool`、`decimal`、`DateTime`、`Guid` 每读一次都要装箱成一个堆对象。[.NET 7 起反射调用会在多次调用后切换到动态生成的 IL](https://devblogs.microsoft.com/dotnet/performance_improvements_in_net_7/?wt.mc_id=MVP_324329#reflection)，参数校验和装箱却省不掉。

### 3. 表达式树

表达式树的思路是：用反射把「赋值规则」描述成一棵语法树，等价于下面这个 lambda：

```csharp
source => new User { UserName = source.UserName, Email = source.Email, ... }
```

然后调用 `Compile()`，运行时会把这棵树编译成 IL 再交给 JIT，得到一个和手写 lambda 几乎一样的委托。构造过程如下：

```csharp
// 所在文件需要 using System.Linq.Expressions; 和 using System.Reflection;

/// <summary>
/// 构造等价于 source =&gt; new TTarget { A = source.A, B = source.B, ... } 的表达式树
/// </summary>
public static Expression<Func<TSource, TTarget>> BuildLambda<TSource, TTarget>() where TTarget : new()
{
    const BindingFlags flags = BindingFlags.Public | BindingFlags.Instance;
    var source = Expression.Parameter(typeof(TSource), "source");

    var sourceProps = typeof(TSource).GetProperties(flags)
        .Where(p => p.CanRead)
        .ToDictionary(p => p.Name);

    // 每个可写的目标属性生成一条 Target.X = source.X 绑定
    var bindings = typeof(TTarget).GetProperties(flags)
        .Where(tp => tp.CanWrite
                     && sourceProps.TryGetValue(tp.Name, out var sp)
                     && sp.PropertyType == tp.PropertyType)
        .Select(tp => Expression.Bind(tp, Expression.Property(source, sourceProps[tp.Name])));

    var body = Expression.MemberInit(Expression.New(typeof(TTarget)), bindings);
    return Expression.Lambda<Func<TSource, TTarget>>(body, source);
}
```

`Expression.Property` 读源属性，`Expression.Bind` 对应对象初始化器里的一条 `X = ...`，`Expression.MemberInit` 把 `new TTarget()` 和所有绑定组合成完整的对象初始化表达式。因为属性类型已经静态确定，整棵树里没有 `object`，也就没有装箱。

`Compile()` 是重操作：要遍历表达式树、生成 IL、创建动态方法、再 JIT。同样分两档来测，一档每次调用都 `Compile`，用来量出它的真实代价；另一档 `Compile` 一次后缓存委托：

```csharp
public static class ExpressionMapper
{
    /// <summary>每次调用都构造表达式树并 Compile：用来量出 Compile 本身的代价</summary>
    public static TTarget MapCompileEveryTime<TSource, TTarget>(TSource source) where TTarget : new()
        => BuildLambda<TSource, TTarget>().Compile()(source);

    /// <summary>Compile 一次，之后调用的就是一个普通委托</summary>
    public static TTarget MapCached<TSource, TTarget>(TSource source) where TTarget : new()
        => CompiledMapper<TSource, TTarget>.Map(source);

    private static class CompiledMapper<TSource, TTarget> where TTarget : new()
    {
        public static readonly Func<TSource, TTarget> Map = BuildLambda<TSource, TTarget>().Compile();
    }
}
```

AutoMapper、Mapster 这类运行时映射库的核心就是这个模式，只是规则配置、类型转换、嵌套对象处理要复杂得多。

表达式树还有一个容易忽视的限制：[NativeAOT 下不能在运行时生成代码](https://learn.microsoft.com/dotnet/core/deploying/native-aot/?wt.mc_id=MVP_324329#limitations-of-native-aot-deployment)，`Compile()` 会退化为解释执行表达式树，性能会大幅下降。

### 4. 源生成器

源生成器把「找同名属性」这件事挪到编译期。Roslyn 编译项目时会调用生成器，生成器读取语法树和符号信息，输出新的 C# 文件参与同一次编译。源生成器的基础概念可以参考 [C# 源代码生成器](/2020/06/15/SourceGenerators/)，这里直接写一个满足需求的最小增量生成器（`IIncrementalGenerator`）。

#### 4.1 生成器项目

生成器必须面向 `netstandard2.0`，因为它要被加载进编译器进程，而 Visual Studio 中的编译器仍运行在 .NET Framework 上：

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <IsRoslynComponent>true</IsRoslynComponent>
    <EnforceExtendedAnalyzerRules>true</EnforceExtendedAnalyzerRules>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.8.0" PrivateAssets="all" />
  </ItemGroup>

</Project>
```

生成器本身分三步：注入 `[GenerateMapper]` 特性，找到所有标了这个特性的类，为每个类输出一份实现。

```csharp
using System.Collections.Generic;
using System.Linq;
using System.Text;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Text;

namespace MapperBench.Generator;

[Generator]
public sealed class MapperGenerator : IIncrementalGenerator
{
    private const string AttributeFullName = "MapperBench.Generated.GenerateMapperAttribute";

    private const string AttributeSource = """
        namespace MapperBench.Generated;

        [System.AttributeUsage(System.AttributeTargets.Class)]
        internal sealed class GenerateMapperAttribute : System.Attribute
        {
        }
        """;

    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        // 1. 把特性本身注入到消费方的编译里，使用方无需额外引用
        context.RegisterPostInitializationOutput(ctx =>
            ctx.AddSource("GenerateMapperAttribute.g.cs", SourceText.From(AttributeSource, Encoding.UTF8)));

        // 2. 找到标了 [GenerateMapper] 的类，提前转成纯字符串模型
        var mappers = context.SyntaxProvider.ForAttributeWithMetadataName(
            AttributeFullName,
            predicate: static (node, _) => node is ClassDeclarationSyntax,
            transform: static (ctx, _) => Emit((INamedTypeSymbol)ctx.TargetSymbol));

        // 3. 每个类输出一个 .g.cs 文件
        context.RegisterSourceOutput(mappers, static (spc, file) =>
            spc.AddSource(file.HintName, SourceText.From(file.Source, Encoding.UTF8)));
    }

    private static GeneratedFile Emit(INamedTypeSymbol mapper)
    {
        var sb = new StringBuilder();
        sb.AppendLine("// <auto-generated />");
        sb.AppendLine("#nullable enable");
        sb.AppendLine($"namespace {mapper.ContainingNamespace.ToDisplayString()};");
        sb.AppendLine();
        sb.AppendLine($"static partial class {mapper.Name}");
        sb.AppendLine("{");

        // 只处理「static partial、没有实现体、一个参数、有返回值」的方法
        foreach (var method in mapper.GetMembers().OfType<IMethodSymbol>())
        {
            if (!method.IsStatic || !method.IsPartialDefinition
                || method.Parameters.Length != 1 || method.ReturnsVoid)
                continue;

            var source = method.Parameters[0];
            var target = method.ReturnType;
            var sourceProps = GetProperties(source.Type, p => p.GetMethod is not null)
                .ToDictionary(p => p.Name);

            sb.AppendLine($"    public static partial {target.ToDisplayString()} {method.Name}({source.Type.ToDisplayString()} {source.Name})");
            sb.AppendLine("    {");
            sb.AppendLine($"        return new {target.ToDisplayString()}");
            sb.AppendLine("        {");

            foreach (var tp in GetProperties(target, p => p.SetMethod is not null))
            {
                // 同名且同类型才赋值，其余属性（如 Id、CreatedAt）保持默认值
                if (sourceProps.TryGetValue(tp.Name, out var sp)
                    && SymbolEqualityComparer.Default.Equals(sp.Type, tp.Type))
                {
                    sb.AppendLine($"            {tp.Name} = {source.Name}.{tp.Name},");
                }
            }

            sb.AppendLine("        };");
            sb.AppendLine("    }");
        }

        sb.AppendLine("}");
        return new GeneratedFile($"{mapper.Name}.g.cs", sb.ToString());
    }

    private static IEnumerable<IPropertySymbol> GetProperties(ITypeSymbol type, System.Func<IPropertySymbol, bool> filter) =>
        type.GetMembers().OfType<IPropertySymbol>()
            .Where(p => !p.IsStatic && p.DeclaredAccessibility == Accessibility.Public && filter(p));

    // record 自带值相等，输入没变时增量管线会跳过输出阶段
    private sealed record GeneratedFile(string HintName, string Source);
}
```

几个值得注意的细节：

- `ForAttributeWithMetadataName` 是 Roslyn 4.3 起提供的 API，编译器内部对特性查找做了专门优化，比自己写 `CreateSyntaxProvider` 再逐个检查特性快得多。
- `transform` 阶段直接把符号转成字符串，管线里传递的是 `GeneratedFile` 这个 record 而不是 `ISymbol`。符号对象每次编译都会重建，放进管线会让增量缓存永远命中失败；record 按值比较，用户在别的文件里敲代码时，这一步的输出不变，输出阶段就会被跳过。
- `netstandard2.0` 没有 `IsExternalInit`，使用 record 需要在生成器项目里补一个空类型，代码见下方。
- 同名但类型不同的属性直接跳过。实际项目里应当用 `context.ReportDiagnostic` 报一条警告，否则字段漏映射又回到了手写时的老问题。

```csharp
// netstandard2.0 没有这个类型，record / init 需要它
namespace System.Runtime.CompilerServices;

internal static class IsExternalInit
{
}
```

#### 4.2 使用生成器

业务项目以 Analyzer 的方式引用生成器项目，`ReferenceOutputAssembly="false"` 表示运行时并不需要这个程序集：

```xml
<ItemGroup>
  <PackageReference Include="BenchmarkDotNet" Version="0.15.4" />
  <ProjectReference Include="..\MapperBench.Generator\MapperBench.Generator.csproj"
                    OutputItemType="Analyzer"
                    ReferenceOutputAssembly="false" />
</ItemGroup>
```

然后声明一个 partial 方法，不写实现：

```csharp
using MapperBench.Generated;

namespace MapperBench.Mappers;

[GenerateMapper]
public static partial class UserMapper
{
    public static partial User Map(CreateUserDto source);
}
```

在项目文件里加上 `<EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>`，编译后可以在 `obj/Release/net10.0/generated/` 下看到生成的 `UserMapper.g.cs`。下面是原样输出：

```csharp
// <auto-generated />
#nullable enable
namespace MapperBench.Mappers;

static partial class UserMapper
{
    public static partial MapperBench.User Map(MapperBench.CreateUserDto source)
    {
        return new MapperBench.User
        {
            UserName = source.UserName,
            Email = source.Email,
            DisplayName = source.DisplayName,
            Phone = source.Phone,
            Age = source.Age,
            IsActive = source.IsActive,
            Balance = source.Balance,
            BirthDate = source.BirthDate,
            TenantId = source.TenantId,
            Country = source.Country,
        };
    }
}
```

它和手写版本逐行一致。不同之处在于：DTO 新增一个同名属性后，重新编译就会自动多出一行赋值，在 IDE 里还可以直接 F12 跳进生成的代码、打断点调试。

### 5. 基准测试

#### 5.1 测试代码

六个基准方法对应前面的六种实现，手写赋值标记为 `Baseline`，`[MemoryDiagnoser]` 用来统计每次调用的内存分配：

```csharp
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net10_0)]
public class MapBenchmarks
{
    private readonly CreateUserDto _dto = SampleData.Dto;

    [Benchmark(Baseline = true)]
    public User HandWritten() => HandWrittenMapper.Map(_dto);

    [Benchmark]
    public User ReflectionNoCache() => ReflectionMapper.MapNoCache<CreateUserDto, User>(_dto);

    [Benchmark]
    public User ReflectionCached() => ReflectionMapper.MapCached<CreateUserDto, User>(_dto);

    [Benchmark]
    public User ExpressionCompileEveryTime() => ExpressionMapper.MapCompileEveryTime<CreateUserDto, User>(_dto);

    [Benchmark]
    public User ExpressionCached() => ExpressionMapper.MapCached<CreateUserDto, User>(_dto);

    [Benchmark]
    public User SourceGenerated() => UserMapper.Map(_dto);
}
```

性能比较的前提是结果正确。`Program.cs` 在启动基准之前，先把其余实现的输出与手写版本做递归比对，任何一处不一致就直接退出。这里同时覆盖了平铺的用户映射和带嵌套对象的订单映射：

```csharp
using System.Collections;
using BenchmarkDotNet.Running;
using MapperBench;
using MapperBench.Mappers;
using MapperBench.Mappers.Deep;

// 跑基准之前先确认各实现的输出与手写版本完全一致，否则比较没有意义
var dto = SampleData.Dto;
var expectedUser = HandWrittenMapper.Map(dto);
var users = new Dictionary<string, User>
{
    ["ReflectionNoCache"] = ReflectionMapper.MapNoCache<CreateUserDto, User>(dto),
    ["ReflectionCached"] = ReflectionMapper.MapCached<CreateUserDto, User>(dto),
    ["ExpressionCompileEveryTime"] = ExpressionMapper.MapCompileEveryTime<CreateUserDto, User>(dto),
    ["ExpressionCached"] = ExpressionMapper.MapCached<CreateUserDto, User>(dto),
    ["SourceGenerated"] = UserMapper.Map(dto),
};

var orderDto = SampleData.CreateOrder(10);
var expectedOrder = HandWrittenOrderMapper.Map(orderDto);
var orders = new Dictionary<string, Order>
{
    ["DeepReflection"] = DeepReflectionMapper.Map<CreateOrderDto, Order>(orderDto),
    ["DeepExpression"] = DeepExpressionMapper.Map<CreateOrderDto, Order>(orderDto),
    ["DeepSourceGenerated"] = OrderMapper.Map(orderDto),
};

if (expectedOrder.Lines.Count != 10 || expectedOrder.ShippingAddress is null || expectedOrder.BillingAddress is not null)
{
    Console.Error.WriteLine("手写订单映射结果不符合预期");
    return 1;
}

foreach (var (name, actual) in users.Select(p => (p.Key, (object)p.Value))
             .Concat(orders.Select(p => (p.Key, (object)p.Value))))
{
    var expected = actual is User ? (object)expectedUser : expectedOrder;
    if (!DeepEquals(expected, actual))
    {
        Console.Error.WriteLine($"{name} 与手写版本不一致");
        return 1;
    }
}
Console.WriteLine("所有实现输出一致");

if (args.Contains("--verify-only")) return 0;

BenchmarkSwitcher.FromAssembly(typeof(Program).Assembly).Run(args);
return 0;

// 递归比较：值类型和 string 比值，List 逐项比，其它对象逐属性比
static bool DeepEquals(object? a, object? b)
{
    if (a is null || b is null) return a is null && b is null;
    if (a.GetType() != b.GetType()) return false;
    if (a is string || a.GetType().IsValueType) return a.Equals(b);
    if (a is IList la && b is IList lb)
        return la.Count == lb.Count && Enumerable.Range(0, la.Count).All(i => DeepEquals(la[i], lb[i]));

    return a.GetType().GetProperties().All(p => DeepEquals(p.GetValue(a), p.GetValue(b)));
}
```

运行命令：

```bash
# 两组基准全部运行
dotnet run -c Release -- --filter '*'

# 只运行订单映射
dotnet run -c Release -- --filter '*OrderMapBenchmarks*'
```

> 在 Windows 上，如果项目放在很深的目录里，BenchmarkDotNet 会在 `bin/Release/net10.0/` 下再生成一个子项目来构建，中间文件路径很容易超过 260 个字符，报错表现为 `MSB3030: Could not copy the file ...\apphost.exe because it was not found`。把解决方案挪到一个短路径下即可。

#### 5.2 测试结果

BenchmarkDotNet 输出的测试环境原样如下：

```
BenchmarkDotNet v0.15.4, Windows 11 (10.0.26200.9448)
Intel Core Ultra 7 155H 1.40GHz, 1 CPU, 22 logical and 16 physical cores
.NET SDK 10.0.204
  [Host]    : .NET 10.0.8 (10.0.8, 10.0.826.23019), X64 RyuJIT x86-64-v3
  .NET 10.0 : .NET 10.0.8 (10.0.8, 10.0.826.23019), X64 RyuJIT x86-64-v3

Job=.NET 10.0  Runtime=.NET 10.0
```

下表数值原样取自 BenchmarkDotNet 报告，省略了 Error、Median、Gen0 等辅助列：

| 实现 | 平均耗时 | 标准差 | 相对基线 | 每次分配 |
|---|---:|---:|---:|---:|
| 手写赋值（基线） | 15.04 ns | 1.676 ns | 1.01 | 120 B |
| 反射，每次 GetProperties | 544.72 ns | 82.506 ns | 36.63 | 376 B |
| 反射，缓存 PropertyInfo | 197.13 ns | 18.311 ns | 13.26 | 256 B |
| 表达式树，每次 Compile | 179,975.20 ns | 5,775.731 ns | 12,102.73 | 9344 B |
| 表达式树，缓存委托 | 14.89 ns | 1.761 ns | 1.00 | 120 B |
| 源生成器 | 13.39 ns | 0.679 ns | 0.90 | 120 B |

六个结果横跨四个数量级，换成对数刻度更容易看出分组：

![六种映射方式的单次耗时](/assets/imgs/dto-mapping-benchmark.svg)

#### 5.3 结果解读

**第一组：手写、源生成器、缓存后的表达式树，三者没有实质差别。** 源生成器的 13.39 ns 比手写的 15.04 ns 低，但两者编译出的是逐行相同的代码，这 1.65 ns 只是测量波动。测试机是一台混合架构（性能核加能效核）的笔记本，各行的标准差都在 10% 左右，Ratio 列 0.90 ± 0.10 也说明了这一点。表达式树缓存委托后同样落在这个区间：委托调用多一次间接跳转，但相比创建对象、复制十个字段，这点开销测不出来。

**第二组：反射，比基线慢一个数量级以上。** 缓存 `PropertyInfo` 后是 197 ns，约为基线的 13 倍；不缓存是 545 ns，约 37 倍。缓存消除了大约三分之二的耗时，说明朴素写法里元数据查找比读写值本身还贵。

分配列能精确地对上账：

| 实现 | 分配 | 构成 |
|---|---:|---|
| 手写 / 源生成器 / 缓存表达式树 | 120&nbsp;B | 一个 `User` 对象 |
| 反射（缓存） | 256&nbsp;B | `User` 对象，加上 `int`、`bool`、`DateTime` 各装箱 24 B，`decimal`、`Guid` 各装箱 32 B，合计 136 B |
| 反射（不缓存） | 376&nbsp;B | 再加上 `GetProperties` 每次新建的 12 元素 `PropertyInfo[]`，120 B |

也就是说，反射缓存后的额外分配全部来自装箱。属性里值类型越多，这部分开销越大。

**第三组：每次都 `Compile` 的表达式树，约 180 μs，是基线的一万两千倍。** 每次调用都要分配约 9 KB 内存，生成一个新的动态方法并 JIT。这一行的意义是说明 `Compile` 有多贵：只要被放进了请求路径，比如在没有缓存的工具方法里每次都 `Compile`，它就是整个映射链路里最大的瓶颈。即便缓存了，这 180 μs 也会在每个类型对第一次使用时付一次，类型越多，冷启动越慢。

换一个更直观的尺度，一个接口每秒处理一万次注册请求，四种做法每秒花在映射上的 CPU 时间分别是：

| 实现 | 每秒耗时 |
|---|---:|
| 手写 / 源生成器 / 缓存表达式树 | 约 0.15 ms |
| 反射（缓存） | 约 2 ms |
| 反射（不缓存） | 约 5.4 ms |
| 表达式树（每次 Compile） | 约 1.8 s |

除了最后一种，前三种在一次涉及数据库往返的请求里都只是零头。反射的代价不在单次延迟，而在高吞吐场景下累积的 CPU 和 GC 压力。

### 6. 更复杂的映射：嵌套对象与集合

`User` 只有平铺的字段，映射逻辑就是十次赋值。真实的请求往往更复杂，比如下单请求里带着收货地址对象和一个明细列表。这时映射不再是一条直线，而是要递归创建子对象、循环映射列表元素。理论上，逻辑越复杂，表达式树和源生成器之间越可能拉开差距：动态方法不参与分层编译，享受不到动态 PGO；委托也不能被调用方内联。下面用实测来检验。

#### 6.1 订单模型

```csharp
/// <summary>前端提交的下单请求：含嵌套对象和集合</summary>
public sealed class CreateOrderDto
{
    public string CustomerName { get; set; } = "";
    public string Email { get; set; } = "";
    public DateTime OrderDate { get; set; }
    public decimal Discount { get; set; }
    public string? Remark { get; set; }
    public AddressDto ShippingAddress { get; set; } = new();
    public AddressDto? BillingAddress { get; set; }
    public List<OrderLineDto> Lines { get; set; } = [];
}

public sealed class AddressDto
{
    public string Country { get; set; } = "";
    public string City { get; set; } = "";
    public string Street { get; set; } = "";
    public string PostalCode { get; set; } = "";
}

public sealed class OrderLineDto
{
    public Guid ProductId { get; set; }
    public string ProductName { get; set; } = "";
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
}
```

实体一侧的 `Order`、`Address`、`OrderLine` 属性同名，`Order` 和 `OrderLine` 各多一个 `Id`，`Order` 还多一个 `CreatedAt`。测试数据里 `BillingAddress` 为 `null`，用来覆盖嵌套对象为空的分支。

映射规则在平铺版「同名同类型才赋值」的基础上扩充为四条：

- 类型相同，直接赋值；
- 两边都是 `List<T>`，新建目标列表并逐个映射元素；
- 两边都是有无参构造函数的类，递归映射；
- 源值为 `null` 时，目标也为 `null`。

三种实现都按这四条规则写成通用代码，而不是只针对订单类型。简单版的类保持不变，新逻辑放在单独的类和单独的生成器里。

#### 6.2 反射：运行时按类型对查缓存

平铺版可以用泛型静态类缓存属性对，嵌套之后就不行了：子对象的类型只能在运行时从 `PropertyType` 取得，没法作为泛型参数，只能用 `(源类型, 目标类型)` 作为键，到 `ConcurrentDictionary` 里查映射计划。

```csharp
private static object? MapObject(object? source, Type sourceType, Type targetType)
{
    if (source is null) return null;

    var target = Activator.CreateInstance(targetType)!;
    foreach (var step in Plans.GetOrAdd((sourceType, targetType), BuildPlan))
    {
        var value = step.Source.GetValue(source);
        value = step.Kind switch
        {
            Kind.Object => MapObject(value, step.Source.PropertyType, step.Target.PropertyType),
            Kind.List => MapList((IList?)value, step.SourceElement!, step.TargetElement!, step.Target.PropertyType),
            _ => value,
        };
        step.Target.SetValue(target, value);
    }
    return target;
}

private static IList? MapList(IList? source, Type sourceElement, Type targetElement, Type targetListType)
{
    if (source is null) return null;

    var result = (IList)Activator.CreateInstance(targetListType, source.Count)!;
    foreach (var item in source)
    {
        result.Add(sourceElement == targetElement ? item : MapObject(item, sourceElement, targetElement));
    }
    return result;
}
```

`BuildPlan` 在第一次遇到某个类型对时，按四条规则把每个属性归类为 `Copy`、`Object` 或 `List`，结构与平铺版的 `PropertyPairs.Build` 相同。每个对象都要多付一次字典查找、一次 `Activator.CreateInstance`，列表还要通过非泛型的 `IList` 操作，枚举器会被装箱。

#### 6.3 表达式树：把循环也写进树里

表达式树可以表达完整的控制流，包括局部变量、条件和循环。嵌套对象生成一段带空值判断的 `MemberInit`，列表则直接在树里生成一个 `for` 循环，最终编译出的委托里没有任何反射：

```csharp
private static Expression? BuildValue(Expression value, Type targetType)
{
    if (value.Type == targetType)
        return value;

    if (MappingRules.IsList(value.Type, out var se) && MappingRules.IsList(targetType, out var te))
        return NullSafe(value, targetType, v => BuildList(v, se, te, targetType));

    if (MappingRules.IsMappableClass(value.Type) && MappingRules.IsMappableClass(targetType))
        return NullSafe(value, targetType, v => BuildNew(v, targetType));

    return null;
}

/// <summary>var v = value; v == null ? null : build(v)</summary>
private static Expression NullSafe(Expression value, Type targetType, Func<Expression, Expression> build)
{
    var v = Expression.Variable(value.Type);
    return Expression.Block(targetType, [v],
        Expression.Assign(v, value),
        Expression.Condition(
            Expression.Equal(v, Expression.Constant(null, value.Type)),
            Expression.Constant(null, targetType),
            build(v),
            targetType));
}

/// <summary>
/// 等价于：
/// var result = new List&lt;TT&gt;(source.Count);
/// for (var i = 0; i &lt; count; i++) result.Add(map(source[i]));
/// </summary>
private static Expression BuildList(Expression source, Type sourceElement, Type targetElement, Type targetListType)
{
    var result = Expression.Variable(targetListType, "result");
    var count = Expression.Variable(typeof(int), "count");
    var i = Expression.Variable(typeof(int), "i");
    var item = Expression.Variable(sourceElement, "item");
    var end = Expression.Label("end");

    var element = BuildValue(item, targetElement)
        ?? throw new NotSupportedException($"{sourceElement} -> {targetElement}");

    return Expression.Block(targetListType, [result, count, i],
        Expression.Assign(count, Expression.Property(source, "Count")),
        Expression.Assign(result, Expression.New(targetListType.GetConstructor([typeof(int)])!, count)),
        Expression.Assign(i, Expression.Constant(0)),
        Expression.Loop(
            Expression.IfThenElse(
                Expression.LessThan(i, count),
                Expression.Block([item],
                    Expression.Assign(item, Expression.Property(source, "Item", i)),
                    Expression.Call(result, targetListType.GetMethod("Add")!, element),
                    Expression.PostIncrementAssign(i)),
                Expression.Break(end)),
            end),
        result);
}
```

`BuildNew` 与平铺版的 `BuildLambda` 主体相同，只是对每个属性改为调用 `BuildValue`。`NullSafe` 先把源值存进局部变量，保证属性只读取一次。代码量明显上来了，这也是表达式树方案真实的维护成本：树里写错一个类型，要到运行时 `Compile` 才会抛出异常。

#### 6.4 源生成器：为每个类型对生成一个私有方法

生成器新增一个 `[GenerateDeepMapper]` 特性。处理属性时，遇到嵌套类型对就放进队列，稍后为它生成一个私有的 `__Map` 重载：

```csharp
/// <summary>同类型直接赋值；List 和可映射的类调用 __Map，并保留 null</summary>
private static string? Value(string access, ITypeSymbol source, ITypeSymbol target,
    Queue<(ITypeSymbol, ITypeSymbol)> pending)
{
    if (SymbolEqualityComparer.Default.Equals(source, target))
        return access;

    if ((IsList(source, out _) && IsList(target, out _))
        || (IsMappableClass(source) && IsMappableClass(target)))
    {
        pending.Enqueue((source, target));
        return $"{access} is null ? null : __Map({access})";
    }

    return null;
}
```

声明方式与平铺版相同：

```csharp
[GenerateDeepMapper]
public static partial class OrderMapper
{
    public static partial Order Map(CreateOrderDto source);
}
```

生成结果原样如下：

```csharp
// <auto-generated />
#nullable disable
namespace MapperBench.Mappers.Deep;

static partial class OrderMapper
{
    public static partial MapperBench.Order Map(MapperBench.CreateOrderDto source)
    {
        return new MapperBench.Order
        {
            CustomerName = source.CustomerName,
            Email = source.Email,
            OrderDate = source.OrderDate,
            Discount = source.Discount,
            Remark = source.Remark,
            ShippingAddress = source.ShippingAddress is null ? null : __Map(source.ShippingAddress),
            BillingAddress = source.BillingAddress is null ? null : __Map(source.BillingAddress),
            Lines = source.Lines is null ? null : __Map(source.Lines),
        };
    }

    private static MapperBench.Address __Map(MapperBench.AddressDto source)
    {
        return new MapperBench.Address
        {
            Country = source.Country,
            City = source.City,
            Street = source.Street,
            PostalCode = source.PostalCode,
        };
    }

    private static System.Collections.Generic.List<MapperBench.OrderLine> __Map(System.Collections.Generic.List<MapperBench.OrderLineDto> source)
    {
        var result = new System.Collections.Generic.List<MapperBench.OrderLine>(source.Count);
        foreach (var item in source)
        {
            result.Add(item is null ? null : __Map(item));
        }
        return result;
    }

    private static MapperBench.OrderLine __Map(MapperBench.OrderLineDto source)
    {
        return new MapperBench.OrderLine
        {
            ProductId = source.ProductId,
            ProductName = source.ProductName,
            Quantity = source.Quantity,
            UnitPrice = source.UnitPrice,
        };
    }
}
```

手写基线 `HandWrittenOrderMapper` 与这份生成代码结构一致，同样是三个私有 `Map` 重载加一个 `foreach` 循环。这个生成器有一个已知边界：同一个源类型如果要映射到两个不同的目标类型，两个 `__Map` 重载的参数相同，会编译失败，实际使用时需要按目标类型区分方法名。

#### 6.5 基准测试与结果

明细行数用 `[Params]` 分为 1、10、100 三档。平铺场景已经说明了不缓存的写法没有意义，这里只比较四种稳态方案：

```csharp
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net10_0)]
public class OrderMapBenchmarks
{
    [Params(1, 10, 100)]
    public int LineCount;

    private CreateOrderDto _dto = null!;

    [GlobalSetup]
    public void Setup() => _dto = SampleData.CreateOrder(LineCount);

    [Benchmark(Baseline = true)]
    public Order HandWritten() => HandWrittenOrderMapper.Map(_dto);

    [Benchmark]
    public Order ReflectionCached() => DeepReflectionMapper.Map<CreateOrderDto, Order>(_dto);

    [Benchmark]
    public Order ExpressionCached() => DeepExpressionMapper.Map<CreateOrderDto, Order>(_dto);

    [Benchmark]
    public Order SourceGenerated() => OrderMapper.Map(_dto);
}
```

测试环境与平铺场景相同。下表是平均耗时，数值原样取自 BenchmarkDotNet 报告：

| 实现 | 1 行 | 10 行 | 100 行 |
|---|---:|---:|---:|
| 手写赋值（基线） | 41.57 ns | 114.24 ns | 878.84 ns |
| 反射，缓存 | 633.34 ns | 1,557.31 ns | 15,490.46 ns |
| 表达式树，缓存委托 | 39.64 ns | 114.93 ns | 852.48 ns |
| 源生成器 | 38.04 ns | 147.36 ns | 880.61 ns |

每次映射的内存分配：

| 实现 | 1 行 | 10 行 | 100 行 |
|---|---:|---:|---:|
| 手写 / 表达式树 / 源生成器 | 288 B | 1008 B | 8208 B |
| 反射，缓存 | 920 B | 2432 B | 17552 B |

![订单映射的单次耗时](/assets/imgs/dto-mapping-order-benchmark.svg)

10 行那一档里，源生成器比手写慢了 30%，标准差 31.379 ns，是手写的近三倍，BenchmarkDotNet 还对它给出了「分布呈多峰」的警告。两者的代码结构完全相同，这个差距不合常理，于是单独把这两组重跑了一遍：

| 实现 | 1 行 | 10 行 | 100 行 |
|---|---:|---:|---:|
| 手写赋值（基线） | 49.29 ns | 129.63 ns | 753.62 ns |
| 源生成器 | 40.31 ns | 107.31 ns | 745.39 ns |

重跑后差距的方向反了过来，源生成器反而快 17%。同一份代码两次运行能差出这么多，说明在这台笔记本上，十几纳秒到二三十纳秒的差异都不能当真，只有成倍的差距才有意义。这也是读基准结果时值得记住的一点：单次运行里的「赢家」未必可信，可疑的数字要重跑确认。

#### 6.6 结果解读

**表达式树仍然和手写持平，前面的理论推测没有被实测证实。** 1 行、10 行、100 行三档，缓存委托的表达式树都落在手写基线的误差范围内。原因在于映射的主要成本是分配对象和复制字段：每多一行明细就要创建一个 `OrderLine`，这部分开销对所有实现都一样。表达式树版本把循环直接写进了树里，整个订单只在入口处调用一次委托，循环体内没有额外的间接调用，动态 PGO 能优化的空间也很小。如果改用「每个元素调用一次缓存的子委托」的写法，每行会多一次委托调用，这种写法没有测，不能套用这里的结论。

**源生成器与手写相同。** 两轮结果互相抵消，结论与平铺场景一致。

**反射的差距随数据量拉大。** 三档分别是基线的 15.5 倍、13.8 倍和 18.0 倍。按 100 行折算，手写每行明细约 8.5 ns，反射约 150 ns。分配同样能对上账：

| 实现 | 每行增量 | 构成 |
|---|---:|---|
| 手写 / 表达式树 / 源生成器 | 80&nbsp;B | 一个 `OrderLine` 对象 72 B，加上列表底层数组的一个槽位 8 B |
| 反射 | 168&nbsp;B | 再加上 `Guid`、`int`、`decimal` 三个值类型各装箱一次，共 88 B |

1 行时反射比手写多出 632 B，除去这一行的 88 B 装箱，其余来自 `Order` 上两个值类型的装箱、带参数的 `Activator.CreateInstance` 以及装箱后的列表枚举器。对象图越大，反射在装箱和动态创建实例上付出的代价就越多。

**结论没有变：** 稳态下表达式树和源生成器都能达到手写的性能，反射是唯一明显落后的方案。复杂映射拉开的不是这两者的运行时性能，而是开发体验。两者的代码量都明显增加：表达式树映射器写到了近百行，生成器也从 92 行增加到 161 行。区别在于出错的方式：表达式树里写错一个类型，要到运行时 `Compile` 才会抛出异常，而且生成的委托无法单步调试；源生成器拼错代码，会在编译期直接报错，生成的结果也是可以阅读、可以打断点的 C# 代码。

### 7. 怎么选

性能之外，三种方式在工程上的差别同样重要：

| 维度 | 反射 | 表达式树 | 源生成器 |
|---|---|---|---|
| 稳态性能 | 慢 13 到 18 倍，有装箱分配 | 缓存后与手写相当，嵌套与集合下同样如此 | 与手写相同 |
| 冷启动 | 首次查元数据 | 每个类型对首次约 180 μs | 无额外开销 |
| NativeAOT / 裁剪 | 需要保留元数据 | `Compile` 退化为解释执行 | 完全兼容 |
| 可调试性 | 只能调试通用循环 | 委托体无法单步进入 | F12 跳进生成代码直接打断点 |
| 映射错误何时暴露 | 运行时 | 运行时 | 编译时（可报诊断） |
| 类型何时必须已知 | 运行时 | 运行时 | 编译时 |
| 实现成本 | 最低 | 中等 | 最高，需要单独的生成器项目 |

据此可以得出几条结论：

- **新项目的 DTO 到实体映射，优先用源生成器。** 映射的源类型和目标类型在编译时就确定，正是源生成器最擅长的场景。不必自己写生成器，Mapperly、Facet 等库已经提供了成熟实现。
- **已有的反射映射代码，第一步是加缓存。** 一个泛型静态类就能把耗时砍掉约三分之二；再往下优化，就要换成表达式树或源生成器。
- **表达式树适合类型在运行时才确定的场景。** 例如根据配置动态映射、插件系统、通用的导入导出工具。用的时候务必缓存编译后的委托，并且注意 NativeAOT 下的退化问题。
- **任何方案都不要在请求路径上 `Compile`。** 一万两千倍的差距，足以让一个不起眼的工具方法成为整个接口的瓶颈。

### 引用与资源

- [表达式树（C#）](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/?wt.mc_id=MVP_324329)
- [源生成器概述](https://learn.microsoft.com/dotnet/csharp/roslyn-sdk/source-generators-overview?wt.mc_id=MVP_324329)
- [增量生成器设计文档](https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md?wt.mc_id=MVP_324329)
- [Native AOT 部署及其限制](https://learn.microsoft.com/dotnet/core/deploying/native-aot/?wt.mc_id=MVP_324329)
- [Performance Improvements in .NET 7：反射部分](https://devblogs.microsoft.com/dotnet/performance_improvements_in_net_7/?wt.mc_id=MVP_324329#reflection)
- [BenchmarkDotNet](https://benchmarkdotnet.org/)
