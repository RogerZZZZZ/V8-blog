> Article: What's up with monomorphism?

> Author: Vyacheslav Egorov

> [Original Link](https://mrale.ph/blog/2015/01/11/whats-up-with-monomorphism.html)

有关JS性能的博客和演讲都经常强调单态的重要性。然而他们一般都没有对单态/多态是什么给出易理解的解释。不出所料的是我得到关于性能方面最常见的问答请求之一就是解释单态究竟是什么，多态是如何产生的，它为什么不好。

无济于事的是"多态"这个词本身就被过度重载(overloaded)。在经典面向对象编程的概念中，多态往往意味着类型多态以及重写基类行为的能力。Haskell程序员则会认为是参数多态(`parametric polymorphism`). 然而多态在JS性能中的意义则不同，它被称为`site polymorphism`.

在我决定写一篇关于它的博客之前，我已经通过很多方法来解释这个概念吗，所以下次我可以直接给出链接而不是现场再临时解释了。

## 动态查找101

![simple implementation](https://github.com/RogerZZZZZ/V8-blog/blob/master/translation/What's-up-with-monomorphism/img/1.png)

为了简单起见，这个内容将会以JS中最简单的属性访问为例，比如下面的代码`o.x`。同时这是十分重要的去理解关于任何`动态范围(dynamically bound)`中的属性查找操作或是算式的计算。

```javascript
function f(o) {
  return o.x
}

f({ x: 1 })
f({ x: 2 })
```

想象你正在面试一家公司中的一个很好的职位，你的面试官问你JS虚拟机中属性查找是如何设计和实现的，如何才能以最简单和直接的方式回答这个问题。

显然你无法找到任何比给出`ECMAScript Language Specification (aka ECMA 262)`中描述的JS语法更简单的解释。之后就可以将它用C++, Java, Rust, 或者Malbolge来重新抄写出来。

实际上，如果你随机打开一个JS解释器，你就会发现如下的代码：

```javascript
jsvalue Get(jsvalue self, jsvalue property_name) {
  // 8.12.3 [[Get]] implementation goes here
}

void Interpret(jsbytecodes bc) {
  // ...
  while (/* has more bytecodes */) {
    switch (op) {
      // ...
      case OP_GETPROP: {
        jsvalue property_name = pop();
        jsvalue receiver = pop();
        push(Get(receiver, property_name));
        // TODO(mraleph): throw exception in strict mode per 8.7.1 step 3.
        break;
      }
      // ...
    }
  }
}
```

这显然是属性查找的一种可行性实现，但是它有一个明显的问题:如果我们将这个实现与现代JS虚拟机的内部实现相比的话，我们就会发现它实在太慢了。

我们的解释器是健忘的: 每次我们需要一个属性的值我们就要执行一次属性查找的算法，它并没有从之前的尝试中学习到任何东西，而是每次付出的代价都相同。这就是为什么面向性能的虚拟机中属性查找的实现是不同的。

![real implementation](https://github.com/RogerZZZZZ/V8-blog/blob/master/translation/What's-up-with-monomorphism/img/2.png)

要是程序中每次属性的访问都有能力从之前见过的对象中学习，并且将这个知识运用到相似的新对象上会怎么样呢？可能这会让我们省去很多执行一般查找算法的时间，而去在一些特定形式的对象上使用更快的方法。

我们知道在任意对象中寻找一个属性是十分昂贵的，所以我们想要只寻找一次，之后在缓存中用`对象形状（Object's shape）`作为key存入这个属性的路径。下一次我们看到一个有相同形状的对象时我们可以直接从缓存中读取路径，而不是重新再计算一次。

这种优化技术被称为`内敛缓存(inline caching)`，我只有的文章有写过[link](https://mrale.ph/blog/2012/06/03/explaining-js-vms-in-js-inline-caches.html)。在这篇文章中我将会不管其中细节的实现，而是更加专注于之前被我忽视的一个概念。每个内联缓存首先它就是一个缓存，与其他缓存一样，它有大小(当前被缓存的条目数量)以及容量`capacity`(最大可缓存的条目数量)。

让我们再看看上面的例子:
```javascript
function f(o) {
  return o.x
}

f({ x: 1 })
f({ x: 2 })
```

对于`o.x`我们期望的内联缓存条目数量是多少呢？`{x: 1} & {x: 2}`有相同的形状(也被称为 `_hidden_class 或者是 _map_`)，答案是`1`。恰恰这时缓存的状态我们可以称为单态，因为我们只看到了一种形状的对象。

那如果我们输入不同形状的对象并且执行函数f会怎么样呢？

`{x: 3} & {x: 3, y: 1}`是不同形状的对象，所以缓存就不再是单态的了，现在它包含两条缓存记录，一个为`{x: *}`，另一个为`{x: *, y: *}`，现在我们的操作就是在一个维度为2的多态状态下运行。

如果我们继续调用函数f，并且输入不同形状的对象，多态的维度会继续增加直到它达到一个预设的阈值 - 最大值也许就是内敛缓存的存储能力大小。到那个零界点，缓存将会转化为`megamorphic`状态。

```javascript
f({ x: 4, y: 1 }) // polymorphic, degree 2
f({ x: 5, z: 1 }) // polymorphic, degree 3
f({ x: 6, a: 1 }) // polymorphic, degree 4
f({ x: 7, b: 1 }) // megamorphic
```

![monomorphic state](https://github.com/RogerZZZZZ/V8-blog/blob/master/translation/What's-up-with-monomorphism/img/3.png)

`megamorphic`状态的存在是为了防止多态缓存不受控制的增长，意味着“我看到太多不同形状的对象了，我放弃追踪他们了”。在V8中`megamorphic ICs`也会继续进行缓存，但是将会将想要缓存的东西放入全局的哈希表中，而不是在本地进行操作。这张哈希表有固定的大小，如果出现碰撞，将会简单的进行覆盖。

![megamorphic ICs](https://github.com/RogerZZZZZ/V8-blog/blob/master/translation/What's-up-with-monomorphism/img/4.png)

现在我们用个小的练习来看看大家的理解:
```javascript
function ff(b, o) {
  if (b) {
    return o.x
  } else {
    return o.x
  }
}

ff(true, { x: 1 })
ff(false, { x: 2, y: 0 })
ff(true, { x: 1 })
ff(false, { x: 2, y: 0 })
```

- 有多少内联缓存的属性访问在函数ff中？
- 他们的状态是怎么样的？

> 答案是，有两个缓存，都是单态的，因为斗志看到一种形状的对象。

### 性能影响

现在不同内联缓存的状态的性能特征应该很明了了：

- 单态是最快的内联缓存状态，如果你一直能命中缓存；
- 多态性的缓存在缓存记录中执行线性搜索；
- 在`megamorphic`状态下的缓存将会在全局哈希表中进行搜索，所以是所有内联缓存中最慢的，但是也要好过没有命中到缓存的情况；
- 没有命中到内联缓存的情况是很昂贵的，你需要首先付出转化的运行时间加上常规操作的时间。

然而这仅仅只是一半的原理(`truth`)。除了加速你的代码，内联缓存同样也作为间谍来优化编译器 - 最终再次更加深度地加速你的代码。

### 推测和优化

内联缓存不能带来巅峰性能的原因有两个：

- 每个IC都独立运行，对其他IC一无所知
- 每个IC当它不能处理输入时最终都会退回到运行时。意思为IC的本质是一个带有副作用的通用操作，而且经常不知道结果的类型。

```javascript
function g(o) {
  return o.x * o.x + o.y * o.y
}

g({x: 0, y: 0})
```

以上述函数为例，每个IC（含有7个：`.x, .x, *, .y, .y, *, +`）都独自运行。每个属性在IC中以与自己形状相同的缓存在`o`中进行检查。算术IC`+`将会检查它的输入是否是数字(以及是什么类型的数字 - 在V8中有把不同的数字表示(如：`SignedSmall`, `Number`, `NumberOrOddball`等)) - 即使这个结果可以由`* ICs`推导出。

JS中的算术操作是`inherently typed`,比如 `a | 0`永远返回32位整数，`+a`永远放回数字。但是大多数其他的操作无法有这样的保证。这就让在JS中写一个提前作用的优化编译器称为一个难题。对比与将JS只进行一次预编译`，大多JS虚拟机分为多个执行层。比如V8中，代码第一次执行时不会有任何优化，被一个无优化的基础编译器编译。被调用频率高的函数将会被优化编译器重编译。

等待代码加载到虚拟机(`warm up`)主要有两个原因

> 译者: Once class-loading is complete, all important classes (used at the time of process start) are pushed into the JVM cache (native code) – which makes them accessible faster during runtime. Other classes are loaded on a per-request basis. ---- warm up in JVM

- 减少启动延迟：优化编译器慢于无优化的编译器。意味着被优化的代码要被使用一定次数才有优化的意义
- 它给了内敛缓存收集类型反馈的机会

被书写JS的人们强调的一点是，JS没有包含足够的内部类型信息(`inherent type information`)来支持全部静态类型以及预编译。`JIT`不得不进行预测: JIT必须对它优化的代码中的用法和行为做出有根据的推测，然后生成在特定假设下个性化的代码。 换句话说编译器需要假设在需要优化的函数中会见到什么类型的对象。 幸运的是这恰好就是内联缓存收集的信息。

- 单态缓存说“我只见过类型A”
- N维的多态缓存说”我只见过A1....An“
- Megamorphic 缓存说”我见过很多东西“

![type guard](https://github.com/RogerZZZZZ/V8-blog/blob/master/translation/What's-up-with-monomorphism/img/5.png)

优化编译器查看IC收集来的信息，然后生成对应的`intermediate representation(IR)`。 IR指令通常比通常的JS操作更加底层以及明确。比如`.x` IC只看到`{x, y}`形状的对象，之后优化器会使用IR指令去对象中特定偏移量的地方读取属性，再用它加载`.x`。当然在任意对象上使用这样的指令是不安全的，所以优化器预先设置了一个类型哨兵(`type guard`)。类型哨兵在对象到达特定操作前检查他的形状(`shape of object`)，如果没有得到预期的对象，将不会在进行下去，而是会以未优化的代码继续运行。这个过程被称为逆优化(`deoptimization`)。逆优化的发生不仅仅只由类型哨兵产生，比如：算术操作只用于32位整数，如果计算结果溢出将会执行逆优化，`arr[idx]`中的idx需要在其长度范围内，否则会因为idx越界，`arr[idx]`不存在导致逆优化的发生。

![deoptimization](https://github.com/RogerZZZZZ/V8-blog/blob/master/translation/What's-up-with-monomorphism/img/6.png)

现在可以列出两条优化过程的缺点：

未优化 | 优化
--- | ---
每个操作会有任意不可知的副作用，因为它是通用的，并且实现所有语义 | 代码的特殊化或是消除不可预见性，副作用被很好的定义(通过偏移量读取属性没有副作用)
每个操作都是独立的，没有信息的交流 | 操作被分解为更底层的IR指令，之后都将会被一起优化，这就给发现和消除冗余提供了机会

实际上根据类型反馈构建明确的IR只是优化流程中的第一步。一旦IR准备就绪，编译器就会运行多次来寻找不变量和消除冗余。 在这一部运行过程的分析是过程中的(`intraprocedural`)，编译器也被强制在每次执行中假设最差的任意副作用。我们需要知道的是通常非个性化的操作本质都是调用自己。比如 `+` 求值就是调用`valueOf`以及调用getter方法来获得`o.x`属性访问。这就意味着那些由于某些原因特化失败的优化器将会阻塞接下来的优化进程。

![eliminate redundancies](https://github.com/RogerZZZZZ/V8-blog/blob/master/translation/What's-up-with-monomorphism/img/7.png)

一个常见的例子为当在相同形状的对象中取取相同值时类型哨兵冗余。下面是最初函数g的IR的样子：
```
    CheckMap v0, {x,y}  ;; shape check 
v1  Load v0, @12        ;; load o.x @12 is offset
    CheckMap v0, {x,y} 
v2  Load v0, @12        ;; load o.x 
i3  Mul v1, v2          ;; o.x * o.x 
    CheckMap v0, {x,y} 
v4  Load v0, @16        ;; load o.y 
    CheckMap v0, {x,y} 
v5  Load v0, @16        ;; load o.y 
i6  Mul v4, v5          ;; o.y * o.y 
i7  Add i3, i6          ;; o.x * o.x + o.y * o.y 
```

这个IR对`v0`在相同形状下检查了四遍，即使之后检查不会影响到`v0`的形状。认真的读者会注意到读取`v2 & v5`也是冗余的
，因为没有对相关属性有任何写入。幸运的是[GVN](https://en.wikipedia.org/wiki/Value_numbering)将会作用于IR，然后消除其中的冗余，下面为结果:

```
    ;; After GVN 
    CheckMap v0, {x,y} 
v1  Load v0, @12 
i3  Mul v1, v1 
v4  Load v0, @16 
i6  Mul v4, v4 
i7  Add i3, i6 
```

然而上述提到的消除冗余只可能在冗余操作之间没有干扰项时才可能。当有个调用出现在`v1, v2`之间时，我们就需要非常谨慎的假设被调用者有权限访问`v0`，并且可以执行add、remove和更改属性的操作，这就让消除`v2`或是`CheckMap`的访问变为不可能。

现在我们队优化编译器有了基本的理解，知道了它喜欢什么(个性化的操作指令)，不喜欢什么(调用以及一般操作指令)，现在剩下一个需要讨论的东西: 优化编译器对非单态操作的处理。

如果操作不是单态的，显然优化编译器就不会使用我们之前讨论过得简单个性化准则`type-guard + specialized-op`。它也同时不能为类型哨兵选择一个单独的类型，以及选择一个单独个性化操作。内联缓存告诉编译器，这个操作会见到不同类型的值，所以选择其中一个而忽视其他会带来有风险的逆优化，这是非常不可取的。取而代之的是优化编译器会尝试构建一个决策树。比如说一个看到了A,B,C三种形状的多态属性访问`o.x`将会被如下展开(下面为伪代码 - 优化编译器将会构建一个CFG`Control flow graph, 控制流图`):

```javascript
var o_x
if ($GetShape(o) === A) {
  o_x = $LoadByOffset(o, offset_A_x)
} else if ($GetShape(o) === B) {
  o_x = $LoadByOffset(o, offset_B_x)
} else if ($GetShape(o) === C) {
  o_x = $LoadByOffset(o, offset_C_x)
} else {
  // o.x saw only A, B, C so we assume
  // there can be *nothing* else
  $Deoptimize()
}
// Note: at this point we can only say that
// o is either A, B or C. But we lost information
// which one.
```

有一件事需要注意的是多态访问相比单态访问来说缺少了有用的属性。个性化的访问在遇到干扰项之前我们都可以保证这个对象只有特定的一种形状。这就允许我们在单态访问之间去除冗余。多态访问则只能保证对象的形状有可能是A,B,C中的一个，因此我们不能使用这个信息来消除相似多态访问中的冗余，最多我们只可以消除最后的条件比较以及逆优化模块，但是V8并不会这么做。

然而当属性位于任意形状对象的相同位置时，V8会构建了一个更加有效率的IR。下面的例子就是使用一个多态类型哨兵而不是决策树。

```javascript
// Check that o's shape is one of A, B or C - deoptimize otherwise.
$TypeGuard(o, [A, B, C])
// Load property. It's in the same place in A, B and C.
var o_x = $LoadByOffset(o, offset_x)
```

这个IR对于冗余消除有一个重要的好处就是，如果在两个`$TypeGuard(o, [A, B, C])`指令中间没有出现干扰项，那么第二个指令将会成为冗余项，和单态的例子相同。

如果类型反馈告诉优化编译器，属性访问看到了不同的形状，接下来优化编译器将会考虑在行内处理，之后优化器将会建一个以普通操作结尾的稍微不同的决策树，而不是执行逆优化过程。

```javascript
var o_x
if ($GetShape(o) === A) {
  o_x = $LoadByOffset(o, offset_A_x)
} else if ($GetShape(o) === B) {
  o_x = $LoadByOffset(o, offset_B_x)
} else if ($GetShape(o) === C) {
  o_x = $LoadByOffset(o, offset_C_x)
} else {
  // We know that o.x is too polymorphic (megamorphic).
  // to avoid deoptimizations leave escape hatch to handle
  // arbitrary object:
  o_x = $LoadPropertyGeneric(o, 'x')
  //    ^^^^^^^^^^^^^^^^^^^^ arbitrary side effects
}
// Note: at this point nothing is known about
// o's shape and furthermore arbitrary
// side-effects could have happened.
```

最后有几种情况，优化编译器会完全放弃个性化操作：

- 如果它不知道如何高效地进行优化，个性化
- 操作是多态的，并且优化器不知道如何根据这个操作正确地构建一个决策树。（比如，V8之前有过的一个例子，`arr[i]`中的多态键访问，现在已经不存在）
- 操作没有任何类型反馈用于个性化（操作永远不会执行，垃圾回收清空了类型反馈等）

在上面的情况下（很小概率），优化器就会输出带有一个普通的变量的IR。

# To be continued

