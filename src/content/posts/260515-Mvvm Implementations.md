---
title: 几种Mvvm的架构实现思路
published: 2026-05-15
description: WinUI绑定、手搓`INPC`、mobx
tags: [MVVM, mobx, WinUI]
category: 编程
draft: false
---

## 前言

入职后接触的第一个项目使用的[mobx](https://github.com/mobxjs/mobx)，虽然一眼看出来是MVVM的思路但是实际使用上感觉踩了很多的坑orz。由于以前用的C#的`INotifyPropertyChanged`+`CommunityToolkit.Mvvm`比较多，生成代码逻辑都很透明，导致一时间没弄懂mobx的黑盒Proxy是怎么实现绑定的。然后经过几个月的偷闲尝试和研究目前应该是弄懂了mobx的实现机制（虽然没看源码看着太累了）。

此外为了在Unity项目里手搓了一个小的`INotifyPropChanged`，没有自动生成纯手搓绑定，去看过WinUI生成的绑定代码。

考虑到`INotifyPropertyChanged`和mobx都是MVVM，但是实际的实现机制差距很大，感觉比较有意思，因此写篇blog玩玩。

## WinUI Xaml的`INotifyPropertyChanged`

C#自带了两个接口`INotifyPropertyChanged`和`INotifyCollectionChanged`作为MVVM的基石，后者实现比较复杂但是理念逻辑和前者一致所以就仅考虑前者了。

接口的实现很简单

``` csharp
interface INotifyPropertyChanged
{
    event PropertyChangedEventHandler? PropertyChanged;
}

class PropertyChangedEventArgs : EventArgs
{
    public string? PropertyName { get; }
}
```

`INPC`的设计就是让Model和ViewModel实现这个接口，并在property的setter中手动触发通知方法，这样只要UI注册到VM的事件，就可以实现实时同步了。

.NET自家的Mvvm重度框架WPF和WinUI用的自然就是这一套逻辑，并通过xaml超级简化了双向绑定的逻辑。可惜WPF重度使用反射导致我们很难看到具体生成的逻辑。但是！WinUI使用了生成器，因此我们可以很轻松地反编译找到生成的代码。

但是生成的代码依然比较多，简单起见我们摘一个最简单的绑定出来。对于一个简单的单向绑定

``` xml
<TextBox Text="{x:Bind ViewModel.Project.Name}">
```

虽然乍一看只需要在初始化时
``` csharp
this.ViewModel.Project.PropertyChanged += (s, e) => 
{
    if (e.PropertyName is "Name")
        this.TextBox.Text = s.Name;
}
```
就行了，对`Name`的修改都会更新到UI。

但是这种简单的处理，一没有在初始化时赋值，二当重新赋值`ViewModel.Project`时，不仅没有绑定到新的`Project`，而且旧`Project`上的事件没有unregister。

哦吼，完蛋。复杂度肉眼可见地上升了。

因此在我手搓绑定时，不得不提供一些额外的扩展方法来做一些`RegisterAndInvoke`，`RegisterNestedPropertyAndInvoke`来简化调用，此外因为需要取消注册之前的`Project`上的事件，`ViewModel`还需要实现`INotifyPropertyChanging`。

那么我们来看看WinUI生成了什么代码（有简化）：

``` csharp

void UpdateChildListener<T>(ref T cache, T obj, PropertyChangedEventHandler handler)
{
    if (obj != cache)
    {
        cache?.PropertyChanged -= handler;
        cache = obj;
        cache?.ProeprtyChanged += handler;
    }
}

void PropertyChanged_ViewModel(object s,PropertyChangedEventArgs e)
{
    if (string.IsNullOrEmpty(e.PropertyName))
    {
        if (obj != null)
            Update_ViewModel_Project(obj.Project);
        return;
    }
    if (e.PropertyName is nameof(ViewModel.Project) && obj != null)
        Update_ViewModel_Project(obj.Project);
}

void PropertyChanged_ViewModel_Project(object s, PropertyChangedEventArgs e)
{
    if (string.IsNullOrEmpty(e.PropertyName))
    {
        if (obj != null)
            Update_ViewModel_Project_Name(obj.Name);
        return
    }
    if (e.PropertyName is nameof(Project.Name) && obj != null)
        Update_ViewModel_Project_Name(obj.Name);
}

void Update_ViewModel(ViewModel obj)
{
    UpdateChildListeners(ref cache_ViewModel, obj, PropertyChanged_ViewModel);
    if (obj != null)
        Update_ViewModel_Project(obj.Project);
}

void Update_ViewModel_Project(Project obj)
{
    UpdateChildListeners(ref cache_ViewModel_Project, obj, PropertyChanged_ViewModel_Project);
    if (obj != null)
        Update_ViewModel_Project_Name(obj.Name);
}

void Update_ViewModel_Project_Name(string obj) => TextBox.Text = obj;

void Update_() => Update_ViewModel(ViewModel)
```

很多boilerplate，除了用cache来代替`INotifyPropertyChanging`外思路也基本差不多（废话我就抄的这个），相当丝滑。此外我们还发现，`e.PropertyName`为`null`时表示的是更新所有property，很欧亨利。

## 手搓绑定`INotifyPropertyChanged`

### 嵌套属性

我通过两个方法来简化这些调用。因为优化了GC加避免了sender类型是object，所以我对接口做了一些修改，不过不重要，看得懂。

``` csharp
public static void RegisterPropertyChangedAndInvoke<T>(this T self, Action<T, PropertyEventArgs> action)
    where T : INotifyPropertyChanged<T>
{
    self.PropertyChanged += action;
    action(self, PropertyEventArgs.AllProperties);
}

public static void RegisterNestedPropertyChangedAndInvoke<T, T1>(this T self, Func<T, T1?> property, string propName, Action<T1, PropertyEventArgs> action)
    where T : INotifyPropertyChanged<T>, INotifyPropertyChanging<T>
    where T1 : INotifyPropertyChanged<T1>
{
    self.RegisterPropertyChangingAndInvoke((s, e) =>
    {
        if (e.MatchProperty(propName)) {
            var prop = property(s);
            if (prop is not null)
                prop.PropertyChanged -= action;
        }
    });
    self.RegisterPropertyChangedAndInvoke((s, e) =>
    {
        if (e.MatchProperty(propName)) {
            property(s)?.RegisterPropertyChangedAndInvoke(action);
            // 顺带一提，有些情况下property(s)的值可以为null，所以还需要一个方法提供这个特殊情况
        }
    });
}
```

调用方使用方式
``` csharp

viewModel.RegisterNestedPropertyChangedAndInvoke(x => x.Project, nameof(viewModel.Project), (s, e) =>
{
    if (e.PropertyName is null or nameof(s.Name))
        this.TextBox.Text = s.Name;
})
```

勉强可以接受，但也很大程度上依赖于我的项目需要的嵌套不深。顺带由于WinUI的绑定对xaml的高度依赖，反而对于有些情况并不方便（比如converter），但是对于手搓绑定，反而converter随手写。

### Computed属性

另一种麻烦的情况是computed property，这个是Xaml也会有的问题。常见的例子是
``` csharp
string FullName => $"{FirstName} {LastName}";
```

UI侧，这个无所谓。但是ViewModel侧，我们需要在`FirstName`里手动Notify `nameof(FullName)`，这个设计上就很不人体工学，改来改去就容易漏。

此外如果`FullName`需要`Compute(FirstName, LastName)`，而这个`Compute`方法运算及其复杂，意味着你必须做手动缓存，否则很可能会在多个listener里重复计算。

而手动缓存又意味着，在`FirstName`中通知时需要清理缓存。

而即使以上都做好了，如果我在同一个方法中一次修改了`FirstName`和`LastName`，`FullName`的运算依然至少会执行两次，要解决这个问题，还需要添加新的scope的概念，否则无解。

## mobx

> 注，笔者不写网页，所以以下的ts代码是真的纯ts代码，不含html。也因此可能有些并不是原生mobx的东西

`INPC`的设计中，一切的绑定（逻辑上）都在编译时确定，因此很容易就能看出来谁绑了谁，整个事件的触发链是什么样的。

而mobx为什么一开始让我十分迷惑，原因就是其不仅是看不到触发链的黑盒，还是个运行时绑定的思路。

相比`INPC`，mobx的使用方式及其简单：

``` typescript
this.reactive.autoRun(() =>
{
    this.textBox.text = this.viewModel.project.name
})
```

嗯。。一眼就能看出底层有一大堆黑科技。

mobx的view model侧通过decorator用Proxy代替property，以此封装了通知相关的功能（此外还有缓存，不过这个暂时不重要）

而view侧，仅通过一个`autoRun`传入一个方法，就完成了`INPC`里及其繁琐的嵌套绑定和初始化。一开始我以为是`autoRun`解析了传入的function（由于我并不熟js我也不知道能不能实现），但是看了一点`autoRun`的实现就发现并不对。而经过几个月的使用似乎看出了一些端倪。

`autoRun`通过一次调用，完成ui初始化的同时，记录这次调用中使用了使用到了哪些observable property，将这个方法绑定到使用过的observable property的“ValueChanged”事件。以此完成一系列绑定。

由此，`autoRun`仅需要一次运行，就可以自动完成一系列绑定，甚至没有`INPC`需要思考哪个view对应哪个property的问题，一切都在编写listener时自动完成了。

### observable/computed

`autoRun`如何收集依赖，这就和mobx里view model property侧的设计有关系。mobx侧手动让一个property变成可通知的方式是
``` typescript
get name() {
    atom.reportObserved()
    return this._name
}
```

作为一个csharper，我看到之后的只有满脑子问号。但是结合`autoRun`的收集机制，似乎就可以理解出一些东西：
- mobx底层有一个全局变量记录当前运行的listerner（很简单，一个flag在运行前后标记一下）
- `reportObserved()`内部执行`this.valueChanged.register(mobx.runningListener)`
- 在运行过程中，如果执行到了某个property内的`atom.reportObserved()`，则`atom`的内部就会将这个listener记录到property的注册列表里
- 于是，当property变化后，就可以通知其记录的listerner执行，并且执行过程中会刷新一遍依赖，因此也不会泄漏

> NOTE again，我没看源码所以以上逻辑纯推测，api更是瞎编

当我理解出以上的逻辑后惊呼牛逼，一种与`INPC`完全不一样的思考方向。

mobx的思路极致简化了view绑定view model的过程，但是这种实现思路对于`INPC`更难理解，以至于如果不了解其实现思路，就藏着很多坑。

### property缓存

mobx对于computed property做了缓存，这意味着你可以在computed property内做任意复杂的运算，只要其依赖的其他property没变，这个属性就不会重新计算。

同时，mobx自带了action的概念做复数属性的scope，且`makeAutoObservable`时把method都标记为了action，其也轻松避免了多个property同时修改造成的多次刷新。

---

和`INPC`相比，mobx的易用性确实相当牛逼，但是代价是理解成本比`INPC`高了不知道多少，以及将很多复杂度转移到了运行时。

另外，mobx的黑盒实际上造成了亿些理解负担，东西全在运行时，绑定过程不透明。至少我踩的坑真挺多的，但是这方面的东西和本文离题了，所以就不谈了。

## 后记·Reactive

`INPC`和mobx几乎就是两个极端，一个把东西全堆编译期，一个把东西全堆运行时。在看懂mobx之后，我又简单看了一下[R3](https://github.com/cysharp/r3)的文档。发现就很好理解了。Reactive的`ReactiveProperty`等类型可以映射到mobx的property Proxy，将一部分复杂度转移到运行时，同时也可以解决`INPC`过于简单带来的嵌套、computed property的一些问题，将一些复杂的boilerplate包裹进`ReactiveProperty`。不过流似乎就是另外的东西了

## 后记

由于最开始C#的学习就是从WPF开始的，用了几年的MVVM的架构思路实际上已经是我思考的最优先架构了，到现在也基本熟悉了MVVM在实际执行中的各种考虑。但是由于一直接触的是C#的这一套MVVM，在遇到mobx后依然被一种全新的实现思路惊到了，尤其当我发现mobx也并没有我想像的那么“黑科技”，甚至实际上在C#不依赖织入就能实现一大半，确实是一种很新奇的体验。
