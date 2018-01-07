---
title: 什么是gRPC（翻译）
date: 2018-01-06 14:20:27
categories: gRPC
tags: []
---

> 英文原文：https://grpc.io/docs/guides/index.html

本文介绍gRPC和protocol buffers。gRPC使用protocol buffers作为IDL（接口描述语言）,同时也使用protocol buffers作为底层消息交换格式。如果你对gRPC或者protocol buffers不甚了解，请继续阅读本文。如果你想立刻开始使用gRPC，请阅读我们的[Quick Starts][1]。

## 概述

在gRPC中，客户端应用可以直接跨机器调用服务端应用的方法，就如同调用本地对象，这样就使得创建分布式应用服务变得更为简单。与其它许多RPC系统一样，gRPC也是基于这样的理念：定义服务（service）、指定可被远程调用的方法（methods）以及方法参数和返回类型。服务端实现了这些接口，并通过运行一个gRPC服务处理客户端的调用。客户端保留的存根（stub）（在某些语言中直接被称为客户端）提供了一套和服务端相同的方法。

<div align=center>![](/images/landing-2.svg)</div>

gRPC客户端和服务端能够在多种环境中运行并相互通信，大至Google内部的服务器，小到你的电脑，并且可以使用gRPC支持的任何一种语言实现。所以你可以很方便地使用Jave创建一个gRPC服务端，然后使用Go、Python或Ruby实现客户端。此外，最新的Google APIs将会拥有gRPC版本的接口，你的应用将更容易接入Google功能。

## 使用protocol buffers



[1]: https://grpc.io/docs/quickstart
