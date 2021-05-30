---
layout: post
title:  "Azure Static Web Apps"
date:   2020-09-23 21:42:55 +0800--
categories: [Azure]
tags: [Azure, Azure Static Web Apps]  
---

Azure Static Web Apps是一项从 GitHub 存储库自动构建和部署完整堆栈 Web 应用程序的服务。Azure 静态 Web 应用由静态 Web 前端和基于 Azure Functions 的后端组成。创建静态 Web 应用资源时，Azure 会在应用的源代码存储库中设置一个 GitHub 操作工作流，用于监控您选择的分支。每次将提交推送到受监视的分支时，GitHub 操作都会自动构建和部署您的应用程序及其 API。

### 1. 快速开始
为了帮助您入门，我们创建了一个[GitHub存储库模板](https://aka.ms/blazor-swa/template)，您可以将其用作您自己项目的起点。

在 GitHub 中，单击使用此模板按钮以使用模板创建新的 GitHub 存储库，为其提供您选择的名称（我们将在此处使用my-blazor-app），然后单击从模板创建存储库。

模板中有三个文件夹：

客户端： Blazor WebAssembly 示例应用程序
API： AC# Azure Functions API，Blazor 应用程序将调用它
共享：在 Blazor 和 Functions 应用程序之间具有共享数据模型的 AC# 类库
要在本地运行应用程序，请同时启动 API 和客户端项目。Blazor 应用程序中的“获取数据”页面从后端函数 API 请求天气预报数据并将其显示在页面上。

### 2. 部署
要将此应用程序部署为 Azure 静态 Web 应用程序，请登录到您的 Azure 帐户并搜索静态 Web 应用程序。
![图片](https://devblogs.microsoft.com/aspnet/wp-content/uploads/sites/16/2020/09/swa-landing-screen.png)

单击Create，为应用程序提供订阅、资源组和名称。

接下来，登录 GitHub 并找到您的 GitHub 存储库 ( my-blazor-app ) 并选择您要部署的分支。

最后，从Build Presets 中选择Blazor，它将使用Client、API和wwwroot填充App location、API location和App artifact location。第一个和第二个值是 Blazor 和 Functions 应用程序的项目文件所在的 Git 存储库中的路径，因此如果您修改了 Git 存储库的结构，请确保更新这些值以反映。第三个值是 Blazor 将编译到的输出路径，不需要更新。

![图片](https://devblogs.microsoft.com/aspnet/wp-content/uploads/sites/16/2020/09/complete-blazor-info.png)

完成向导，静态 Web 应用程序将为您创建 GitHub 操作工作流文件并将您的应用程序部署到 Azure。

![图片](https://devblogs.microsoft.com/aspnet/wp-content/uploads/sites/16/2020/09/blazor-swa-in-action.gif)
