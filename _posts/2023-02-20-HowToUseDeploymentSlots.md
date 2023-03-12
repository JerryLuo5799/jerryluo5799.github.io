---
layout: post
title:  "如何使用Deployment slots (部署槽) 切换部署环境"
date:   2023-02-20 21:42:55 +0800--
categories: [Azure]
tags: [Rules, Redundant Storage]  
---

### 1. 前言

Azure App Services 是一个非常强大并且易于使用的应用。许多开发人员选择它发布Web应用。当我们需要同时考虑生产环境和Staging环境部署的时候, 我们可以选择创建第二个资源组或订阅来托管资源, 与此同时, 我们可以选择一个更好的方案作为替代方案，那就是Deployment slots (部署槽) 功能。

### 2. 如何使用

要开始使用插槽部署，我们可以启动另一个web应用程序-它位于您的原始web应用程序旁边，具有不同的url。您的生产url可以是production.website.com，对应的Staging插槽可以是staging.website.com。你的用户将访问你的产品web应用程序，而你可以部署一个新版本的web应用程序到你的Staging插槽。这样，更新后的web应用程序就可以在上线之前进行测试。只需要一键配置，您就可以轻松地交换Staging插槽和生产插槽。

### 3. 其他优点

使用Deployment slots (部署槽) 的另一个好处是，如果你的生产web应用程序出现了任何问题，你可以通过与Staging插槽交换来轻松地将其回滚-你的上一个版本的web应用程序位于Staging插槽上-准备在新版本被推入Staging插槽之前随时交换回去。

部署槽也可以与灰度发布策略一起工作-您可以逐步选择用户到Staging插槽。

### 4. 案例

首先, 我们运行着两个插槽, 一个是生产环境插槽, 一个是Staging插槽, 可以看到生产环境插槽接口返回是v2, Staging插槽接口返回的是v3。

![Before Swap - Production slot](https://ssw.com.au/rules/static/5838293634436856854ae3db3904618d/2bef9/azure-slot-1.png)

![Before swap - Staging slot](https://ssw.com.au/rules/static/86f1a0158ddd8afe1930cdf5d1839d14/2bef9/azure-slot-2.png)

接下去， 让我们在Azure后台一键配置对应的插槽。

![Swap the slot with one click](https://ssw.com.au/rules/static/dc0b982fd0351befe10b583b545625c9/2bef9/azure-slot-3.png)

切换成功后，生产环境插槽接口返回是v3, Staging插槽接口返回的是v2，非常的便捷， 同样， 如果我们发现v3的接口有问题，可以同样一键配置，回退版本。

**注意：这里生产环境的URL没有变，只是接口返回变成了v3**

![After swap – Production slot](https://ssw.com.au/rules/static/f293bcef838a9fe90bb4e8ba047af6cc/2bef9/azure-slot-4.png)

![After swap – Staging slot](https://ssw.com.au/rules/static/c023071468d612e177501a668f674068/2bef9/azure-slot-5.png)