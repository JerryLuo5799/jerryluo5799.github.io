---
layout: post
title:  "使用Visual Studio自带Dev Tunnels实现端口转发"
date:   2023-03-19 21:42:55 +0800--
categories: [Tools]
tags: [Visual Studio]  
---

### 1. 前言

大家应该都遇到过这样的场景, 需要接受第三方的推送信息(Webhook), 开发后如果要调试的话, 一般只能发布到服务器上, 通过打印日志的方式。

Visual Studio的Dev Tunnels提供了一种便捷的端口转发的方式, 自动化的生成一个公网可访问的URL地址, 并且会转发端口请求到你的本地环境, 这样其实可以大大简化线上环境的调试。

### 2. 版本要求

Visual Studio 2022 17.5+

### 3. 如何使用

演示视频地址: [https://www.bilibili.com/video/BV1gv4y1V7Lz/](https://www.bilibili.com/video/BV1gv4y1V7Lz/)

1. 在菜单 **Tools | Options | Environment | Preview Features** 启用 Dev Tunnels
![Enable Dev Tunnels](https://www.ssw.com.au/rules/static/1fa6e7be9985bbd5e2858bbf7e17b94b/2bef9/screen1.png)
2. 在菜单  **View \| Other Windows \| Dev Tunnels** 打开 Dev Tunnels 窗口
3. 创建一个新的Dev Tunnel
![Create Dev Tunnel](https://www.ssw.com.au/rules/static/93579fb5cf32b1faf2331e47407ff09e/2bef9/screen2.png)
4. 运行项目
5. 在菜单 **Dev Tunnels | Tunnel URL** 获取公网可访问的的URL地址
![Get URL](https://www.ssw.com.au/rules/static/3495c5bad508a68c4d75b00e2497c021/d4c13/screen4.png)
6. 通过公网URL地址访问本机应用
![Web](https://www.ssw.com.au/rules/static/2cbd4ce9eb9039bc1bb70465c55421b6/2bef9/screen3.png)
![Mobile](https://www.ssw.com.au/rules/static/4684d912be95d2de09660e7f550b566f/2bef9/screen5.png)







