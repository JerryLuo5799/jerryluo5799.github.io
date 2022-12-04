---
layout: post
title:  "Playwright: 新的Web应用自动化测试框架"
date:   2022-11-13 21:42:55 +0800--
categories: [Tools]
tags: [Playwright, Web Testing]  
---

### 1. 什么是Playwright

Playwright是微软开源的Web应用自动化测试框架, 它可以使用非常简单的API去操作Chromium, Firefox, WebKit内核的浏览器。为了让不同语言的开发者都可以使用Playwright，微软提供了不同版本的Playwright

- Playwright（NodeJS）- <https://github.com/microsoft/playwright>
- Playwright for .NET - <https://github.com/microsoft/playwright-dotnet>
- Playwright for Java – <https://github.com/microsoft/playwright-java>
- Playwright for Python - <https://github.com/microsoft/playwright-python>

### 2. 如何使用 (NodeJS)

#### 2.1 环境准备

- NodeJS, v14.0以上
- 代码编辑器
- Playwright
  
```npm
npm install playwright 
```

#### 2.2 创建Playwright项目

```npm
npm init playwright
```

#### 2.3 代码
   
```JavaScript
const { chromium } = require('playwright');

const device = devices['iPhone 12'];

(async () => {
  const browser = await chromium.launch({
    headless: false
  });
  const context = await browser.newContext({
    recordVideo:{ 
        dir:"video"
    },
  });

  context.tracing.start({ screenshots: true, snapshots: true, sources: true});

  const page = await context.newPage();
  await page.goto('https://playwright.dev/');
  await page.getByRole('button', { name: 'Node.js' }).click();
  await page.getByRole('navigation').getByRole('link', { name: 'Python' }).click();
  await page.getByRole('button', { name: 'Python' }).click();
  await page.getByRole('navigation').getByRole('link', { name: '.NET' }).click();
  await page.getByRole('link', { name: 'Docs' }).click();
  await page.getByRole('navigation').filter({ hasText: 'Playwright for .NETDocsAPI.NET.NETNode.jsPythonJavaCommunitySearchK' }).getByRole('link', { name: 'API' }).click();
  await page.getByRole('button', { name: 'Search' }).click();
  await page.getByPlaceholder('Search docs').click();
  await page.getByPlaceholder('Search docs').fill('click');
  await page.getByRole('link', { name: 'Mouse.ClickAsync(x, y, options)​ Mouse' }).click();

  context.tracing.stop({path: "trace.zip"});

  // ---------------------
  await context.close();
  await browser.close();
})();
```
