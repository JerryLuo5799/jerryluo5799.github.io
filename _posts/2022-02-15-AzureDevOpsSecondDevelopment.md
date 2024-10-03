---
layout: post
title:  "使用Microsoft.TeamFoundationServer.Client实现Azure DevOps的二次开发"
date:   2022-02-22 21:42:55 +0800--
categories: [Azure DevOps]
tags: [Azure DevOps, Microsoft.TeamFoundationServer.Client, 二次开发]  
---

### 1. 前言
博主的公司使用Azure DevOps Server来管理整个软件开发的周期, 期间开发部门有需要需要基于Azure DevOps平台进行二次开发, 实现一些只用的产品和工具。

### 2. 方案

经过调研, 发现有两个方案:
1. 直接调用Azure DevOps Server的[Rest Api](https://docs.microsoft.com/en-us/rest/api/azure/devops/account/?view=azure-devops-rest-6.1&wt.mc_id=MVP_324329) 
2. 使用官方的包[Microsoft.TeamFoundationServer.Client](https://www.nuget.org/packages/Microsoft.TeamFoundationServer.Client), 详情请查阅[文档](https://docs.microsoft.com/en-us/azure/devops/integrate/concepts/dotnet-client-libraries?view=azure-devops&wt.mc_id=MVP_324329)

最后我们选择了方案二, 因为相对来说比较简单。

### 3.如何使用

不管是使用Rest Api或者引用包的方式来实现, 第一步都是要进行身份认证,之后才能调用接口进行操作。微软官方提供了各种身份认证的方式，可以查阅[文档](https://docs.microsoft.com/zh-cn/azure/devops/integrate/get-started/authentication/authentication-guidance?view=azure-devops&wt.mc_id=MVP_324329)

我们在实际使用中, 采用了PATs (personal access tokens)的方式，简而言之，就是每个用户都可以在登录Azure DevOps后，手工创建一些Token并给Token授权，然后在程序中就可以使用这些Token，调用接口做对应的事情。 [什么是PATs，如何创建？](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows&wt.mc_id=MVP_324329)

创建完PAT之后, 我们就可以用Microsoft.TeamFoundationServer.Client和Azure DecOps进行交互了。

首先，需要安装 **Microsoft.TeamFoundationServer.Client** 包：

```CSharp
Install-Package Microsoft.TeamFoundationServer.Client
```

实现代码, 代码非常简单,就是获取当前用户有权访问的项目列表。

```Csharp

Uri orgUrl = new Uri("[Azure DevOps集合地址]"); // 如:https://dev.azure.com/Default               
string personalAccessToken = "[个人的PAT]";  // 

// Create a connection
using (VssConnection connection = new VssConnection(orgUrl, new VssBasicCredential("", personalAccessToken)))
{
    ProjectHttpClient projectClient = connection.GetClient<ProjectHttpClient>();

    IEnumerable<TeamProjectReference> projects = projectClient.GetProjects(null, 500).Result;

    string projectId = projects.FirstOrDefault()?.Id.ToString();

    Console.WriteLine($"总计项目 {projects.Count()}, 第一个项目的ID为: {projectId}");
}

````

最后的执行结果为：
![执行结果](/assets/imgs/AzureDevOpsProjectList.png)