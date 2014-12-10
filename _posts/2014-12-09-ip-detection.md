---
layout: post
title: web应用中的IP检测方法
category: 技术
tags: Infosec
description: 在web应用中常见的IP检测方法及其优缺点
---

### 简述

刚刚群里有人问web应用中的IP检测问题，正好之前也遇到过，就来总结下，由于环境原因，观点可能有所差异，后续如果发现更好的方案再补充。

### 详细介绍

确保web应用安全性之一的就是访问控制，常见做法有两种：基于角色的访问控制（RBAC, Role Based Access Control），网络访问控制（IP）。本文主要针对基于IP的访问控制。

依靠IP进行访问控制的通常是一些限定只能内网访问的，如：监控页面（敏感信息），定时任务接口等。那么如何获取客户端IP地址呢？

由于平时接触较多的是java，因此以下以java代码为例。

javax.servlet.ServletRequest类中提供一个getRemoteAddr方法，用于获取客户端IP地址。该方法的说明如下：

>Returns the Internet Protocol (IP) address of the client or last proxy that sent the request. For HTTP servlets, same as the value of the CGI variable REMOTE_ADDR.

即获取客户端或最后一个代理的IP地址，可以理解为仅能获取到与web server直接连接的客户端IP。

但是很多web应用并不直接通过getRemoteAddr获取客户端IP，以下是一个网上常见的获取客户端IP的java代码示例：
    
    public static String getRemoteIpAddr() {
        HttpServletRequest request = getRequest();
        String ip = request.getHeader("x-forwarded-for");
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getHeader("WL-Proxy-Client-IP");
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
            ip = request.getRemoteAddr();
        }
        return ip;
    }
以上代码通过HTTP头部的三个字段X-Forwarded-For, Proxy-Client-IP, WL-Proxy-Client-IP来获取客户端的IP地址，为什么不直接用getRemoteAddr呢？

原因在于目前大多数web应用并不是简单的  用户---server 这种结构，很多应用为了负载均衡会在server前架设nginx, apache traffic server或者其他硬件式的负载均衡设备，因此使用getRemoteAddr获取的IP地址就是web server前面的负载均衡设备，这样永远也获取不到用户的IP地址。

为什么用X-Forwarded-For等HTTP头部字段获取IP地址？

X-Forwarded-For并不是RFC中定义的标准请求头信息，它是由Squid开发人员最早引入的，目前大多数代理都会使用X-Forwarded-For用来记录客户端的IP地址，X-Forwarded-For的标准格式如下：
    
    client, proxy1, proxy2

为了支持这一格式，有些负载均衡需要手动配置，如nginx的配置中要加上：

    proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded-for;

我们来通过一个具体场景分析一下X-Forwarded-For的格式：

用户A通过代理B访问某个web应用D，D前面部署的有nginx C做负载均衡。
那到达web应用时的X-Forwarded-For是什么样的？以下是详细过程：

>1. 用户A的请求经过代理B，代理B将用户A的ip地址放入到X-Forwarded-For中，此时HTTP头中为： X-Forwarded-For: ip-A
2. 代理B请求web应用，经过nginx C，此时： X-Forwarded-For: ip-A, ip-B

而这就是webserver获取到的X-Forwarded-For的信息，但如果代理很多获取到的IP地址，而通常我们只需要一个IP地址，怎么办呢，有些程序员就将X-Forwarded-For分割，只取第一个IP作为客户端IP。

这就是问题所在了，我们知道X-Forwarded-For是很容易被伪造的，使用Fillder等工具都可轻易修改，因此只取第一个IP显然是不可信的，以此作为访问控制可以被轻易绕过。

### 那么如何获取真实的客户端IP呢？

答案是没办法，至少我没想到。

### 那如何基于IP进行访问控制呢？

有以下几种做法：

1. X-Forwarded-For虽然可信度不高，但其中有可信的部分，如上例由nginx写入的部分ip-B就是可信的，因为nginx是由web应用自己搭建的，因此很多程序都把这部分IP作为客户端IP来对待，缺点就是这个IP很可能是代理服务器的IP，比如某个公司公共代理的出口IP，以此进行访问控制的结果就是如果封锁IP就会将使用同一代理的其他用户也封锁了，相信有些人已经有体会，如一下论坛注册或者抽奖活动等。

2. 第二种适用于做内网和外网区分，如监控页面，计划任务等，不需要获取到具体IP，只要能区分内外网即可，而区分的关键在于网络结构，如上例，用户到达web服务器至少要经过nginx，而此时如果内部请求不需要nginx即可到达web服务器，就可以根据这两种情况中X-Forwarded-For的不同来区分内外网。

3. 我们架构师提出一种方案，在外层nginx上另开一个仅限内网访问的端口，将敏感接口放于此端口下。这样做部署略复杂，很多开发不愿意做。

### 页面广告是怎么做防刷的？

后来群里的朋友又问那目前那些页面广告是如何防刷的。我之前也没有做过这种项目，只能在这里纸上谈兵一下了。

1. 显然防刷不能依靠IP，因此IP在这里仅供参考。
2. HTTP请求头中的referer也可以作为参考。
3. 使用js等技术分析用户指纹信息，如浏览器版本，操作系统，用户点击等信息，通过cookie，参数等形式来对用户唯一性和真实性进行判断。具体实现方法应该是一些算法专家在研究了。

