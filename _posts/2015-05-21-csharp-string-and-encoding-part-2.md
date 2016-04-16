---
layout: post
title: C#字符串与编码 (下)
tags: string, encoding, unicode
---

[上一篇文章](/2015/05/18/csharp-string-and-encoding-part-1/)我们介绍了Unicode，它定义了一个供全人类使用的字符集合以及各自对应的码位 (Code position / code point)，而 UTF-32 和 UTF-16 是两种编码形式 (Encoding forms)，它们负责把码位以不同的方式映射到各自的编码单元 (Code unit)，比如 UTF-32 将码位一一映射到其 4 字节的编码单元，而 UTF-16 将 0x0000 - 0xFFFF 之间的码位映射到其 2 字节的编码单元，但对 0xFFFF 之上的码位则映射到两个编码单元。

但事实上，除了 UTF-32 和 UTF-16，我们更耳熟能详的应该是 UTF-8。


## UTF-8 ##

UTF-8 应用非常广泛，即使是个刚入行的小白，也应该会经常听到前辈说，“把文件保存成 UTF-8”，“这个讨厌的网站居然用的是 GB2312 编码”，等等。

之所以这么流行，是因为 UTF-8 完全兼容 ASCII，对于 ASCII 字符，UTF-8 使用和 ASCII 完全一样的编码方式，同样只使用一个字节，这就意味着，如果被编码的字符仅含 ASCII 字符，那即使是用 UTF-8 进行编码，只支持 ASCII 的旧系统仍然能够准确地解码。同时，如果被编码的字符大部分是 ASCII 字符，因为只占用一个字节，UTF-8 也最节省空间。

但有得必有失，对于其它字符，UTF-8 则需要采用二到四字节进行编码。

```csharp
// 输出 UTF-8: 3 bytes
Console.WriteLine(
	"UTF-8: " + Encoding.UTF8.GetBytes("ABC").Length + " bytes");

// 输出 UTF-16: 6 bytes
Console.WriteLine(
	"UTF-16: " + Encoding.Unicode.GetBytes("ABC").Length + " bytes");

// 输出 UTF-32: 12 bytes
Console.WriteLine(
	"UTF-32: " + Encoding.UTF32.GetBytes("ABC").Length + " bytes");
```

上面的代码对比了三种编码对"ABC"进行编码的结果，UTF-8 只需要三字节。

```csharp
// 输出 UTF-8: 6 bytes
Console.WriteLine(
	"UTF-8: " + Encoding.UTF8.GetBytes("我们").Length + " bytes");

// 输出 UTF-16: 4 bytes
Console.WriteLine(
	"UTF-16: " + Encoding.Unicode.GetBytes("我们").Length + " bytes");

// 输出 UTF-32: 8 bytes
Console.WriteLine(
	"UTF-32: " + Encoding.UTF32.GetBytes("我们").Length + " bytes");
```

上面的代码对比了三种编码分别对"我们"进行编码的结果，UTF-8 需要六字节，而 UTF-16 只需要 四字节。所以，如果大部分是中文字符，UTF-16 相对会更节省空间。而 UTF-32，无论哪种情况它基本上都是最差的，所以它应用不是很广泛，但它有一个好处是每个字符统一占用 4 字节，处理起来简单，所以在内存中使用时也可以看情况考虑 UTF-32。

UTF-8 的编码单元是 8 位，是面向字节的 (Byte-oriented)，但具体的编码算法这里不做过多介绍，网上搜索一下会有很多相关内容。

## 字节序 (Byte-order / Endianness) ##

现在我们知道了，Unicode 定义了字符集及它们各自对应的码位，Encoding Forms (UTF-32, UTF-16 和 UTF-8) 将码位映射到编码单元 (Code unit)，但这里实际上还没涉及具体的存储，涉及具体的存储时，我们需要把 UTF-32 细分为 UTF-32BE 和 UTF-32LE，相对应的 UTF-16 需要细分为 UTF-16BE 和 UTF-16LE，这些称为编码方案 (Encoding schemes)，其中的 BE 是 Big-endian 的意思，LE 是 Little-endian 的意思。

### Big-endian 和 Little-endian ##

吃一个鸡蛋的时候，要先从大头吃起呢，还是从小头吃起？这实际上是一个很纠结的问题，故事里的人们曾因为这个问题爆发过战争。

> 故事来源于英国作家斯威夫特的《格列佛游记》，在该书中，小人国里爆发了内战，战争起因是人们争论，吃鸡蛋时究竟是从大头(Big-End)敲开还是从小头(Little-End)敲开。为了这件事情，前后爆发了六次战争，一个皇帝送了命，另一个皇帝丢了王位。

当一个数据类型需要占用两个或两个以上的字节时，需要怎么在内存里存储呢？比如 C# 里声明为 Int16 类型的 31，占用两个字节，二进制表示是 0000 0000 0001 1111，这看起来有点头疼，我们把它转成 16 进制，就变成 00 1F，所以，Int16 类型的 31 的第一个字节是 00，第二个字节是 1F，这个顺序我们把它称作逻辑顺序。那我们也很自然的想到，在内存里存储时，先存 00，然后在下一个内存地址上存储 1F，如下所示：

```
内存地址增长方向
------------>

         0x100    0x101
------+--------+--------+------
  ..  |   00   |   1F   |  ..  
------+--------+--------+------
```
(0x100 和 0x101 表示内存地址，仅作示例用，其值没有特殊含义)

但是有些人却不这么认为，他们觉得应该要先存 1F，然后再存 00，于是，同样是 31，在他们看来应该要这样存储：

```
         0x100    0x101
------+--------+--------+------
  ..  |   1F   |   00   |  ..  
------+--------+--------+------
```

那么，就存在了两种选择:

(1) 先存高位字节 (Most significant byte)，然后再存低位字节，那 00 1F 就会被存储成 00 1F，这种叫做 Big-endian ("大头"先来);

(2) 先存低位字节 (Least significant byte)，然后再存高位字节，那 00 1F 就会被存储成 1F 00，这种叫做 Little-endian ("小头"先来);

不同的处理器可能会采用不同的字节序，Intel 的处理器大部分是 Little-endian，在 C# 中，可以通过`BitConverter.IsLittleEndian`获得这个信息。

> x86，MOS Technology 6502，Z80，VAX，PDP-11 等处理器为 Little endian; Motorola 6800，Motorola 68000，PowerPC 970，System/370，SPARC (除V9外) 等处理器为 Big endian; ARM，PowerPC (除PowerPC 970外)，DEC Alpha，SPARC V9，MIPS，PA-RISC and IA64 的字节序是可配置的;
>
> -- Wikipedia

### 字节序的影响 ###

首先，对于只占用一个字节的对象，是不影响的 (只有一个字节，何来谁先谁后)。其次，即使是占用两个字节或以上的对象，大部分情况下也不会影响，因为 C# 已经把这个事情做的相当透明了，但当我们要在代码中直接处理内存数据时，就不能不考虑字节序了；或者网络传输时，也要保证通信的双方都能理解传输的数据所采用的字节序。

现在我们通过 C# 来看一下，Int16 是不是真的像前面说的这样存储：

```csharp
Int16 value = 31;

byte[] bytes = BitConverter.GetBytes(value);

Console.WriteLine("Little-endian: " + BitConverter.IsLittleEndian);
Console.WriteLine(BitConverter.ToString(bytes));
```

上面的代码在我的电脑上运行时会输出：

```
Little-endian: True
1F-00
```

这是因为我的机器用的是 Little-endian。如果你用的是 Big-endian 的机器，上面的字节数组就不是`[0x1F, 0x00]`，而是`[0x00, 0x1F]`，这时如果你把你机器上拿到的这个字节数组丢给我 (Little-endian)， 而我却不经处理地直接将它转成 Int16，得到的 Int16 值就不正确：

```csharp
// 假设这是来自于 Bit-endian 的机器的字节数组，期望值是 31
byte[] bytes = new byte[] { 0x00, 0x1F };

// 在我 Little-endian 的机器上直接将它转成 Int16
Int16 value = BitConverter.ToInt16(bytes, 0);

// 输出结果
Console.WriteLine(value);
```

上面的代码在我的电脑上运行时会输出 7936，就不再是我们所期望的 31 了。

### Int32 和字节序 ###

好了，再来一个例子。Int16 占用两个字节，如果是占用四个字节的 Int32，比如 0x01234567，在 Big-endian 的机器和 Little-endian 的机器上进行存储时，分别是怎样的呢？思考一下

.<br/>
.<br/>
.

好了，也比较简单，如果是 Big-endian 的机器，那它和我们的直觉是一样的，高位字节先来：

```
         0x100    0x101   0x102    0x103
------+--------+--------+--------+-------+------
  ..  |   01   |   23   |   45   |   67  |  ..
------+--------+--------+--------+-------+------
```

如果是 Little-endian，就“反转”一下：

```
         0x100    0x101   0x102    0x103
------+--------+--------+--------+-------+------
  ..  |   67   |   45   |   23   |   01  |  ..
------+--------+--------+--------+-------+------
```

## UTF-16BE 和 UTF-16LE ##

UTF-16 的编码单元是两字节 (16位)，已经超过了一个字节，这时就需要考虑字节序。

比如字符"A"，它的 Unicode 码位是 0x0041，对应的 UTF-16 编码也是 0x0041，如果对它进行编码时，把 00 放在 41 前面（高位字节先来），就是 UTF-16 Big-Endian (UTF-16BE)，如果把 41 放在 00 前面（低位字节先来），就是 UTF-16 Little-Endian (UTF-16LE)。

UTF-16 属于 Encoding Form，而 UTF-16BE 或 UTF-16LE 属于 Encoding Scheme，区别是，前者将 Unicode 码位映射成编码单元 (还没有涉及实际存储)，而后者将 Code unit 映射成 serialized byte sequence (翻译不出来，但是大家应该明白我的意思)。

我们同样可以通过 C# 来验证一下：

```csharp
// UTF-16 Little-endian
var utf16_LE = Encoding.Unicode;

var littleEndianBytes = utf16_LE.GetBytes(str);
Console.WriteLine(BitConverter.ToString(littleEndianBytes));

// UTF-16 Big-endian
var utf16_BE = Encoding.BigEndianUnicode;
var bigEndianBytes = utf16_BE.GetBytes(str);
Console.WriteLine(BitConverter.ToString(bigEndianBytes));
```

上面的代码会输出：

```
41-00
00-41
```

注意，这个输出不会因为硬件字节序的不同而产生差异，因为每种编码方式都已经指定了确切的字节序。

## UTF-32BE 和 UTF-32LE ##

UTF-32 的编码单元是四字节 (32位)，同样超过了一个字节，所以也有 Big-endian 和 Little-endian 的区分，这和 UTF-16 类似，不再赘述。在 .NET 中，静态属性 `System.Text.Encoding.UTF32` 是 Little-endian 的，如果需要 Big-endian，则需要手工创建`System.Text.UTF32Encoding`实例，其构造函数的第一个参数可以指定字节序。

## Byte Order Mark (BOM) ##

如果拿到一个 UTF-16 编码的文本文件，但却不知道到底是哪种字节序，UTF-16LE 还是 UTF-16LE? 怎么办？说实话这个问题真比较难办。

不过有一个办法，就是在文件保存时，在文件起始处添加几个特殊的字节来标识用的是哪种字节序，这几个特殊的字节就称为 Byte Order Mark (BOM)。在 UTF-16 中，BOM 是两个字节 (一个编码单元)，Big-endian 对应的 BOM 是 0xFEFF，Little-endian 对应的 BOM 是 0xFFFE。但注意 BOM 并不是强制要写入到文件里的。

所谓的“自动检测文件编码”，也就是检测文件开头的几个字节，比如你发现文件的前两字节是 0xFEFF，那就可以用 UTF-16BE 对它进行解码，如果前两字节是 0xFFFE，就用 UTF-16LE。但需要注意的是，这个是没有办法保证完全正确的，有可能有的文件用的其它编码，但前两字节还真就刚好是 0xFEFF，出于什么目的我不懂，反正就是这样，所以你也没办法。

UTF-32 的 BOM 也类似，只不过 UTF-32 的编码单元是 4 字节，所以它用 0x0000FEFF 表示 Big-endian，而用 0xFFFE0000 表示 Little-endian。

UTF-8 的 BOM 永远都是 0xEFBBBF，这是因为 UTF-8 是 Byte-oriented 的，因为其特殊的编码规则，所以不需要 BOM。UTF-8 的 BOM 据说 M$ 的一个发明，目的就是标识这个文件用的是 UTF-8 编码，在其它平台不用，所以在保存 UTF-8 文件时，推荐不添加 BOM。

在 Windows 下，可以通过记事本来试验一下 BOM 是如何影响记事本程序对文件编码的理解的。

```csharp
var filePath = "C:\\Work\\tmp.txt";

var bytes = Encoding.Unicode.GetBytes("你好");

// 将字节数组写入文件
using (var fs = new FileStream(filePath, FileMode.Create, FileAccess.Write))
{
    fs.Write(bytes, 0, bytes.Length);
    fs.Flush();
}
```

运行上面的 C# 代码，然后在 Windows Explorer 中打开 tmp.txt，会发现我们看到的是乱码，这是因为我们用了 UTF-16LE，但没有将 BOM 写入到文件，所以记事本不知道文件是采用什么编码，所以它只好使用默认的编码打开，就变成乱码了。

如果我们把 BOM 写入到文件中：

```csharp
var filePath = "C:\\Work\\tmp.txt";

var bytes = Encoding.Unicode.GetBytes("你好");

// 将字节数组写入文件
using (var fs = new FileStream(filePath, FileMode.Create, FileAccess.Write))
{
    // 写入 BOM
    byte[] bom = new byte[] { 0xFF, 0xFE };
    fs.Write(bom, 0, bom.Length);

    fs.Write(bytes, 0, bytes.Length);
    fs.Flush();
}
```

这次再打开 tmp.txt 就会正常地看到“你好”了。这说明记事本在打开文本文件时，是会先检查 BOM 的。

## 总结 ##

大部分情况下我们都可以忽略字节序的存在，但需要在字节这个级别上处理数据时，就不得不考虑字节序了。采用什么样的字节序，更多的是当时硬件设计人员的一个选择而已，虽然不同人的不同选择给我们造成了困扰，但现在事实就是如此，也无法逃避，只能去认识它。因为字节序的原因，我们就有了 UTF-16LE 和 UTF16BE，以及 UTF-32LE 和 UTF32BE，为了方便读取文件内容的程序，我们可以在文件头部添加 BOM，这样它们就知道文件用的是什么编码了。

### 参考资料 ###

- [The Absolute Minimum Every Software Developer Absolutely, Positively Must Know About Unicode and Character Sets](http://www.joelonsoftware.com/articles/Unicode.html)
- [Unicode in .NET](http://csharpindepth.com/Articles/General/Unicode.aspx)
- [Why does C# use UTF-16 for strings](http://blog.coverity.com/2014/04/09/why-utf-16/)