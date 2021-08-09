---
layout: post
title:  "使用PuppeteerSharp实现监听抓取WebSocket消息"
date:   2021-07-30 21:42:55 +0800--
categories: [.NET Core]
tags: [.NET Core, PuppeteerSharp, WebSocket]  
---

### 1. 前言
最新在项目中遇到了一个监听系统Socket消息的需求, 由于项目是基于.NET Core的, 最后使用**PuppeteerSharp**实现了该功能。

### 2. 备选方案

#### 2.1 方案一
方案一是采用模拟的方案, 通过各种接口获取到参数, 然后利用对应参数直接发起Socket连接, 这种方式感觉是再造一个虚拟前端, 有点小复杂, 最后放弃使用。

#### 2.2 方案二
在研究站点请求的时候，突发奇想，PuppeteerSharp如果可以像Chrome一样打开开发者工具就好了，因为开发者工具里面可以直接监听所有的网络请求， 最后发现该方案可行， 并且很简单就实现了对WebSocket消息的监听

### 3. 实现

#### 3.1 安装SDK包
   
```csharp
dotnet add package PuppeteerSharp
```

#### 3.2 引用命名空间
   
```csharp
using PuppeteerSharp;
```
#### 3.3 关键代码
   
```csharp
public async Task ListenWebSocket()
{
    var launchOptions = new LaunchOptions()
    {
        Headless = true,
        ExecutablePath = [Chrome浏览器安装路径]
    };
    ns);
    var page = await browser.NewPageAsync();
    var navOptions = new NavigationOptions
    {
        WaitUntil = new[] { WaitUntilNavigation.Networkidle2 }
    };
    //打开页面地址
    await page.GoToAsync([页面地址], navOptions);

    //关键是下面的步骤...

    //打开浏览器的开发者工具, 监听Socket
    await client.SendAsync("Network.enable");
    client.MessageReceived += OnChromeDevProtocolMessage;
}

/// <summary>
/// Socket消息监听和处理
/// </summary>
/// <param name="sender"></param>
/// <param name="eventArgs"></param>
public void OnChromeDevProtocolMessage(object sender, MessageEventArgseventArgs)
{
    if (eventArgs.MessageID == "Network.webSocketCreated")
    {
        Console.WriteLine($"Network.webSocketCreated: {eventArgs.MessageData}");
    }
    else if (eventArgs.MessageID == "Network.webSocketFrameSent")
    {
        Console.WriteLine($"Network.webSocketFrameSent: {eventArgs.MessageData}");
    }
    else if (eventArgs.MessageID == "Network.webSocketFrameReceived")
    {
        Console.WriteLine($"Network.webSocketFrameReceived: {eventArgs.MessageData}");
    }
}

```

**更多Chrome开发者工具协议可参考: [点击此处](https://chromedevtools.github.io/devtools-protocol/tot/Network/)**

