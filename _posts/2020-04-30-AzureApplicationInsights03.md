---
layout: post
title:  "使用Azure Application Insights监测你的应用 (三) - Snapshot Debugger"
date:   2020-02-29 21:42:55 +0800--
categories: [Azure]
tags: [Application Insights, 监控平台, 日志系统, Snapshot Debugger]  
---

### 前言
很多时候, 即使系统记录了异常日志, 但是信息的不全, 技术人员还是难以定位到具体问题, 而Snapshot Debugger不然记录了异常发生时的堆栈信息, 同时也记录了当时所有对象和临时变量的信息, 技术人员可以根据这些快速的定位和排查问题。