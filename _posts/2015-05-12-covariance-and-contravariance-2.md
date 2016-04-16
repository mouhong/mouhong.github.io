---
layout: post
title: C#中的协变与逆变(续)
tags: Covariance, Contravariance
---

有同学反馈[上一篇关于协变与逆变](/2015/05/10/covariance-and-contravariance/)的文章不好理解，所以本文换个角度来说明。

## 一切都是为了"类型安全" ##

在理解什么协变与逆变之前，需要明白，在 C# 中，"类型安全"是非常重要的。

曾经我们抱怨为什么 C# 不支持把`IEnumerable<Cat>`类型的对象赋给声明为`IEnumerable<Animal>`的变量(`Cat`是`Animal`的子类)，C# 团队听到了(可能本来就在计划之中)，所以在 C# 4 中这个问题就不存在了，但我们不禁要问，既然支持把`IEnumerable<Cat>`赋值给`IEnumerable<Animal>`，为什么不同时支持把`IList<Cat>`赋值给`IList<Animal>`？

答案是为了**类型安全**。假如支持把`IList<Cat>`赋值给`IList<Animal>`，那下面的代码就可以正常通过编译:

```csharp
IList<Cat> cats = ...;
IList<Animal> animals = cats;

animals.Add(new Tiger());
```

但问题是，运行时上面代码的最后一行会抛出一个类型不匹配的异常，也就是说，上面的代码类型不安全。C# 想尽力避免这样的情况发生，让尽可能多的错误在编译时(而不是运行时)就可以被发现，所以它干脆就不允许把`IList<Cat>`赋给`IList<Animal>`，这样出现运行时错误的机会就会大大下降。

可以发现，上面的代码之所以会有运行时错误，主要原因在于`IList<Animal>.Add`方法的参数可接受任意的`Animal`参数，如果泛型接口的泛型参数不用在方法参数中，而只用在方法返回值中，会不会有问题呢？答案是没有问题，比如`IEnumerable<T>`:

```csharp
IEnumerable<Cat> cats = ...;
IEnumerable<Animal> animals = cats;

// 这里除了遍历 animals 还是只能遍历 animals
```

可以发现，如果把`IEnumerable<Cat>`赋给`IEnumerable<Animal>`，那我们除了遍历`IEnumerable<Animal>`还是只能遍历`IEnumerable<Animal>`（因为`IEnumerable<T>`上没有定义添加元素的方法)，自然就不会有上个例子中的运行时错误，所以 C# 4 支持把`IEnumerable<Cat>`赋给`IEnumerable<Animal>'。

## 为什么`IList<T>`不继承`IList` ##

`IEnumerable<T>`接口继承了非泛型的`IEnumerable`，那为什么看起来那么相似的`IList<T>`不继承非泛型的`IList`?

答案也是为了类型安全。非泛型`IList.Add()`方法的参数类型是`Object`，如果`IList<T>`继承`IList`，那就意味着不管`T`是什么类型，调用者都可以把任意对象添加到`IList<T>`中，编译可以通过，但运行时就会报错。而非泛型`IEnumerable`没有定义添加元素的方法，自然不会存在这个问题。

> **为什么`List<T>`同时实现了`IList<T>`和`IList`？**<br/>
> 这主要是为了更好地[兼容](http://blogs.msdn.com/b/ericlippert/archive/2011/04/04/so-many-interfaces.aspx) C# 1.0 (无泛型)。另外，`List<T>`是实现类，不是接口，我们可以发现`List<T>`是显式实现了`IList`，所以对于使用者来说，若不是把`List<T>`强制转换为`IList`，是看不到非泛型的`Add`方法的。

## 协变与逆变 ##

协变与逆变是比较理论化的术语，如果因为`Cat`可以赋给`Animal`，所以`IEnumerable<Cat>`也可以赋给`IEnumerable<Animal>`，那我们就把这种能力称为协变；如果因为`Cat`可以赋给`Animal`，所以`IEnumerable<Animal>`可以赋给`IEnumerable<Cat>`，我们就把这种能力称为逆变。

但上面这样的描述无法揭示协变与逆变的本质，所以我们对这个描述进行泛化：

```
假设存在 T 到 T' 的一个映射，记为 T -> T'，
然后在 T 和 T' 上分别应用一个操作 F，

(1) 如果 F(T) -> F(T') 成立，则称 F 操作是协变的（映射方向不变）；
(2) 如果 F(T) <- F(T') 成立，则称 F 操作是逆变的（映射方向反转）；

```

把`T`替换成`Cat`，`T'`替换成`Animal`，"映射"替换成"赋值"，`F`替换成`IEnumerable<>`，就变成：

```
Cat 可以赋值给 Animal，
然后把 Cat 和 Animal 分别套上 IEnumerable<>

(1) 如果 IEnumerable<Cat> 可以赋值给 IEnumerable<Animal>  --> 协变
(2) 如果 IEnumerable<Animal> 可以赋值给 IEnumerable<Cat>  --> 逆变
```

显然，将`IEnumerable<Animal>`赋给`IEnumerable<Cat>`类型不安全，所以 C# 不支持`IEnumerable<T>`上的逆变（但支持协变）。而对于`Action<T>`，则支持`T`上的逆变而不支持协变，原因同样也是因为类型安全性，就不赘述了。


## 总结 ##

协变与逆变本身并没有什么可不可以之说，如果能把`IEnumerable<Animal>`赋给`IEnumerable<Cat>`，那这种能力就称为逆变，只是这种逆变会导致代码类型不安全，所以 C# 不支持而已。明白 C# 中类型安全的重要性，应该就能更好地理解 C# 4 中的协变和逆变。

### 参考资料 ###

- [Eric Lippert: Covariance and Contravariance in C# (11 Parts)](http://blogs.msdn.com/b/ericlippert/archive/2007/10/16/covariance-and-contravariance-in-c-part-one.aspx)
- [Wikipedia: Covariance and contravariance (computer science)](http://en.wikipedia.org/wiki/Covariance_and_contravariance_%28computer_science%29)

