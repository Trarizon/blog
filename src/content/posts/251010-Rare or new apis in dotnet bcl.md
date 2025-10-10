---
title: 发现.NET BCL中的不常见/新API
published: 2025-10-10
tags: [C#, .NET, BCL]
category: C#
draft: false
---

## 前言

本篇用于记录本人在网上冲浪时发现的.NET BCL更新时一些没有被特别指出的有点用但不常见或者新的API，常不常见的基准基于本人主观，且由于本人从.NET 6学起，可能会集中记录大量.NET 7 ~ 10的新API。

大概无规律持续更新（

## 目录

- [`Delegate.EnumerateInvocationList`](#delegate)

## 正片

### Delegate

``` c#
Delegate.EnumerateInvocationList<T>(T? d) where T : Delegate 
```

|Release|Blog update|
|-------|-----------|
|.NET 9 |25-10-10   |

`Delegate`上的静态方法。顾名思义，遍历多播委托，可以用于避免`GetInvocationList`的数组分配。

> [!NOTE]
> 前两天面试提到了如何获取event的委托列表，当时一下宕机忘了`GetInvocationList`，面试官提了突然想起来。今天去[Source Browser](https://source.dot.net)看了一下发现多了个Enumerate的api，于是突发奇想写个blog。
> 顺带一提blog不能更新太可惜了，于是决定手动更新.jpg
