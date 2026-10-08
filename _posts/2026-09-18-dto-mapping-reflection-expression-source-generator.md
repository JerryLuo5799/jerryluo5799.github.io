---
layout: post
title:  "反射、表达式树与源生成器：DTO 到实体映射的性能对比"
date:   2026-09-18 10:00:00 +0800--
categories: [.NET]
tags: [Source Generators, Expression, Reflection, BenchmarkDotNet]
---

### 前言

把前端提交的请求 DTO 转换成数据库实体，是几乎每个 Web API 都在写的代码。项目里可能用的是手写赋值、AutoMapper、Mapster，或者 [Facet](/2025/08/15/using-facet-for-mapping/) 这类新一代映射库，但无论封装成什么样子，底层只有三条技术路线：

- **反射**：运行时通过 `PropertyInfo` 读写属性；
- **表达式树**：运行时用 `System.Linq.Expressions` 拼出赋值逻辑，`Compile` 成委托后反复调用；
- **源生成器**：编译时由 Roslyn 生成普通的 C# 赋值代码，运行时没有任何额外机制。

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

性能比较的前提是结果正确。`Program.cs` 在启动基准之前，先把另外五种实现的输出与手写版本逐个属性比对，任何一处不一致就直接退出：

```csharp
// 跑基准之前先确认 6 种实现的输出完全一致，否则比较没有意义
var dto = SampleData.Dto;
var expected = HandWrittenMapper.Map(dto);
var candidates = new Dictionary<string, User>
{
    ["ReflectionNoCache"] = ReflectionMapper.MapNoCache<CreateUserDto, User>(dto),
    ["ReflectionCached"] = ReflectionMapper.MapCached<CreateUserDto, User>(dto),
    ["ExpressionCompileEveryTime"] = ExpressionMapper.MapCompileEveryTime<CreateUserDto, User>(dto),
    ["ExpressionCached"] = ExpressionMapper.MapCached<CreateUserDto, User>(dto),
    ["SourceGenerated"] = UserMapper.Map(dto),
};

foreach (var (name, actual) in candidates)
{
    foreach (var prop in typeof(User).GetProperties())
    {
        if (!Equals(prop.GetValue(expected), prop.GetValue(actual)))
        {
            Console.Error.WriteLine($"{name}.{prop.Name} 不一致");
            return 1;
        }
    }
}
Console.WriteLine("6 种实现输出一致");

if (args.Contains("--verify-only")) return 0;

BenchmarkRunner.Run<MapBenchmarks>(args: args);
return 0;
```

运行命令：

```bash
dotnet run -c Release -- --filter '*'
```

> 在 Windows 上，如果项目放在很深的目录里，BenchmarkDotNet 会在 `bin/Release/net10.0/` 下再生成一个子项目来构建，中间文件路径很容易超过 260 个字符，报错表现为 `MSB3030: Could not copy the file ...\apphost.exe because it was not found`。把解决方案挪到一个短路径下即可。

#### 5.2 测试结果

BenchmarkDotNet 输出的环境信息与结果表原样如下：

```
BenchmarkDotNet v0.15.4, Windows 11 (10.0.26200.9448)
Intel Core Ultra 7 155H 1.40GHz, 1 CPU, 22 logical and 16 physical cores
.NET SDK 10.0.204
  [Host]    : .NET 10.0.8 (10.0.8, 10.0.826.23019), X64 RyuJIT x86-64-v3
  .NET 10.0 : .NET 10.0.8 (10.0.8, 10.0.826.23019), X64 RyuJIT x86-64-v3

Job=.NET 10.0  Runtime=.NET 10.0
```

| Method                     | Mean          | Error        | StdDev       | Median        | Ratio     | RatioSD  | Gen0   | Allocated | Alloc Ratio |
|--------------------------- |--------------:|-------------:|-------------:|--------------:|----------:|---------:|-------:|----------:|------------:|
| HandWritten                |      15.04 ns |     0.587 ns |     1.676 ns |      14.63 ns |      1.01 |     0.15 | 0.0095 |     120 B |        1.00 |
| ReflectionNoCache          |     544.72 ns |    29.775 ns |    82.506 ns |     529.10 ns |     36.63 |     6.70 | 0.0296 |     376 B |        3.13 |
| ReflectionCached           |     197.13 ns |     6.648 ns |    18.311 ns |     191.72 ns |     13.26 |     1.83 | 0.0203 |     256 B |        2.13 |
| ExpressionCompileEveryTime | 179,975.20 ns | 3,576.717 ns | 5,775.731 ns | 179,126.27 ns | 12,102.73 | 1,299.26 | 0.4883 |    9344 B |       77.87 |
| ExpressionCached           |      14.89 ns |     0.624 ns |     1.761 ns |      14.22 ns |      1.00 |     0.16 | 0.0095 |     120 B |        1.00 |
| SourceGenerated            |      13.39 ns |     0.315 ns |     0.679 ns |      13.29 ns |      0.90 |     0.10 | 0.0095 |     120 B |        1.00 |

六个结果横跨四个数量级，换成对数刻度更容易看出分组：

![六种映射方式的单次耗时](/assets/imgs/dto-mapping-benchmark.svg)

#### 5.3 结果解读

**第一组：手写、源生成器、缓存后的表达式树，三者没有实质差别。** 源生成器的 13.39 ns 比手写的 15.04 ns 低，但两者编译出的是逐行相同的代码，这 1.65 ns 只是测量波动。测试机是一台混合架构（性能核加能效核）的笔记本，各行的标准差都在 10% 左右，Ratio 列 0.90 ± 0.10 也说明了这一点。表达式树缓存委托后同样落在这个区间：委托调用多一次间接跳转，但相比创建对象、复制十个字段，这点开销测不出来。

**第二组：反射，比基线慢一个数量级以上。** 缓存 `PropertyInfo` 后是 197 ns，约为基线的 13 倍；不缓存是 545 ns，约 37 倍。缓存消除了大约三分之二的耗时，说明朴素写法里元数据查找比读写值本身还贵。

分配列能精确地对上账：

| 实现 | 分配 | 构成 |
|---|---:|---|
| 手写 / 源生成器 / 缓存表达式树 | 120 B | 一个 `User` 对象 |
| 反射（缓存） | 256 B | `User` 对象，加上 `int`、`bool`、`DateTime` 各装箱 24 B，`decimal`、`Guid` 各装箱 32 B，合计 136 B |
| 反射（不缓存） | 376 B | 再加上 `GetProperties` 每次新建的 12 元素 `PropertyInfo[]`，120 B |

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

### 6. 怎么选

性能之外，三种方式在工程上的差别同样重要：

| 维度 | 反射 | 表达式树 | 源生成器 |
|---|---|---|---|
| 稳态性能 | 慢 13 倍以上，有装箱分配 | 缓存后与手写相当 | 与手写相同 |
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
