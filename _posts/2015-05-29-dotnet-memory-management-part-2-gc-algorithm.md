---
layout: post
title: .NET内存管理(2) - 垃圾回收算法
tags: gc, clr, memory
series: .NET Memory Management
---

```csharp
static void Main(string[] args){
	 // 创建一个 Timer，每 2 秒调用一次 OnTimerTick    var timer = new Timer(OnTimerTick, null, 0, period: 2000);    Console.ReadKey();}static void OnTimerTick(object state){    Console.WriteLine("Tick tick");
    // Timer 的 Callback 每次调用完都回收一次垃圾    GC.Collect();}```

在 Release 模式下编译上面的代码，然后执行，你会期望看到什么结果?

期待每两秒就会得到一个 Tick tick 的输出? 可惜事实并非如此，我们只会看到输出一次 Tick tick，然后，然后就再也没有然后了 (可以在 Visual Studio 中试一下)。

但如果不用 Release ，而是用 Debug 模式编译，那我们就会如期看到每两秒输出一次 Tick tick。说实话，这种事情最可怕，试想你在开发机上一切正常，部署到服务器后各种错误，这会是怎样一种感受！不过本文会解释这个“奇怪”现象发生的原因和解决办法。

## Mark and Sweep 垃圾回收算法 ##

所谓垃圾回收，就是当我们创建的对象“不再被程序使用”时，把它占用的内存回收回来，那就需要明确一个问题，一个对象怎样才是“不再被程序使用”? Mark and Sweep 算法认为，如果一个对象是不可达的 (unreachable)，那它就“不再被程序使用”，也就成了可以回收的垃圾，反之则不是垃圾，CLR GC 用得便是这个算法。

每个程序都有一些根对象 (roots)，它们不被其它对象引用，比如静态变量、局部变量，或方法参数，但只有引用类型的对象才被认为是根对象，值类型的变量不关 GC 什么事，不去管它。

> 在 C# 中，如果一个局部变量在匿名方法中被用到了，那因为闭包的原因，这个局部变量会变成编译器生成的一个匿名类的属性，这时就有一个我们“看不见”的根对象产生了。

在 CLR GC 进行垃圾回收时，遍历所有的根对象，给它们做上标记，表示它们是可达的 (reachable)，同时，也遍历它们所引用的其它对象，也给它们做上同样的标记，完了之后，GC 就知道当前哪些对象是可达的，哪些是不可达的了，这称为**标记阶段 (Mark Phase)**。

于是，托管堆上未被标记为可达的对象就都被当作垃圾了，所以接下来 GC 的工作就是回收这些不可达对象占用的内存，然后压缩托管堆，完了 CLR 还得纠正所有对象引用的地址 (因为压缩后托管堆上对象的内存地址发生了变化)。这称为**压缩阶段 (Compact Phase)**。

> “回收对象的内存”只是一种形象的表达，事实上，GC 只要把托管堆上分散开的可达对象移到一起 (压缩)，然后再移动`NextObjPtr`指针到相应位置就可以了。
> 另外，Mark and Sweep 算法中第二个阶段称为 Sweep 阶段，但为了突出“压缩”动作，CLR GC 的第二个阶段称为 Compact 阶段更合适;

### 引用计数 ###

曾经还有一个称为“引用计数 (Reference Counting)”的垃圾回收算法，这个算法认为，如果一个对象没有被其它对象引用 (引用计数为 0)，那它就是可被回收的“垃圾”。

但引用计数有个比较严重的问题: 循环引用。假设 A 对象引用了 B，B 也引用了 A，那它们的引用计数永远都大等于1，也就意味着它们永远都不会被回收，这就可能导致内存泄漏。而如果是 Mark and Sweep 算法，因为这时 A 和 B 都是不可达的 (假设它们没被其可达对象引用)，所以它们都会被回收，也就不存在内存泄漏的问题。所以，引用计数后来就逐渐被 Mark and Sweep 取代了。

## CLR 的优化 ##

显然，一个对象占用内存的时间越短，内存的压力就越小，当然就越好，所以 CLR 对对象生命周期的管理相当苛刻，这也就直接导致了本文开头处的现象。

```csharp
 1. static void Main(string[] args) 2. {
 3.    // 创建一个 Timer，每 2 秒调用一次 OnTimerTick 4.    var timer = new Timer(OnTimerTick, null, 0, period: 2000); 5. 6.    Console.ReadKey(); 7. } 8.  9. static void OnTimerTick(object state)10. {11.    Console.WriteLine("Tick tick");
12.    // Timer 的 Callback 每次调用完都回收一次垃圾13.    GC.Collect();14. }```

直观上，我们可能会觉得`Main`方法中创建的`Timer`对象在方法返回前都不应该被回收，这和局部变量在线程栈上的生命周期一致，也好理解。但 CLR 认为这不够优化，如果一个局部变量从方法的某一行代码开始就再也没用过，那它所指向的托管对象就可以被当作垃圾了。`Main`方法中的`timer`自从声明并赋值后就再也没被用过，所以从第 5 行开始，这个才刚刚创建的`Timer`对象就已经是垃圾了 (但尚未回收)。

假如执行到第 5 行时，GC 刚好开始工作，那连一次 Tick tick 的输出都看不到 (试下在第 5 行加上`GC.Collect()`)。我们之所以能看到一次输出，是因为示例程序中没分配什么对象，所以第 5 行时 GC 还没有被触发。而我们在第 13 行处强制进行了垃圾回收，所以此时`Timer`对象就被回收了，也就看不到第二次输出了。

实际上，JIT 在编译一个方法时，会在内部创建一个表结构来存储各个局部变量的使用情况，它会记录一个局部变量在哪一行开始被使用，及哪一行结束使用。比如上面的`timer`，JIT 会记录它最后一次被使用的位置是第 3 行 (实际上，JIT 记录的是编译后指令的偏移地址)。

假设执行到第 5 行时 GC 开始工作，GC 检查 JIT 创建的内部表结构，发现`timer`已不再被使用，于是就知道`timer`所指向的`Timer`对象可以当成垃圾了，于是在 Compact 阶段中就将它给回收了。

### Debug 模式下的不同行为 ###

有意思的是，上面描述的现象只在 Release 模式下发生，在 Debug 模式时就不会，这是因为 CLR 团队考虑到，如果在 Visual Studio 中调试代码时，也那么早就把`Timer`对象当成垃圾，那开发人员调试代码时可能就没办法在 Watch 窗口中方便地检查`timer`，所以如果是在 Debug 模式下编译的，JIT 在编译方法时会自动将局部变量的生命周期延长到方法的末尾。

CLR 的这些优化是好事，但同时也可能给开发人员带来一些难解的困惑，比如一段代码，在开发期明明一切正常，部署后就不正常了。所幸这种情况只有在类似`Timer`这样的对象上才会发生，普通的对象，比如`String`，不论它在方法未执行完时就被回收，还是到方法返回时才被回收，对程序的正确性都没有影响。

### `GC.KeepAlive(obj)` ###

显然`Timer`的例子中我们不希望`Timer`被过早回收，这有很多种解决办法，一种是在方法的后面“使用”一下`timer`，比如调用其`Dispose`方法:

```csharp
static void Main(string[] args){    var timer = new Timer(OnTimerTick, null, 0, period: 2000);    Console.ReadKey();    // 通过“使用” timer 来防止被回收    timer.Dispose();}```

这样就可以顺利防止`Timer`被过早回收，但要注意下面的这种“使用”法是无效的:

```csharp
static void Main(string[] args){    var timer = new Timer(OnTimerTick, null, 0, period: 2000);    Console.ReadKey();    // 无法防止 Timer 被回收    timer = null;}
```

另一种方法是使用`GC.KeepAlive()`方法:

```csharp
static void Main(string[] args){    var timer = new Timer(OnTimerTick, null, 0, period: 2000);    Console.ReadKey();    // 防止 Timer 被过早回收    GC.KeepAlive(timer);}```

如果去看`GC.KeepAlive()`方法的源代码，会发现它什么事都没做，是一个空方法，它存在的意义就是让 JIT 认为`timer`还在被使用。所以，我们也可以自己加一个空方法来防止`Timer`被回收:

```csharp
static void Main(string[] args){    var timer = new Timer(OnTimerTick, null, 0, period: 2000);    Console.ReadKey();    // 防止 Timer 被过早回收    DoNotCollectMe(timer);}[MethodImpl(MethodImplOptions.NoInlining)]static void DoNotCollectMe(object obj){    // 什么都不做}
```

这样也能防止`Timer`过早被回收。但在开启优化选项时 (Release 模式下编译默认会开启优化选项)，编译器可能会内联`DoNotCollectMe`方法，导致`DoNotCollectMe(timer)`这行调用在优化编译后丢掉了，所以我们要使用`MethodImplOptions.NoInlining`来告诉编译器不要内联这个方法。

### 另一个例子 ###

下面的这段代码，你能看出它有什么潜在问题吗?

```csharp
public class SomeClass{    // 非托管资源    private IntPtr _unmanagedResource;    public SomeClass()    {        // 这里分配非托管资源        // _unmanagedResource = ...;    }    ~SomeClass()    {        // 这里释放 _unmanagedResource    }    public void DoSomeWork()    {        // 接下来准备使用 _unmanagedResource
        // (假设后面没再用到 this)    }}

static void Main() 
{
	 // 调用 DoSomeWork 方法
    new SomeClass().DoSomeWork();
    
    Console.ReadKey();
}
```

> GC 回收垃圾时，如果发现一个对象定义了`Finalizer`(C# 中用 C++ 中析构函数的语法来定义`Finalizer`)，就不会回收它的内存，而是把它放到一个`To be finalized`队列中，而另一个专门的线程会去执行`To be finalized`队列中对象的`Finalizer`方法，完了之后下一次垃圾回收时，该对象才会被真正被回收。

这个类在构造函数中分配了非托管资源 (`IntPtr`)，而在`Finalizer`中释放了相应的非托管资源，这一切看起来都很正常。但实际上，它有一个潜在的问题: 假如执行`DoSomeWork`方法时，恰好 GC 开始工作，那在`DoSomeWork`方法后面使用`_unmanagedResource`时，`_unmanagedResource`可能已被释放! 

也就是说，一个实例方法还没执行完时，它所在的类实例就可能已经被垃圾回收。什么?! 哥已经凌乱了，但事实就是如此。

> 不知道大家如何感觉，反正我第一次知道这个的时候感觉很惊讶，因为我下意识里觉得方法是“属于”类实例的，要是实例都没了，方法怎么还能执行? 但这种归属关系其实只是面向对象语言高度的抽象带给我们的错觉。

如前面所述，`Main`函数中`DoSomeWork`方法调用后就没再用过这个`SomeClass`对象，并且，在`DoSomeWork`方法内部没有用到`this`，因此，如果 GC 在执行`DoSomeWork`方法时开始工作，它就会认为`this`(即当前的`SomeClass`实例)是垃圾，可以被回收，因为`SomeClass`定义了`Finalizer`，所以此时`this`会被放入`To be finalized`队列，如果恰好此时负责执行`Finalizer`的线程没什么事干，马上把`SomeClass`的`Finalizer`给执行了，那等到`DoSomeWork`继续往下执行时，`_unmanagedResource`就已经失效了(在`Finalizer`执行时被释放)，后面的事情有多危险我就不说了。

下面的代码可以模拟这个问题的发生:

```csharp
public class SomeClass{    public SomeClass()    {        Console.WriteLine("分配非托管资源");    }    ~SomeClass()    {        Console.WriteLine("释放非托管资源");    }    public void DoSomeWork()    {        // 强制垃圾回收，并等待 Finalizer 执行完        GC.Collect();        GC.WaitForPendingFinalizers();        Console.WriteLine("使用非托管资源");    }}static void Main(string[] args){    new SomeClass().DoSomeWork();    Console.ReadKey();}```

在 Release 模式下编译并执行，会得到输出:

```
分配非托管资源
释放非托管资源
使用非托管资源
```

知道问题的原因后，解决办法也很简单，就是在`DoSomeWork`方法的后面加上一行`GC.KeepAlive(this)`:

```csharp
public void DoSomeWork(){    // 这里好好享用 _unmanagedResource
    
    GC.KeepAlive(this);}
```

但对于日常开发来说，这种情况其实很少会碰到，.NET 中有一个`SafeHandle`类，它包装了`IntPtr`，并且实现了标准的 Dispose 模式，所以我们只要记得，把使用`IntPtr`的地方改成使用`SafeHandle`，生活就会变得更加轻松 (又安全又少写代码)。

## 总结 ##

本文主要探讨了 CLR GC 中使用的垃圾回收算法，以及 CLR 对对象生命周期近乎苛刻的管理，从中也可以感觉到 CLR 团队在性能优化上花的心思，向他们致敬。文中也稍稍提到了`Finalizer`和 Dispose 模式，下一篇文章我们将进一步研究一下它们。

### 参考资料 ###

- 《CLR via C#》第3版
- [MDN: Memory Management](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Memory_Management)
- [Understanding Garbage Collection in .NET](http://stackoverflow.com/questions/17130382/understanding-garbage-collection-in-net/17131389)
- [Does the .NET garbage collector perform predictive analysis of code?](http://stackoverflow.com/questions/3161119/does-the-net-garbage-collector-perform-predictive-analysis-of-code)