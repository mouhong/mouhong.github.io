---
layout: post
title: .NET内存管理(3) - Finalizer
tags: gc, clr, memory
series: .NET Memory Management
---

## Finalization ##

在`System.Object`类中，有一个`Finalize`虚方法，它定义为:

```csharpprotected virtual void Finalize() { }
```

`Finalize`方法的目的是，在一个对象被回收前，留下个最后的机会来做一些清理操作 (比如释放非托管资源)。.NET 中所有的类都继承自`Object`，因此每个类都可以通过重写`Finalize`方法来实现自己的“清理操作”。

`Finalize`是一个被 CLR 特殊照顾的方法，当 GC 准备回收一个对象时，会先检查其是否重写了`Finalize`，如果是，则暂不回收其内存，而把它放入一个叫做`ToBeFinalized`队列中，一个专门的线程会逐一执行`ToBeFinalized`队列中对象的`Finalize`方法。当一个对象的`Finalize`方法执行完后，它就变成了普通的垃圾对象，GC 下一次工作时会将它回收。所以，重写`Finalize`方法是有代价的，它会延长一个对象的存活时间，增加内存的压力。

> 其实一个对象可以在执行`Finalize`方法时让自己“复活”，比如把自己赋值给一个静态属性，这样它就不再是一个垃圾对象，也就不会被 GC 回收，但这是一种非常不建议的做法。

C# 设计者觉得应该在语言层面来支持这个特殊的`Finalize`方法，因此 C# 提供了一个称为析构函数的语法:

```csharp
public class MyObject
{
    ~MyObject()
    {
        // 清理操作
    }
}
```

上面的代码相当于 (C# 中无法显式重写`Finalize`):

```csharp
public class MyObject
{
    protected override void Finalize()
    {
        // 清理操作
    }
}
```

如果你来自 C++ 背景，千万不要被这里的“析构函数”迷惑，虽然是一样的语法，但效果是完全不一样的。因此，在 C# 中我们也尽量不用“析构函数”这样的称呼，而称它为 Finalizer。

## 为什么要使用 Finalizer ##

这个问题的答案可能会让你失望，如果你使用的是 .NET Framework 2.0+，99.9% 的情况下，你都不需要使用`Finalizer`，如果你用了，反而要考虑这可能是个错误 (除非你穿越到 10 年以前，在 .NET Framework 2.0 还没有发布的时候)。

.NET 中有 GC 来做自动内存管理，但它管理的仅仅只是内存，而程序中可能会用到许多内存之外的资源，比如文件句柄，网络连接等。当你使用这些资源的时候，要时刻警惕着，一定要在使用完后将其释放，否则会导致资源泄漏。但是人都会犯错，开发人员有可能某天心情不太好，导致写下的代码中忘记释放某个文件句柄。

CLR 想把这种失误带来的影响控制到最低，所以它提出了`Finalizer`来作为最后一道防线，它通常由类库开发人员来编写。比如`System.IO.FileStream`，我们可以用它来读写文件流，使用完毕后，调用`FileStream.Close()`来释放相关资源。因为`FileStream`封装了非托管资源的使用，所以作为`FileStream`使用者来说，需要关心的事情不多，惟一需要注意的就是使用完记得调用`FileStream.Close()`。

但要是忘记调用`FileStream.Close()`怎么办？这时`FileStream`的`Finalizer`就隆重登场了。`FileStream`实现人员为其添加了`Finalizer`，里面有释放相关非托管资源的代码。如果`FileStream`的使用者记得调用`FileStream.Close()`，那就打5星，然后加薪，如果忘记调用`FileStream.Close()`，那也还有`Finalizer`来做最后一道防护。GC 在准备回收`FileStream`对象时，看到它有定义`Finalizer`，会把它放入`ToBeFinalized`队列，这样，即使忘记调用`FileStream.Close()`，也还有最后一次机会来释放非托管资源。

如果开发人员有及时调用`FileStream.Close()`，那就不需要再执行`Finalizer`，所以 .NET  基础库中提供了`GC.SuppressFinalize(object)`方法来告诉 CLR 不要再执行某个对象的 Finalizer。例如下面是`FileStream.Close()`方法 (实际上是定义在`FileStream`的基类`Stream`中):

```csharp
public virtual void Close(){	this.Dispose(true);	GC.SuppressFinalize(this);}
```

`Finalizer`是最后一道防线，但我们不能依赖它，因为它执行的时机是不确定的，你可能现在独占使用完某个资源，但忘记释放它，而`Finalizer`可能在一小时后才执行，那这一小时内简直就是占着茅坑不拉屎，自己不用还导致别人不能用。`Finalizer`甚至还可能永远都没机会执行 (具体原因后续博文再介绍)，所以我们一定要记得用完就及时释放。

## Finalizer 的代价 ##

其中一个代价在文章开始时就已经提到了，`Finalizer`会导致对象被放入一个`ToBeFinalized`队列中，导致对象的存活时间变长。GC 把对象分成了三代，新创建的对象是第 0 代 (Generation 0)，GC 对这一代对象的回收频率较高，如果一个 0 代对象在一次垃圾回收中侥幸活下来了，那它就变成 1 代对象，GC 对 1 代对象的回收频率就明显降低了，如果 GC 回收 1 代对象时它还能侥幸活下来，它就变成 2 代对象，而 GC 对 2 代对象的回收频率就更低了。所以我们平常写程序要尽量避免对象“逃”到下一代，而`Finalizer`则恰好会导致一个对象在一次垃圾回收中侥幸逃脱，导致它进入下一代，进而延长了对象的存活时间。

另外，带有`Finalizer`的对象的创建开销比普通对象要大，CLR 在创建这些对象后还要把它们放到一个特殊的列表中，这样垃圾回收时才知道哪些对象有`Finalizer`。这个开销是不受`GC.SuppressFinalize()`影响的，`GC.SupressFinalize()`只是让`Finalizer`不执行而已，额外的对象创建开销还是需要的。

当前 CLR 只分配了一个线程来执行`Finalizer`，如果程序中创建了大量需要执行`Finalizer`的对象，这个线程的压力就很大，尤其当多个 CPU 都在不停创建这类对象的时候，一对多怎么扛得住啊 (未来 CLR 有可能会使用多个线程并发执行`Finalizer`)。

除此之外，`Finalizer`还很难写对。本想在这篇博文里把`Finalizer`写完，但现在看来这篇博文已经太长了，所以，下篇博文里我们再继续探讨为什么`Finalizer`很难写对，同时还会研究下为什么在 .NET Framework 2.0 以后不需要再写`Finalizer`。

## 总结 ##

只要写过 .NET 代码的，应该都知道`IDisposable`，但`Finalizer`则相应显得陌生一些，就我自己而言，我得承认我在很长的一段时间内对`Finalizer`都存在着错误的理解，所幸用到的不多，也没有太大影响。更多内容，我们下回分解。