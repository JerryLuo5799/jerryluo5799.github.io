---
layout: post
title:  "什么是QUIC"
date:   2021-08-30 21:42:55 +0800--
categories: [.NET]
tags: [.NET Core, Http/3, QUIC]  
---

### 1. 前言
回到过去，大约在 1990 年代中期，当 Internet 是新事物时，Internet 连接非常的慢, 加载一个简单的网页可能需要 1-2 分钟以上的时间。良好的视频流，忘记它！高分辨率视频最初是 640x480，我们现在称之为 480p 或标清视频，但大多数早期的视频是 320x240。即使在较低的分辨率下，您也会开始播放视频，然后立即暂停，以便视频可以缓冲，大约 5-10 分钟后你就可以开始播放视频了。

之后, 处理器和网络速度显着提高。显示器变得更大，分辨率更高。新的互联网连接技术无处不在。然后智能手机出现了，第二波创新浪潮袭来。从这个角度来看，当前的旗舰智能手机比1995年最快的超级计算机强大十倍（10 倍）。

尽管实现了现代互联网体验的技术出现了巨大飞跃，但我们从一开始就一直使用相同的主要互联网协议，我们久经考验的朋友 TCP。

### 2. TCP 和 什么是QUIC

首先让我说 TCP 或传输控制协议不会很快消失。它是，也将是未来几年世界上占主导地位的网络协议。这引出了一个问题，为什么要更换它？

TCP 最初是在 1974 年为 Internet 的前身 ARPAnet 开发的。有一点数学，就会发现TCP将拥有50个很快的生日。在 1990 年代初 ARPAnet 成为商业互联网之前，ARPAnet 主要由军事和教育机构跨闭路使用。ARPAnet 使用的 TCP/IP 协议套件被转移到 Internet。TCP 作为弹性传输协议，成为大多数 Internet 活动的事实上的标准。如果没有 TCP，Internet 就不会正常工作。

这通常是我们说 TCP 坏了需要更换之类的东西的地方，但 TCP 并不一定不好，因为它是一个很棒的协议。问题是我们无法改变 TCP 以使其更好并满足需要疯狂快速的现代互联网服务的需求。TCP 协议过于嵌入，无法对其进行任何重大更改，而不会破坏数百万台设备的风险。

多年来，人们提出了很多想法来改善互联网体验，但直到 QUIC（快速 UDP 互联网连接协议）在 Google 使用独特的概念开发出来之前，这些想法都没有得到解决。QUIC 协议不是开发一种需要对互联网主干进行大规模升级的全新协议，而是扩展了现有协议 UDP。我说扩展是因为 QUIC 是一个传输层协议，而不是一个应用程序协议，即使 QUIC 是在 UDP 段内传输的。可以这样想：

UDP + QUIC = 传输层

QUIC 使用 UDP 进行端口和无连接传输，然后添加 TCP 的弹性和 TLS 1.3 的安全性，加入一些来自 SMB 等协议的命令和版本控制，然后混合一组新的协议概念和效率来创建在协议世界中完全独一无二的东西。

### 3. QUIC的改进
大型科技公司对 QUIC 感到兴奋，因为它增加了一些有助于改善互联网服务的变化，推而广之，会使每个人都能获得更好的 Internet 体验。以下是 QUIC 的一些改进：

#### 3.1 加密
 
QUIC 1.0 要求对所有数据进行基于 TLS 1.3 的加密。这使得 QUIC 上的数据无论服务如何都具有内在的安全性。

#### 3.2 更低的连接延迟
 
QUIC 不会改变物理定律，但它不必等待两次握手（TCP 然后 TLS）来完成安全的网络连接。与 TCP + TLS 相比，连接设置所需的数据包更少，并且在关闭后可以恢复。这意味着您在第一次连接到服务时开始更快地获取数据，第二次可能更快。

#### 3.3 连接重用

QUIC 可以通过两种方式重用会话：流和会话票证。单个 QUIC 会话可以同时具有多个数据流。服务器还可以向客户端授予会话票证，该票证可用于重新连接到服务器，而无需进行完整的握手。这减少了客户端-服务器连接的数量，并允许快速、安全的重新连接。

#### 3.4 连接迁移
 
在我看来，这是最酷的功能之一，因为它允许 QUIC 连接在 IP 更改后继续存在。假设您有一台带 WiFi 和 LAN 的笔记本电脑，并从有线 LAN 连接切换到 WiFi。使用 TCP 必须关闭连接，并使用 WiFi IP 地址打开新连接。使用 QUIC，客户端可以向服务器提供证据以证明他们是谁，并在新 IP 上继续现有连接，就好像什么都没有改变一样。使移动用户体验更加无缝。

#### 3.5 安全

除了加密之外，QUIC 还可以防止或减轻拒绝服务 (DoS)、重放、反射、欺骗和其他类型的攻击等事件的影响。QUIC 不能消除所有攻击，但它确实试图使其更难成功攻击。

#### 3.6 版本控制

QUIC 可以改变并适应新的互联网需求，因为它具有版本控制。QUIC 1.0 于 2021 年 5 月通过 RFC 9000 以及 RFC 8999、9001 和 9002 发布。 QUIC 的未来版本可以根据需要自由更改协议。计算机可以同时支持多个 QUIC 版本。这允许新版本修复和改进 QUIC 协议，同时在过渡期间支持旧的实现。随着时间的推移，旧版本可以被删除，以在未来几十年保持 QUIC 安全和最新。

#### 3.7 扩展框架

个人和公司可以扩展 QUIC 以满足他们自己的需求。这是通过 QUIC 扩展帧处理的，它可以是公共的，并且可能被添加到未来版本的 QUIC 中，或者是私有的，仅用于内部服务。必须遵循的规则很少，但除此之外，人们可以通过自定义框架自由地扩展 QUIC，只要他们认为合适。

### 4. QUIC目前的问题
QUIC 1.0 是全新的，目前还不到 4 个月。大多数设备支持 QUIC 需要时间。现代浏览器已经支持 QUIC。最新版本的 Windows、Window 10 21Hx、Windows 11 和 Windows Server 2022 具有原生 QUIC 支持。一些较旧版本的 Windows 应该会在 2022 年初看到对 MsQuic 的支持。Apple 从 Big Sur 开始具有本机 QUIC 支持。Linux 和 FreeBSD 当前需要在用户应用程序中安装或实现 QUIC 驱动程序，但未来版本可能会提供本地支持。

较旧的操作系统版本是这里的问题，并且可能在浏览器之外没有任何 QUIC 功能。如果 IT 教会了我一件事，那就是很多人通常升级缓慢。

然后是为 QUIC 开发时的学习曲线。QUIC 协议的处理方式与 TCP 和普通 UDP 略有不同。学习利用 QUIC 流为您带来优势的最佳方式需要一些时间和精力。

最后，还有流量整形问题。一些网络和 ISP 更喜欢 TCP 流量而不是 UDP，这是正确的。TCP 是目前 Internet 上事实上的传输协议。然而，当达到某些网络拥塞条件时，这可能会在短期内导致 QUIC 性能出现问题。严重拥塞的网络可能会丢弃 UDP 流量以通过 TCP 流量。随着骨干网络越来越意识到 QUIC，这种情况将会改变。短期来看，高峰时段可能会出现性能问题。
 
这是否意味着要忽略QUIC？不应该。微软、苹果、谷歌、亚马逊、Facebook、Cloudflare 和其他大型科技公司已经在使用 QUIC，因为它比 TCP 有优势。科技界已经为 QUIC 开关做好了准备，所以你也应该考虑一下。

### 5. MsQuic
微软的 QUIC 实现称为[MsQuic](https://github.com/microsoft/msquic?wt.mc_id=MVP_324329)。它是一个开源跨平台项目。从版本 1.5.0 开始完全支持 Windows 和 Linux，MacOS 目前处于 alpha 阶段。MsQuic 支持为 Windows Server 2022 宣布的UDP卸载功能，该功能也将在 Windows 11 中可用。UDP 功能确实需要兼容的网络适配器。MsQuic 的表现也相当令人印象深刻。