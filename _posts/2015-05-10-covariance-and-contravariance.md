---
layout: post
title: C#中的协变与逆变
tags: Covariance, Contravariance
---

个人感觉协变(Covariance)与逆变(Contravariance)是 C# 4 中最难理解的一个特性了，因为 C# 4 用了一个非常直观的语法（`in`和`out`关键字），在很多情况下，这似乎很简单，`in`用于输入的参数，`out`用于输出的返回值，但事实上不完全如此，比如`Method(Action<T> action)`（会让人抓狂，一会再说）。这也是困扰了我相当久的问题，所以今天打算分享一下我自己的理解。

## 协变与逆变 ##

我们先引入一些记号，假设 T 和 U 是两个类型，那它们之间会有几种关系：

```
T < U
T > U
T = U
T 和 U 无关
```

比如`Animal`和`Cat`两个类型，`Cat`是`Animal`的子类，那我们就记为`Animal > Cat`或`Cat < Animal`，可以理解为，`Animal`表示所有的动物，而`Cat`只表示"猫"，`Animal`可以表示的范围比`Cat`更广，所以`Animal > Cat`。

现在假设我们分别在 T 和 U 上应用一个操作，我们用 f 函数来表示这个操作，即应用了 f 以后，T 和 U 对应地变成 f(T) 和 f(U)。

**如果应用了 f 操作以后，T 和 U 的大小关系被保留了下来，就称这个操作是协变的；反之，如果 T 和 U 的大小关系被反转了，就称这个操作是逆变的**：

```
协变：T < U，应用 f 操作后，f(T) < f(U)
逆变：T < U，应用 f 操作后，f(T) > f(U)
```

这可能有点抽象（但很重要，是后续内容的基础），我们举一个 C# 数组的例子。

把上面的 T 替换成 `Cat`，U 替换成 `Animal`，用大小关系来表示，即 `Cat < Animal`，然后把 f 操作替换为"数组化"，也就是说，应用了数组化操作后，`Cat`就成变`Cat[]`，`Animal`就变成`Animal[]`。

C# 从 1.0 开始就支持数组上的协变，这是什么意思呢？用我们上面提到的协变和逆变的定义，它可以描述为：

```
数组上的协变：
Cat < Animal，所以 Cat[] < Animal[]
```

我们都知道，在 C# 中，如果`Cat`是`Animal`的子类，即`Cat < Animal`，那下面的语法是合法的：

```csharp
Cat cat = ...;
Animal animal = cat;
```

也就是说，如果两个类型 T 和 U 满足 T < U，那么下面的代码是合法的：

```csharp
T obj = ...;
U u = obj;
```

前面我们说了，C# 从 1.0 开始就支持数组上的协变，也就是说，如果`Cat < Animal`，那么`Cat[] < Animal[]`就可以成立，那是不是意味着，我可以将一个`Cat`数组赋值给一个`Animal`数组呢？答案是确定的：

```csharp// 定义 Cat 数组Cat[] cats = new[] { new Cat { Name = "Kitty" } };// 将 Cat 数组赋值给 Animal 数组Animal[] animals = cats;```

上面的代码可以编译通过，也可以正常的跑起来。这就是 C# 1.0 在数组上对协变的支持。

## 类型安全 ##

C# 是一门类型安全的语言，比如`Animal animal = new Person()`这样的代码是没办法通过编译的（`Person`类型和`Animal`不兼容），C# 语言在设计上就在尽可能地避免类型不安全的发生，但可惜的是，数组上的协变不是类型安全的协变，我们可以通过一个例子来看：

```csharp
// 定义 Cat 数组Cat[] cats = new[] { new Cat { Name = "Kitty" } };// 将 Cat 数组赋值给 Animal 数组Animal[] animals = cats;// 修改 Animal 数组的第一个元素animals[0] = new Tiger { Name = "Tiger Lei" };```

上面的代码是可以通过编译的，但是运行时，会抛出一个`System.ArrayTypeMismatchException`的异常。在对数组的协变性的支持上，C#编译器团队曾经是有过[争议](http://blogs.msdn.com/b/ericlippert/archive/2007/10/17/covariance-and-contravariance-in-c-part-two-array-covariance.aspx)的，但是由于其它一些原因，还是加上了。

但这并不意味着 C# 会从此毫不顾忌的支持一切协变，比如`Action<>`上的协变就不被也永远不会被支持，试想一下，如果支持`Action<>`操作的协变，那会是怎样？

按前面说的，已知`Cat < Animal`，若支持`Action<>`操作上的协变，则有`Action<Cat> < Action<Animal>`，那就意味着下面的代码是合法的：

```csharp
Action<Cat> miao = cat => cat.Miao();
Action<Animal> action = miao;

action(new Tiger());
```

如果支持`Action<>`上的协变，则上面的代码可以通过编译，但很明显，最后一行在执行时会抛出一个运行时的异常，因为`Tiger`虽然长得有那么点像猫，但人家可不会`Miao`的叫啊。所以，C# 是不会允许这种情况发生的，所以上面的代码在实际中会编译错误（不管是哪个版本的编译器）。

既然无法在`Action<>`上支持类型安全的协变，那可以支持类型安全的逆变吗？我们可以来试一下，已知`Cat < Animal`，假设支持`Action<>`操作的逆变，则有`Action<Cat> > Action<Animal>`，那就意味着下面的代码是合法的：

```csharp
Action<Animal> sayHello = { it => Console.WriteLine(it.Name); };
Action<Cat> catSayHello = sayHello;

catSayHello(new Cat());
```

`sayHello`这个委托永远都只会调用`Animal`上的属性和方法，而我们永远都只会向`catSayHello`传入`Cat`或`Cat`的子类。`sayHello`既然可以处理`Animal`，那一定可以处理`Cat`，所以，上面的代码是类型安全的，也就是说，`Action<>`上的逆变是类型安全的。

虽然`Action<>`上的逆变是类型安全的，但在 C# 4.0 之前，你没有办法在代码中使用这种逆变性，所以大家可能会发牢骚，把`Action<Animal>`赋给`Action<Cat>`明明是类型安全的，会什么编译器不让我通过！不过幸运的是，C# 4.0 开始，类型安全的协变和逆变都得到了支持，但要注意的是，我们在 C# 4.0 中谈到的对协变和逆变的支持，都是在"类型安全"的前提下，类型不安全的协变和逆变是不支持的，并且，我们谈的都是对泛型参数的协变性和逆变性的支持。

## 协变逆变与泛型参数位置 ##

(1) 泛型参数若处于输出的位置，那它的协变性是类型安全的。

例如：

```csharp
public interface IEnumerator<T>
{
	T Current { get; }
}

public interface IEnumerable<T>
{
	IEnumerator<T> GetEnumerator();
}

public delegate TResult Func<TResult>();
```

`IEnumerator<T>`、`IEnumerable<T>`和`Func<T>`中的T，都是处于"输出"的位置，所以`T`是可以支持类型安全的协变的，我们可以试一下，已知`Cat < Animal`，若支持`IEnumerable<>`操作上的协变，则`IEnumerable<Cat> < IEnumerable<Animal>`，那按照"小的"可以赋值给"大的"的原则：

```csharp
IEnumerable<Cat> cats = ...;
IEnumerable<Animal> animals = cats;

// 接下来随便对 animals 怎么操作，都是类型安全的，强制类型转换除外
```

同样对，对于`Func<T>`，已知`Cat < Animal`，那`Func<Cat> < Func<Animal>`，因此：

```csharp
Func<Cat> findCat = () => new Cat();
Func<Animal> findAnimal = findCat;

Animal animal = findAnimal();
// 接下来不管怎么对 animal 操作，都是类型安全的，强制类型转换除外
```

(2) 若泛型参数处于输入的位置，则它的逆变性一般是类型安全的（不完全成立，但是我们先这么认为）。

例如：

```csharp
public interface IComparer<T>
{
	int Compare(T x, T y);
}

public delegate void Action<T>(T obj);
```

`IComparer<T>`和`Action<T>`中的`T`都是处于输入的位置，所以它们的逆变性都是类型安全的。我们可以试一下，已知`Cat < Animal`，若支持`IComparer<>`操作上的逆变，则有`IComparer<Cat> > IComparer<Animal>`，也就意味着下面的代码是合法的：

```csharp
IComparer<Animal> animalComparer = ...;
IComparer<Cat> catComparer = animalComparer;

catComparer(new Cat(), new Cat());
```

`animalComparer`可以处理任意的动物 (`Animal`)，而我们只可能向`catComparer`传入`Cat`或`Cat`的子类，既然`animalComparer`可以处理任意的动物，那当然就可以处理任意的猫了，所以上面的代码是类型安全的。`Action<>`前面已举过例子，不再重复。

因此，我们可以得到一个大致的结论，如果泛型参数处于输出的位置，那它就可以支持类型安全的协变，若泛型参数处于输入的位置，就可以支持类型安全的逆变（不完全正确，后面再细说），这也就是为什么 C# 用`out`来表示对应的泛型参数支持协变，而用`in`来表示对应的泛型参数支持逆变。`out`和`in`显然比协变和逆变这样的术语来得通俗易懂，所以这也是 C# 设计团队的聪明之处。

## 抓狂的时候到了 ##

如果说上面的内容都很好理解，那接下来的这个例子也许就会让人抓狂了。

前面说过，当泛型参数处于"输入"的位置时，它的逆变是类型安全的，这时只要在相应的泛型参数前加个`in`关键字，就可以让它支持逆变，C# 编译器就不会为难我们，但C# 编译器真的这么仁慈吗？

```csharp
public interface IFoo<in T> { }public interface IBar<in T>{    void Method(IFoo<T> foo);}```

在`IBar<T>`中，T 是处于输入的位置，所以上面的代码理应会在 C# 4.0 的编译器下编译通过，但事实上，我们会得到一个编译错误。好吧，和前一节讲的不一样，这是为什么？要怎么做才能让它编译过过？

为简化问题，我们用`X`来代指 `IFoo<T>`，用`X'`来代指 `IFoo<T'>`（`'`没有特殊含义，如果不觉得太多字母迷乱双眼的话，也可以用`Y`啊`A`啊什么的），即：

```
X  = IFoo<T>
X' = IFoo<T'>
```
```csharp
public interface IFoo<in T> { }

// 下面的问号有三种可能性，
// in, out 或什么都不加（不可变）
// 接下来我们会推导出一个合适的结果
public interface IBar<? T>
{
	void Method(X foo);
}
```

对于`IBar`来说，泛型参数`X`处于输入的位置，这和上一节中提到的`IComparer<X>`的情形是一样的，所以对于`X`来说，是可以支持类型安全的逆变的（注意，是`X`，不是`T`）。根据`X`的逆变性：

```
如果 IBar<X>  <  IBar<X'>，则必有 X  >  X'；
如果 IBar<X>  >  IBar<X'>，则必有 X  <  X';

```

我们下面只取第一种情况进行推导（两种情形可推出一致的结论）：

```
[1] 因为 IFoo<T> 中的 T 是逆变的（根据 IFoo<in T> 接口定义），因此，
[2] 若 IFoo<T>  >  IFoo<T'>，则必有 T < T'

[3] 若 IBar<X>  <  IBar<X'>，
[4] 因为 IBar<X> 上的 X 支持类型安全的逆变，因此，
[5] 必有 X  >  X'，即 IFoo<T>  >  IFoo<T'>
[6] 根据 [2] 中的结论，
[7] 必有 T < T'
```

接下来是见证奇迹的时刻，把上面推导过程的第[3]和第[7]行留下，其它的全部抹掉，就变成：

```
如果 IBar<X>  <  IBar<X'>，
必有 T < T'
```

看到了吗？要让`IBar<X>  <  IBar<X'>`成立，`T < T'`必须成立，也就意味着，如果把`T`作为`IBar`的泛型参数，那`T`只能支持类型安全的协变，而我们一开始的代码中，`IBar<T>`中的`T`被标记为`in`（要求支持逆变），当然编译器就不答应了，如果我们把它改成`out`（要求支持协变），那编译器就没有意见了，因为根据前面的推导，这样是类型安全的。

所以，可以编译的代码应该是：

```csharp
public interface IFoo<in T> { }public interface IBar<out T>{    void Method(IFoo<T> foo);}```

当然，每次这么推导也是很痛苦的，但是我们可以记住，把`T`直接作为输入参数时，那它就可以支持逆变，但如果`T`上被套了另一个操作，比如`IFoo<T>`，那可变性就会被扭转。所以，上面代码中的`in`和`out`互调位置后也可以编译通过。不过这对于方法返回值则不会有这种”扭转“。

## 不可变 (Invariance) ##

一个泛型参数如果既是输入参数，又是输出参数，那它无法支持协变和逆变（即不可变），例如 .NET 框架中的`IList<T>`接口的`T`即是不可变的，因为无法同时保证它的协变和逆变都是类型安全的。

##总结 ##

如果你使用的是 C# 4.0+，即使你没听说过协变和逆变，也很可能已经在你的代码中大量地用到了协变和逆变，例如当你把一个`IEnumerable<Cat>`赋值给一个`IEnumerable<Animal>`的时候，这在早期的 C# 中是不允许的。但把协变和逆变理解清楚却不是件容易的事，至少对我来说，困惑了我太久。

### 参考资料 ###

- [Eric Lippert: Covariance and Contravariance in C# (11 Parts)](http://blogs.msdn.com/b/ericlippert/archive/2007/10/16/covariance-and-contravariance-in-c-part-one.aspx)
- [Wikipedia: Covariance and contravariance (computer science)](http://en.wikipedia.org/wiki/Covariance_and_contravariance_%28computer_science%29)


