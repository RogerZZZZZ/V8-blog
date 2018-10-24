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

`SpiderMonkey`，Mozilla用在FireFox和SpiderNode中的JS引擎。其工作原理有一点不同于V8，它有两个优化编译器。解释器优化后到`Baseline`编译器中生成稍微优化后的代码。`IonMonkey`编译器则在使用代码运行时收集的分析数据来生成深度优化的代码。如果推测优化失败了，`IonMonkey`会回到`Baseline`阶段。

![5]()

`Chakra`，微软的在`Edge & Node-ChakraCore`中使用的JS引擎，也有相似的两个优化编译器作为开始。解释器优化后进入`SimpleJIT` - `JIT`意思为`Just-In-Time compile(及时编译器)`，在这里生成轻度优化的代码。再运用上分析数据，`FullJIT`会生成深度优化代码。

![6]()

`JavaScriptCore`（简称JSC），苹果在Safari和React Native中使用的JS引擎，使用了三个不同的优化编译器。`LLInt`，低级的解释器，之后到达`Baseline`编译器，再之后进入`DFG`(数据流图`Data Flow Graph`)编译器，最后进入FTL(Faster Than Light)编译器。

为什么有一些引擎会比其他引擎有更多的优化编译器？这都是一些权衡和取舍。一个解释器可以快速的生成字节码，但是通常来说字节码不是那么地高效。在另一方面一个优化编译器花更长的时间来最终生成更加有效率的机器码。这就是快速得到可以运行的代码或者是牺牲一些时间来让代码运行有更好的性能间的抉择。一些引擎选择加入一些有不同时间/效率特性的优化编译器，允许对这些权衡进行更细粒度的控制，但是代价是额外的复杂性。另一个权衡是内存的使用，可以在[我们后续的文章](https://mathiasbynens.be/notes/prototypes#tradeoffs)中寻找更多的观点。

我们只是提到了每个JS引擎中解释器和优化编译器管道的主要区别。但是除了这些区别，在更高的层面，所有的JS引擎都是相同架构的：都有一个分析器和一些解释器/编译器管道。

##TO BE CONTINUED                                                                                                                          