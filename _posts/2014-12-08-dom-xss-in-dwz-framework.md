---
layout: post
title: DWZ富客户端框架DOM XSS分析
category: 技术
tags: Infosec
description: DWZ富客户端框架DOM XSS分析

### 简述

公司部分后台使用DWZ，在一次代码审计过程中发现了DWZ框架使用不当可造成的XSS漏洞。这个XSS漏洞让我意识到之前对DOM类的XSS存在诸多误解。

### 过程分析

公司的某个应用大概过程是这样的：用户在前端发帖，管理员可以在后台审核帖子内容。而此应用采用的是输入过滤的方式，重写了java的HttpServletRequest类，在里面加上了转义函数，即输入过滤的方式，（这里就不吐槽输入过滤的种种缺点了）。

问题出在后台的一处话题报表中，其中话题标题为：&lt;script&gt;alert(42873)&lt;/script&gt;如下图：

![Alt text](/images/20141208164721.jpg)

