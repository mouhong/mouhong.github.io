---
layout: post
title: C#字符串与编码 (上)
tags: string, encoding, unicode
---

## C#字符串 ##

我们都知道，C# 中`System.Char`用来表示一个字符，它是一个 16 位的值类型，而一个字符串 (`String`) 则是连续的一串字符，我们可以通过`String.Length`属性来获取字符串的长度。那么，下面的代码的会输出什么呢?

```
Console.WriteLine("𠬠".Length);```

答案是 2。呃，怎么不是 1? 好像有点不太合逻辑。

要回答这个问题，需要知道`Char`到底是一个什么东西。MSDN对`Char`的描述是：

> Represents a character as a UTF-16 code unit.

UTF-16 code unit? 所以，我们还要从 UTF-16 说起。

## Unicode, UTF-32 和 UTF-16 ##

早期美帝的程序员没有意识到英语只是全世界所有语言中的一种，他们以为26个英文字母再加上一些其它符号就够用了，所以就有了当时的 ASCII。但是随着互联网的发展，他们终于意识到软件原来还是需要给不同国家不同语言的人来使用的，所以就开始有了其它的编码方法，但因为缺少一个一统天下的标准，所以乱码问题非常常见 (相信不少码农在初入行时都有被乱码问题折磨过的经历)。

而 Unicode 就是要来解决这个问题，它是一个字符集，目标是定义一个能满足全人类需要的字符集合，除了定义哪些字符会被涵盖外，它还要定义每个字符所对应的码位 (Code point 或 Code position)。

> **什么是码位？**<br/>
Unicode 字义了字符集合后，需要为每个字符指定一个数字，这样计算机才有办法处理。假如字符集中有 1 万个字符，那就需要 1 万个数字，每个字符对应一个数字，这所有的 1 万个数字就构成了编码空间 (Code space)，而每个数字就是对应的字符的码位。 

> 假设字符集前三个字符是 A, B, C，我们用 0 到 9999 来编码，那 A 对应 0，B 对应 1， C 对应 2。0 就是 A 的码位，1 就是 B 的码位，以此类推。码位也就是字符在编码空间中的位置，所以叫码“位”。

在 Unicode 标准中，编码空间由 0x0000 - 0x10FFFF 之间的整数构成，一共可以容纳 1,114,112 个码位 (17 &times; 2<sup>16</sup>)，目前 (Unicode 7.0) 已分配字符的[还不到一半](http://en.wikipedia.org/wiki/Plane_(Unicode))。

好了，到现在为止，我们讨论的还只是字符集，没有涉及具体的编码。Unicode 定义了三种编码形式 (Encoding forms)，分别是 UTF-32, UTF-16 和 UTF-8，从 C# 开发人员的角度，Unicode 字符集就好像是一个接口，规定了所有“实现类”必须能正确编码的所有 Unicode 中定义的字符，而 UTF-32, UTF-16 和 UTF-8 则像是实现类，它们都可以编码 Unicode 中定义的字符，但编码方法则不一样。

### UTF-32 ###

Unicode 的编码空间为 0xFFFF - 0x10FFFF，那可以想到的最简单的办法就是让每个码位对应一个 32 位 (4 bytes) 二进制数，这就是 UTF-32 编码。所以在 UTF-32 中，每个字符占用 4 个字节。

```csharp
var encoding 
	= new UTF32Encoding(bigEndian: true, byteOrderMark: false); 
							var bytes = encoding.GetBytes("ABC");var hex = BitConverter.ToString(bytes);// 输出 12 bytesConsole.WriteLine(bytes.Length + " bytes");// 输出 00-00-00-41-00-00-00-42-00-00-00-43Debug.WriteLine(hex);
```

上面的 C# 代码使用`System.Text.UTF32Encoding`类对字符串"ABC"进行编码，编码得到的字节数组长度为 12，即每个字符占用 4 个字节。即使是"A"这么简单的字母，UTF-32 中都要占用 4 个字节 (在 ASCII 中只要一个字节)，这无疑是一种巨大的浪费，所以 UTF-32 在实际中用的不多。

UTF-32 对每个码位都使用 32 位 (4 字节) 进行编码，也就是说，32 位是 UTF-32 的最小编码单元 (Code unit)，如果有人给我一个 10 字节的字节数组，说这是 UTF-32 对某个字符串的编码结果，让我猜猜源字符串是什么，那我一定拿臭鸡蛋砸他，居然敢骗我，UTF-32 编码的结果怎么可能是 10 字节。

> 下篇再细说上面代码中的`bigEndian`和`byteOrderMark`。

### UTF-16 ###

事实上，Unicode 中常用的字符定义在 0x0000-0xFFFF 之间，那我们是不是可以对常用字符做一些优化呢？是的，这就是UTF-16。

在UTF-16中，码位在 0x0000 - 0xFFFF 之间的字符用两字节来编码，比如“我”，Unicode 码位是 U+6211，在 UTF-16 中也编码为 0x6211。而对于 0xFFFF 以上的码位 (显然两字节不够用了)，UTF-16 用四个字节来编码 (Surrogate pairs)。例如文章开头的“𠬠”，Unicode 码位是 U+20B20，在 UTF-16 中编码为 0xD842 0xDF20。我们可以用 C# 代码来试验一下：

```csharp
var encoding 
	= new UnicodeEncoding(bigEndian: true, byteOrderMark: false);
						var bytes = encoding.GetBytes("ABC");var hex = BitConverter.ToString(bytes);// 输出 6 bytesConsole.WriteLine(bytes.Length + " bytes");// 输出 00-41-00-42-00-43Console.WriteLine(hex);bytes = encoding.GetBytes("我");hex = BitConverter.ToString(bytes);// 输出 62-11Console.WriteLine(hex);bytes = encoding.GetBytes("𠬠");hex = BitConverter.ToString(bytes);// 输出 D8-42-DF-20Console.WriteLine(hex);
```

> 上面的代码使用了 .NET 中的`System.Text.UnicodeEncoding`类，它实际上指的是 UTF-16，`UnicodeEncoding`这个命名有一定的误导性 (历史原因)，其类名中的 Unicode 和我们前面说的 Unicode 不是一回事。

所以 UTF-16 是一种变长编码 (UTF-32 是定长编码)，它的编码单元是 16 位 (2 字节)，对于0xFFFF 以上的码位，需要使用两个编码单元。

## 再说 C# 字符串 ##

```
// 输出 2
Console.WriteLine("𠬠".Length);```

好了，现在我们来看看为什么文章开头的代码输出结果是 2 而不是 1。

我们先看下 MSDN 是怎么对`Char`进行描述的：

> Represents a character as a UTF-16 **code unit**.

C# 使用 UTF-16 来表示字符串 (Java 和 JavaScript 也一样)，一个字符串则由一串连续的`Char`组成，而一个`Char`则表示了一个 UTF-16 编码单元 (Code unit)。

前面说了，UTF-16 的编码单元是 16 位 (这就是为什么`Char`是 16 位的值类型)，而 16 位的编码单元只能表示码位在 0x0000 - 0xFFFF 之间的 Unicode 字符，对于码位在 0xffff 以上的字符，UTF-16 需要用两个编码单元，也就是两个`Char`来表示。所以，**`Char`和我们所认为的字符并不一定是一一对应的关系**。

`"𠬠".Length`输出"𠬠"这个字符串所包含的`Char`的数量，而“𠬠”的 Unicode 码位是 U+20B20，在 UTF-16 使用两个编码单元 (0xD842 0xDF20) 进行编码，所以它需要两个`Char`，第一个`Char`用来表示 0xD842，第二个`Char`用来表示 0xDF20。可以通过程序来证明我说的是真的：

```csharp
var str = "𠬠";var hexChar1 = ((int)str[0]).ToString("x");var hexChar2 = ((int)str[1]).ToString("x");// 输出 d842 df20Console.WriteLine(hexChar1 + " " + hexChar2);
```

而对于其它字符，比如"A"，在 UTF-16 中只使用了一个编码单元，所以用一个`Char`就可以了，这就是为什么`"A".Length`的结果是 1。

## 总结 ##

直觉上，我们很容易会以为 C# `String.Length`属性会输出字符串中我们所认为的字符的数量，而事实上，它只是输出`Char`的数量，而一个`Char`只是对应了一个 UTF-16 编码单元，它和我们所认为的字符并不一定有一一对应的关系。明白了这一点，就知道为什么`"𠬠".Length`的结果是 2 这种“奇怪”的事了。

### 参考资料 ###

- [The Absolute Minimum Every Software Developer Absolutely, Positively Must Know About Unicode and Character Sets](http://www.joelonsoftware.com/articles/Unicode.html)
- [Unicode in .NET](http://csharpindepth.com/Articles/General/Unicode.aspx)
- [Why does C# use UTF-16 for strings](http://blog.coverity.com/2014/04/09/why-utf-16/)