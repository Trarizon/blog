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

- [无GC遍历委托的InvocationList](无GC遍历委托的InvocationList)
- [通过委托创建的`Comparer`/`EqualityComparer`](#通过委托创建comparerequalitycomparer)

## 正片

### 无GC遍历委托的InvocationList

|Release|Blog update|
|-------|-----------|
|.NET 9 |25-10-10   |

``` c#
Delegate.EnumerateInvocationList<T>(T? d) where T : Delegate 
```

`Delegate`上的静态方法。顾名思义，遍历多播委托，可以用于避免`GetInvocationList`的数组分配。

> [!NOTE]
> 前两天面试提到了如何获取event的委托列表，当时一下宕机忘了`GetInvocationList`，面试官提了突然想起来。今天去[Source Browser](https://source.dot.net)看了一下发现多了个Enumerate的api，于是突发奇想写个blog。
> 顺带一提blog不能更新太可惜了，于是决定手动更新.jpg

### 通过委托创建`Comparer`/`EqualityComparer`

|Release|Blog update|
|-------|-----------|
|All(`Comparer`) <br/> .NET 8(`EqualityComparer`) |25-11-4   |

``` c#
static EqualityComparer<T> EqualityComparer<T>.Create(Func<T?, T?, bool> equals, Func<T, int>? getHashCode = null)
static Comparer<T> Comparer<T>.Create(Comparison<T> cmp)
```

两个通过委托创建`Comparer`/`EqualityComparer`的静态方法。
`EqualityComparer<T>.Create`的`getHashCode`参数不传入时，调用会抛出异常。

> [!NOTE]
> 之前在dotnet/csharplang翻[discussion](https://github.com/dotnet/csharplang/discussions/8583)时提到的一个方法。`Comparer<T>`的版本倒是始终存在，`EqualityComparer`的版本在.NET 8加入了。