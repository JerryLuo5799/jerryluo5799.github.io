---
layout: post
title:  "使用Azure Application Insights监测你的应用 (一) - 如何使用"
date:   2020-02-29 21:42:55 +0800--
categories: [Azure]
tags: [Application Insights, 监控平台, 日志系统]  
---

### 00 前言
随着应用系统越来越复杂, 如何监测应用的运行情况, 如何排查线上问题成为了一个很大的难点, 大部分企业会使用开源的组件来搭建自己的日志平台, 但是, 如果你希望可以减轻技术人员和运维人员的压力, Azure Application Insights是一个不错的选择。

### 01 什么是Azure Application Insights
>Application Insights 是 Azure Monitor 的一项功能，是面向开发人员和 DevOps 专业人员的可扩展应用程序性能管理 (APM) 服务。 使用它可以监视实时应用程序。 它将自动检测性能异常，并且包含了强大的分析工具来帮助诊断问题，了解用户在应用中实际执行了哪些操作。 它旨在帮助持续提高性能与可用性。 它适用于本地云、混合云或任何公有云中托管的各种平台（包括 .NET、Node.js 和 Java EE）中的应用。 它与 DevOps 进程集成，并且具有与不同开发工具的连接点。 可以通过与 Visual Studio App Center 集成来监视和分析移动应用的遥测数据。

### 02 工作原理
![原理图](https://docs.azure.cn/zh-cn/azure-monitor/app/media/app-insights-overview/01-scheme.png)

如图所示, Azure Application Insights主要由以下几部分组成:
1. 日志记录组件, 不仅可以检测 Web 服务应用程序，还可以检测所有后台组件以及前端代码
2. Application Insights服务本身提供了强大的分析和搜索工具
3. 强大的第三方集成工具，可以方便的整合到其他系统

### 03 监测内容

>Application Insights 主要面向开发团队，旨在帮助用户了解应用的运行性能和使用方式。 监视：
1. 请求率、响应时间和失败率 - 了解最受欢迎的页面、时段以及用户的位置。 查看哪些页面效果最好。 当有较多请求时，如果响应时间长且失败率高，则可能存在资源问题。
2. 依赖项速率、响应时间和失败率 - 了解外部服务是否正拖慢速度。
异常 - 分析聚合的统计信息，或选择特定实例并钻取堆栈跟踪和相关请求。 报告服务器和浏览器异常。
3. 页面查看次数和负载性能 - 由用户的浏览器报告。
4. AJAX 调用（从网页） - 速率、响应时间和失败率。
用户和会话计数。
5. Windows 或 Linux 服务器计算机中的性能计数器，例如 CPU、内存和网络使用情况。
6. Docker 或 Azure 中的主机诊断。
7. 应用中的诊断跟踪日志- 可以将跟踪事件与请求相关联。
8. 在客户端或服务器代码中自行编写的自定义事件和指标，用于跟踪业务事件（例如销售的商品或赢得的游戏）。

### 04 创建应用
登录Azure门户, 依次点击 **All resources | Add | DevOps | Application Insights**, 进入应用的创建界面
![创建Application Insights](/assets/imgs/ApplicationInsightsCreate.png)
选择/创建资源组, 输入实例名称, 选择实例所在的区域, 点击下一步, 即可创建完成。
创建完成后，打开实例的**Overview**页面， 找到对应的**Instrumentation Key**
![Application Insights Overview](/assets/imgs/ApplicationInsightsOverview.png)

### 05 .NET Core中启用Application Insights
1. 安装Application Insights SDK包
   
   ```csharp
   dotnet add package Microsoft.ApplicationInsights.AspNetCore
   ```
2. 在Startup类中启用Application Insights

```csharp
    public void ConfigureServices(IServiceCollection services)
    {
        // 必须在services.AddMvc()前面
        services.AddApplicationInsightsTelemetry();

        // This code adds other services for your application.
        services.AddMvc();
    }   
```
3. 在appsettings.json文件中, 添加配置
```Json
   {
      "ApplicationInsights": {
        "InstrumentationKey": "[上述创建应用的Instrumentation Key]"
      }
    }
```
### 06 JavaScript中启用Application Insights
```JavaScript
<script type="text/javascript">
var sdkInstance="appInsightsSDK";window[sdkInstance]="appInsights";var aiName=window[sdkInstance],aisdk=window[aiName]||function(n){var o={config:n,initialize:!0},t=document,e=window,i="script";setTimeout(function(){var e=t.createElement(i);e.src=n.url||"https://az416426.vo.msecnd.net/scripts/b/ai.2.min.js",t.getElementsByTagName(i)[0].parentNode.appendChild(e)});try{o.cookie=t.cookie}catch(e){}function a(n){o[n]=function(){var e=arguments;o.queue.push(function(){o[n].apply(o,e)})}}o.queue=[],o.version=2;for(var s=["Event","PageView","Exception","Trace","DependencyData","Metric","PageViewPerformance"];s.length;)a("track"+s.pop());var r="Track",c=r+"Page";a("start"+c),a("stop"+c);var u=r+"Event";if(a("start"+u),a("stop"+u),a("addTelemetryInitializer"),a("setAuthenticatedUserContext"),a("clearAuthenticatedUserContext"),a("flush"),o.SeverityLevel={Verbose:0,Information:1,Warning:2,Error:3,Critical:4},!(!0===n.disableExceptionTracking||n.extensionConfig&&n.extensionConfig.ApplicationInsightsAnalytics&&!0===n.extensionConfig.ApplicationInsightsAnalytics.disableExceptionTracking)){a("_"+(s="onerror"));var p=e[s];e[s]=function(e,n,t,i,a){var r=p&&p(e,n,t,i,a);return!0!==r&&o["_"+s]({message:e,url:n,lineNumber:t,columnNumber:i,error:a}),r},n.autoExceptionInstrumented=!0}return o}(
{
  instrumentationKey:"[上述创建应用的Instrumentation Key]"
}
);(window[aiName]=aisdk).queue&&0===aisdk.queue.length&&aisdk.trackPageView({});
</script>
```
### 07 SDK支持情况
1. 官方支持 .NET, .NET Core，Java，Node.js，JavaScript几种语言;
2. 对于其他语言和框架, 如Python，PHP, React等, 社区提供了对应的SDK包

大部分语言和框架都可以直接找到对应的SDK, 方便快捷的启用Application Insights

### 08 Azure Global VS Azure中国
**Note: 由于Azure Global和Azure中国的节点并不互通, 所以如果Application Insights的实例是创建在Azure中国上的, 需要额外配置终结点**

1. .NET Core

```json
"ApplicationInsights": {
    "InstrumentationKey": "[上述创建应用的Instrumentation Key]",
    "TelemetryChannel": {
      "EndpointAddress": "[Azure中国的终结点地址:https://dc.applicationinsights.azure.cn/v2/track]"
    }
  }
```

2. JavaScript
```JavaScript
<script type="text/javascript">
    var sdkInstance="appInsightsSDK";window[sdkInstance]="appInsights";var aiName=window[sdkInstance],aisdk=window[aiName]||function(e){function n(e){t[e]=function(){var n=arguments;t.queue.push(function(){t[e].apply(t,n)})}}var t={config:e};t.initialize=!0;var i=document,a=window;setTimeout(function(){var n=i.createElement("script");n.src=e.url||"https://az416426.vo.msecnd.net/scripts/b/ai.2.min.js",i.getElementsByTagName("script")[0].parentNode.appendChild(n)});try{t.cookie=i.cookie}catch(e){}t.queue=[],t.version=2;for(var r=["Event","PageView","Exception","Trace","DependencyData","Metric","PageViewPerformance"];r.length;)n("track"+r.pop());n("startTrackPage"),n("stopTrackPage");var s="Track"+r[0];if(n("start"+s),n("stop"+s),n("setAuthenticatedUserContext"),n("clearAuthenticatedUserContext"),n("flush"),!(!0===e.disableExceptionTracking||e.extensionConfig&&e.extensionConfig.ApplicationInsightsAnalytics&&!0===e.extensionConfig.ApplicationInsightsAnalytics.disableExceptionTracking)){n("_"+(r="onerror"));var o=a[r];a[r]=function(e,n,i,a,s){var c=o&&o(e,n,i,a,s);return!0!==c&&t["_"+r]({message:e,url:n,lineNumber:i,columnNumber:a,error:s}),c},e.autoExceptionInstrumented=!0}return t}(
    {
      instrumentationKey:"[上述创建应用的Instrumentation Key]",
      endpointUrl: "[Azure中国的终结点地址:https://dc.applicationinsights.azure.cn/v2/track]"
    }
    );window[aiName]=aisdk,aisdk.queue&&0===aisdk.queue.length&&aisdk.trackPageView({});
</script>
```