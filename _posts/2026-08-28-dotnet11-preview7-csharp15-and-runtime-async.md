---
layout: post
title:  ".NET 11 Preview 7 中的 C# 15 与 runtime-async 更新"
date:   2026-08-28 14:00:00 +0800--
categories: [.NET]
tags: [C#15, dotnet11, async, WebAssembly]
---

### 前言

[.NET 11 Preview 7](https://devblogs.microsoft.com/dotnet/dotnet-11-preview-7/?wt.mc_id=MVP_324329) 同时更新了语言与运行时实验。本文分成两部分来看：前半部分是 C# 15 的预览语法和编译器诊断，后半部分是 `runtime-async` 与 CoreCLR WebAssembly 的实现进展。

这些功能的成熟度并不相同。C# 15 中的 union、`closed` 类型和 Unsafe Evolution 仍可能调整；`runtime-async` 与 WebAssembly 上的 CoreCLR 也不应仅凭当前基准或测试进展视为生产替代方案。

### 1. 带标签的 break 和 continue

`break` 和 `continue` 现在可以指名一个外层的循环或 `switch`，直接从内层跳出或继续外层。

以前处理这种控制流，常见办法包括标志位、`goto`、提取方法或使用局部函数。下面先看标志位写法：

```csharp
string? foundValue = null;
bool found = false;
for (int x = 0; x < xMax && !found; x++)
{
    for (int y = 0; y < yMax; y++)
    {
        if (GetValue(x, y) is { } value && value == target)
        {
            foundValue = value;
            found = true;
            break;
        }
    }
}
```

也可以使用 `goto`，但不少团队会因为控制流可读性而限制它。

现在给目标循环加个标签，然后在 `break` 里指名：

```csharp
string? foundValue = null;
outer: for (int x = 0; x < xMax; x++)
{
    for (int y = 0; y < yMax; y++)
    {
        if (GetValue(x, y) is { } value && value == target)
        {
            foundValue = value;
            break outer;
        }
    }
}
```

同一个标签也能驱动 `continue`——跳过本行剩下的列，直接进入下一行：

```csharp
row: for (int i = 0; i < rows; i++)
{
    for (int j = 0; j < cols; j++)
    {
        if (ShouldSkipRestOfRow(i, j)) continue row;
        Process(i, j);
    }
}
```

两条规则需要记住：标签必须直接附着在 `for`、`foreach`、`while`、`do` 或 `switch` 语句上，而带标签的 `break` / `continue` 必须出现在该语句内部。

带标签的写法可以省掉只为退出嵌套循环而存在的状态，但不应机械替换所有标志位。标签较多或跨越较长代码块时，提取方法并通过返回值结束流程通常更容易阅读。

### 2. 联合类型的模式匹配：Try-Both

当前 C# 15 预览设计中的 `union` 类型采用 **Try-Both** 匹配策略：模式作用于一个联合值时，编译器先测试联合实例本身，如果没有匹配，再用同一模式测试联合内含的 `Value`。类型模式、`var` 模式、声明模式、列表模式和递归模式都适用。

![union 的 Try-Both 匹配](/assets/imgs/dotnet11-union-try-both.svg)

```csharp
public record class Dog(string Name);
public record class Cat(int Lives);

public union Pet(Dog, Cat);

Pet pet = new Cat(9);

// 匹配联合实例本身
if (pet is Pet)              // true —— 联合类型本身就是合法的匹配目标
    Console.WriteLine("got a pet");

// 匹配内含的值
if (pet is Cat { Lives: > 0 } cat)
    Console.WriteLine($"cat has {cat.Lives} lives");
```

配合这次调整，绑定期的类型检查在输出值收窄到具体分支类型后，不再继续传播联合类型信息。自定义联合声明则使用新的 `UnionMatchingMode` 属性控制模式如何降级，取代此前的临时标志。这里描述的是 Preview 7 的设计，后续预览仍可能调整属性和匹配规则。

联合类型仍然是预览特性，需要 `<LangVersion>preview</LangVersion>` 才能用，接口也还会继续变。

### 3. 泛型参数约束到封闭类型时的穷尽性

`switch` 表达式的穷尽性检查现在认得约束到 `closed` 类型的泛型参数。当封闭基类型的每个直接子类型都被处理了，编译器不再警告 `switch` 不完备——即便输入的静态类型是那个类型参数，而不是基类型本身。

```csharp
public closed record class Shape;
public record class Circle(double Radius) : Shape;
public record class Square(double Side) : Shape;

static double Area<T>(T shape) where T : Shape => shape switch
{
    Circle(var r) => Math.PI * r * r,
    Square(var s) => s * s
};
```

根据 `closed` 层次结构提供的信息，编译器可以把 `T` 的分支限制在已声明的派生类型内。此前这里仍会报告 switch 不完备，开发者通常需要添加 `_ =>` 兜底；保留兜底后，将来新增派生类型时也无法再依靠穷尽性诊断发现遗漏。

Preview 7 还调整了封闭层次结构的元数据表示，`IsClosedTypeAttribute` 现在带有 `DerivedTypes` 属性。由于 `closed` 仍是预览设计，这里不把当前格式视为最终契约。

这项改动减少了仅为消除穷尽性诊断而添加兜底分支的需要。另一个 C# 15 预览功能见[集合表达式参数](/2026/02/19/csharp15-collection-expression-arguments/)。

### 4. Unsafe Evolution：兼容模式与 nameof

Unsafe Evolution 仍是预览中的安全规则调整，目标是区分仅引用相关成员和实际执行非托管内存操作的场景。Preview 7 包含两项后续规则：

- **兼容模式扩展到旧调用方。** 在更新后的内存安全规则下标记为 requires-unsafe 的成员，即使被尚未选用新规则的代码调用，也要求 `unsafe` 上下文。这样可以避免调用方仅升级部分语言设置后，在安全上下文中绕过成员声明的 requires-unsafe 要求。
- **`nameof` 不再报 requires-unsafe 错误。** 在 `nameof(...)` 里引用 requires-unsafe 成员不再触发 unsafe 上下文错误，这与 `nameof` 处理其他大多数成员种类的方式一致。

Unsafe Evolution 同样是预览特性，具体规则和诊断都还可能变。

### 5. 内联数组的合成辅助方法加了空检查

`[InlineArray]` 值的元素访问和 `Span<T>` 转换会被降级成 `<PrivateImplementationDetails>` 里的合成辅助方法调用：`InlineArrayElementRef`、`InlineArrayElementRefReadOnly`、`InlineArrayAsSpan`、`InlineArrayAsReadOnlySpan`。

这些辅助方法此前可以接收空 byref，再按偏移量返回 `ref` 或 `Span<T>`。经由 `Unsafe.NullRef<T>()` 等路径，安全代码也可能触发未受保护的内存访问。

Preview 7 为传入的缓冲区增加空检查，从空的内联数组引用构造 `ref` 或 span 时会抛出 `NullReferenceException`。在 JIT 能证明基址非空的常见路径上，这项检查可以被消除，因此不应仅凭新增检查推断普通元素访问会产生固定开销。

这一条不需要改任何源码，用 Preview 7 的编译器重新构建即可。

### 6. runtime-async：异步版本走完分层编译

`runtime-async` 是仍在演进中的运行时内建 `async`/`await` 实现。Preview 7 重点补齐分层编译，并针对若干异步模式增加 JIT 优化。下面的数字来自官方发布说明中的特定基准，用于解释实现变化，不代表普通应用会获得相同比例的提升。

![runtime-async 走完分层编译](/assets/imgs/dotnet11-runtime-async-tiering.svg)

#### 6.1 异步版本纳入分层编译

方法的异步版本现在进入分层编译管线。此前异步版本会停留在偏向快速编译的 tier0 代码，无法获得后续层级针对稳态执行的优化；这在官方测试负载的预热阶段表现为较高的分配速率。

两项 tier0 层面的改动让这次过渡本身也变得更便宜：

- **tail-await 优化下放到 tier0。** 官方在 TechEmpower `platform-json` 场景中记录到，预热期最大分配速率从 110,580,952 B/s 降到 8,030,616 B/s。该数据只描述这一配置下的预热表现。
- **JIT 可以内联 `AsyncHelpers.TransparentAwait`。** 对已经完成的 `Task` 执行热路径 `await` 时，可以折叠为任务状态检查。官方微基准中，一亿次循环的耗时从约 191 ms 降到约 32 ms；这种紧凑循环会放大调用开销，不能直接换算成应用延迟。

#### 6.2 Task 与 ValueTask 工厂方法的内建识别

JIT 现在能识别出现在异步版本里的常见 `Task` / `ValueTask` 工厂方法，并把它们折叠成直接的快速路径。覆盖 `Task.FromResult`、`Task.CompletedTask`、`ValueTask.FromResult`、`ValueTask.CompletedTask`、`default(ValueTask)`、`new ValueTask()` 和 `new ValueTask<T>(T)`。

`Task` 和 `ValueTask` 之间的异步适配包装也会被识别并拆掉：

```csharp
[MethodImpl(MethodImplOptions.NoInlining)]
private static ValueTask TestTaskToValueTask() => new ValueTask(TestTask());

[MethodImpl(MethodImplOptions.NoInlining)]
private static async Task TestTask() => await Task.Yield();
```

在官方给出的这个示例中，外层 `new ValueTask(...)` 生成代码从 115 字节降到 29 字节：原来的栈帧、构造函数和辅助方法调用被简化为尾调用与状态检查。这说明 JIT 能识别该适配模式，不意味着所有 `Task`/`ValueTask` 转换都会得到同样结果。

#### 6.3 尾调用与 await Task.Yield()

当被调用方返回的任务正是方法自己要返回的那个时，异步方法的隐式尾调用重新启用了。一个在两个 `async Task` 方法之间做分派的方法（`return b ? Bar() : Baz();`）从 52 字节、19 条指令降到 20 字节、6 条指令，两个分支都变成了 `tail.jmp`。

runtime-async 方法里的 `await Task.Yield()` 现在可以跳过内部线程池装箱分配，方式与状态机模型使用 `IStateMachineBoxAwareAwaiter` 类似。官方一千万次循环的微基准从约 723 ms 降到约 534 ms，并在该测试中接近编译器生成状态机的结果。

另外还修了个 bug：`await` 现在会正确地为返回 `ValueTask` 的方法保存和恢复异步上下文，此前一处标志检查错误导致这一步被跳过了。

这些优化不要求应用改写对应的异步方法。只有在使用相同 runtime-async 配置并升级到 Preview 7 后，才适合比较预热期分配和稳态吞吐；如果基准或告警阈值按旧实现建立，应在相同硬件、负载和预热策略下重新测量。

### 7. CoreCLR 跑上 WebAssembly

.NET 11 正在实验让 CoreCLR 运行在 WebAssembly 上，采用可移植解释器与 ReadyToRun 组合，并复用 RyuJIT 生成提前编译代码。Preview 7 的进展是可以端到端运行库测试套件；这是实现里程碑，不等于已经满足生产部署要求。

相关工作包括迁移到 WebAssembly 的 `exnref` 异常处理提案、补充 RyuJIT 的 SIMD 与 ReadyToRun 支持、增加 CoreCLR-WASI host，以及让 cDAC 支持 WebAssembly 栈遍历。它们分别解决异常、代码生成、测试宿主和诊断能力问题。

Emscripten 工具链升级到 6.0.2，浏览器运行时改用 `-O3` 链接。官方发布说明报告了产物大小和性能改善，但实际影响仍需在具体应用中测量。

运行库测试套件通过并不表示 Blazor WebAssembly 已经切换到 CoreCLR，也不表示当前方案具备生产支持。现阶段更适合把它理解为验证 CoreCLR、RyuJIT 和诊断组件能否在 WebAssembly 环境中协同工作的基础进展。

### 小结

C# 15 的这些改动分别涉及控制流表达、联合类型匹配、穷尽性诊断和 unsafe 规则。带标签跳转可以减少某些嵌套循环中的辅助状态，但是否更易读仍要结合代码规模判断；union、`closed` 类型和 Unsafe Evolution 目前都应按预览设计使用。

runtime-async 在 Preview 7 中获得了分层编译和若干 JIT 快速路径。官方基准说明这些优化解决了特定实现开销，但是否改善应用性能仍需要在相同配置下测量。CoreCLR WebAssembly 则仍处于验证运行时组件和测试覆盖的阶段。

完整清单在[官方发布说明](https://github.com/dotnet/core/discussions/10529?wt.mc_id=MVP_324329)里，另外 [C# 15 的新增功能](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-15?wt.mc_id=MVP_324329)会随每个预览版更新。

同一批发布里偏工程侧的改动——MSBuild Server 和 NativeAOT CLI 默认开启、`dotnet test` 的运行级策略、Blazor 的电路自动暂停与 `CacheView`——见 [.NET 11 Preview 7：CLI、构建、测试与 Blazor 更新](/2026/08/21/dotnet11-preview7-build-and-cost/)。
