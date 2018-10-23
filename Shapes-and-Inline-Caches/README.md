> Article: JavaScript engine fundamentals: Shapes and Inline Caches

> Author: Mathias

> [Original Link](https://mathiasbynens.be/notes/shapes-ics)

这篇文章描述了一些对于所有Javascript引擎来说常见且非常重要的基础--而不是仅仅针对V8（作者[Benedikt Meurer](https://twitter.com/bmeurer)&[Mathias Bynens](https://twitter.com/mathias)开发的引擎），作为一个JS开发者，理解JS引擎如何工作的可以帮助你优化你的代码。

### Javascript的管道

全部都开始于你所写出的JS代码。JS引擎解析源代码，然后将它转化为抽象语法树（AST）。基于AST解释器开始生成字节码，非常好！到这个时候，引擎开始真正的运行JS代码。

![1]()

为了让它运行的更快，字节码将会与分析数据一同被输入到优化编译器中。优化编译器根据这些信息作出一定的假设，之后生成高度优化的机器码。

如果在某个时刻其中一个假设结果为不正确，那么优化编译器将会进行逆优化，然后回到解释器中。

### 解释器/编译器在JS引擎中的管道

现在，让我们开始看真正运行你JS代码的管道部分，即代码被解释和优化的地方，之后复习一下一些主要JS引擎的区别。

通常来说，有一个包含一个解释器和一个优化编译器的管道。解释器快速的生成没有被优化的字节码，优化编译器花稍微长一点的时间并最后生成高度优化的机器码。

![2]()

下面的管道几乎就是V8(Chrome和Node.js所使用的引擎)的工作原理：

![3]()

V8中的解释器被称为`点火器-Ignition`，它的职责是生成和运行字节码。在运行的同时收集分析数据，这些数据将会加速之后的运行过程。当一个函数变`hot`，比如说执行更加频繁，生成的字节码和分析数据将会被传到`TurboFan`中，我们的优化编译器，去根据收集来的分析数据来生成优化的机器码。

![4]()

`SpiderMonkey`，Mozilla用在FireFox和SpiderNode中的JS引擎。其工作原理有一点不同于V8，它有两个优化编译器。


##TO BE CONTINUED                                                                                                                          