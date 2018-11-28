> Article: JavaScript engine fundamentals: Shapes and Inline Caches

> Author: Mathias

> [Original Link](https://mathiasbynens.be/notes/shapes-ics)

这篇文章描述了一些对于所有Javascript引擎来说常见且非常重要的基础--而不是仅仅针对V8（作者[Benedikt Meurer](https://twitter.com/bmeurer)&[Mathias Bynens](https://twitter.com/mathias)开发的引擎），作为一个JS开发者，理解JS引擎如何工作的可以帮助你优化你的代码。

### Javascript的工作流程

全部都开始于你所写出的JS代码。JS引擎解析源代码，然后将它转化为抽象语法树（AST）。基于AST解释器开始生成字节码，非常好！到这个时候，引擎开始真正的运行JS代码。

![1](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/1.svg)

为了让它运行的更快，字节码将会与分析数据一同被输入到优化编译器中。优化编译器根据这些信息作出一定的假设，之后生成高度优化的机器码。

如果在某个时刻其中一个假设结果为不正确，那么优化编译器将会进行逆优化，然后回到解释器中。

### 解释器/编译器在JS引擎中的工作流程

现在，让我们开始看真正运行你JS代码的工作流程部分，即代码被解释和优化的地方，之后复习一下一些主要JS引擎的区别。

通常来说，有一个包含一个解释器和一个优化编译器的工作流程。解释器快速的生成没有被优化的字节码，优化编译器花稍微长一点的时间并最后生成高度优化的机器码。

![2](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/2.svg)

下面的工作流程几乎就是V8(Chrome和Node.js所使用的引擎)的工作原理：

![3](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/3.svg)

V8中的解释器被称为`点火器-Ignition`，它的职责是生成和运行字节码。在运行的同时收集分析数据，这些数据将会加速之后的运行过程。当一个函数变`hot`，比如说执行更加频繁，生成的字节码和分析数据将会被传到`TurboFan`中，我们的优化编译器，去根据收集来的分析数据来生成优化的机器码。

![4](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/4.svg)

`SpiderMonkey`，Mozilla用在FireFox和SpiderNode中的JS引擎。其工作原理有一点不同于V8，它有两个优化编译器。解释器优化后到`Baseline`编译器中生成稍微优化后的代码。`IonMonkey`编译器则在使用代码运行时收集的分析数据来生成深度优化的代码。如果推测优化失败了，`IonMonkey`会回到`Baseline`阶段。

![5](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/5.svg)

`Chakra`，微软的在`Edge & Node-ChakraCore`中使用的JS引擎，也有相似的两个优化编译器作为开始。解释器优化后进入`SimpleJIT` - `JIT`意思为`Just-In-Time compile(及时编译器)`，在这里生成轻度优化的代码。再运用上分析数据，`FullJIT`会生成深度优化代码。

![6](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/6.svg)

`JavaScriptCore`（简称JSC），苹果在Safari和React Native中使用的JS引擎，使用了三个不同的优化编译器。`LLInt`，低级的解释器，之后到达`Baseline`编译器，再之后进入`DFG`(数据流图`Data Flow Graph`)编译器，最后进入FTL(Faster Than Light)编译器。

为什么有一些引擎会比其他引擎有更多的优化编译器？这都是一些权衡和取舍。一个解释器可以快速的生成字节码，但是通常来说字节码不是那么地高效。在另一方面一个优化编译器花更长的时间来最终生成更加有效率的机器码。这就是快速得到可以运行的代码或者是牺牲一些时间来让代码运行有更好的性能间的抉择。一些引擎选择加入一些有不同时间/效率特性的优化编译器，允许对这些权衡进行更细粒度的控制，但是代价是额外的复杂性。另一个权衡是内存的使用，可以在[我们后续的文章](https://mathiasbynens.be/notes/prototypes#tradeoffs)中寻找更多的观点。

我们只是提到了每个JS引擎中解释器和优化编译器工作流程的主要区别。但是除了这些区别，在更高的层面，所有的JS引擎都是相同架构的：都有一个分析器和一些解释器/编译器工作流程。


### JavaScript的对象模型

让我们通过研究一些方面是如何实现的来看看还有什么是JS引擎都具有的。

比如，JS引擎是如何实现JS对象模型，它们又是用什么奇技淫巧来加速在JS对象上的属性访问？最终你会发现所有主要的引擎实现都十分的类似。

`ECMAScript规范`将所有的对象如字典一样的定义过，每一个属性都有一个key值与之匹配。

![7](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/7.svg)

除了`[[Value]]]`本身以外，规范还定义了这些属性：

- `[[Writable]]` 决定属性是否可以被重新赋值
- `[[Enumerable]]` 决定属性是否可以在`for-in`循环中被遍历
- `[[Configurable]]` 决定属性是否可以被删除

`[[双中括号]]`表示看起来很funky，但是它确实是规范中定义不被直接暴露在外的属性的方法。你可以通过API`Object.getOwnPropertyDescriptor`来获得这些属性的值：

```javascript
const object = { foo: 42 };
Object.getOwnPropertyDescriptor(object, 'foo');
// → { value: 42, writable: true, enumerable: true, configurable: true }
```

好了，这就是JS如何定义对象，那数组又怎么样呢？

你可以将数组看作是对象的一个特例。一个区别在于数组对于数组下标有特殊的处理。数组索引在`ECMAScript规范`中时特殊的一个条目。JS中数组不能超过2的32次方-1个。在这个限制中所有索引值都是合法的，即从0到2的32次方-2的整数。

另一个区别在于数组有神奇的`length`属性。

```javascript
const array = ['a', 'b'];
array.length; // → 2
array[2] = 'c';
array.length; // → 3
```

在这个例子中，创建了一个长度为2的数组，当在下标为2的地方赋值一个新的元素时，`length`会自动的更新。

JS定义了数组与对象相似。比如，所有keys包括数组索引都可以被用字符串来表示。数组中的第一个元素被存在key为`'0'`的地方。

![8](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/8.svg)

`length`属性恰好是一个不可枚举以及不可删除的属性。

一旦一个元素被加入到数组中，JS将会自动更新length的`[[Value]]`。

![9](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/9.svg)

一般来说，数组的行为和对象十分相似。

### 优化属性访问

现在我们知道了对象在JS中是怎么定义的了，让我们深入讨论JS引擎是如何有效的处理对象的。

目前为止属性访问在JS程序中是最常见的操作，所以对JS引擎来说让属性访问操作加速十分的重要。

```javascript
const object = {
	foo: 'bar',
	baz: 'qux',
};

// Here, we’re accessing the property `foo` on `object`:
doSomething(object.foo);
//          ^^^^^^^^^^
```

### Shapes - 形状、结构

在JS程序中，多个对象拥有相同属性的key是很普遍的。这样的对象就有相同的`shape`。

```javascript
const object1 = { x: 1, y: 2 };
const object2 = { x: 3, y: 4 };
// `object1` and `object2` have the same shape.
```

使用相同`shape`去访问对象上的相同属性也十分的普遍。

```javascript
function logX(object) {
	console.log(object.x);
	//          ^^^^^^^^
}

const object1 = { x: 1, y: 2 };
const object2 = { x: 3, y: 4 };

logX(object1);
logX(object2);
```

考虑到这一点，JS引擎可以根据对象的`shape`来优化对象属性访问。下面就是它如何工作的。

让我们假设我们有一个对象有`x`和`y`两个属性，它使用我们之前讨论的字典数据类型：包含字符串类型的key，它们指向对应的属性值。

![10](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/b_1.svg)

当你访问一个属性，比如`Object.y`，JS引擎会在`JSObject`中寻找key `y`，之后加载对应的属性值，最后返回`[[Value]]`.

但是这些属性值存在内存的什么位置呢？我们需要把它们作为`JSObject`的一部分存储吗？如果假设我们之后会看到更多对象具有这个`shape`，存储所有包含`JSObject`属性名和属性值的‘字典’将会非常的浪费。因为属性名在所有相同`shape`的对象中都是重复的。这就造成了大量的重复和没必要的内存使用。作为优化，引擎分开存储对象的`Shape`.

![11](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/10.svg)

`Shape`包含了除了`[[Value]]`以外所有的属性名和属性值。转而存储`JSObject`中值得偏移量，因此JS引擎知道在哪找到值。每个有相同`shape`的`JSObject`都指向这个`shape`的实例。现在每个`JSObject`只需要存储对于每个对象不同的值就可以了。

![12](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/11.svg)

当我们有多个对象时，优点就十分的明显。不管有多少个对象，只要它们的`shape`相同，我们只需要存储一次`shape`和属性信息！

所有引擎都使用`shape`来作为优化的一种手段，但是并不是多有的引擎都这么称呼：

- 学术论文一般称为`Hidden Classes`
- V8称它为`Map`
- Chakra称它为`Types`
- JavaScriptCore称为`Structures`
- SpiderMonkey称为`Shape`

在接下来的文章中，我们将继续称它为`shapes`

### 转换链和树

当你有一个具有一定`shape`的对象被添加一个属性时会发生什么？JS引擎是如何找到新的`shape`的？

```javascript
const object = {};
object.x = 5;
object.y = 6;
```

新的`shape`来源于一个在JS引擎中被称为转换链的东西，这里有一个例子：

![13](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/12.svg)

对象开始时没有任何的属性，所以指向一个空的`shape`。下一步就是添加了一个值为5的属性`x`到对象中，所以JS引擎将`shape`转换到包含属性`x`的形态，并且在`JSObject`偏移量为`0`的地方添加值`5`。在下一行中添加属性`y`，所以这时引擎转换到下一个包含`x & y`的`shape`，并且附加上值`6`到`JSObject`中(在偏移量为`1`的地方)。

> 注意: 属性添加的顺序印象`shape`，比如`{x: 4, y: 5}`生成出的shape与`{y: 5, x: 4}`不相同。

我们不需要存储每个`Shape`中所有的属性，而是，每个`Shape`只需要知道新引入的属性。比如：在这个例子中我们并没有必要再最后一个`shape`中存储`x`的信息，因为它可以在之前的结构中被找出。为了实现这个，每个`Shape`都有一个指向前一个`shape`的链接。

![14](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/13.svg)

如果你在JS代码中写`o.x`，JS引擎将会通过追溯转换链直到在一个`Shape`中招待属性`x`为止。

但是当没有办法创建转换链时会发生什么？比如，如果你有两个空对象，之后你在每个对象中加上不同的属性。

```javascript
const object1 = {};
object1.x = 5;
const object2 = {};
object2.y = 6;
```

在这种情况下，我们需要分支，而不是一个转换链，我们最后会得到一个转换树。

![15](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/14.svg)

这里我们创建了空对象`a`，在其中加入属性`x`。最后我们得到一个包含单个值得`JSObject`以及两个`shapes`：一个空`shape`以及一个带有属性`x`的`shape`。

第二个例子中同样开始于一个空对象`b`，但是之后我们附加上不同的属性`y`，最后我们得到两条`shape`链以及总共三个`shapes`.

但这就意味着我们总要由一个空`shape`开始吗？其实没有必要。引擎对已包含属性的对象做了一些优化。假设我们要么从空对象开始添加`x`，或是开始于一个包含`x`的对象。

```javascript
const object1 = {};
object1.x = 5;
const object2 = { x: 6 };
```

在第一个例子中，我们开始于一个空`shape`然后过渡到一个含有`x`的`shape`就像而我们之前看到的一样。

在`Object2`的例子中，直接生成从头开始已经有`x`的对象，而不是从空对象开始转换是有意义的。

![16](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/15.svg)

一个包含`x`的对象开始于一个包含`x`的`shape`，有效的跳过了空`shape`的阶段。这就是（至少）V8以及SpiderMonkey的做法。这样的优化缩短了转换链，也更加高效从字面量中构建对象。

Benedikt的博客[surprising polymorphism in React applications](https://medium.com/@bmeurer/surprising-polymorphism-in-react-applications-63015b50abc)讨论了这些微妙的操作是如何影响日常的性能的。

这里有一个例子，一个3D的点包含`x, y, z`三个属性。

```javascript
const point = {};
point.x = 4;
point.y = 5;
point.z = 6;
```

根据我们之前学到的，这会创建三个`shapes`在内存中(不包括空的`shape`)，当我们去访问对象上的`x`属性时，例如，当我们在程序中写下`point.x`，JS引擎将会顺着转换链去寻找：开始时位于最低端的`Shape`，最后在最上层的`Shape`中找到`x`。

![17](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/16.svg)

当我们频繁这么做的话将会特别的慢，特别是当一个对象有很多属性的时候。寻找属性的时间复杂度为`O(n)`，即随着对象中的属性增加而线性的增加。为了加速搜寻属性，JS引擎加入了`ShapeTable`的数据结构。这个`ShapeTable`类似一个字典，将每个key匹配上不同的`Shape`。

![18](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/17.svg)

等一下，现在我们回到了字典查询...在我们开始引入`Shapes`之前，我们就是这样！那么为什么我们要为`Shapes`烦恼呢？

原因就是`shapes`启用了另一个优化，称为内联缓存。

### 内联缓存(ICs)

`Shapes`背后的主要动机就是`内联缓存或是ICs`的概念。内联缓存是让JS运行快速的核心！JS引擎使用ICs来记住对象上属性的位置，以此来减少昂贵的查找开销。

下面是一个函数`getX`，作用是加载对象中的属性`x`:

```javascript
function getX(o) {
	return o.x;
}
```

如果我们在`JSC`下运行这个函数，它会生成如下的字节码：

![18](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/18.svg)

第一个指令`get_by_id`从第一个参数(`arg1`)中加载了属性`x`，然后将结果存储到`loc0`中，第二个指令返回存在`loc0`中的数值。

`JSC`还在指令`get_by_id`中嵌入一个内联缓存，由两条没有被初始化的插槽组成。

![19](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/19.svg)

现在我们假设将对象`{x: 'a'}`传入到函数`getX`中。根据我们之前学到的，这个对象又要一个包含属性`x`的`shape`，而这个`shape`存储了属性`x`以及它的偏移量。当你在第一次执行函数的时候，指令`get_by_id`会寻找属性`x`然后发现值被存在偏移量为`0`的位置。

![20](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/20.svg)

内联缓存将会被嵌入到指令`get_by_id`中去记忆`shapes`以及被寻找到的属性的偏移量：

![21](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/21.svg)

在接下来的运行中，内联缓存只需要去比较`Shape`间的差异，如果与之前相同，呢嘛额直接从被缓存的偏移量中读取值。特别的是，当JS引擎看到一个对象的`shape`是之前IC所记录的，它就不再需要去触及属性信息 - 而是完全跳过这些昂贵的属性信息查找。这相对于每次寻找属性来说有明显的加速。

### 高效存储数组

对于数组来说，储存属性即数组索引是很平常的。每个属性的值被称为数组元素。存储单个数组中的每个元素是十分浪费内存的。取而代之的，JS引擎根据这个现实使得数组索引类型的属性默认是可修改的，可枚举的，并且可以可配置的，并将数组元素和其他命名属性分开存储。

思考下面的数组：

```javascript
const array = [
	'#jsconfeu',
];
```

引擎存储了数组的长度(`1`)，然后指向一个含有偏移量和`length`属性的`shape`

![22](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/22.svg)

这和我们之前见过的情况类似....但是数组中的数值存在哪里呢？

![23](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/23.svg)

每个数组都有一个分开的`element backing store`来存储所有数组索引的属性值。JS引擎不需要存储数组元素的任何属性，因为通常来说他们都是可写，可枚举以及可配置的。

那如果遇到不寻常的情况会怎么样呢？ 当你改变了数组元素的属性时？

```javascript
// Please don’t ever do this!
const array = Object.defineProperty(
	[],
	'0',
	{
		value: 'Oh noes!!1',
		writable: false,
		enumerable: false,
		configurable: false,
	}
);
```

上面的片段定义了一个名为`0`的属性（恰巧是一个数组的索引），但是将他的属性改变，与默认值不同。

在这样的边缘例子中，JS引擎将会把`element backing store`表示为由数组索引隐射到属性特性的字典。

![24](https://github.com/RogerZZZZZ/V8-blog/raw/master/Shapes-and-Inline-Caches/img/24.svg)

即使当单个数组元素没有默认属性时，整个数组的`backing store`将会变得很慢，并进入一个不高效的模式。**避免在数组索引上使用`Object.defineProperty`**(我也不确定为什么你会想这么做，这件事看起来很奇怪并且也没有什么用)。

### 补充

我们学习了JS引擎是如何存储对象和数组的，以及`Shape`和内联缓存是如何帮助优化常规操作的。根据这些知识，我们可以确定一些可以帮助我们提高JS编码性能的技巧：

- 始终以相同的方式去初始化对象，以保证它们不会最终拥有不同的`Shape`
- 不要去混淆数组元素的属性，这样的话你将会更加高效地储存和使用它们。

## DONE                                                                                                                       