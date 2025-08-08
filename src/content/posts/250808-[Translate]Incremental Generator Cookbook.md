---
title: 【译】增量源生成器使用手册
published: 2025-08-08
description: 原标题：Incremental Generators Cookbook
tags: [C#, Roslyn, 源生成器, 翻译]
category: C#
draft: false
---

> [!NOTE]
> 本文为[dotnet/roslyn](https://github.com/dotnet/roslyn)项目中[incremental-genrators.cookbook.md](https://github.com/dotnet/roslyn/blob/73ea6887cb8abecabf3dcd3b8109d4336690afe9/docs/features/incremental-generators.cookbook.md)文档的中文翻译，原文由.NET团队撰写，并采用MIT License发布。


# 译者序

最近没事翻翻又翻到了这篇文章，便详细读了一下。发现之前在网上冲浪学到的东西里有很多关于**Incremental**的要点没提（也就是说之前我写的生成器其实和旧版生成器没什么性能差别orz）。

正好想试着翻译点东西看看，就开动了。网上可以直接搜到另一篇[Incremental Generator](https://github.com/dotnet/roslyn/blob/main/docs/features/incremental-generators.md)的中文机译，所以我直接先译这篇。

由于某些专有名词在中文太过容易混淆（点名批评Attribute），本文在部分情况下保留了一些英文名词（比如Attribute）。

# 增量源生成器使用手册

## 概述

本文旨在通过提供一系列以常见模式的指南，以帮助用户创建源生成器。同时也说明哪些类型的生成器当前是可用的、哪些在特性发布的最终设计里明确不会实现。

**本文扩展了[完整设计文档](https://github.com/dotnet/roslyn/blob/73ea6887cb8abecabf3dcd3b8109d4336690afe9/docs/features/incremental-generators.md)的细节，请确保先阅读该文章**

## 目录

- 源生成器使用手册
  - [概述](#概述)
  - [目录](#目录)
  - [提案](#提案)
  - [非设计目标](#非设计目标)
    - [语言特性](#语言特性)
    - [代码重写](#代码重写)
  - [约定](#约定)
    - [流水线模型设计](#流水线模型设计)
    - [使用`ForAttributeWithMetadataName`](#使用forattributewithmetadataname)
    - [使用缩进文本编写器(indented text writer)，而非`SyntaxNode`进行生成](#使用缩进文本编写器indented-text-writer而非syntaxnode进行生成)
    - [将`Microsoft.CodeAnalysis.EmbededAttribute`标记在生成的标记类型上](#将microsoftcodeanalysisembededattribute标记在生成的标记类型上)
    - [不要查找非直接实现的接口、非直接继承的类型、或从接口、基类继承的非直接标记的Attribute](#不要查找非直接实现的接口非直接继承的类型或从接口基类继承的非直接标记的attribute)
  - [设计](#设计)
    - [生成类](#生成类)
    - [转换附加文件](#转换附加文件)
    - [增强用户代码](#增强用户代码)
    - [发出诊断](#发出诊断)
    - [INotifyPropertyChanged](#inotifypropertychanged)
    - [将生成器打包进NuGet包](#将生成器打包进nuget包)
    - [使用NuGet包提供的功能](#使用nuget包提供的功能)
    - [访问分析器配置属性](#访问分析器配置属性)
    - [使用MSBuild属性与元数据](#使用msbuild属性与元数据)
    - [生成器的单元测试](#生成器的单元测试)
    - [自动实现接口](#自动实现接口)
  - [破坏性更改](#破坏性更改)
  - [未解决的问题](#未解决的问题)

## 提案

再次重提，源生成器的高层设计目标为：

- 生成器生成一个或多个表示C#源代码的字符串，用于添加到编译中。
- 明确只允许*添加*。生成器可以添加新的源代码到编译中，但**不**允许修改现有的用户代码。
- 可以访问*附加文件*，也就是非C#源代码文本。
- *不保证运行顺序*，所有源生成器会获取相同的输入编译结果，不能访问由其他源生成器创建的文件。
- 用户通过程序集列表指定要运行的生成器，类似分析器。
- 生成器创建流水线(pipline)，从基础的输入源开始，将它们映射到想要生成的输出。暴露的、能正确判等的状态越多，编译器就能越早中止映射，并重用相同的输出。

## 非设计目标

我们将以不可解决的问题为例，简要说明哪些*不是*源生成器设计用来解决的问题。

### 语言特性

源生成器并不是用来替换新的语言特性的：比如，我们可以想象使用源生成器实现[记录类型(Records)](https://github.com/dotnet/roslyn/blob/73ea6887cb8abecabf3dcd3b8109d4336690afe9/docs/features/records.md)，将特定的语法转换为可编译的C#表示。

我们明确地将此认定为反模式。语言会继续升级以及添加新的特性，我们不希望使用源生成器来大成这一点。这么做会产生新的C#的“方言”，无法兼容没有生成器的编译器。更进一步，因为源生成器在设计上不允许互相交互，通过这种方法实现的语言特性会很快无法与其他语言新增的特性兼容。

### 代码重写

如今有很多用户在程序集上执行的后处理任务，我们将其粗略的定义为“代码重写”。包括且不限于：
- 优化
- 日志注入(Logging injection)
- IL织入(IL Weaving)
- 调用站点重写(Call site re-writing)

这些技术有很多有价值的使用案例，它们并不符合*源生成*的思想。它们在定义上是改变代码的操作，而源生成器订单提案明确排除了这些操作。

目前已有很多工具和技术可以很好地实现这些操作，源生成器提案的目标并不是并不代替它们。我们正在尝试一种新的方法实现调用站点重写（见[拦截器](https://github.com/dotnet/roslyn/blob/73ea6887cb8abecabf3dcd3b8109d4336690afe9/docs/features/interceptors.md)），但这些特性仍然处于实验状态，可能会有重大变更，甚至被移除。

## 约定

### 流水线模型设计

作为一般准则，源生成器流水性需要传递*值相等*的模型。这对于`IIncrementalGenerator`的增量性质十分重要；一旦流水线的某一步返回了与上次运行相同的结果，生成器的驱动程序就可以停止运行生成器，并重用生成器在上一次运行中缓存的数据。大多数时间，当生成器触发时（尤其是需要使用`ForAttributeWithMetadataName`查找类型或方法定义的生成器），触发生成器的编辑并不对生成器关注的内容产生影响。然而，由于每次编辑的基础语义都可能发生变化，生成器的驱动程序*必须*每次都重新运行生成器来确保状况正确。如果生成器产生了与上一次运行相同的值，就可以短路掉流水线，避免了很多执行。以下是关于确保相等性的模型设计的一般准则：

- 使用记录类型(`record`)而不是普通类(`class`)，这样可以自动生成值相等性的对象。
- Symbol（`ISymbol`以及继承自该接口的任何类型）永远不会相等，持有这些对象的模型可能潜在的含有旧的编译结果，并要求Roslyn持有本可以释放的大量内存。永远不要将Symbol包含在你的模型类型中。作为替代，获取你需要从Symbol获取的信息，并将其转化为可值相等的表示形式：`string`一般是个很好的选择。
- `SyntaxNode`在两次运行之间通常也不会相等，但在流水线的开始阶段并不像Symbol那样强烈不推荐，[后续](..)的一个例子展示了需要将`SyntaxNode`包含在模型中的一个案例。语法结点通常也不会需要保存像Symbol那么多的内存。然而，对某个文件任何修改都会是该文件内的`SyntaxNode`不再相等，因此你仍然应该今早将其移除模型实例中。
- 上一点同样对`Location`有效[^1]
- 注意模型类型中使用的集合类型。.NET中绝大多数内置的集合类型默认并不是值相等的。比如数组、`ImmutableArray<T>`以及`List<T>`是引用相等，而非值相等。我们建议大多数生成器作者对集合使用包装类新来扩展值相等性。

[^1]: 译注：参考[`RegexGenerator`源码](https://source.dot.net/#System.Text.RegularExpressions.Generator/RegexGenerator.Parser.cs,252)你可以发现，其实有办法让Location可以参与值相等判断

### 使用`ForAttributeWithMetadataName`

我们强烈推荐所有需要检查语法结点的源生成器作者使用标记特性(Marker attribute)来只是需要检查的类型或成员。这有对作者以及用户都有很多好处：

- 作为作者，你可以使用`SyntaxProvider.ForAttributeWithMetadataName`。这个实用的方法的效率比`SyntaxProvider.CreateSyntaxProvider`高至少99倍。，在某些情况下可能更高。这样可以避免你的用户在编辑器中出现性能问题。
- 用户可以清晰地指定他们需要使用源生成器。这种设计对于良好的用户体验十分有用。这表示当用户想要使用你的生成器但在某些情况下违反了你的规则时，你可以使用Roslyn分析器来帮助用户。比如，如果你想生成某个方法的方法体，你的生成器要求用户返回某个特定的类型，一个`GenerateMe` Attribute的存在意味着你可以编写分析器来告诉用户他们的方法返回了错误的类型。

### 使用缩进文本编写器(indented text writer)，而非`SyntaxNode`进行生成

我们不推荐在为`AddSource`生成语法时创建`SyntaxNode`。这么做会很复杂，并且十分难以进行良好的格式化；调用`NormalizeWhitespace`开销巨大，并且这个API并不是为此设计的。另外，为确保不可变性，`AddSource`不接受`SyntaxNode`作为参数，而是需要字符串(`string`)的表示形式，并存储为`SourceText`。代替`SyntaxNode`，我们推荐使用`StringBuilder`的包装类型，通过记录缩进等级，并在调用`AppendLine`时添加正确的缩进。[^2]关于`NormalizeWhitespace`的性能的更多案例、性能测量、以及关于为什么我们不认为`SyntaxNode`不是生成代码的合适方式可以见[这个会话](https://github.com/dotnet/roslyn/issues/52914#issuecomment-1732680995)。

[^2]: 译注：.NET内置了[`IndentexTextWriter`](https://learn.microsoft.com/zh-cn/dotnet/api/system.codedom.compiler.indentedtextwriter?view=net-8.0)，不需要手搓wrapper

### 将`Microsoft.CodeAnalysis.EmbededAttribute`标记在生成的标记类型上

用户可能会在一个解决方案的多个项目中使用生成器，这些项目通常会有`InternalsVisibleTo`标记。这意味着`internal`的标记类型会在多个项目中定义，而编译器会发出警告。阻止编译会让用户不满。为了避免这个问题，为你的标记特性类型添加`Microsoft.CodeAnalysis.EmbeddedAttribute`；当编译器获取到其他程序集或项目上的这个类型时，编译器不会将其包括在查找结果中。为确保`Microsoft.CodeAnalysis.EmbededAttribute`在编译中可用，在`RegisterPostInitializationOutput`的回调中调用`AddEmbededAttributeDefination`方法。

另一个选项是在你的nuget包中提供一个定义了你的标记特性类型的程序集，但这对作者来说会更困难。我们推荐使用`EmbededAttribute`，除非你需要支持Roslyn 4.14一下的版本。

### 不要查找非直接实现的接口、非直接继承的类型、或从接口、基类继承的非直接标记的Attribute

使用生成器时，使用接口/基类标记十分方便且自然。然而，查找这些类型上的标记类型开销*十分*巨大，并且无法应用增量性质完成。这么做会对IDE以及命令行性能产生巨大影响，即使对轻量使用的用户来说也是如此。这些场景包括：

- 用户在`BaseModelType`上实现了某个接口，生成器查找所有`BaseModelType`的派生类。由于生成器无法提前知道`BaseModelType`是什么，因此生成器需要在所有编译结果的所有类型上获取`AllInterfaces`来查找标记接口。视用户运行生成器的模式而定，在每次输入或每次保存文件时都会造成这个问题；无论哪种情况，都是对IDE性能的灾难性影响，哪怕通过缩小扫描范围，只扫描具有基类列表的类型来进行优化也是如此。
- 用户从生成器生成的`BaseSerializerType`派生了新类型，生成器直接或间接地查找派生自该类型的所有类型。类似上一个场景，生成器将需要扫描所有具有基类的类型来查找`BaseSerializerType`的派生类，这将严重影响IDE性能。
- 生成器查找某个类型的所有标记了生成器提供的标记特性的基类/实现的接口。这和场景1、场景2没什么区别，只是搜索标准不同。
- 生成器允许标记特性类型被继承，并希望用户能够通过继承该类型来实现参数的自定义。这有几个问题：首先，所有带Attribute的类型都需要检查Attribute是否继承自标记特性类型。这仍然会影响性能，尽管没有前三种那么严重。其次，更重要的一点是，没有好的办法从派生的Attribute中获取自定义信息。这些Attribute不是由源生成器创建的，因此生成器无法获取任何传递给`base()`构造调用的参数和赋值到基类Attribute的属性的值。建议使用`ForAttributeWithMetadataName`进行开发[^3]，并使用分析器来告知用户生成器的工作是否需要继承某个基类。

[^3]: 译注喷：原文FAWMN-driven，我特么还去查了下什么术语，结果copilot告诉我这个可能是`ForAttributeWithMetadataName`的缩写。。。

## 设计

本节就用户场景差分，先列出一些通用解决方案，再提供更具体的示例。

### 生成类

**用户场景：** 我希望将某个类型添加到编译中，并且可以被用户的代码引用。一般的案例包括创建一个可用于其他生成器的Attribute。

**解决方案：** 让用户如同类型已经存在一样编写代码。使用`RegisterPostInitializationOutput`根据编译信息生成缺少的类型。

**示例：**

对如下用户代码：

``` csharp
public partial class UserClass
{
    [GeneratedNamespace.GeneratedAttribute]
    public partial void UserMethod();
}
```

创建生成器在运行时创建缺少的类型。

``` csharp
[Generator]
public class CustomGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        context.RegisterPostInitializationOutput(static postInitializationContext => {
            postInitializationContext.AddEmbeddedAttributeDefinition();
            postInitializationContext.AddSource("myGeneratedFile.cs", SourceText.From("""
                using System;
                using Microsoft.CodeAnalysis;

                namespace GeneratedNamespace
                {
                    [Embedded]
                    internal sealed class GeneratedAttribute : Attribute
                    {
                    }
                }
                """, Encoding.UTF8));
        });
    }
}
```

**其他方案：** 如果你为用户提供了一个类库，除了使用源生成器，也可以直接在类库中包含Attribute定义。

### 转换附加文件

**用户场景：** 我希望能够将外部的非C#文件转化为等效的C#表示。

**解决方案：** 使用`AdditionalTextsProvider`来筛选并获取你想要的文件。将其转化为你的类型，并将代码输出进解决方案。

**示例：**

``` csharp
[Generator]
public class FileTransformGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var pipeline = context.AdditionalTextsProvider
            .Where(static (text) => text.Path.EndsWith(".xml"))
            .Select(static (text, cancellationToken) =>
            {
                var name = Path.GetFileName(text.Path);
                var code = MyXmlToCSharpCompiler.Compile(text.GetText(cancellationToken));
                return (name, code);
            });

        context.RegisterSourceOutput(pipeline,
            static (context, pair) => 
                // Note: this AddSource is simplified. You will likely want to include the path in the name of the file to avoid
                // issues with duplicate file names in different paths in the same project.
                context.AddSource($"{pair.name}generated.cs", SourceText.From(pair.code, Encoding.UTF8)));
    }
}
```

你的csproj文件需要在ItemGroup中使用`AdditionalFiles`导入这些文件。

``` xml
<ItemGroup>
    <AdditionalFiles Include="file1.xml" />
    <AdditionalFiles Include="file2.xml" />
<ItemGroup>
```

### 增强用户代码

**用户场景：** 我希望能够检查并为用户代码增加新功能。

**解决方案：** 需要用户将你想增强的类改为`partial class`，并使用一个特性标注。在`RegisterPostInitializationOutput`中提供这个特性类。使用`ForAttributeWithMetadataName`在该特性上注册回调以收集生成代码所需的信息，使用元组（或自定义值相等模型）来传递信息。传递的信息应当从语法和Symbol获取，**切忌将语法和Symbol对象直接放入模型类**。

**示例：**

``` csharp
public partial class UserClass
{
    [GeneratedNamespace.Generated]
    public partial void UserMethod();
}
```

``` csharp
[Generator]
public class AugmentingGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        context.RegisterPostInitializationOutput(static postInitializationContext =>
            postInitializationContext.AddEmbeddedAttributeDefinition();
            postInitializationContext.AddSource("myGeneratedFile.cs", SourceText.From("""
                using System;
                using Microsoft.CodeAnalysis;
                namespace GeneratedNamespace
                {
                    [AttributeUsage(AttributeTargets.Method), Embedded]
                    internal sealed class GeneratedAttribute : Attribute
                    {
                    }
                }
                """, Encoding.UTF8)));

        var pipeline = context.SyntaxProvider.ForAttributeWithMetadataName(
            fullyQualifiedMetadataName: "GeneratedNamespace.GeneratedAttribute",
            predicate: static (syntaxNode, cancellationToken) => syntaxNode is BaseMethodDeclarationSyntax,
            transform: static (context, cancellationToken) =>
            {
                var containingClass = context.TargetSymbol.ContainingType;
                return new Model(
                    // Note: this is a simplified example. You will also need to handle the case where the type is in a global namespace, nested, etc.
                    Namespace: containingClass.ContainingNamespace?.ToDisplayString(SymbolDisplayFormat.FullyQualifiedFormat.WithGlobalNamespaceStyle(SymbolDisplayGlobalNamespaceStyle.Omitted)),
                    ClassName: containingClass.Name,
                    MethodName: context.TargetSymbol.Name);
            }
        );

        context.RegisterSourceOutput(pipeline, static (context, model) =>
        {
            var sourceText = SourceText.From($$"""
                namespace {{model.Namespace}};
                partial class {{model.ClassName}}
                {
                    partial void {{model.MethodName}}()
                    {
                        // generated code
                    }
                }
                """, Encoding.UTF8);

            context.AddSource($"{model.ClassName}_{model.MethodName}.g.cs", sourceText);
        });
    }

    private record Model(string Namespace, string ClassName, string MethodName);
}
```

### 发出诊断

**用户场景：** 我希望能够为用户的编译过程添加诊断。

**解决方案：** 我们不推荐通过生成器添加诊断。这可行，但在不破坏增量性质的前提下做到这一点是超出了本手册的范围的高级话题。作为代替，我们建议使用额外的分析器来报告诊断。

### INotifyPropertyChanged

**用户场景：** 我希望自动为用户实现`INotifyPropertyChanged`模式。

**解决方案：** “明确只允许添加”的设计宗旨似乎与实现该功能的能力相悖，这项功能似乎需要修改用户代码。但是我们可以充分发挥使用显示字段定义而不是*修改*用户定义的属性，直接通过字段提供该功能。

**示例：**

已知用户类：

``` csharp
using AutoNotify;

public partial class UserClass
{
    [AutoNotify]
    private bool _boolProp;

    [AutoNotify(PropertyName = "Count")]
    private int _intProp;
}
```

生成器可以输出如下内容：

``` csharp
using System;
using System.ComponentModel;

namespace AutoNotify
{
    [AttributeUsage(AttributeTargets.Field, Inherited = false, AllowMultiple = false)]
    sealed class AutoNotifyAttribute : Attribute
    {
        public AutoNotifyAttribute()
        {
        }
        public string PropertyName { get; set; }
    }
}


public partial class UserClass : INotifyPropertyChanged
{
    public bool BoolProp
    {
        get => _boolProp;
        set
        {
            _boolProp = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs("UserBool"));
        }
    }

    public int Count
    {
        get => _intProp;
        set
        {
            _intProp = value;
            PropertyChanged?.Invoke(this, new PropertyChangedEventArgs("Count"));
        }
    }

    public event PropertyChangedEventHandler PropertyChanged;
}
```

### 将生成器打包进NuGet包

**用户场景：** 我希望将生成器打包为NuGet包使用。

**解决方案：** 生成器可以用与打包分析器相同的方法打包。保证生成器放在包的`analyzer\dotnet\cs`文件夹下，这样就会在安装时自动添加到用户项目。

比如，要在构建时生成器打包进NuGet包，将以下内容添加到你的项目文件

``` xml
  <PropertyGroup>
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild> <!-- 在构建时生成Nuget包 -->
    <IncludeBuildOutput>false</IncludeBuildOutput> <!-- 不将生成器包含为类库依赖 -->
  </PropertyGroup>

  <ItemGroup>
    <!-- 将生成器打包到nuget的分析器目录下 -->
    <None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />
  </ItemGroup>
```

### 使用NuGet包提供的功能

**用户场景：** 我希望在生成器中使用NuGet包提供的功能。

**解决方案：** 在生成器中使用NuGet包是可能的，但是在发布时有些需要特别考虑的情况。

对于任何*运行时*依赖，也就是用户的程序需要依赖的代码，可以通过通常的引用机制添加为生成器NuGet包的依赖。

比如，假设一个生成器依赖`Newtonsoft.Json`创建代码。生成器不直接使用这个依赖，它只是会输出依赖于这个库的代码。生成器作者应该添加`Newtonsoft.Json`为公开依赖，当用户添加包时，它会被自动引用。

``` xml
<Project>
  <PropertyGroup>
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild> <!-- 在构建时生成Nuget包 -->
    <IncludeBuildOutput>false</IncludeBuildOutput> <!-- 不将生成器包含为类库依赖 -->
  </PropertyGroup>

  <ItemGroup>
    <!-- 添加Json.Net的公开依赖。生成器的用户能够直接引用这个包 -->
    <PackageReference Include="Newtonsoft.Json" Version="12.0.1" />

    <!-- 将生成器打包到nuget的分析器目录下 -->
    <None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />
  </ItemGroup>
</Project>
```

然而，然和*生成时*依赖，也就是生成器在运行且生成代码时使用的依赖，必须和生成器的程序集一起直接打包进生成器的NuGet包。没有自动工具完成这件事，你需要手动指定依赖。

假设一个生成器使用`Newtonsoft.Json`在生成时将一些东西转为json，但是不会输出任何在运行时依赖它的代码。用户可以添加`Newtonsoft.Json`的引用，并讲其所有资产(assets)标记为*私有(private)*。这么做可以保证生成器的用户不会直接继承该库的引用。

作者需要将`Newtonsoft.Json`与生成器一起打包进NuGet包。可以通过以下方法实现：添加`GeneratePathProperty="true"`让依赖生成一个路径属性，来创建新的MSBuild属性，格式为`PKG<PackageName>`，其中`<PackageName>`是将`.`替换为了`_`的包名。在我们的示例中，这个MSBuild属性为`PKGNewtonsoft_Json`，该属性含有一个指示了NuGet文件的二进制内容在硬盘上的路径。之后我们就可以使用该属性添加二进制内容到NuGet包中，和对生成器项目的操作一样。
``` xml
<Project>
  <PropertyGroup>
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild> <!-- 在构建时生成Nuget包 -->
    <IncludeBuildOutput>false</IncludeBuildOutput> <!-- 不将生成器包含为类库依赖 -->
  </PropertyGroup>

  <ItemGroup>
    <!-- 添加Newtonsoft.Json的私有依赖（PrivateAssets=all）。生成器的用户不会引用到该包
         设置GeneratePathProperty=true，以此可以通过PKGNewtonsoft_Json属性引用二进制内容 -->
    <PackageReference Include="Newtonsoft.Json" Version="12.0.1" PrivateAssets="all" GeneratePathProperty="true" />

    <!-- 将生成器打包到nuget的分析器目录下 -->
    <None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />

    <!-- 将Newtonsoft.Json依赖与生成器程序集一起打包 -->
    <None Include="$(PkgNewtonsoft_Json)\lib\netstandard2.0\*.dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />
  </ItemGroup>
</Project>
```

``` csharp
[Generator]
public class JsonUsingGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var pipeline = context.AdditionalTextsProvider.Select(static (text, cancellationToken) =>
        {
            if (!text.Path.EndsWith("*.json"))
            {
                return default;
            }

            return (Name: Path.GetFileName(text.Path), Value: Newtonsoft.Json.JsonConvert.DeserializeObject<MyObject>(text.GetText(cancellationToken).ToString()));
        })
        .Where((pair) => pair is not ((_, null) or (null, _)));

        context.RegisterSourceOutput(pipeline, static (context, pair) =>
        {
            var sourceText = SourceText.From($$"""
                namespace GeneratedNamespace
                {
                    internal sealed class GeneratedClass
                    {
                        public static const (int A, int B) SerializedContent = ({{pair.A}}, {{pair.B}});
                    }
                }
                """, Encoding.UTF8);

            context.AddSource($"{pair.Name}generated.cs", sourceText)
        });
    }

    record MyObject(int A, int B);
}
```

### 访问分析器配置属性

**用户场景：**

- 作为作者，我希望访问分析器配置属性处理语法树和附加文件。
- 作为作者，我希望访问访问键值对来自定义生成器输出。
- 作为用户，我希望使用时能够自定义生成的代码，覆盖默认配置。

**解决方案：** 生成器可以通过`AnalyzerConfigOptionsProvider`访问分析器配置值。分析器配置值可以通过`SyntaxTree`、`AdditionalFile`访问，也可以通过`GlobalOptions`全局访问。全局选项是“环境”选项，不适用于任何特定上下文，但在通过特定上下文请求时会被包括在内。

注意这是少数几种需要将`SyntaxNode`放入流水线的情况之一，你需要语法树来获取生成器选项。请尽快将`SyntaxNode`移出流水线以避免模型无法正确判等。

生成器可以自由使用全局选项来自定义输出。比如，假设一个生成器可以选择是否输出日志。作者可以选择检查全局分析器配置的值来控制是否输出日志代码。用户可以通过`.globalconfig`文件选择是否在每个项目中开启设置

``` .globalconfig
mygenerator_emit_logging = true
```

``` csharp
[Generator]
public class MyGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {

        var userCodePipeline = context.SyntaxProvider.ForAttributeWithMetadataName(... /* 收集用户代码信息 */);
        var emitLoggingPipeline = context.AnalyzerConfigOptionsProvider.Select(static (options, cancellationToken) =>
            options.GlobalOptions.TryGetValue("mygenerator_emit_logging", out var emitLoggingSwitch)
                ? emitLoggingSwitch.Equals("true", StringComparison.InvariantCultureIgnoreCase)
                : false); // 默认

        context.RegisterSourceOutput(userCodePipeline.Combine(emitLoggingPipeline), (context, pair) => /* 输出代码 */);
    }
}
```

### 使用MSBuild属性与元数据

**用户场景：**

- 作为作者，我希望基于项目文件中包含的值进行决定
- 作为用户，我希望能够自定义生成的代码，覆盖默认配置。

**解决方案：** MSBuild会将指定属性和元数据自动转换为全局分析器配置，以此可以被生成器读取。生成器作者通过添加`CompilerVisibleProperty`和`CompilerVisibleItemMetadata`到ItemGroup来启用特定的属性和元数据。可以通过props或targets文件在打包为NuGet包时进行添加。

比如，假设一个生成器通过附加文件创建了代码，并希望那个用户能够通过项目文件启用或关闭日志。作者可以在props文件中指定需要的MSBuild属性对编译器可见：

``` xml
<ItemGroup>
    <CompilerVisibleProperty Include="MyGenerator_EnableLogging" />
</ItemGroup>
```

`MyGenerator_EnableLogging`属性的值会在构建前以`build_property.MyGenerator_EnableLogging`为名输出到生成的分析器配置文件。之后生成器可以通过`AnalyzerConfigOptionsProvider`的`GlobalOptions`属性读取该配置属性：

``` csharp
context.AnalyzerConfigOptionsProvider.Select((provider, ct) =>
    provider.GlobalOptions.TryGetValue("build_property.MyGenerator_EnableLogging", out var emitLoggingSwitch)
        ? emitLoggingSwitch.Equals("true", StringComparison.InvariantCultureIgnoreCase) : false);
```

于是用户可以通过设置项目文件中的属性来启用或禁用日志。

现在，假设生成器作者想要在每个附加文件的基础上选择性地允许启用/禁用日志。作者可以通过添加`CompilderVisibleItemMetadata`到ItemGroup要求MSBuild输出指定文件的元数据的值。作者同时指定了其想要获取元数据的MSBuild项类型（此处为`AdditionalFiles`），以及其想要获取的元数据的名称。

``` xml
<ItemGroup>
    <CompilerVisibleItemMetadata Include="AdditionalFiles" MetadataName="MyGenerator_EnableLogging" />
</ItemGroup>
```

`MyGenerator_EnableLogging`的值会为编译中的每个附加文件输出名为`build_metadata.AdditionalFiles.MyGenerator_EnableLogging`的值到生成的分析器配置文件中。生成器可以通过每个附加文件的上下文获取该值。

``` csharp
context.AdditionalTextsProvider
       .Combine(context.AnalyzerConfigOptionsProvider)
       .Select((pair, ctx) =>
           pair.Right.GetOptions(pair.Left).TryGetValue("build_metadata.AdditionalFiles.MyGenerator_EnableLogging", out var perFileLoggingSwitch)
               ? perFileLoggingSwitch : false);
```

在用户配置文件中，用户可以对每个独立的附加文件标记其是否需要启用日志：

``` xml
<ItemGroup>
    <AdditionalFiles Include="file1.txt" />  <!-- 通过默认值或全局值控制是否启用日志 -->
    <AdditionalFiles Include="file2.txt" MyGenerator_EnableLogging="true" />  <!-- 启用该文件的日志功能 -->
    <AdditionalFiles Include="file3.txt" MyGenerator_EnableLogging="false" /> <!-- 禁用该文件的日志功能 -->
</ItemGroup>
```

**完整示例：**

MyGenerator.props:

``` xml
<Project>
    <ItemGroup>
        <CompilerVisibleProperty Include="MyGenerator_EnableLogging" />
        <CompilerVisibleItemMetadata Include="AdditionalFiles" MetadataName="MyGenerator_EnableLogging" />
    </ItemGroup>
</Project>
```

MyGenerator.csproj:

``` xml
<Project>
  <PropertyGroup>
    <GeneratePackageOnBuild>true</GeneratePackageOnBuild> <!-- 在构建时生成Nuget包 -->
    <IncludeBuildOutput>false</IncludeBuildOutput> <!-- 不将生成器包含为类库依赖 -->
  </PropertyGroup>

  <ItemGroup>
    <!-- 将生成器打包到nuget的分析器目录下 -->
    <None Include="$(OutputPath)\$(AssemblyName).dll" Pack="true" PackagePath="analyzers/dotnet/cs" Visible="false" />

    <!-- 打包props文件 -->
    <None Include="MyGenerator.props" Pack="true" PackagePath="build" Visible="false" />
  </ItemGroup>
</Project>
```

MyGenerator.cs:

``` csharp
[Generator]
public class MyGenerator : IIncrementalGenerator
{
    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        var emitLoggingPipeline = context.AdditionalTextsProvider
            .Combine(context.AnalyzerConfigOptionsProvider)
            .Select((pair, ctx) =>
                pair.Right.GetOptions(pair.Left).TryGetValue("build_metadata.AdditionalFiles.MyGenerator_EnableLogging", out var perFileLoggingSwitch)
                ? perFileLoggingSwitch.Equals("true", StringComparison.OrdinalIgnoreCase)
                : pair.Right.GlobalOptions.TryGetValue("build_property.MyGenerator_EnableLogging", out var emitLoggingSwitch)
                  ? emitLoggingSwitch.Equals("true", StringComparison.OrdinalIgnoreCase)
                  : false);

        var sourcePipeline = context.AdditionalTextsProvider.Select((file, ctx) => /* 收集构建信息 */);

        context.RegisterSourceOutput(sourcePipeline.Combine(emitLoggingPipeline), (context, pair) => /* 添加源 */);
    }
}
```

### 生成器的单元测试

**用户场景：** 我希望能够对生成器进行单元测试来确保开发更渐染，并保证正确。

**解决方案A：**

推荐的方式是使用[Microsoft.CodeAnalysis.Testing](https://github.com/dotnet/roslyn-sdk/tree/main/src/Microsoft.CodeAnalysis.Testing#microsoftcodeanalysistesting)包

- `Microsoft.CodeAnalysis.CSharp.SourceGenerators.Testing.MSTest`
- `Microsoft.CodeAnalysis.VisualBasic.SourceGenerators.Testing.MSTest`
- `Microsoft.CodeAnalysis.CSharp.SourceGenerators.Testing.NUnit`
- `Microsoft.CodeAnalysis.VisualBasic.SourceGenerators.Testing.NUnit`
- `Microsoft.CodeAnalysis.CSharp.SourceGenerators.Testing.XUnit`
- `Microsoft.CodeAnalysis.VisualBasic.SourceGenerators.Testing.XUnit`

TODO: [原文未编写](https://github.com/dotnet/roslyn/issues/72149)

### 自动实现接口

**用户场景：** 我希望能够为用户自动实现作为参数传递给类的标记特性的接口的属性[^4]

**解决方案：** 需要用户将类标记为`[AutoImplement]`特性，并将需要自动实现的接口的类型作为参数；标记了改Attribute的类必须为`partial class`。在`RegisterPostInitializationOutput`中提供该Attribute类型。使用`fullyQualifiedMetadataName: FullyQualifiedAttributeName`通过`ForAttributeWithMetadataName`为标记过的类注册回调，然后使用元组（或创建值相等模型）传递信息。Attribute也可以对结构体生效，以下作为指导实例尽量保持简单。

**示例：**

``` csharp
public interface IUserInterface
{
    int InterfaceProperty { get; set; }
}

public interface IUserInterface2
{
    float InterfacePropertyOnlyGetter { get; }
}

[AutoImplementProperties(typeof(IUserInterface), typeof(IUserInterface2))]
public partial class UserClass
{
    public string UserProp { get; set; }
}
```

``` csharp
#nullable enable
[Generator]
public class AutoImplementGenerator : IIncrementalGenerator
{
    private const string AttributeNameSpace = "AttributeGenerator";
    private const string AttributeName = "AutoImplementProperties";
    private const string AttributeClassName = $"{AttributeName}Attribute";
    private const string FullyQualifiedAttributeName = $"{AttributeNameSpace}.{AttributeClassName}";

    public void Initialize(IncrementalGeneratorInitializationContext context)
    {
        context.RegisterPostInitializationOutput(ctx =>
        {
            ctx.AddEmbeddedAttributeDefinition();
            //生成AutoImplementPropertiesAttribute类
            const string autoImplementAttributeDeclarationCode = $$"""
// <auto-generated/>
using System;
using Microsoft.CodeAnalysis;
namespace {{AttributeNameSpace}};

[AttributeUsage(AttributeTargets.Class, Inherited = false, AllowMultiple = false), Embedded]
internal sealed class {{AttributeClassName}} : Attribute
{
    public Type[] InterfacesTypes { get; }
    public {{AttributeClassName}}(params Type[] interfacesTypes)
    {
        InterfacesTypes = interfacesTypes;
    }
}
""";
            ctx.AddSource($"{AttributeClassName}.g.cs", autoImplementAttributeDeclarationCode);
        });

        IncrementalValuesProvider<ClassModel> provider = context.SyntaxProvider.ForAttributeWithMetadataName(
            fullyQualifiedMetadataName: FullyQualifiedAttributeName,
            predicate: static (node, cancellationToken_) => node is ClassDeclarationSyntax,
            transform: static (ctx, cancellationToken) =>
            {
                ISymbol classSymbol = ctx.TargetSymbol;

                return new ClassModel(
                    classSymbol.Name,
                    classSymbol.ContainingNamespace.ToDisplayString(),
                    GetInterfaceModels(ctx.Attributes[0])
                    );
            });

        context.RegisterSourceOutput(provider, static (context, classModel) =>
        {
            foreach (InterfaceModel interfaceModel in classModel.Interfaces)
            {
                StringBuilder sourceBuilder = new($$"""
                    // <auto-generated/>
                    namespace {{classModel.NameSpace}};

                    public partial class {{classModel.Name}} : {{interfaceModel.FullyQualifiedName}}
                    {

                    """);

                foreach (string property in interfaceModel.Properties)
                {
                    sourceBuilder.AppendLine(property);
                }

                sourceBuilder.AppendLine("""
                    }
                    """);

                //连接类名和接口名来保证如果一个类使用AutoImplementAttribute实现了两个接口时拥有唯一文件名
                string generatedFileName = $"{classModel.Name}_{interfaceModel.FullyQualifiedName}.g.cs";
                context.AddSource(generatedFileName, sourceBuilder.ToString());
            }
        });
    }

    private static EquatableList<InterfaceModel> GetInterfaceModels(AttributeData attribute)
    {
        EquatableList<InterfaceModel> ret = [];

        if (attribute.ConstructorArguments.Length == 0)
            return ret;

        foreach(TypedConstant constructorArgumentValue in attribute.ConstructorArguments[0].Values)
        {
            if (constructorArgumentValue.Value is INamedTypeSymbol { TypeKind: TypeKind.Interface } interfaceSymbol)
            {
                EquatableList<string> properties = new();

                foreach (IPropertySymbol interfaceProperty in interfaceSymbol
                    .GetMembers()
                    .OfType<IPropertySymbol>())
                {
                    string type = interfaceProperty.Type.ToDisplayString(SymbolDisplayFormat.FullyQualifiedFormat);

                    //检查属性是否有setter
                    string setter = interfaceProperty.SetMethod is not null
                        ? "set; "
                        : string.Empty;

                    properties.Add($$"""
                            public {{type}} {{interfaceProperty.Name}} { get; {{setter}}}
                        """);
                }

                ret.Add(new InterfaceModel(interfaceSymbol.ToDisplayString(), properties));
            }
        }

        return ret;
    }

    private record ClassModel(string Name, string NameSpace, EquatableList<InterfaceModel> Interfaces);
    private record InterfaceModel(string FullyQualifiedName, EquatableList<string> Properties);

    private class EquatableList<T> : List<T>, IEquatable<EquatableList<T>>
    {
        public bool Equals(EquatableList<T>? other)
        {
            // 如果other的列表为null或大小不同，那么两者不想等
            if (other is null || Count != other.Count)
            {
                return false;
            }

            // 比较每一对值的相等性
            for (int i = 0; i < Count; i++)
            {
                if (!EqualityComparer<T>.Default.Equals(this[i], other[i]))
                {
                    return false;
                }
            }

            // 如果代码执行到这里，那么两个列表相等
            return true;
        }
        public override bool Equals(object obj)
        {
            return Equals(obj as EquatableList<T>);
        }
        public override int GetHashCode()
        {
            return this.Select(item => item?.GetHashCode() ?? 0).Aggregate((x, y) => x ^ y);
        }
        public static bool operator ==(EquatableList<T> list1, EquatableList<T> list2)
        {
            return ReferenceEquals(list1, list2)
                || list1 is not null && list2 is not null && list1.Equals(list2);
        }
        public static bool operator !=(EquatableList<T> list1, EquatableList<T> list2)
        {
            return !(list1 == list2);
        }
    }
}
```

## 破坏性更改

- 暂无

## 未解决的问题

翻译略。