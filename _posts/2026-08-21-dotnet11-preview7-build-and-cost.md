---
layout: post
title:  ".NET 11 Preview 7：CLI、构建、测试与 Blazor 更新"
date:   2026-08-21 10:30:00 +0800--
categories: [.NET]
tags: [dotnet11, MSBuild, NativeAOT, Blazor]
---

### 前言

[.NET 11 Preview 7](https://devblogs.microsoft.com/dotnet/dotnet-11-preview-7/?wt.mc_id=MVP_324329) 发布于 2026 年 8 月 11 日，是 GA 之前的最后一个 Preview——9 月和 10 月各有一个 RC，正式版定在 11 月 10 日。

这篇不逐项复述发布说明，而是挑出三类值得升级时检查的内容：默认行为发生变化的 NativeAOT CLI 和 MSBuild Server、CI 中可直接使用的测试选项，以及需要显式启用的 Blazor 资源管理功能。前两类可能在不改项目代码的情况下改变行为，后一类则需要安装或配置后才会生效。

### 1. NativeAOT 版 dotnet CLI 默认开启

Preview 6 里，`dotnet` 命令的 NativeAOT 快速路径藏在 `DOTNET_CLI_ENABLEAOT` 开关后面。Preview 7 把默认值翻了过来：AOT 入口在所有平台上默认启用，包括之前被平台闸门挡住的 macOS 和 Linux。

#### 1.1 省下的是第二次 CoreCLR 启动

收益来自跳过托管 CLI 回退时的那次额外 CoreCLR 启动，官方给出的对照数据：

| 命令 | 托管 CLI | NativeAOT | 变化 |
| --- | ---: | ---: | --- |
| `dotnet tool list` | 378 ms | 68 ms | 快 5.5 倍 |
| `dotnet dev-certs https`、`dotnet ef` 一类的工具分发 | 约 700 ms | 200–220 ms | 快 3.2–3.5 倍 |

这些是官方发布说明中的测试结果，用来说明跳过托管 CLI 启动的潜在收益。实际耗时仍会受机器配置、磁盘状态和命令负载影响，不宜把这组数字当成每个开发环境都能复现的结果。

#### 1.2 只有一部分命令走 AOT

AOT 快速路径目前只覆盖部分命令。例如 `dotnet --info`、`sdk check`、部分 `sln` 和本地工具命令，以及全局工具、本地工具和 PATH 上外部命令的解析与调用，都可以走 AOT 入口。

需要在进程内使用 MSBuild 或 NuGet 的 `build`、`run`、`test`、`pack` 和 `publish` 等命令仍会回退到托管 CLI。支持范围还在演进，完整列表应以对应 Preview 的[官方发布说明](https://github.com/dotnet/core/discussions/10529?wt.mc_id=MVP_324329)为准。

因此，`dotnet build` 本身不会因为 NativeAOT CLI 默认开启而获得这里列出的启动收益。连续构建的启动开销主要与下一节的 MSBuild Server 有关。

#### 1.3 关掉它

```bash
# 只对单条命令关闭 AOT 路径
DOTNET_CLI_ENABLEAOT=false dotnet --info
```

`false`、`0`、`no`、`off` 都算关闭。这是一项被官方列入破坏性变更清单的改动，如果发现某条命令的输出格式或行为和 .NET 10 不一致，先用这个开关对比一次，能很快定位是不是 AOT 路径引起的。

### 2. MSBuild Server 默认开启

MSBuild Server 的作用是在两次 CLI 调用之间保留一个热的 MSBuild 工作进程，让接连执行的 `dotnet build`、`dotnet test`、`dotnet run` 跳过 MSBuild 的启动过程。

![MSBuild Server 默认开启前后](/assets/imgs/dotnet11-msbuild-server.svg)

Preview 6 做的是"CLI 不再无条件覆盖 `MSBUILDUSESERVER`"，Preview 7 则直接把默认值改成开启。

#### 2.1 关掉它

```bash
# 两个变量任选其一，都会退回经典的单次性 MSBuild 行为
export DOTNET_CLI_USE_MSBUILD_SERVER=false
export MSBUILDUSESERVER=0
```

`DOTNET_CLI_USE_MSBUILD_SERVER=false` 会向下转发 `MSBUILDUSESERVER=0`。在 Preview 7 的规则下，响应文件、`MSBUILDFORCEMULTITHREADED=1` 或 `/mt` 不会再次将它开启。需要排查服务器是否仍被使用时，可以优先检查这个变量和诊断日志。

#### 2.2 怎么确认服务器到底有没有生效

Preview 7 新增了 `MSBuildServerLifecycleEventArgs` 构建事件，会报告这次构建是新建了服务器、新建了短命服务器、复用了已有服务器，还是根本没用服务器，并附带服务器进程 ID。

该事件以低重要性记录，因此默认的控制台输出不受影响，但它会出现在二进制日志里，也会在 `-v:diag` 下显示：

```bash
dotnet build -v:diag
# 或者抓 binlog 之后用 MSBuild Structured Log Viewer 打开
dotnet build -bl
```

需要判断一次构建为何冷启动时，可以从这个事件和服务器进程 ID 入手，而不是只根据构建耗时推测。

#### 2.3 与多线程构建和节点复用的关系

- 在 Preview 7 的实现中，`-mt` 构建通过 MSBuild Server 使用 Server GC。因此即使设置了 `-nr:false`，仍会启动一个构建结束后退出的短命服务器，以避免跨构建复用。
- 非 `-mt` 的构建在 `-nr:false` 下仍然不使用服务器，这一点没有变化。
- 协调器协议增加了嵌套授权，用于解决任务再次调起 MSBuild 时可能出现的嵌套构建死锁。

这些规则属于 Preview 7 的当前实现。升级后建议用 `-v:diag` 或 binlog 确认自己的构建是否创建、复用或绕过了服务器。

### 3. dotnet test 多了两个运行级策略

在 Microsoft.Testing.Platform 下，`dotnet test` 新增了两个**运行级**选项。位置很关键：写在 `--` **之前**是整个测试运行的策略，写在 `--` 之后则退化为按测试应用逐个生效。

```bash
# 整个测试运行超过 90 秒就中止，退出码 3（TestSessionAborted）
dotnet test --timeout 90s

# 所有测试应用累计失败 5 个就停下来，退出码 13
dotnet test --maximum-failed-tests 5
```

`--timeout` 接受 `ms`、`s`、`m` 后缀，而且计时只在至少有一个测试应用正在运行时推进——挂在排队上的时间不算。两个策略都通过一条新的反向控制管道下发协作式取消，SDK 会协商到 MTP 2.4 以便宿主能响应 `CancelSession` 消息。

两个选项解决的是不同问题：`--timeout` 用于限制挂起或异常缓慢的测试运行，`--maximum-failed-tests` 则适合在失败数量达到阈值后尽早结束。CI 可以根据测试规模和失败诊断需求分别设置，而不是用其中一个替代另一个。

同批还有几个顺手的改进：

```bash
# 发现的测试可以直接输出成 JSON 给工具消费，不用再去扒人类可读的输出
dotnet test --list-tests json
```

多模块运行时，`dotnet test` 现在会在给出最终摘要之前自动合并各测试应用的 TRX 与代码覆盖率产物；如果更希望每个测试应用留一份自己的产物，用 `--no-artifact-post-processing` 关掉。另外，整个解决方案的运行不再因为某一个项目一个测试都没匹配上就判定失败——结论现在按全量测试数计算一次，但每个模块的 `Exit code: 8` 诊断信息仍然保留。

### 4. 容器发布会优先挑平台原生的运行时

SDK 的容器发布流程现在认得平台原生的本地容器 CLI：Windows 上的 `wslc` 和 macOS 上的 `container`，它们排在 Docker 和 Podman 前面。选择是自动的，装了谁就用谁，Docker 和 Podman 依次作为兜底。

每种运行时现在各自处理就绪探测、归档格式、加载命令、清单和多平台能力。部分引擎差异因此可以给出更有针对性的错误，例如 WSLC 不支持多架构本地加载时会报告对应限制；这并不表示不同容器引擎之间的兼容性问题都已消失。

想固定用某一个，显式设置 `LocalRegistry`，它现在接受 `Docker`、`Podman`、`Wslc` 和新的 `MacOSContainer`：

```xml
<PropertyGroup>
  <PublishProfile>DefaultContainer</PublishProfile>
  <LocalRegistry>Wslc</LocalRegistry>
</PropertyGroup>
```

这同样被列为一项破坏性变更：装有 `wslc` 或 macOS `container` 的机器可能不再默认选择 Docker。依赖特定引擎行为的构建机应显式设置 `LocalRegistry`，并重新验证镜像加载和多平台发布流程。另外，独立的 `containerize` CLI 不再随 SDK 打包。

### 5. Blazor 的两个资源使用相关改动

#### 5.1 电路在标签页隐藏时自动暂停

Interactive Server 的 Circuit 会在服务器上保存状态。Preview 6 加入了 `Circuit.RequestCircuitPauseAsync`，Preview 7 又提供了根据标签页可见性请求暂停的可选功能。它可能减少长期隐藏标签页对应的闲置资源，但实际效果取决于连接数量、应用状态大小和用户行为，需要通过应用自己的指标评估。

![Blazor 电路自动暂停的流程](/assets/imgs/dotnet11-blazor-circuit-pause.svg)

这是一个独立的可选包：

```xml
<PackageReference Include="Microsoft.AspNetCore.Components.Server.AutoPause" />
```

```csharp
app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode()
    .WithBrowserOptions(options =>
    {
        options.AddAutoPause(pause =>
        {
            pause.Enabled = true;                        // 默认值
            pause.HiddenDelay = TimeSpan.FromSeconds(30); // 默认是 2 分钟
        });
    });
```

暂停在可能丢数据或打断工作的场景下会自动推迟，内置的条件包括：值与默认值不同的文本输入框或 `contenteditable` 元素（含 shadow DOM 和同源 iframe 里的）、正在播放且未静音的 `<audio>` / `<video>`、打开着的画中画窗口、持有中的 Web Lock，以及任何进行中的电路活动——未完成的 `IJSRuntime` 调用、`DotNetStreamReference` / `JSStreamReference` 传输、排队中的渲染。

应用自己的业务状态需要自己兜。注册一个实现了 `onCircuitPausing` 的客户端电路处理器，Blazor 在暂停前会等待每一个已注册的处理器——自动暂停和服务器主动发起的暂停都会等：

```javascript
// wwwroot/{ASSEMBLY NAME}.lib.module.js
export function beforeWebStart(options) {
  options.circuit ??= {};
  options.circuit.circuitHandlers ??= [];

  options.circuit.circuitHandlers.push({
    onCircuitPausing: async (signal) => {
      // 持久化进行中的状态。如果暂停被取消（比如标签页又可见了），signal 会中止。
      await savePendingWork(signal);
    },
  });
}
```

配合这项改动，`Circuit.RequestCircuitPauseAsync` 的返回类型变成了 `Task<bool>`，并接受一个可选的取消令牌，这样连接被拆除时可以中止推迟暂停的收尾工作。

#### 5.2 CacheView：把 SSR 渲染结果缓存下来

`CacheView` 是一个新组件，用于缓存服务端渲染子树的输出。命中缓存时，子组件不会再次实例化和渲染，而是回放此前捕获的 HTML，因此可以减少重复 SSR 工作。收益取决于命中率和子树的渲染成本，缓存本身也会占用内存或分布式存储空间。

```razor
@using Microsoft.AspNetCore.Components

<CacheView ExpiresAfter="TimeSpan.FromMinutes(10)"
           VaryByRoute="productId"
           VaryByQuery="page,pageSize"
           VaryByCulture="true">
    <ExpensiveProductSummary ProductId="productId" />
</CacheView>
```

过期策略支持绝对（`ExpiresAfter`、`ExpiresOn`）和滑动（`ExpiresSliding`）；vary-by 的维度覆盖查询字符串、路由、请求头、Cookie、已认证用户（`VaryByUser`）、当前区域性（`VaryByCulture`），以及任意字符串 `VaryBy`。同一个父级下有多个缓存边界时，用 `CacheKey` 区分。非 GET 请求、`Enabled="false"`、以及流式 SSR 内部都会跳过缓存。

默认使用受 `RazorComponentsServiceOptions.CacheViewSizeLimit`（默认 100 MB）限制的内存存储。如果应用注册了 `HybridCache`，`CacheView` 会从 DI 中使用它，从而支持本地与分布式缓存组合：

```csharp
builder.Services.AddHybridCache();

builder.Services.AddRazorComponents()
    .AddInteractiveServerComponents();
```

缓存键和失效策略需要覆盖所有会改变输出的请求状态。遗漏用户、区域性、查询参数或权限维度，可能把不该共享的 HTML 返回给其他请求。

输出依赖每请求状态的组件可以通过 `[CacheBehavior]` 和 `[CacheCondition]` 标注：`CacheBehavior.Rerender` 让组件在缓存命中时重新渲染；`CacheBehavior.Throw` 则拒绝不满足缓存条件的组合。几个有代表性的内置组件如下：

| 组件 | 标注 | 效果 |
| --- | --- | --- |
| `AuthorizeView` | `Throw` + `CacheCondition(CacheVaryBy.User)` | 必须设置 `VaryByUser="true"` |
| `QuickGrid<TGridItem>` | `Throw` + `CacheCondition(CacheVaryBy.Query)` | 必须设置 `VaryByQuery` |

违规时的报错会点名组件并给出修法：

```text
System.InvalidOperationException: Component 'QuickGrid`1[...]' cannot be used inside a CacheView
because its output depends on per-request state ([CacheBehavior(CacheBehavior.Throw)],
[CacheCondition(CacheVaryBy.Query)]) that cannot be safely cached and replayed. To fix this,
configure the CacheView to vary by Query, or move the component outside the CacheView.
```

### 6. 顺手要留意的两处行为变化

#### 6.1 CSRF 中间件改回只校验显式加入的端点

Preview 6 引入的自动跨域 CSRF 保护会默认校验每一个不安全的请求。Preview 7 收窄了范围：中间件现在只校验元数据里带有 `IAntiforgeryMetadata { RequiresValidation: true }` 的端点，和经典的 `AntiforgeryMiddleware` 规则一致。

这意味着没有防伪元数据的裸 `MapPost` 和裸 MVC `[HttpPost]` 端点会直接放行，与 .NET 10 的行为相同——在 .NET 10 上能跑的端点，不会一升到 .NET 11 就开始失败。Blazor SSR 表单、Razor Pages / MVC 的表单绑定，以及绑定表单数据的最小 API 处理器（`[FromForm]`、`IFormFile`、`IFormCollection`）依然受保护，因为它们都会自动附加 `RequireAntiforgeryTokenAttribute`。

想让某个端点重新纳入校验，要么让它绑定表单数据，要么给它加 `[ValidateAntiForgeryToken]`；`DisableAntiforgery()` 仍然是退出的方式。另外 `QUERY` 现在被当作 CSRF 与防伪意义上的安全 HTTP 方法。

Preview 6 升上来的项目值得专门回归一遍：**上一版收紧、这一版放宽**，中间那段时间写下的防护假设可能已经不成立了。

#### 6.2 NoBuild=true 不再构建项目引用

SDK 项目在 `NoBuild=true` 时会把 `BuildProjectReferences` 默认为 `false`，于是 `dotnet publish --no-build` 和 `dotnet pack --no-build` 不再触发那个隐蔽的 `NETSDK1085`。

如果流水线恰好依赖 `--no-build` 仍然去构建过期的项目引用，需要显式设置 `BuildProjectReferences=true` 才能恢复旧行为：

```bash
dotnet publish --no-build -p:BuildProjectReferences=true
```

### 小结

### 升级检查清单

NativeAOT CLI、MSBuild Server、容器运行时选择、CSRF 校验范围和 `NoBuild=true` 属于可能改变现有行为的部分；Blazor AutoPause 与 `CacheView` 则需要显式安装或配置，不会仅因升级 SDK 自动进入应用。

升级时可以先做三项检查：

1. 运行 `dotnet build -v:diag` 或查看 binlog，确认 MSBuild Server 的生命周期符合预期。
2. 检查 CI 机器是否安装了 `wslc` 或 macOS `container`；如果流水线依赖 Docker，显式指定 `LocalRegistry`。
3. 回归 Preview 6 期间围绕自动 CSRF 保护建立的测试，确认 Preview 7 收窄校验范围后仍符合安全预期。

更完整的清单在[官方发布说明](https://github.com/dotnet/core/discussions/10529?wt.mc_id=MVP_324329)和 [.NET 11 的新增功能](https://learn.microsoft.com/dotnet/core/whats-new/dotnet-11/overview?wt.mc_id=MVP_324329)里。
