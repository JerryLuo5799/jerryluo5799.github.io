---
layout: post
title:  "Azure基础知识"
date:   2021-04-30 21:42:55 +0800--
categories: [.NET Core]
tags: [.NET Core, PuppeteerSharp]  
---

### 1. 前言
最新在项目中遇到了一个在后台自动给页面截图的需求, 由于项目是基于.NET Core的, 最后使用PuppeteerSharp实现了该功能

### 2. 什么是PuppeteerSharp

#### 2.1 Puppeteer
Puppeteer是一个Node库，它提供了高级API来通过DevTools协议控制Chrome或Chromium 。Puppeteer默认情况下无头运行，但可以配置为运行完整（无头）的Chrome或Chromium。

**Github**: https://github.com/puppeteer/puppeteer

#### 2.2 PuppeteerSharp
Puppeteer Sharp是Puppeteer的.NET 移植, 是一个NetStandard 2.0库，最低支持的平台版本是.NET Framework 4.6.1和.NET Core 2.0。

**Github**: https://github.com/hardkoded/puppeteer-sharp

#### 2.3 实现的功能
您可以在浏览器中手动执行的大多数操作都可以使用**Puppeteer**或**PuppeteerSharp**完成！
1. 生成页面的屏幕截图和PDF。
2. 爬取SPA（单页应用程序）并生成预渲染的内容（即“ SSR”（服务器端渲染））。
3. 自动进行表单提交，UI测试，键盘输入等。
4. 创建最新的自动化测试环境。使用最新的JavaScript和浏览器功能，直接在最新版本的Chrome中运行测试。
5. 捕获站点的时间线跟踪，以帮助诊断性能问题。
6. 测试Chrome扩展程序。

### 3. 如何使用

#### 3.1 安装SDK包
   
```csharp
dotnet add package PuppeteerSharp
```

#### 3.2 引用命名空间
   
```csharp
using PuppeteerSharp;
```
#### 3.3 代码
   
```csharp
/// <summary>
/// 页面截图
/// </summary>
/// <param name="outputFile">图片保存的物理路径</param>
/// <param name="pageUrl">需要截图的页面地址</param>
/// <param name="width">页面宽度</param>
/// <param name="height">页面高度</param>
/// <param name="sleepTime">等待页面打开的时间, 单位毫秒</param>
/// <param name="executablePath">Chrome安装路径</param>
/// <param name="isNeedDownload">是否需要下载Chromium</param>
/// <returns></returns>
public static async Task GetScreenShot(string outputFile, string pageUrl, int width = 0, int height = 0, int sleepTime = 0, string executablePath = "", bool isNeedDownload = false)
{
    if (isNeedDownload)
    {
        var result = await new BrowserFetcher().DownloadAsync(BrowserFetcher.DefaultChromiumRevision);
    }
    var launchOptions = new LaunchOptions()
    {
        Headless = true
    };
    if(!string.IsNullOrEmpty(executablePath))
    {
        //@"C:\Program Files\Google\Chrome\Application\chrome.exe"
        launchOptions.ExecutablePath = executablePath;
    }
    using (Browser browser = await Puppeteer.LaunchAsync(launchOptions))
    {
        try
        {
            using (var page = await browser.NewPageAsync())
            {
                if (width > 0 && height > 0)
                {
                    await page.SetViewportAsync(new ViewPortOptions
                    {
                        Width = width,
                        Height = height
                    });
                }

                if (sleepTime > 0)
                {
                    //超时时间比等待时间多加几秒
                    await page.GoToAsync(pageUrl, 0);

                    System.Threading.Thread.Sleep(sleepTime);
                }
                else
                {
                    var navOptions = new NavigationOptions
                    {
                        WaitUntil = new[] { WaitUntilNavigation.Networkidle0 }

                    };

                    await page.GoToAsync(pageUrl, navOptions);
                }

                //将页面保存为图片
                await page.ScreenshotAsync(outputFile, new ScreenshotOptions() { FullPage = true, Type = ScreenshotType.Png });

            }
        }
        catch (Exception ex)
        {
            throw ex;
        }
        finally
        {
            if (!browser.IsClosed)
            {
                await browser.CloseAsync();
            }
        }
    }
}
```
PuppeteerSharp可以Headless的方式打开服务器上已经安装的Chrome, 也可以在第一次运行的时候下载Chromium, 一般建议现在服务器上安装Chrome, 因为自动下载Chromium的地址是被屏蔽的。

另外，我在这里使用sleepTime的原因是，我的页面是一致在后台调用接口刷新的， 所以PuppeteerSharp无法判断页面已经加载完成， 正常情况下可以不使用这个参数。

最后，尽量即时的调用browser.CloseAsync()， 否则在服务器上会发现Chrome进程会越来越多。

### 4. 异常
博主基于BackgroundService实现了后台服务， 托管在IIS中， 最后发布到服务器端的时候， 出现了异常： System.IO.FileNotFoundException: Failed to launch browser! path to executable does not exist

最后的解决方案是，使用WorkerService实现了Window服务，运行起来就OK了。
