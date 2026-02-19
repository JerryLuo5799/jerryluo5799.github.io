---
layout: post
title:  ".NET Framework升级.NET8之九: 如何管理不同目标框架"
date:   2024-07-03 21:42:55 +0800--
categories: [.NET]
tags: [NET Framework升级.NET8]  
---

### 前言

将项目迁移到新的目标框架（TFM）是一项复杂的任务，尤其是在处理不同TFM之间的兼容性问题时。建议将迁移相关的PBIs（产品待办事项）集中处理，并将主分支过渡到新的TFM。做出这一判断需要仔细考虑PBIs的数量及其预计完成时间等因素。

以下是一些在管理与新旧TFM不兼容的更改时的主要方案：

### 使用#if编译指令

可以使用 编译指示指令来专门为某个TFM编译代码。这种技术还可以简化迁移后清理过程中不兼容代码段的移除工作。

如果可能，考虑使用依赖注入或工厂模式，根据目标TFM注入适当的实现。这种方法通过抽象TFM特定的细节，提高了代码的灵活性和可维护性。

```cs
public static class WebClientFactory
{
  public static IWebClient GetWebClient()
  {
#if NET472
    return new CustomWebClient();
#else
    return new CustomHttpClient();
#endif
  }
}
```

### 使用MSBuild条件

可以使用MSBuild条件来添加仅与特定TFM兼容的不同库的引用。这使你能够根据使用的TFM动态管理引用。

```cs
<ItemGroup Condition="'$(TargetFramework)' == 'net472'">
    <Reference Include="System.Web" />
    <Reference Include="System.Web.Extensions" />
    <Reference Include="System.Web.ApplicationServices" />
</ItemGroup>
```