> Article: Explaining JavaScript VMs in JavaScript - Inline Caches

> Author: Vyacheslav Egorov

> [Original Link](https://mrale.ph/blog/2012/06/03/explaining-js-vms-in-js-inline-caches.html)


我用一个语言（或说是一个语言的子集）实现了虚拟机，而这个虚拟机同时也可以执行这种语言。如果我在大学中或是有一点空闲时间，我一定会致力于用Javascript来写Javascript的虚拟机。实际上这并不是一个独特的项目，因为来自世界各地的人们已经使用`Tachyon`做到了。但是我还是有一些想法：

![1](https://github.com/RogerZZZZZ/V8-blog/blob/master/translation/Explain-Javascript-VMs-in%20Javascript/img/1.png)

我想要帮助JS开发者理解JS引擎是怎么工作的，我想理解一个你使用的工具能够最大程度的帮助到你。这样更多的人就不会将JS虚拟机看做是一个神秘的黑盒。

我需要说的是在解释代码的内部运行原理以及如何写出高效的代码这件事上我并不孤独。来自全世界的很多人都在做相同的事情，但是我觉有一个问题一直限制开发人员高效的吸收这些知识。我们在用一种错误的方式运输这些知识。我也因此而内疚：

![2](https://github.com/RogerZZZZZ/V8-blog/blob/master/translation/Explain-Javascript-VMs-in%20Javascript/img/2.png)

- 有事我将我知道关于V8的事情包装成一个文献列表，上面都是"做这个，而不是那个"的一些建议。这样做的问题在于它没有解释任何东西。就好像是一个神圣的仪式一样被关注，但是也会很容易变得不再那么引人注目。
- 有事尝试去解释V8的内部运作原理，我选择的高度有问题。我喜欢一种想法，当看到一整篇被汇编代码塞的慢慢的讲义时，也许会让人们去学习汇编以及之后再次阅读这篇讲义，但是我也怕因为缺少一些实践过程中有用的信息，导致人们很快的遗忘。

我已经思考这个问题很长时间了，所以用Javascript来解释Javascript虚拟机。我做的演讲 - `V8 Inside Out`很好的实现了这个想法([slide](https://mrale.ph/s3/webrebels2012.pdf))。这篇文章我打算再次回顾我之前的内容。

### 用JS实现动态语言

想象你用JS为一种在语法上相似于JS但是有更简单的对象模型的语言设计一个虚拟机。不同于JS对象，它有一个任何类型都能作为key来对应value的`table`。简单来说，让我们想想`Lua`，与Javascript非常相近，但是作为一个语言有非常的不同。我最喜欢的一个例子`构造一个点的数组，之后计算向量综合`大概是这样：

```javascript
function MakePoint(x, y)
  local point = {}
  point.x = x
  point.y = y
  return point
end

function MakeArrayOfPoints(N)
  local array = {}
  local m = -1
  for i = 0, N do
    m = m * -1
    array[i] = MakePoint(m * i, m * -i)
  end
  array.n = N
  return array
end

function SumArrayOfPoints(array)
  local sum = MakePoint(0, 0)
  for i = 0, array.n do
    sum.x = sum.x + array[i].x
    sum.y = sum.y + array[i].y
  end
  return sum
end

function CheckResult(sum)
  local x = sum.x
  local y = sum.y
  if x ~= 50000 or y ~= -50000 then
    error("failed: x = " .. x .. ", y = " .. y)
  end
end

local N = 100000
local array = MakeArrayOfPoints(N)
local start_ms = os.clock() * 1000;
for i = 0, 5 do
  local sum = SumArrayOfPoints(array)
  CheckResult(sum)
end
local end_ms = os.clock() * 1000;
print(end_ms - start_ms)
```

我有一个习惯是检查至少最后的几个结果，为的是防止让别人发现我的测试用例不是一无是处。

如果你将上面的代码放入Lua的解释器中，你会得到：

```shell
∮ lua points.lua
150.2
```

很好，但是并不能帮助你理解VM是怎么工作的，所以让我们想想如果我们有一个用JS写的准Lua虚拟机的话，会怎么样。之所以用`准`是因为我并不想实现所有Lua的语法。我更想专注于`对象是table`这一个方面上。

```javascript
function MakePoint(x, y) {
  var point = new Table();
  STORE(point, 'x', x);
  STORE(point, 'y', y);
  return point;
}

function MakeArrayOfPoints(N) {
  var array = new Table();
  var m = -1;
  for (var i = 0; i <= N; i++) {
    m = m * -1;
    STORE(array, i, MakePoint(m * i, m * -i));
  }
  STORE(array, 'n', N);
  return array;
}

function SumArrayOfPoints(array) {
  var sum = MakePoint(0, 0);
  for (var i = 0; i <= LOAD(array, 'n'); i++) {
    STORE(sum, 'x', LOAD(sum, 'x') + LOAD(LOAD(array, i), 'x'));
    STORE(sum, 'y', LOAD(sum, 'y') + LOAD(LOAD(array, i), 'y'));
  }
  return sum;
}

function CheckResult(sum) {
  var x = LOAD(sum, 'x');
  var y = LOAD(sum, 'y');
  if (x !== 50000 || y !== -50000) {
    throw new Error("failed: x = " + x + ", y = " + y);
  }
}

var N = 100000;
var array = MakeArrayOfPoints(N);
var start = LOAD(os, 'clock')() * 1000;
for (var i = 0; i <= 5; i++) {
  var sum = SumArrayOfPoints(array);
  CheckResult(sum);
}
var end = LOAD(os, 'clock')() * 1000;
print(end - start);
```

然而如果你用`d8`(V8独立的命令行)来运行翻译后的代码，它将会被拒绝：

```shell
∮ d8 points.js
points.js:9: ReferenceError: Table is not defined
  var array = new Table();
                  ^
ReferenceError: Table is not defined
    at MakeArrayOfPoints (points.js:9:19)
    at points.js:37:13
```

被拒绝的原因很简单，我们还缺少了运行系统的代码(`runtime system code`)来实现对象模型和`loads 和 stores`，也许它非常的明显，但我还是想要强调：VM也许在外部看着是一个单一的黑盒，但是在内部是一支管乐队为了输出高性能而一起合作，其中有编译器，运行时的任务(`runtime routines`)，对象模型，垃圾收集等。幸运的是我们的语言和例子都很简单，所以我们的运行系统只有几十行代码的大小：

```javascript
function Table() {
  // Map from ES Harmony is a simple dictionary-style collection.
  this.map = new Map;
}

Table.prototype = {
  load: function (key) { return this.map.get(key); },
  store: function (key, value) { this.map.set(key, value); }
};

function CHECK_TABLE(t) {
  if (!(t instanceof Table)) {
    throw new Error("table expected");
  }
}

function LOAD(t, k) {
  CHECK_TABLE(t);
  return t.load(k);
}

function STORE(t, k, v) {
  CHECK_TABLE(t);
  t.store(k, v);
}

var os = new Table();

STORE(os, 'clock', function () {
  return Date.now() / 1000;
});
```

注意我必须使用`Harmony Map`而不是不同的Javascript对象，因为其可以把任意对象当做key，而不是仅仅是string。

```shell
∮ d8 --harmony quasi-lua-runtime.js points.js
737
```

现在我们翻译的代码运行了，但是速度很慢，因为每次加载和储存都要跨越所有的抽象层，在最终拿到值之前。让我们通过使用一个现在绝大多数JS虚拟机使用的基础优化来减少开支，那就是IC，即内联缓存。即使是用Java写的JS虚拟机也最终会使用它，因为`invokedynamic`本质上也是一个结构化的内联缓存，而它暴露在字节码层。内联缓存（同在在V8中缩写为IC）实际上是一个30年前为`SmallTalk VMs`而开发的很老的技术。

![3](https://github.com/RogerZZZZZ/V8-blog/blob/master/translation/Explain-Javascript-VMs-in%20Javascript/img/3.png)

### 好的鸭子叫声都相同

内敛缓存背后的原理很简单，当我们对对象以及它的属性假设正确时，我们希望创建一个支路或者是一条快速路径能够允许我们不用进入运行系统快速的拿到属性值。在用充满动态类型、动态绑定(`late binding`)以及类似`eval`这样奇怪的操作的语言写的系统中，对对象形状进行假设是很困难的。因此我们想要让我们的`loads/stores`有学习和观察的能力：即一旦他们看到一些对象可以用一种方法自己转换，使得后续加载相似形状的对象更快。因此我们将要缓存一些之前看到过对象的结构在`loads/stores`中，这也就是内联缓存。只要你能找到一条有意义的快速路径，内联缓存就可以结合动态行为被运用在几乎任何操作上：算术操作，函数调用等。一些内联缓存也可以也可以缓存不止一条快速路径，这也就是所谓的`多态`.



## CONTINUED