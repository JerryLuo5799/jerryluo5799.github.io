---
layout: post
title:  "如何创建Azure资源"
date:   2022-12-17 21:42:55 +0800--
categories: [Azure]
tags: [Azure Service, 资源创建方式]  
---

### 1. 前言

Azure支持通过多种方式创建资源, 一般来说, 我们应该尽量减少手工创建资源。

### 2. 手动创建

这是最常见和最不推荐的，当需要的资源比较多的时候, 每次都需要手动工作来重现，并为非常容易人为错误留下余地。

![手动创建](https://ssw.com.au/rules/89969e39c1a92926f4c850cec736ecdf/azure%20resources.gif)

### 3. 手动创建资源并保存脚本

有些人通过手动创建和保存脚本来解决问题。这也一样会有问题。将Azure资源导出为Json格式的ARM模板时, 通常需要手工调整。

![手动创建资源并保存脚本](https://ssw.com.au/rules/static/e94735b63c5b7d7a941c55a4a9316c83/2bef9/azure-bad-1.png)

![手动创建资源并保存脚本](https://ssw.com.au/rules/static/60637adb7d477dba6147163874c728c0/2bef9/azure-bad-2.png)

### 4. 推荐选项

因此，如果不手动创建 Azure 资源，则有以下几种选项。

#### 4.1 Farmer

Farmer是一个.net的一个领域特定语言(DSL: domain-specific-language)，用于快速生成Azure资源管理器(ARM)模板。Farmer是商业支持的，开源的，免费使用的。

- [Farmer - 使可重复的 Azure 部署变得简单！](https://compositionalit.github.io/farmer/)
- 从 F# 生成 ARM 模板

#### 4.2 Bicep (推荐)

```text
Bicep 是微软官方发布的一种领域特定语言（DSL），用于以声明方式部署 Azure 资源。它旨在通过更简洁的语法、改进的类型安全性以及对模块化和代码重用的更好支持来大大简化创作体验。Bicep 是对 ARM 和 ARM 模板的透明抽象，这意味着可以在 ARM 模板中完成的任何操作都可以在 Bicep 中完成。

Bicep 代码被转译为标准 ARM 模板 JSON 文件，从而有效地将 ARM 模板视为中间语言 （IL）。
```
- [Bicep - 用于描述和部署 Azure 资源的声明性语言](https://github.com/Azure/bicep)
- 免费，完全受微软支持
- 具有["az"命令](https://learn.microsoft.com/en-us/cli/azure/bicep?view=azure-cli-latest)集成
- VS Code的[扩展](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-bicep)，用于编写ARM Bicep文件 ⭐️
- 幕后 - 编译成 ARM JSON 模板进行部署
- 语法比 ARM JSON 简单得多
- 自动处理资源依赖关系
- 用于发布版本化和可重用体系结构的[专用模块注册表](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/private-module-registry?tabs=azure-powershell)
  
[Bicep 案例](https://github.com/william-liebenberg/BicepFlex)
![Bicep 案例](https://ssw.com.au/rules/static/46eeacc4d394177db4a3758a9b5b6d22/681f1/Bicep.png)

#### 4.3 其他收费服务 

迁移到自动化基础设施即代码（IaC）解决方案时，另一种选择是迁移到 [Pulumi](https://www.pulumi.com/) 或 [Terraform](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs) 等付费提供商。如果你使用多个云，或者想要直接控制软件安装和基础架构，这些解决方案是理想的选择。

- 这两种工具都很棒，并且提供免费版本
- 付费版本为大型团队提供更多好处，提供管理大型的基础结构解决方案的能力
- Terraform 使用 HashiCorp Configuration Language HCL
  - 像 YAML 一样，但功能更强大
  - [Terraform Quick Start Demo](https://learn.hashicorp.com/tutorials/terraform/cdktf-install?in=terraform/cdktf)
- Pulumi 使用真实代码（C#、TypeScript、Go 和 Python）作为基础设施，而不是 JSON/YAML

Pulumi的Demo, 其中用C#定义Azure资源, 并从控制台运行"pulumi up"将资源部署到Azure

![Pulumi Demo](https://ssw.com.au/rules/static/6197d662da23d03bc6dc1fc561f2ab3d/2bef9/pulumi3.png)


![Pulumi Demo](https://ssw.com.au/rules/static/c58c652de09d4883d5771fd114969bd6/2bef9/pulumi2.png)