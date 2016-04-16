---
layout: post
title: JavaScript对象模型
tags: javascript, oop
---

JavaScript 因为其 Java 前缀，使得这门语言变成最让人误解的语言。而作为一门面向对象的编程语言，则容易让来自于其它如 Java 或 C# 背景的人员在 JavaScript 中寻找“类”的踪迹。

前人已经告诉我们, JavaScript 中没有 class，但是它有 function，相当于 class 的效果，比如：

```javascript
function Animal() {    this.name = null;
        this.sing = function () {        console.log('Singing ' + this.name);    }}var cat = new Animal();cat.name = 'Kitty';
cat.sing(); // 输出 Singing Kitty
```

上面的代码定义了一个 Animal 的类，包含 name 属性和 sing 方法。类似于 Java，它使用 new 关键字来创建类的实例，同时通过点号来调用属性和方法。

## 继承 ##

作为一门面向对象编程语言，继承是必须的。JavaScript 通过原型链 (Prototype Chain) 来实现继承。现在我们来定义一个 Cat 子类：

```javascript
function Cat() { 
	this.jump = function () { }
}Cat.prototype = new Animal;var cat = new Cat();cat.name = 'Kitty';cat.sing();
```

新定义的 Cat 只有一个 jump 方法，没有 sing，但上面的代码运行后，同样会输出 "Singing Kitty"，而不会报错，这说明 sing 方法已经从 Animal 继承下来了，而这一切，是通过`Cat.prototype = new Animal`做到的。

每个 JavaScript 对象都有一个`__proto__`属性，它是一个对象，是与生俱来的，不用显式去定义，新建的对象的`__proto__`都是指向了 Object 的 prototype，这也是为什么所有对象都可以调用 toString 等方法。

而重点在于，`__proto__`是一个特殊属性，当我们调用对象上的属性或方法时，JavaScript 会先在对象上找相应的属性或方法，如果没找到，就会顺着其`__proto__`去找，若还没找到，就再顺着其`__proto__`的`__proto__`去找，这就形成了一个原型链，原型链最终会以 null 结束。

如果上面代码中没有`Cat.prototype = new Animal`这一行，JavaScript 找不到 sing 方法，会抛出一个错误，但幸运的是，我们通过`Cat.prototype = new Animal`修改了 Cat 的 prototype，所以 JavaScript 在 cat 对象的 prototype 上找到了 sing 方法。

上面我们定义出来的对象结构构造出来的原型是这样的：

```
{ jump } --> { name, sing } --> ... --> null
```

类似的，我们还可以继续定义 Cat 的子类：

```javascript
function RobotCat() {    this.os = 'Andriod';}RobotCat.prototype = new Cat;var cat = new RobotCat();cat.name = 'Kitty';cat.sing();
```

于是原型链就变成了：

```
{ os } --> { jump } --> { name, sing } --> ... --> null
```

当然，我们也可以重写一个方法，比如：

```javascript
RobotCat.prototype.sing = function () {    console.log('Sorry, I cant sing.');}
```

这时再调用 cat.sing 就变成了输出 "Sorry, I cant sing"。方法重写并不是特例，画出原型链就知道为什么可以实现方法重写:

```
{ os } --> { jump, sing } --> { name, sing } --> ... --> null
```

{ jump, sing } 对应的是 RobotCat 的 prototype，而 { name, sing } 对应的是 Animal，它现在有自己的 sing 方法了，所以 JavaScript 在检查 RobotCat 的 prototype 时就找到了 sing，也就不会再去 Animal 上面找了。

## JavaScript对象模型 ##

上面我们用 JavaScript 的 function 去“模拟”了“类”的效果，并且用原型链实现了继承。说得仿佛 JavaScript 是一个天生有缺陷的孩子，只能通过一些 hack 的手段来模拟面向对象的效果。但 JavaScript 是一门面向对象编程语言，只不过它不是我们所熟悉的基于类，而是基于原型。这也说明它和 Java 除了语法相似，根本就是两门完全不同的语言。

在基于类的语言中，“类”和“实例”是两个独立的概念，我们先定义类，然后再创建类的实例。而在 JavaScript 中，则没有这种区别。

前面例子中我们用 function 来模拟“类”，然后用 new 创建这个模拟“类”的实例，但现在我要说，我们定义的 function，本身也是一个对象。什么？！好凌乱的感觉。

JavaScript 有两种类别的数据类型，一种是 Primitive Type，包括 Boolean, null, undefined, Number, String, Symbol (ES6)，另一种是 Object，而 Function 则属于 Object 的一种，注意，这里的 Function 用的是大写 F。

我们在代码中写下一个 function 时，在效果上，相当于创建了一个 Function 对象，例如：

```javascript
function hello(name) {    console.log('Hello, ' + name);}

hello('Mouhong');
```

上面的代码相当于：

```javascript
var hello = new Function('name', 'console.log("Hello, " + name)');hello('Mouhong');
```

两段代码不同的地方在于写法和性能（通过 function 定义的函数会有优化)，但它们都是在创建 Function 对象，换句话说，在 JavaScript 中，函数就是一种对象，这也能解释为什么我们可以把一个函数赋给一个变量：

```javascript
var func = function (name) {    console.log('Hello, ' + name);};func('Mouhong');```

上面我们把一个函数赋给了 func 变量，再次重申一下，我们这里定义了一个函数，但实际上是创建了一个 Function 对象，Function 对象和其它对象的不同之处在于，它可以被调用，所以它被称为 Callable Object。

回到前面我们”模拟“出来的 Animal 类：

```javascript
function Animal() {    this.name = null;
        this.sing = function () {        console.log('Singing ' + this.name);    }}
```

我们说 Animal 类中定义了一个 sing 方法，而实际上，JavaScript 并没有区分属性和方法，对 JavaScript 来说，Animal 中的 name 和 sing 都是属性，sing 属性的值是一个 Function 对象，仅此而已。

## `new`关键字 ##

使用 new 关键字来创建一个对象的时候，都发生了什么呢？继续拿 Animal 举例：

```javascript
function Animal() {    this.name = null;
        this.sing = function () {        console.log('Singing ' + this.name);    }}

var cat = new Animal();
```

1. 当 JavaScript 看到 new 关键字时，它先创建一个空对象;
2. 将新创建的对象绑定到 this，然后执行 Animal 函数;
3. Animal 函数体开始执行，在 this 上添加 name 和 sing ，注意此时 this 是新创建的对象，所以相当于在第1步中创建的对象上添加属性。同时将 this 对象的 prototype 设置为 Animal.prototype (不是把 prototype 设置成 Animal 对象);

可以用代码来模拟上面的过程：

```javascript
var cat = {};Animal.call(cat);cat.__proto__ = Animal.prototype;
```

不过要注意，设置新创建对象的 prototype 是很重要的，还记得我们的原型链吗？假设 Animal 的原型链上有其它属性，而在创建 cat 对象时没有把 cat 的 prototype 设置为 Animal 的 prototype，那 cat 就没办法把 Animal 的”基类属性”给继承下来。自动设置 prototype 的过程只有用 new 时才会发生，所以上面的模拟代码中，我们要手工设置 `__proto__`。

此时的原型链如下所示：

```
{ name, sing } --> { call, apply, ... } --> { toString, ... } --> null
```

后三项有点晕，是吗？还记得我们提到过，用 function 来定义一个函数，相当于是 new 出一个  Function 对象，所以我们把代码转换成下面的形式：

```javascript
var Animal = new Function('\                            this.name = "Kitty";\                            this.sing = function () {\                                console.log("Singing " + this.name);\                            }'                );var cat = new Animal();
```

然后再用 JavaScript 代码模拟出其执行流程：

```javascript
var Animal = {};
Function.call(Animal, '...'); // 这里忽略函数体字符串
Animal.__proto__ = Function.prototype;

var cat = {};Animal.call(cat);cat.__proto__ = Animal.prototype;
```

Function.prototype 中定义了 call, apply 等方法，而 Function.prototype 又是基于Object.prototype 构建出来的，而 Object.prototype 又包含了 toString 等，所以，原型链就变成了：

```
cat { name, sing } 
  --> { call, apply, ... } 
    --> { toString, ... } 
      --> null
```

另外，当我们知道 JavaScript 中没有所谓的“类”，连 function 都是一个对象时，也可以发现，通过设置一个对象的 prototype，可以非常方便的让一个对象从另一个对象上“继承”一些东西，比如：

```javascript
var Thing = {    die: function () {        console.log('I am dying...');    }};function Animal() {    this.name = null;    this.sing = function () {        console.log('Singing ' + this.name);    }}Animal.prototype = Thing;var cat = new Animal();cat.die(); // 输出 I am dying...```
从 C# 码农的背景来看，上面的 Thing 可以充当“静态类”的效果，但是既然我们在写 JavaScript，还是要从 JavaScript 的角度来看，不要硬往其它语言上面靠，所以 Thing 是一个对象，它有一个 die 属性，属性值是一个 Function 对象，而通过设置 `Animal.prototype = Thing`，Animal 就把 die 从 Thing 上继承下来了。

当我们了解清楚 JavaScript 的原型链，就会发现它有多么的灵活。JavaScript 并没有限制一定要用怎么实现继承，只要设置对原型链，就可以达到我们的效果。

比如，我们可以这样：

```javascript
function Animal() {    this.name = 'Kitty';    this.sing = function () {        console.log('Singing ' + this.name);    }}function Cat() {    Animal.call(this);    this.jump = function () { }}Cat.prototype = Object.create(Animal.prototype);var cat = new Animal();cat.sing(); // 输出 Singing Kitty
```

主要的不同点在于，Cat 函数的第一行，我们调用了 `Animal.call(this)`，回忆一下刚才 new 的原理，可以画出 `var cat = new Animal()` 的执行流程：

```javascript
var cat = {};
Cat.call(cat);
  - Animal.call(cat)
    - cat.name = 'Kitty'
    - cat.sing = function () { }
  - cat.jump = function () { }
  - cat.__proto__ = Cat.prototype
```

所以，最终 cat 对象就变成了这样:

```javascript
var cat = {
	name: 'Kitty',
	sing: function () { },
	jump: function () { },
	__proto__: { }
}
```

## 总结 ##

JavaScript 看起来像 Java，但和 Java 完全不相干（硬要扯上点关系的话，大概就是当年 JavaScript 想借 Java 之名火一把，人家本来叫 LiveScript)。

从 Java 的角度，可以认为 JavaScript 的 function 相当于是“类”，但其实更适合把 function 看成构造函数，事实上，它也被称为 Constructor Function。如果从 Java 的背景，也许不好理解一个构造函数如何能够脱离于类而存在，但函数在 JavaScript 中是一等公民，所以这也不奇怪。

从另一个角度，Java 中的类封装了函数和状态，而 JavaScript 通过闭包，也能把函数和状态封装在一起，不也就实现了 Java 中的类的效果。当然，闭包就是另一个话题了。

### 参考资料 ###

[1] [Details of the Object Model](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Details_of_the_Object_Model)