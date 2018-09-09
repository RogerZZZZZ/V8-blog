> Article: What's up with monomorphism?

> Author: Vyacheslav Egorov

> Original Link: [Link](https://mrale.ph/blog/2015/01/11/whats-up-with-monomorphism.html)

有关JS性能的博客和演讲都经常强调单态的重要性。然而他们一般都没有对单态/多态是什么给出易理解的解释。不出所料的是我得到关于性能方面最常见的问答请求之一就是解释单态究竟是什么，多态是如何产生的，它为什么不好。

无济于事的是"多态"这个词本身就被过度重载(overloaded)。在经典面向对象编程的概念中，多态往往意味着类型多态以及重写基类行为的能力。Haskell程序员则会认为是参数多态(`parametric polymorphism`). 然而多态在JS性能中的意义则不同，它被称为`site polymorphism`.

在我决定写一篇关于它的博客之前，我已经通过很多方法来解释这个概念吗，所以下次我可以直接给出链接而不是现场再临时解释了。

## 动态查找101

~[simple implementation]()

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

![real implementation]()

要是程序中每次属性的访问都有能力从之前见过的对象中学习，并且将这个知识运用到相似的新对象上会怎么样呢？可能这会让我们省去很多执行一般查找算法的时间，而去在一些特定形式的对象上使用更快的方法。
