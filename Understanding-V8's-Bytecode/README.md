> Article: Understanding V8’s Bytecode

> Author: Franziska Hinkelmann

> [Original Link](https://medium.com/dailyjs/understanding-v8s-bytecode-317d46c94775)



V8是Google开源JS引擎。Chrome, Node.js以及很多应用都使用V8，这篇文章将解释什么是V8的字节码格式--一旦你了解一些基础概念后这篇文章将会很容易阅读。

![1](https://github.com/RogerZZZZZ/V8-blog/raw/master/Understanding-V8's-Bytecode/img/1.png)

当V8编译JS代码时，解析器会生成抽象语法生成树，语法树是JS代码的语法结构表示。`Ignition`--解释器，从语法生成树中生成字节码。`TurboFan`--优化编译器，最终会用字节码生成优化过的机器码。

![2](https://github.com/RogerZZZZZ/V8-blog/raw/master/Understanding-V8's-Bytecode/img/2.png)

如果你想知道为什么我们有两个执行模式，你可以查看我在`JSConfEU`上的视频： [link](https://www.youtube.com/watch?v=p-iiEDtpy6I)

字节码是机器码的抽象。如果字节码被设计为与物理CPU相同的计算模型的话，将字节码编译为机器码会更加的简单。这就是为什么解释器通常为寄存器或是堆栈。**Ignition是一个具有累加器的寄存器**。

![3](https://github.com/RogerZZZZZ/V8-blog/raw/master/Understanding-V8's-Bytecode/img/3.png)

你可以将V8的字节码理解为一个小的构建模块，它们可以组合起来建造任何的JS功能。V8有几百种字节码，它们有类似`Add`或者`TypeOf`的操作符，或是用于属性加载的`LdaNamedProperty`。V8也有一些类似`CreateObjectLiteral`或是`SuspendGenerator`这样非常特别的字节码。头文件[bytecodes.h](https://github.com/v8/v8/blob/master/src/interpreter/bytecodes.h)定义了V8中所有的字节码。

每个字节码将它的输入和输出指定为寄存器操作数。`Ignition`使用寄存器`r0, r1, r2, ...`以及一个累加器，几乎所有的字节码都使用累加器。它类似一个常规的寄存器，除了一点是字节码不会指定它。例如： `Add r1`将`r1`寄存器中的值加到累加器中的值中。这使得字节码更短小，也更节省内存。

许多字节码以`Lda`和`Sta`开始。`Lda`和`Sta`中的`a`代表累加器。例如：`LdaSmi[42]`将小整数(Smi)`42`加载到累加器中。`Star r0`将目前累加器中的值存储到寄存器`r0`中。

到目前为止都是基础知识，现在是时候看一下实际函数中的字节码了。

```javascript
function incrementX(obj) {
  return 1 + obj.x;
}
incrementX({x: 42});  // V8的编译器是惰性的，如果你不执行这个函数，它将不会被解释。
```

> 如果你想看到JS代码的对应V8字节码，你可以通过D8或者是在Node.js(8.3或更高的版本)中带上`--print-bytecode`来执行代码来打印出对应的字节码。在Chrome中，在打开Chrome的命令后跟上`--js-flags="--print-bytecode"`

```shell
$ node --print-bytecode incrementX.js
...
[generating bytecode for function: incrementX]
Parameter count 2
Frame size 8
  12 E> 0x2ddf8802cf6e @    StackCheck
  19 S> 0x2ddf8802cf6f @    LdaSmi [1]
        0x2ddf8802cf71 @    Star r0
  34 E> 0x2ddf8802cf73 @    LdaNamedProperty a0, [0], [4]
  28 E> 0x2ddf8802cf77 @    Add r0, [6]
  36 S> 0x2ddf8802cf7a @    Return
Constant pool (size = 1)
0x2ddf8802cf21: [FixedArray] in OldSpace
 - map = 0x2ddfb2d02309 <Map(HOLEY_ELEMENTS)>
 - length: 1
           0: 0x2ddf8db91611 <String[1]: x>
Handler Table (size = 16)
```

我们可以忽略大多数输出，而注意在实质的字节码上。下面是每一行字节码的意思：

#### LdaSmi[1]

`LdaSmi[1]`将常量`1`放入累加器中.

![4](https://github.com/RogerZZZZZ/V8-blog/raw/master/Understanding-V8's-Bytecode/img/4.png)


#### Star r0

接下来，`Star r0`存储了目前在累加器中的值，即`1`，到寄存器`r0`中。

![5](https://github.com/RogerZZZZZ/V8-blog/raw/master/Understanding-V8's-Bytecode/img/5.png)


#### LdaNamedProperty a0, [0], [4]

`LdaNamedProperty`加载一个命名属性`a0`到累加器中，`ai`代表了函数`incrementX()`的第i个参数。在这个例子中，我们寻找一个命名属性`a0`，即`incrementX()`的第一个参数。这个名字由常量`0`决定。`LdaNamedProperty`使用`0`在分散的table中寻找名称：

```shell
- length: 1
           0: 0x2ddf8db91611 <String[1]: x>
```

在这里，`0`匹配`x`，所以字节码加载`obj.x`

> 译者：这一步操作应该是由内联缓存提前做出的优化，`0`此处应该就为offset。

那值为`4`的操作符又有什么用呢？这就是函数`incrementX()`的所谓反馈向量的索引(index of feedback vector)。反馈向量包含了用于性能优化的运行时信息。

现在寄存器的情况如下：

![6](https://github.com/RogerZZZZZ/V8-blog/raw/master/Understanding-V8's-Bytecode/img/6.png)


#### Add r0, [6]

最后一个指令就是将`r0`的值加到累加器中，结果为`43`，`6`是另一个反馈向量的索引。

![7](https://github.com/RogerZZZZZ/V8-blog/raw/master/Understanding-V8's-Bytecode/img/7.png)


#### Return

`Return`返回累加器中的值。这也是函数`incrementX()`的结束。此时`incrementX()`的调用者可以从累加器中得到值`43`，并可以将这个值用在之后的步骤中。


当刚开始接触V8的字节码时，它十分的神秘，尤其是带着额外信息一同被打印出来时。但是一旦你知道了`Ignition`是一个带有累加器的寄存器时，你就可以知道字节码的作用了。


## DONE