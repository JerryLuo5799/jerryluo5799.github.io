---
layout: post
title:  "使用工具扫描第三方引用库的漏洞"
date:   2022-07-29 21:42:55 +0800--
categories: [Other]
tags: [漏洞扫描, .NET Core, dotnet Cli, npm, yarn]  
---

### 1. 前言
在开发的过程中, 我们总是会引用各类第三方的库, 项目交付后，进入维护模式时, 通常没有开发人员会跟进升级这些包，如果在项目中引用的库发现了严重的漏洞，会给项目带来很大的风险。

那么如何对这些引用的包进行漏洞扫描呢？现在的包管理器其实都提供了对应检测漏洞的方法，以下的命令可以帮助到你。
- dotnet cli: dotnet list package --vulnerable
- npm: npm audit
- yarn: yarn audit

### 2. dotnet Cli
打开命令窗口，进入.Net Core的项目目录, 输入命令:
```
dotnet list package --vulnerable
```
![dotnet Cli结果](/assets/imgs/UsingToolsToScanForVulnerabilities01.png)

上图说明在 **VulnerableRepo.Application** 这个项目下, 引用的 UmbracoForms 库存在高危漏洞, 漏洞的细节在 [https://github.com/advisories/GHSA-8m73-w2r2-6xxj](https://github.com/advisories/GHSA-8m73-w2r2-6xxj)

同时, 在项目 **VulnerableRepo.Domain** 项目下, 没有存在高危漏洞的库

### 3. npm: npm audit
打开命令窗口，进入前端的项目目录, 输入命令:
```
npm: npm audit
```
![npm audit结果](/assets/imgs/UsingToolsToScanForVulnerabilities02.png)

### 4. yarn: yarn audit
同理, 打开命令窗口，进入前端的项目目录, 输入命令
```
yarn: yarn audit
```
可以得到和上述内容类似的结果。

### 5. 更多
更多内容可以查看：[Do you monitor your application for vulnerabilities?](https://www.ssw.com.au/rules/monitor-packages-for-vulnerabilities)