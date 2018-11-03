> Article: An internship on laziness: lazy unlinking of deoptimized functions

> Author: Juliana Franco

> [Original Link](https://v8.dev/blog/lazy-unlinking)


大约在三个月前，我作为一个实习生加入到了V8(Google Munich)，自从那时候开始我就开始解决`VM's Deoptimizer`的问题，它对于我来说是全新的，也同时是一个有趣和挑战性的项目。我实习的第一部分专注于[提升虚拟机的安全](https://docs.google.com/document/d/1ELgd71B6iBaU6UmZ_lvwxf_OrYYnv0e4nuzZpK05-pg/edit)。第二个部分专注于性能的提升。换句话说，是去除先前逆优化功能的数据结构，这是在垃圾回收过程中的性能瓶颈。这篇博客主要内容就是我实习的第二部分工作内容。我会解释V8之前是如何取消对逆优化方法的连接的，是如何改变这一点的，以及性能得到了多少提高。

让我们简短的回顾一下V8的整个流程：V8的解释器，`Ignition`，在解释过程中收集函数的信息。一旦函数变`热`，这些信息将会被传到V8的编译器中，即`TurboFan`，它会生成优化后的机器码。当这些剖析数据不再有效时--比如说因为一个对象在运行时变为另一种类型(译：`Shape`发生改变)--优化过的机器码将会变为无效。在这种情况下，V8就需要逆优化。

![1.An overview of V8](https://github.com/RogerZZZZZ/V8-blog/tree/master/Lazy-unlinking-of-deoptimized-functions/img/1.png)

在优化时，`TurboFan`会为优化下的函数生成代码对象，即优化后的机器码。当这个函数被再次调用时，V8会寻找指向该函数的优化代码并执行它。在函数的逆优化过程中，我们需要解开之前与优化代码的链接，以避免下次再次执行，那这是怎么发生的呢？

举个例子，在下面的代码中，函数`f1`将会被调用很多次（总是传入整数作为参数）。`TurboFan`之后会为这个特殊情况生成机器码。

```javascript
function g() {
  return (i) => i;
}

// 创建闭包
const f1 = g();
// 优化 f1
for (var i = 0; i < 1000; i++) f1(0);
```

每个函数都需要有一个`蹦床`给解释器--更多的细节在这些[slides](https://docs.google.com/presentation/d/1Z6oCocRASCfTqGq1GCo1jbULDGS-w-nzxkbVF7Up0u0/edit#slide=id.p)中--每个函数同样也会在`SharedFunctionInfo(SFI)`中保存一个指向`蹦床`的指针。每当V8需要回到未优化的代码时，`蹦床`就会被使用。因此，在逆优化过程会被传入不同类型的参数而被触发，比如，逆优化器可以简单地将Javascript函数的字段设置为`蹦床`。

![2.An overview of V8](https://github.com/RogerZZZZZ/V8-blog/tree/master/Lazy-unlinking-of-deoptimized-functions/img/1.png)

尽管看起来很简单，它也让V8去维护一个优化过得HS方法的列表。这是因为有可能不同的函数指向同一个优化代码对象。我们可以将我们的例子像下面一样扩展一下，这时函数`f1`和`f2`就都指向了同一个优化代码。

```javascript
const f2 = g();
f2(0);
```

如果函数`f1`被逆优化了（比如调用时传入不同类型的对象`{x: 0}`），我们需要保证当`f2`被调用时这个无效的优化代码不会再次执行。

所以，在逆优化过程中，V8之前会遍历所有的优化函数，然后删除那些指向已经被逆优化代码对象的链接。这在有很多优化函数的应用中的遍历就成为了性能的瓶颈，此外，除了减慢逆优化过程，V8曾经遍历垃圾回收中的`stop-the-world cycles`中的列表，使其变得更糟。

为了了解这种数据结构对V8的性能的影响，我们编写了一个[`小型检查程序（micro-benchmark）`](https://github.com/v8/v8/blob/master/test/js-perf-test/ManyClosures/create-many-closures.js)，通过创建很多JS函数之后触发很多清楚循环的操作来强调它的作用。

```javascript
function g() {
  return (i) => i + 1;
}

// 创建一个初始的闭包以及优化
var f = g();

f(0);
f(0);
%OptimizeFunctionOnNextCall(f);
f(0);

// 创建2M个闭包：这将会得到之前的优化代码。
var a = [];
for (var i = 0; i < 2000000; i++) {
  var h = g();
  h();
  a.push(h);
}

// 现在触发清扫： 所有都会很慢
for (var i = 0; i < 1000; i++) {
  new Array(50000);
}
```

当我们在运行这个检查程序时，我们观察到V8花费了大约98%的执行时间来进行垃圾回收。我们之后去除了数据结构，转而使用了一个延迟取消链接的方法，这就是我们在x64上观察到的：

![3](https://github.com/RogerZZZZZ/V8-blog/tree/master/Lazy-unlinking-of-deoptimized-functions/img/2.png)

尽管这只是一个生成很多触发垃圾回收的JS函数的小型检查程序，它让我们了解到了这个数据结构所引入的开销。在[router benchmard](https://github.com/delvedor/router-benchmark)这样更实际的应用中我们看到了许多开销，这也激励我们完成这个工作。

### 延迟取消链接

不是在逆优化过程中将优化代码与JS函数进行解绑，V8会推迟这个操作到下一次这样函数的执行时完成。当这样的函数被调用时，V8会检查他们是否被逆优化，如果是则进行解绑然后继续它们延迟编译过程。如果这些函数不会再次被调用，那么它们将永远不会被解绑，逆优化的代码对象也不会被回收。然而，鉴于在逆优化期间，我们使得代码对象所有的嵌入字段无效，我们只保持该代码对象存活。

这个[commit](https://github.com/v8/v8/commit/f0acede9bb05155c25ee87e81b4b587e8a76f690)删除了虚拟机很多部分需要的优化函数列表，基本的想法如下：当我们组装优化代码对象时，我们检查这是否是JS函数的代码，如果是，在它的序言中，如果代码已经被逆优化，我们就会组装机器码来挽救。在逆优化过程中我们不回去修改逆优化的代码--代码补丁将不会存在。因此当再次调用该函数时，仍然去设置`marked_for_deoptimization`位。`TurboFan`会生成代码去检查它，如果设置了，那么V8将会跳转到一个新的名为`CompileLazyDeoptimizedCode`的`builtin`中，它将函数和逆优化代码进行解绑，然后继续它们的延迟编译。

更详细的说，第一步是去生成加载目前被组装的代码地址的指令。我们可以通过下面的代码在x64上实现这样的操作：

```
Label current;
// 加载目前指令的高效地址到rcx中
__ leaq(rcx, Operand(&current));
__ bind(&current);
```

之后我们需要得到`marked_for_deoptimization`位在代码对象中存放的位置。

```c++
int pc = __ pc_offset();
int offset = Code::kKindSpecificFlags1Offset - (Code::kHeaderSize + pc);
```

我们之后检查这个位的值，如果已经被设置，我们将会进入`CompileLazyDeoptimizedCode builtin`

```
// 检查这个位有没有被设置过，如果是，代码已经被标记为逆优化
__ testl(Operand(rcx, offset),
         Immediate(1 << Code::kMarkedForDeoptimizationBit));
// 如果被设置过，跳转到内置中
__ j(not_zero, /* handle to builtin code here */, RelocInfo::CODE_TARGET);
```

在`CompileLazyDeoptimizedCode builtin`这一侧，剩下需要做的是取消代码字段和函数的链接，并将其设置为`蹦床`以等到解释器的进入，因此，考虑到函数的地址已经在寄存器`rdi`中国，我们可以通过下面的指令来获得指向`SharedFunctionInfo`的指针：

```
// 获取ShredFunctionInfo
__ movq(rcx, FieldOperand(rdi, JSFunction::kSharedFunctionInfoOffset));
```

相似的，用下面代码得到`蹦床`：

```
// 获取代码对象
__ movq(rcx, FieldOperand(rcx, SharedFunctionInfo::kCodeOffset));
```

之后我们可以使用它来更新代码指针的函数插槽：

```
// 使用‘蹦床’来更新函数的代码字段
__ movq(FieldOperand(rdi, JSFunction::kCodeOffset), rcx);
// Write barrier to protect the field.
// 写下障碍来保护域
__ RecordWriteField(rdi, JSFunction::kCodeOffset, rcx, r15,
                    kDontSaveFPRegs, OMIT_REMEMBERED_SET, OMIT_SMI_CHECK);
```

这将会产生与之前一样的结果。但是，我们需要在代码生成过程中关注它，而不是在取消与逆优化代码链接的过程中。

上述就是这个过程是如何在`x64`架构中运行的，我们也同样需要在ia32, arm, arm64, mips, 和mips64中实现。

这个新的技术现在已经被整合到了V8中，正如我们稍后讨论的那样，它可以提高性能。然而，它也带来了很小的缺点：之前V8只会在逆优化过程中考虑解开链接的事情，现在它需要在所有优化函数被执行的时候都做一遍这样的考虑。此外，检查`marked_for_deoptimization`位的操作并没有非常的高效，为了得到结果我们需要去获取代码对象的地址。请注意，在进入每个优化函数时都会发生这样的情况。一个可能的解决方案是，在代码对象中维护一个指向自己的指针。在函数每次被调用的时候，V8不会去做查找代码对象地址的操作，而是在构造之后只执行一次。

### 结果

我们来看这个项目获得的性能提升和回归。

**x64上的提升**

下面的图展示了提升和回归，相比于之前的提交来说，注意越高代表越好

![4](https://github.com/RogerZZZZZ/V8-blog/tree/master/Lazy-unlinking-of-deoptimized-functions/img/3.png)

## TO BE CONTINUED