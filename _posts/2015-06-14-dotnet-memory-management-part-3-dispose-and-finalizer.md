---
layout: post
title: .NET内存管理(4) - Dispose 和 Finalizer
tags: gc, clr, memory
series: .NET Memory Management
---

## Dispose 模式 ##

[上一篇](/dotnet-memory-management-part-3-finalizer/)文章中我们说过，Finalizer 是“最后一道防线”，但我们不能依赖它去释放非托管资源，因为 Finalizer 的执行时机是不确定的。如果我们分配了非托管资源，要及时手工释放。现在轮到 Dispose 模式上场了，这个模式很重要，因为它和语言是紧密结合的，例如在 C# 中 using 块结束时会自动调用相关对象的`Dispose`方法。

下面是 Dispose 模式的模板:

```csharp
public class MyDisposable : IDisposable{
    // 注意: Finalizer 通常是不需要的
    ~MyDisposable()
    {
        Dispose(false);
    }
    public void Dispose()    {
    	 // 调用 Dispose 重载，传入 true        Dispose(true);        GC.SuppressFinalize(this);    }    protected virtual void Dispose(bool disposing)    {
        // 清理操作    }}
```

1. 开发人员手工调用`Dispose`时，调用的是第一个公开的`Dispose`方法，它转而调用`protected`的`Dispose`重载，并传入`disposing = true`，表示是手工调用`Dispose`。如果是 Finalizer 在调用`Dispose`重载，则传入`disposing = false`。所有的清理逻辑都应当写在`protected`的`Dispose`重载中，并通过`disposing`参数来判断`Dispose`是什么时候被调用的 (这很重要，后面再说);
2. 如果第一个`Dispose`被调用，那一定是开发人员手工调用，此时我们要告诉 GC，开发人员已经手工清理过了，不要再调用`Finalizer`，这是通过`GC.SuppressFinalize(this)`实现的。`GC.SuppressFinalize(this)`要放在`Dispose(true)`后面，因为要保证 Dispose 成功调用后才能不执行`Finalizer`;
3. 这里添加 Finalizer 是为了说明`disposing`的意义，事实上，Finalizer 不属于 Dispose 模式的内容，如上一篇博文所说，99.9% 的情况下我们都不要 Finalizer;
4. `Dispose`方法中不要抛出异常，除非我们觉得系统状态已经严重破坏，必须马上中止执行;
5. `Dispose`要允许被多次调用，调用多次和调用一次的效果要一样。我们可以在内部维护一个`bool`字段，用于标识`Dispose`是否已调用过，如果已调用过，再次调用时直接返回即可;

## 什么时候需要 Finalizer ##

.NET 中我们是没有办法直接分配并使用非托管资源的，我们一定是要通过 P/Invoke 调用非托管 API 才可能导致非托管资源的分配，该 P/Invoke 调用通常还会返回`IntPtr`，用来作为相应非托管资源的“句柄”，以便后面释放。简单的说，只有在类中使用了`IntPtr`时，才需要考虑 Finalizer。所以**不能直接套用前面的 Dispose 模板，不需要 Finalizer 时要将模板中的 Finalizer 删除，但其余部分都应保留** (不删除的性能代价在上篇博文中已经说过)。

如果我们只是引用了`FileStream`，则无需添加 Finalizer。`FileStream`是对非托管资源的一个托管包装 (Managed Wrapper)，它内部有 P/Invoke 调用，所以它会去实现 Finalizer，但实现 Finalizer 是`FileStream`要做的事，不是引用了`FileStream`的类要做的事。

常见的一个误解是以为只要我们引用了`FileStream`就要实现 Finalizer，并在里面调用`FileStream.Dispose()`。假如我们真实现了这样的 Finalizer，那当它执行的时候，调用`FileStream.Dispose()`其实已经没有意义了，就算不调用，`FileStream`中分配的非托管资源也会很快因为它的 Finalizer 的执行而被释放。但注意我们说的是不需要再重复实现Finalizer，而不是不实现`IDisposable`，引用了`FileStream`的类还是应该实现`IDisposable`，其中调用`FileStream.Dispose()`。

> 事实上，Finalizer 的调用是无序的，即使是在`MyDisposable`中引用`FileStream`，`FileStream`的 Finalizer 也可能先于`MyDisposable`的 Finalizer 执行。

.NET Framework 2.0 引入了`SafeHandle`类，它是对`IntPtr`的一个包装，并带有Finalizer，所以，现在的我们真的已经没有什么机会再直接使用`IntPtr`了，这也是为什么 99.9% 的情况下我们都不再需要 Finalizer。

## 继承实现了 Dispose 模式的基类 ##

Dispose 模式中第二个`Dispose`方法被标记为`protected virtual`，所以它注定是要给子类用的，子类需要这样来继承实现了 Dispose 模式的基类:

```csharp
public class MyDrivedDisposable : MyDisposable{    protected override void Dispose(bool disposing)    {        // 清理操作写在这里
        // 再调用 base.Dispose(disposing)        base.Dispose(disposing);    }}```

注意重写`Dispose`方法时，要在末尾调用`base.Dispose(disposing)`，以保证父类的清理代码可以执行，并且这个调用要放在方法的末尾，因为子类的清理代码可能还会用到父类中的相关资源，要保证子类使用完相关资源后才能对父类进行清理。

如果子类满足前面说的添加 Finalizer 的条件，且父类未实现 Finalizer，那可在子类加上Finalizer，其中调用`Dispose(false)`。如果父类已实现 Finalizer，那子类就不要再实现Finalizer 了，因为这样会导致 Dispose 方法的重复执行:

```csharp
public class MyFinalizable : BaseFinalizable
{
    ~MyFinalizable()
    {
        Dispose(false);
    }
}
```

上面的代码实际上会被编译器编译成:

```csharp
public class MyFinalizable : BaseFinalizable
{
    protected override void Finalize()
    {
        try 
        {
            // 子类调用一次 Dispose(false)
            Dispose(false);
        } 
        finally 
        {
            // 基类的 Finalize 中还会再调用一次 Dispose(false)
            base.Finalize();
        }
    }
}
```

## `disposing`参数 ##

一般说来，`Dispose(bool disposing)`会这么实现:

```csharp
protected virtual void Dispose(bool disposing)
{
    if (disposing)
    {
        // 这里可以调用其它托管对象的方法
        // 或按需调用其它对象的 Dispose()
    }
    
    // 这里不能调用其它托管对象的方法，
    // 如果要调用，要放到上面的 if 中去。
    // 这里可以释放非托管资源，例如 XXX.CloseHandle(...) 之类的调用
    
    // 如果基类实现了 Dispose 模式，
    // 这里就还要加上 base.Dispose(disposing)
}
```

1. 如果要调用其它托管对象的方法，一定要放到`if`中去，也就是说，只有在开发人员手工调用`Dispose`时，才可以调用其它对象的方法，这是因为 Finalizer 是无序执行的，我们内部引用了`FileStream`，并不意味着我们的 Finalizer 一定会先于`FileStream`的 Finalizer 执行，当我们的 Finalizer 执行时，`FileStream`的 Finalizer 可能已经先执行了，显然，此时调用`FileStream`上的方法是很危险的。也许被引用的对象上确实有些方法总是可以安全调用，但我们很难确定具体哪些方法可以，也许这些方法第一版本时还可以安全调用，但第二版时就不行了，所以最保险的办法就是永远别调用;
2. 可以考虑添加一个内部字段，用来标识`Dispose`是否已调用过，如果已调用过就直接返回，这可以避免`Dispose`的重复调用带来的性能影响;

## Finalizer 执行的无序性 ##

Finalizer 难写的一个主要原因在于它执行的无序性，我们很容易误以为 Finalizer 是按顺序执行的，这是很多问题的根源。考虑下面的代码:

```csharp
var stream = new FileStream("C:\\Work\\temp.txt", 
    FileMode.Create, FileAccess.Write, FileShare.None, bufferSize: 4096);
    var bytes = Encoding.ASCII.GetBytes("Hello World");stream.Write(bytes, 0, bytes.Length);GC.Collect();GC.WaitForPendingFinalizers();
```

这里我特意不调用`Dispose`，因为我要模拟忘记调用`Dispose`的场景。`FileStream`指定了 4KB 的缓冲，而我们要写入的"Hello World"显然远不足 4KB，代码的最后我们强制执行 GC，并等待 Finalizer 执行完毕，然后退出程序。

你觉得上面的代码能不能将"Hello World"写入磁盘呢？答案是确定的，也是合理的，“缓冲”其实是实现细节，对 API 使用者来说，最理想的情况就是调用`FileStream.Write()`方法后，就认为内容已写入磁盘，不用过多关心缓冲之类的东西。

现在我们把代码改成这样:

```csharp
var stream = new FileStream("C:\\Work\\temp.txt", 
    FileMode.Create, FileAccess.Write, FileShare.None, bufferSize: 4096);var writer = new StreamWriter(stream, Encoding.ASCII);writer.Write("Hello World");GC.Collect();GC.WaitForPendingFinalizers();```

如果执行上面的代码，我们会惊奇地发现"Hello World"并没有写入到磁盘，而如果我们显式添加一行`writer.Dispose()`的调用，它竟然又会如期写入磁盘，这可真是...

`StreamWriter`自己有一个缓冲，要想在忘记调用`Dispose`时也能刷新缓冲，就只能考虑在`StreamWriter`上添加 Finalizer，然后在 Finalizer 中刷新缓冲。但问题也在这，Finalizer 是无序执行的，执行`StreamWriter`的 Finalizer 时，其引用的`FileStream`可能已经被`Finalize`，此时刷新`StreamWriter`的缓冲必然报错。如果运气好一点，`FileStream`可能会在`StreamWriter`之后才被`Finalize`，但我们总不能靠运气写代码。

所以`StreamWriter`并没有实现 Finalizer，这意味着我们自己要注意调用`Flush`或`Dispose`来刷新缓冲，也因为这种不一致，我们写代码时不管使用的是哪个`Stream`实现，都最好显式地调用`Flush`。

但有些时候，还真的就需要有个先后顺序，于是，.NET Framework 2.0 引入了 Critical Finalizer。

## Critical Finalizer ##

`CriticalFinalizerObject`是一个超级简单的抽象类，它的源码如下:

```csharp
public abstract class CriticalFinalizerObject{    [ReliabilityContract(Consistency.WillNotCorruptState, Cer.MayFail)]    protected CriticalFinalizerObject()    {    }    [ReliabilityContract(Consistency.WillNotCorruptState, Cer.Success)]    ~CriticalFinalizerObject()    {    }}
```

虽然简单，但 CLR 却对它有特殊照顾，这实际上涉及到了 CER (Constrained Execution Region)，这也是个我们平常几乎用不到的东西，只有在编写可靠性要求极高的代码时才可能用到，本文不对其进行探讨。

这里我们只要知道，如果一个类从`CritialFinalizerObject`继承，CLR 就会保证它的 Finalizer 会在普通的 Finalizer (即非 Critical Finalizer) 执行完后才执行，这就保证了一定的有序性。例如，`FileStream`使用了`SafeHandle`，因为`SafeHandle`是`CriticalFinalizerObject`，而`FileStream`不是，所以 CLR 会保证在执行`FileStream`的 Finalizer 时，`SafeHandle`的 Finalizer 一定还没有执行。

需要使用 Finalizer 的时候本来就已经极少，使用 Critical Finalizer 的时候就更少了，所以我们了解了解即可。

> 除了保证在普通 Finalizer 之后执行，CLR 还为 Critical Finalizer 提供了另外两个保证:<br/>
> 1. 其`Finalize`方法会在对象创建时立即被 JIT 编译，我们知道 CLR 采用的是即时编译，一个方法只有在被用到时才会被 JIT 编译，而编译方法需要内存，在内存受限系统中，如果等到要执行时才编译`Finalize`，可能会导致`OutOfMemoryException`。当然，如果硬要在Finalizer 中做分配内存之类的操作 (比如装箱，字符串上的方法调用等，都可能分配内存)，那 CLR 也搞不定，所以除了 CLR 的努力，我们也要做相应配合;<br/>
> 2. 即使宿主要强制中止 (rude abort) 一个 AppDomain，那 CLR 也尽可能保证该 AppDomain 中的 Critical Finalizer 能得以执行。但 CLR 只是在尽“最大努力”保证 Finalizer 能执行，并不意味着它就能保证 Finalizer 一定会被执行 (也没办法保证)，所以永远都不要假设 Finalizer 一定会执行;

## 总结 ##

Dispose 是一个非常重要的模式，但千万别把它和 Finalizer 混淆，如果不是直接使用 P/Invoke 分配了非托管资源，我们永远都不需要使用 Finalizer (从 .NET Framework 2.0 开始，P/Invoke 返回的`IntPtr`可以全部用`SafeHandle`或其子类替代，意味着几乎没有使用 Finalizer 的时候了)。要真有需要 Finalizer 的时候，就记着，永远不要假设它是按顺序执行的，也永远不要假设它一定会执行。

### 参考资料 ###

- CLR via C# 第三版
- [cbrumme: Finalization](http://blogs.msdn.com/b/cbrumme/archive/2004/02/20/77460.aspx)
- [Eric Lippert: When everything you know is wrong](http://ericlippert.com/2015/05/18/when-everything-you-know-is-wrong-part-one/)