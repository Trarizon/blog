---
title: 【译】增量源生成器 - Incremental Generators
published: 2025-08-08
description: Roslyn 增量源生成器概述
tags: [C#, Roslyn, Source Generator]
category: C#
draft: true
---

本文基于2025年8月8日[dotnet/roslyn](https://github.com/dotnet/roslyn)main分支最新文档译制

# 增量源生成器(Incremental Generators)

## 概述

增量源生成器(Incremental generators)是用于替换[第一版源生成器](https://github.com/dotnet/roslyn/blob/73ea6887cb8abecabf3dcd3b8109d4336690afe9/docs/features/source-generators.md)的一组新API，以允许用户指定生成策略，由托管层以高性能方式应用。

### 高层设计目标

- 允许以更精细的方式定义生成器
- 扩展源生成器以支持Visual Studio中“Roslyn/CoreCLR”规模的项目
- 在各步骤之间使用缓存来减少重复工作
- 支持生成除源文本以外的其他项
- 与基于`ISourceGenrator`的实现能够同时存在

## 基本示例