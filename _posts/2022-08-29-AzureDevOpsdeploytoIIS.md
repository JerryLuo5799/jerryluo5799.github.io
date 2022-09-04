---
layout: post
title:  "使用Azure DevOps实现文件形式发布.NET Core应用到IIS"
date:   2022-08-29 21:42:55 +0800--
categories: [Azure DevOps]
tags: [Azure DevOps, .NET Core, WinServer, IIS]  
---

### 1. 前言
目前, 关于.NET Core应用使用Azure DevOps进行CI/CD的资料大都是使用Docker形式发布的, 作者目前遇到的情况是客户只能支持文件形式发布, 所以进行了一定的探索。

### 2. CI

CI的最终目的为了生成可发布的包，一般情况下需要执行：
1. 编译项目
2. 运行单元测试
3. 发布项目
4. 将发布后生成的文件上传到Azure Pipelines 或 某个文件共享位置

**最简单的情况下, 可以跳过1和2**

**azure-pipelines.yml** 文件如下：
```
trigger:
- master

pool:
  name: 'Gongshu'

variables:
  solution: '**/*.sln'
  buildPlatform: 'Any CPU'
  buildConfiguration: 'Release'

steps:
- task: DotNetCoreCLI@2
  inputs:
    command: 'publish'
    publishWebProjects: true

- task: PublishBuildArtifacts@1
  inputs:
    PathtoPublish: '$(Build.SourcesDirectory)\OneCallCore.BI.Api\bin\Debug\netcoreapp3.1'
    ArtifactName: 'OneCallCoreBIApiPublish'
    publishLocation: 'Container'
```

### 3.CD

CD的目的是发布文件到服务器，所以一般需要已下步骤：
1. 将文件上传到服务器的某个目录
2. 停止IIS下.NET Core应用对应的程序池
3. 覆盖.NET Core应用文件
4. 启动IIS下.NET Core应用对应的程序池

因为停止/启动程序池需要远程执行命令, 所以建议在Windows上启用SSH服务, 上传文件和执行命令都使用SSH实现。[Windows系统开启SSh Server服务](https://zhuanlan.zhihu.com/p/452890363)

