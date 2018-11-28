> Article: Explaining JavaScript VMs in JavaScript - Inline Caches

> Author: Vyacheslav Egorov

> [Original Link](https://mrale.ph/blog/2012/06/03/explaining-js-vms-in-js-inline-caches.html)


我用一个语言（或说是一个语言的子集）实现了虚拟机，而这个虚拟机同时也可以执行这种语言。如果我在大学中或是有一点空闲时间，我一定会致力于用Javascript来写Javascript的虚拟机。实际上这并不是一个独特的项目，因为来自世界各地的人们已经使用`Tachyon`做到了。但是我还是有一些想法：

![1](https://github.com/RogerZZZZZ/V8-blog/raw/master/Explain-Javascript-VMs-in-Javascript/img/1.png)

我想要帮助JS开发者理解JS引擎是怎么工作的，我想理解一个你使用的工具能够最大程度的帮助到你。这样更多的人就不会将JS虚拟机看做是一个神秘的黑盒。

我需要说的是在解释代码的内部运行原理以及如何写出高效的代码这件事上我并不孤独。来自全世界的很多人都在做相同的事情，但是我觉有一个问题一直限制开发人员高效的吸收这些知识。我们在用一种错误的方式运输这些知识。我也因此而内疚：

![2](https://github.com/RogerZZZZZ/V8-blog/raw/master/Explain-Javascript-VMs-in-Javascript/img/2.png)

- 有时我将我知道关于V8的事情包装成一个文献列表，上面都是"做这个，而不是那个"的一些建议。这样做的问题在于它没有解释任何东西。就好像是一个神圣的仪式一样被关注，但是也会很容易变得不再那么引人注目。
- 有时尝试去解释V8的内部运作原理，我选择的高度有问题。我喜欢一种想法，当看到一整篇被汇编代码塞的满满的讲义时，也许会让人们去学习汇编以及之后再次阅读这篇讲义，但是我也怕因为缺少一些实践过程中有用的信息，导致人们很快的遗忘。

我已经思考这个问题很长时间了，所以用Javascript来解释Javascript虚拟机。我做的演讲 - `V8 Inside Out`很好的实现了这个想法([slide](https://mrale.ph/s3/webrebels2012.pdf))。这篇文章我打算再次回顾我之前的内容。

### 用JS实现动态语言

想象你用JS为一种在语法上相似于JS但是有更简单的对象模型的语言设计一个虚拟机。不同于JS对象，它有一个任何类型都能作为key来对应value的`table`。简单来说，让我们想想`Lua`，与Javascript非常相近，但是作为一个语言又有很多的不同。我最喜欢的一个例子`构造一个点的数组，之后计算向量综合`大概是这样：

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

![3](https://github.com/RogerZZZZZ/V8-blog/raw/master/Explain-Javascript-VMs-in-Javascript/img/3.png)

### 好的鸭子叫声都相同

内敛缓存背后的原理很简单，当我们对对象以及它的属性假设正确时，我们希望创建一个支路或者是一条快速路径能够允许我们不用进入运行系统从而快速的拿到属性值。在用充满动态类型、动态绑定(`late binding`)以及类似`eval`这样奇怪的操作的语言写的系统中，对对象形状进行假设是很困难的。因此我们想要让我们的`loads/stores`有学习和观察的能力：即一旦他们看到一些对象可以用一种方法自己转换，使得后续加载相似形状的对象更快。因此我们将要缓存一些之前看到过对象的结构在`loads/stores`中，这也就是内联缓存。只要你能找到一条有意义的快速路径，内联缓存就可以结合动态行为被运用在几乎任何操作上：算术操作，函数调用等。一些内联缓存也可以缓存不止一条快速路径，这也就是所谓的`多态`.

如果我们开始考虑将IC运用到上述的代码中时，我们就会发现需要改变之前的对象模型了。我们没办法以更快的方式从`Map`中读值，我们一直只能访问`get`方法。[如果而我们可以窥视一个Map之后原始的`hashtable`，我们就可以通过缓存`bucket index`让IC为我们服务，甚至不需要新的对象布局(`object layout`)]

![4](https://github.com/RogerZZZZZ/V8-blog/raw/master/Explain-Javascript-VMs-in-Javascript/img/4.png)

### 发现隐藏结构

![5](https://github.com/RogerZZZZZ/V8-blog/raw/master/Explain-Javascript-VMs-in-Javascript/img/5.png)

对于高效的table，比如我们使用的结构化数据需要像C语言中的`structs`一样：一些在固定偏移位置的域的序列。 与table相同的被用作为array：我们想要一些数字属性存于数组中。但十分明显的是：不是所有的table都适用这样的表示方法：一些实际上被使用为table，要不包含非字符串非数字的keys，要不包含很多用字符串命名的属性。不幸的是我们不能执行任何高代价的类型推断，我们只能去发现一种在程序运行时在每个table之后的一种结构，并创建和改变它们。幸运的是有一种有名的技术允许我们这样做，这个技术被称为隐藏类(`hidden classes`)

隐藏类之后的概念可以归结为两件简单的事：

- 运行系统让每一个对象都关联一个隐藏类，就像Java虚拟机让每一个对象关联一个`java.lang.Class`的实例；
- 如果一个对象的结构改变了，运行系统就会创建或者寻找一个能够匹配新结构的隐藏类，之后将它附给这个对象。

隐藏类有一个非常重要的功能：它们允许虚拟机通过简单的比对现在结构与缓存隐藏类的区别，来检查假设的正确性。这正是内联缓存所需要的。让我们为我们的准Lua运行环境来实现简单的隐藏类。每个隐藏类的实质都是一个属性描述符的集合，在集合中每个描述符不是一个真实属性就是一个从没有某些属性的类指向具有此属性的类的转换。

```javascript
function Transition(klass) {
  this.klass = klass;
}

function Property(index) {
  this.index = index;
}

function Klass(kind) {
  // Classes are "fast" if they are C-struct like and "slow" is they are Map-like.
  this.kind = kind;
  this.descriptors = new Map;
  this.keys = [];
}
```

转换的存在是为了让以相同方式创建的对象之间可以共享隐藏类：如果你有两个对象共享隐藏类，当你给它们添加相同属性时，你不希望得到不同的隐藏类。

```javascript
Klass.prototype = {
  // 创建一个之前不存在且带有一个新属性的隐藏类
  // 当前的隐藏类
  addProperty: function (key) {
    var klass = this.clone();
    klass.append(key);
    // 让隐藏类和transition进行连接来实现共享
    // this == add property key ==> klass
    this.descriptors.set(key, new Transition(klass));
    return klass;
  },

  hasProperty: function (key) {
    return this.descriptors.has(key);
  },

  getDescriptor: function (key) {
    return this.descriptors.get(key);
  },

  getIndex: function (key) {
    return this.getDescriptor(key).index;
  },

  // 克隆一个有相同属性的隐藏类
  // 在相同的偏移（但是没有任何的transitions）
  clone: function () {
    var klass = new Klass(this.kind);
    klass.keys = this.keys.slice(0);
    for (var i = 0; i < this.keys.length; i++) {
      var key = this.keys[i];
      klass.descriptors.set(key, this.descriptors.get(key));
    }
    return klass;
  },

  // 将真实属性添加到描述符中
  append: function (key) {
    this.keys.push(key);
    this.descriptors.set(key, new Property(this.keys.length - 1));
  }
};
```

现在你可以让我们的tables更加灵活。

```javascript
 ROOT_KLASS = new Klass("fast");

function Table() {
  // 所有的tables开始于快速的根隐藏类，开始于一棵简单的树。在V8中隐藏类实际上开始于一个 
  // 森林 - 他们有很多根类，比如说：每一个构造器都是一个根类。但这只是部分，因为V8中的隐
  // 藏类封装了构造器的信息，比如原型的指针实际就存储在隐藏类中，而不是在对象自己中，所以
  // 不同原型的类必须有不同的隐藏类即使它们有同样的结构，然而有多根类的结构也同样允许独立
  // 的发展这些树，然而各自捕捉类的特定进化。
  this.klass = ROOT_KLASS;
  this.properties = [];  // 命名属性的数组 'x','y',...
  this.elements = [];  // 属性下标的集合: 0, 1, ...
  // 我们实际有点作弊，因为允许任何int32的数据存入，我们也允许V8来选择数组存储的合适表示
  // 因为太多的细节无法放到一篇文章中。
}

Table.prototype = {
  load: function (key) {
    if (this.klass.kind === "slow") {
      // Slow class => 属性将会被以Map的形式表示
      return this.properties.get(key);
    }

    // 这是一个fast table，只带有索引和命名的属性
    if (typeof key === "number" && (key | 0) === key) {  // 带有索引的属性.
      return this.elements[key];
    } else if (typeof key === "string") {  // 带有命名的属性.
      var idx = this.findPropertyForRead(key);
      return (idx >= 0) ? this.properties[idx] : void 0;
    }

    // 在fast table中只能有string&number的keys
    return void 0;
  },

  store: function (key, value) {
    if (this.klass.kind === "slow") {
      // Slow class => 属性将会被以Map的形式表示
      this.properties.set(key, value);
      return;
    }

    // 这是一个fast table，只带有索引和命名的属性
    if (typeof key === "number" && (key | 0) === key) {  // 带有索引的属性.
      this.elements[key] = value;
      return;
    } else if (typeof key === "string") {  // 带有命名的属性.
      var index = this.findPropertyForWrite(key);
      if (index >= 0) {
        this.properties[index] = value;
        return;
      }
    }

    this.convertToSlow();
    this.store(key, value);
  },

  // 寻找属性或是可能的话添加一个，之后返回属性的索引或者是-1(当我们有太多属性时，之后
  // 切换到slow模式)
  findPropertyForWrite: function (key) {
    if (!this.klass.hasProperty(key)) {  // 当属性不存在时添加它.
      // 太多属性，退出fast模式
      if (this.klass.keys.length > 20) return -1;

      // 切换到有这个属性的类
      this.klass = this.klass.addProperty(key);
      return this.klass.getIndex(key);
    }

    var desc = this.klass.getDescriptor(key);
    if (desc instanceof Transition) {
      // 属性不存在，但是我们有一个transition到一个有这个属性的类
      this.klass = desc.klass;
      return this.klass.getIndex(key);
    }

    // 得到这个属性的索引
    return desc.index;
  },

  // 如果存在找到这个属性的索引，否则返回-1
  findPropertyForRead: function (key) {
    if (!this.klass.hasProperty(key)) return -1;
    var desc = this.klass.getDescriptor(key);
    if (!(desc instanceof Property)) return -1;  // 这里我们不关心transition
    return desc.index;
  },

  // 将所有的属性复制到一个map中，然后切换成slow
  convertToSlow: function () {
    var map = new Map;
    for (var i = 0; i < this.klass.keys.length; i++) {
      var key = this.klass.keys[i];
      var val = this.properties[i];
      map.set(key, val);
    }

    Object.keys(this.elements).forEach(function (key) {
      var val = this.elements[key];
      map.set(key | 0, val);  // Funky JS, force key back to int32.
    }, this);

    this.properties = map;
    this.elements = null;
    this.klass = new Klass("slow");
  }
};
```

[我不会对上述的代码一行一行的解释，因为都是有注释的JS代码，而不是C++或是汇编...这都是由JS书写的。然而当你遇到困惑时也可以通过评论或是邮件的方式来问我]

现在我们在运行系统中有了隐藏类，这允许我们能够快速检查对象结构，通过偏移量快速读取属性值。这些的存在也让我们能够实现内联缓存。编译器和运行系统还需要一些补充的方法(还记得我是怎么谈论虚拟机中不同部分是如何合作的吗？)。

### 生成代码的拼凑被子 (Patchwork quilts of generated code)

![6](https://github.com/RogerZZZZZ/V8-blog/raw/master/Explain-Javascript-VMs-in-Javascript/img/6.png)

其中一种实现一个内联缓存的方法是将它一分为二：生成代码中可变的方法调用，以及可以被方法调用站点调用的`stubs`集合(生成的原生代码的小片段)。`stubs`(或是运行系统)本质上可以找到方法调用的站点: `stubs`只包含在特定假设下编译的快速路径，如果这些假设不适用于`stubs`看到的对象，则`stubs`会开始修改（修补）这个调用`stubs`的调用位置，以便让该位置适应新的环境。我们原生JS中的IC也包含两个部分：

- 每个IC的全局变量将会被用来模拟可修改的调用指令。
- 以及闭包将会代替`stubs`

在原生代码中，V8通过检查栈中返回的的地址来修改IC站点。我们不能再纯JS种做这样的操作(`arguments.caller`没有足够的细粒度)，所以我们明确的传递IC的id到IC stub中。下面就是例子：

```javascript
// 初始时所有IC都是未初始化状态
// 它们将会无法命中缓存，并且在运行系统中将会一直这样
var STORE$0 = NAMED_STORE_MISS;
var STORE$1 = NAMED_STORE_MISS;
var KEYED_STORE$2 = KEYED_STORE_MISS;
var STORE$3 = NAMED_STORE_MISS;
var LOAD$4 = NAMED_LOAD_MISS;
var STORE$5 = NAMED_STORE_MISS;
var LOAD$6 = NAMED_LOAD_MISS;
var LOAD$7 = NAMED_LOAD_MISS;
var KEYED_LOAD$8 = KEYED_LOAD_MISS;
var STORE$9 = NAMED_STORE_MISS;
var LOAD$10 = NAMED_LOAD_MISS;
var LOAD$11 = NAMED_LOAD_MISS;
var KEYED_LOAD$12 = KEYED_LOAD_MISS;
var LOAD$13 = NAMED_LOAD_MISS;
var LOAD$14 = NAMED_LOAD_MISS;

function MakePoint(x, y) {
  var point = new Table();
  STORE$0(point, 'x', x, 0);  // 最后一个数字是 IC 的 id: STORE$0 &rArr; id 为 0
  STORE$1(point, 'y', y, 1);
  return point;
}

function MakeArrayOfPoints(N) {
  var array = new Table();
  var m = -1;
  for (var i = 0; i <= N; i++) {
    m = m * -1;
    // 现在我们也能区分表达式x[p]以及x.p。 第一个被称为键控load/store，而第二个被称为
    // 命名的load/store。
    // 主要的区别在于命名的load/store使用一个固定已知不变的string键，所以可以被优化为
    // 一个固定的属性偏移
    KEYED_STORE$2(array, i, MakePoint(m * i, m * -i), 2);
  }
  STORE$3(array, 'n', N, 3);
  return array;
}

function SumArrayOfPoints(array) {
  var sum = MakePoint(0, 0);
  for (var i = 0; i <= LOAD$4(array, 'n', 4); i++) {
    STORE$5(sum, 'x', LOAD$6(sum, 'x', 6) + LOAD$7(KEYED_LOAD$8(array, i, 8), 'x', 7), 5);
    STORE$9(sum, 'y', LOAD$10(sum, 'y', 10) + LOAD$11(KEYED_LOAD$12(array, i, 12), 'y', 11), 9);
  }
  return sum;
}

function CheckResults(sum) {
  var x = LOAD$13(sum, 'x', 13);
  var y = LOAD$14(sum, 'y', 14);
  if (x !== 50000 || y !== -50000) throw new Error("failed x: " + x + ", y:" + y);
}
```

上述变化再次不言自明： 每个属性输入id调用load/store来得到自己的IC。现在还剩一小步：去实现当没有匹配上`stubs`时，“stub 编译器”将会生成一个特殊的stubs:

```javascript
function NAMED_LOAD_MISS(t, k, ic) {
  var v = LOAD(t, k);
  if (t.klass.kind === "fast") {
    // 为一个固定的类和键k创建一个load stub，之后可以从一个固定的偏移位置读取属性。
    var stub = CompileNamedLoadFastProperty(t.klass, k);
    PatchIC("LOAD", ic, stub);
  }
  return v;
}

function NAMED_STORE_MISS(t, k, v, ic) {
  var klass_before = t.klass;
  STORE(t, k, v);
  var klass_after = t.klass;
  if (klass_before.kind === "fast" &&
      klass_after.kind === "fast") {
    // 为一个类之间固定的transition和固定的键k个性化地创建一个store stub，在固定的
    // 偏移位置存储属性，在必要的时候也替换对象的隐藏类
    var stub = CompileNamedStoreFastProperty(klass_before, klass_after, k);
    PatchIC("STORE", ic, stub);
  }
}

function KEYED_LOAD_MISS(t, k, ic) {
  var v = LOAD(t, k);
  if (t.klass.kind === "fast" && (typeof k === 'number' && (k | 0) === k)) {
    // 为快速存储在元素数组中创建一个stub
    // 实际上不依赖于类，但如果我们有更复杂的存储系统的话也可以依赖类
    var stub = CompileKeyedLoadFastElement();
    PatchIC("KEYED_LOAD", ic, stub);
  }
  return v;
}

function KEYED_STORE_MISS(t, k, v, ic) {
  STORE(t, k, v);
  if (t.klass.kind === "fast" && (typeof k === 'number' && (k | 0) === k)) {
    // 为快速存储在元素数组中创建一个stub
    // 实际上不依赖于类，但如果我们有更复杂的存储系统的话也可以依赖类
    var stub = CompileKeyedStoreFastElement();
    PatchIC("KEYED_STORE", ic, stub);
  }
}

function PatchIC(kind, id, stub) {
  this[kind + "$" + id] = stub;  // 不严格的JS写法，这是一个全局对象
}

function CompileNamedLoadFastProperty(klass, key) {
  // 键被认为是一个常量 (命名的读取)，特定的索引
  var index = klass.getIndex(key);

  function KeyedLoadFastProperty(t, k, ic) {
    if (t.klass !== klass) {
      // 期望的kclass没有匹配上，不能够使用已缓存的索引
      // 回到运行系统中
      return NAMED_LOAD_MISS(t, k, ic);
    }
    return t.properties[index];
  }

  return KeyedLoadFastProperty;
}

function CompileNamedStoreFastProperty(klass_before, klass_after, key) {
  // 键被认为是一个常量 (命名的读取)，特定的索引
  var index = klass_after.getIndex(key);

  if (klass_before !== klass_after) {
    // 在存储过程中发生transition
    // 编译更新隐藏类的stub
    return function (t, k, v, ic) {
      if (t.klass !== klass_before) {
        // 期望的kclass没有匹配上，不能够使用已缓存的索引
        // 回到运行系统中
        return NAMED_STORE_MISS(t, k, v, ic);
      }
      t.properties[index] = v;  // 快速存储.
      t.klass = klass_after;  // T-t-t-transition!
    }
  } else {
    // 写一个存在的属性，无transition
    return function (t, k, v, ic) {
      if (t.klass !== klass_before) {
        // 期望的kclass没有匹配上，不能够使用已缓存的索引
        // 回到运行系统中
        return NAMED_STORE_MISS(t, k, v, ic);
      }
      t.properties[index] = v;  // 快速存储.
    }
  }
}


function CompileKeyedLoadFastElement() {
  function KeyedLoadFastElement(t, k, ic) {
    if (t.klass.kind !== "fast" || !(typeof k === 'number' && (k | 0) === k)) {
      // 如果table是slow模式，或者键不是number，则不能使用快速路径
      // 回到运行系统，它能处理一切
      return KEYED_LOAD_MISS(t, k, ic);
    }
    return t.elements[k];
  }

  return KeyedLoadFastElement;
}

function CompileKeyedStoreFastElement() {
  function KeyedStoreFastElement(t, k, v, ic) {
    if (t.klass.kind !== "fast" || !(typeof k === 'number' && (k | 0) === k)) {
      // 如果table是slow模式，或者键不是number，则不能使用快速路径
      // 回到运行系统，它能处理一切
      return KEYED_STORE_MISS(t, k, v, ic);
    }
    t.elements[k] = v;
  }

  return KeyedStoreFastElement;
}
```

虽然有很多的代码，但是都十分的简单，上面也给出的很全的解释：IC观察，stub的编译器/工厂生层生成的stub[细心地读者会发现我从一开始就用快速模式来初始化所有键来存储IC，或是一旦进入就切换到快速模式]。

如果我们将所有代码合并，并重新运行我们的`benchmark`，我们就会欣喜的发现：

```shell
∮ d8 --harmony quasi-lua-runtime-ic.js points-ic.js
117
```

相比于之前的实现，这次快了6倍。

### JavaScript VM优化永远不会得出结论

但愿在你读这部分时，上面的内容你都已经读过...我作为了一个JS开发者尝试着从一个不同的角度来看一些加强JS引擎的方法。我代码写的越多，越会感觉到这是一个盲人摸象的故事。给你一种望向深渊的感觉：V8有10中描述符，5种元素(加9中外部元素), `ic.cc`包含了绝大多数IC选择的逻辑，而这一部分代码就超过2500行，V8中的内联缓存有多于两种以上的状态(未初始化、预单态，单态，多态，通用状态未提到键控load/stores的内联缓存的特殊状态，或者是算术内联缓存的完全不同层次的状态)。`ia32-specific`手写内联缓存stub有超过5000行代码等等。这些数字只会随着时间的推移不断增加，V8学会区别/适应更多更多的对象结构。我现在甚至都没有触及到对象本身(`objects.cc` 1万3千行代码)，或是垃圾回收器，或是优化编译器。

然而在可预见的未来我确信基本原理是不会改变的，而当他们改变了基本原理将会带来一次突破，也同时会“发出巨响”，你当然也会注意到。所以我认为这个关于尝试使用JS来书写和理解基本原理是十分十分重要的。

我希望明天或者是一周以后你会告诉你的同事，为什么有条件的在代码某个地方对对象添加性能会影响触摸这些对象的其他“热函数”循环的性能。因为你知道隐藏类发生了改变。

## DONE