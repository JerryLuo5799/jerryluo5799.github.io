---
layout: post
title:  "Source Generators VS Lombok (Java)"
date:   2021-12-19 21:42:55 +0800--
categories: [.NET Core]
tags: [.NET Core, Source Generators, Lombok]  
---

### 1. 前言
博主在.NET Conf China 2021分享了Source Generators探索, 具体视频如下:

<iframe src="//player.bilibili.com/player.html?aid=934879516&bvid=BV1pM4y1c78H&cid=464387394&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"  class="bilibili"> </iframe>

链接地址: [https://www.bilibili.com/video/BV1pM4y1c78H/](https://www.bilibili.com/video/BV1pM4y1c78H/)

期间, 有问起Source Generators是否类似于Java的Lombok, 所以简单的对比了一下。

### 2. 什么是Lombok
Lombok项目是一个Java库，它会自动插入您的编辑器和构建工具中，从而为您的Java增光添彩。永远不要再写另一个getter或equals方法，带有一个注释的您的类有一个功能全面的生成器，自动化您的日志记录变量等等。理解一下，使用Lombok，通过注解类，让你不再需要编写getter、equals等方法，减少样板代码的编写。

### 3. Lombok的实现原理
- 运行时解析
  
运行时能够解析的注解，必须将@Retention设置为RUNTIME，这样就可以通过反射拿到该注解。java.lang,reflect反射包中提供了一个接口AnnotatedElement，该接口定义了获取注解信息的几个方法，Class、Constructor、Field、Method、Package等都实现了该接口，对反射熟悉的朋友应该都会很熟悉这种解析方式。

- 编译时解析
  
编译时解析有两种机制，分别简单描述下：

**1）Annotation Processing Tool**

apt自JDK5产生，JDK7已标记为过期，不推荐使用，JDK8中已彻底删除，自JDK6开始，可以使用Pluggable Annotation Processing API来替换它，apt被替换主要有2点原因：

api都在com.sun.mirror非标准包下
没有集成到javac中，需要额外运行

**2）Pluggable Annotation Processing API**

JSR 269自JDK6加入，作为apt的替代方案，它解决了apt的两个问题，javac在执行的时候会调用实现了该API的程序，这样我们就可以对编译器做一些增强，Lombok本质上就是一个实现了“JSR 269 API”的程序, 这时javac执行的过程如下：

1. javac对源代码进行分析，生成了一棵抽象语法树（AST）
2. 运行过程中调用实现了“JSR 269 API”的Lombok程序
3. 此时Lombok就对第一步骤得到的AST进行处理，找到@Data注解所在类对应的语法树（AST），然后修改该语法树（AST），增加getter和setter方法定义的相应树节点
4. javac使用修改后的抽象语法树（AST）生成字节码文件，即给class增加新的节点（代码块）

### 4. Source Generators VS Lombok
1. Source Generator是C#编译器(Roslyn)的一部分功能, Lombok是一个IDE插件, 实现了一些固定的注, 所以如果要类比的话,Source Generator更像是JSR 269 API, 而Lombok更好的类比对象应该是基于Source Generator实现的功能类
2. Lombok支持运行时和编译时, 而Source Generator只支持编译时
3. 和编译时解析的Lombok相比, 两者都是实现了对应的接口(Source Generator的ISourceGenerator 和 Lombok的JSR 269 API), 然后编译器在编译期间调用, 实现扩展编译器的功能, 但是Lombok编译时会修改语法树(意味着实质上修改了源代码文件), 而在C#中,源代码文件和语法树是强对应的,所以Source Generator在编译过程中不会修改语法树,而是通过新增源代码文件(语法树)的方式扩展原先的功能, 在这点上, Lombok的实现更接近于AOP的静态织入,因为都会修改原有的代码逻辑
