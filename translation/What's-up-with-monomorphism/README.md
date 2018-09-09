> Article: What's up with monomorphism?

> Author: Vyacheslav Egorov

> [Original Link](https://mrale.ph/blog/2015/01/11/whats-up-with-monomorphism.html)

有关JS性能的博客和演讲都经常强调单态的重要性。然而他们一般都没有对单态/多态是什么给出易理解的解释。不出所料的是我得到关于性能方面最常见的问答请求之一就是解释单态究竟是什么，多态是如何产生的，它为什么不好。

无济于事的是"多态"这个词本身就被过度重载(overloaded)。在经典面向对象编程的概念中，多态往往意味着类型多态以及重写基类行为的能力。Haskell程序员则会认为是参数多态(`parametric polymorphism`). 然而多态在JS性能中的意义则不同，它被称为`site polymorphism`.

在我决定写一篇关于它的博客之前，我已经通过很多方法来解释这个概念吗，所以下次我可以直接给出链接而不是现场再临时解释了。

## 动态查找101

![simple implementation](https://github.com/RogerZZZZZ/V8-blog/tree/master/translation/What's-up-with-monomorphism/img/1.png)

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

![real implementation](https://github.com/RogerZZZZZ/V8-blog/tree/master/translation/What's-up-with-monomorphism/img/2.png)

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

![monomorphic state](https://github.com/RogerZZZZZ/V8-blog/tree/master/translation/What's-up-with-monomorphism/img/3.png)

`megamorphic`状态的存在是为了防止多态缓存不受控制的增长，意味着“我看到太多不同形状的对象了，我放弃追踪他们了”。在V8中`megamorphic ICs`也会继续进行缓存，但是将会将想要缓存的东西放入全局的哈希表中，而不是在本地进行操作。这张哈希表有固定的大小，如果出现碰撞，将会简单的进行覆盖。

![megamorphic ICs](https://github.com/RogerZZZZZ/V8-blog/tree/master/translation/What's-up-with-monomorphism/img/4.png)

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

