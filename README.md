# FRP介绍

作者：[@andrestaltz](https://twitter.com/andrestaltz)

翻译：[@benjycui](https://github.com/benjycui)

> [原文](https://github.com/benjycui/introrx-chinese-edition/blob/master/introrx.md)

> 在翻译时，术语我尽量不翻译，就算翻译了也会给出原文以作对照。因为就个人观察的情况而言，术语翻译难以统一，不同的译者会把同一个概念翻译成不同的版本，最终只会让读者困惑。而且术语往往就几个单词，记起来也不难。

很明显你是有兴趣学习这样被称作FRP(Functional Reactive Programming)的新技术才来看这篇文章的。

学习FRP是很困难的一个过程，特别是在缺乏优秀资料的前提下。刚开始学习时，我试过去找一些教程，并找到了为数不多的实用教程，但是它们都流于表面，从没有围绕FRP构建起一个完整的知识体系。库的文档往往也无法帮助你去了解它的函数。不信的话可以看一下这个：

> **Rx.Observable.prototype.flatMapLatest(selector, [thisArg])**

> ！@#￥%……&*

尼玛。

我看过两本书，一本只是讲述了一些概念，而另一本则纠结于如何使用FRP库。我最终放弃了这种痛苦的学习方式，决定在开发中一边使用FRP，一边理解它。在[Futurice](https://www.futurice.com)工作期间，我尝试在真实项目中使用FRP，并且当我遇到困难时，得到了[同事们的帮助](http://blog.futurice.com/top-7-tips-for-rxjava-on-android)。

在学习过程中最困难的一部分是 **以FRP的方式思考**。这意味着要放弃命令式且带状态的(Imperative and stateful)编程习惯，并且要强迫你的大脑以一种不同的方式去工作。在互联网上我找不到任何关于这方面的教程，而我觉得这世界需要一份关于怎么以FRP的方式思考的实用教程，这样你就有足够的资料去起步。库的文档无法为你的学习提供指引，而我希望这篇文章可以。

## 什么是FRP?

在互联网上有着一大堆糟糕的解释与定义。[维基百科](https://en.wikipedia.org/wiki/Functional_reactive_programming)一如既往的空泛与理论化。[Stackoverflow](http://stackoverflow.com/questions/1028250/what-is-functional-reactive-programming)的权威答案明显不适合初学者。[Reactive Manifesto](http://www.reactivemanifesto.org/)看起来是你展示给你公司的项目经理或者老板们看的东西。微软的[Rx terminology](https://rx.codeplex.com/) "Rx = Observables + LINQ + Schedulers" 过于重量级且微软味十足，只会让大部分人困惑。相对于你所使用的MV*框架以及钟爱的编程语言，"Reactive"和"Propagation of change"这些术语并没有传达任何有意义的概念。框架的Views层当然要对Models层作出反应，改变当然会传播(分别对应上文的"Reactive"与"Propagation of change"，意思是这一大堆术语和废话差不多，翻译不好，只能靠备注了)。如果没有这些，就没有东西会被渲染了。

所以不要再扯这些废话了。

### FRP是使用异步数据流进行编程

一方面，这并不是什么新东西。Event buses或者Click events本质上就是异步事件流(Asynchronous event stream)，你可以监听并处理这些事件。FRP的思路大概如下：你可以用包括Click和Hover事件在内的任何东西创建Data stream(原文："FRP is that idea on steroids. You are able to create data streams of anything, not just from click and hover events.")。Stream廉价且常见，任何东西都可以是一个Stream：变量、用户输入、属性、Cache、数据结构等等。举个例子，想像一下你的Twitter feed就像是Click events那样的Data stream，你可以监听它并相应的作出响应。

**在这个基础上，你还有令人惊艳的函数去combine、create、filter这些Stream。**这就是函数式(Functional)魔法的用武之地。Stream能接受一个，甚至多个Stream为输入。你可以_merge_两个Stream，也可以从一个Stream中_filter_出你感兴趣的Events以生成一个新的Stream，还可以把一个Stream中的Data values _map_到一个新的Stream中。

既然Stream在FRP中如此重要，那么我们就应该好好的了解它们，就从我们熟悉的"Clicks on a button" Event stream开始。

![Click event stream](http://i.imgur.com/cL4MOsS.png)

Stream就是一个 **按时间排序的Events(Ongoing events ordered in time)序列** ，它可以emit三种不同的Events：(某种类型的)Value、Error或者一个"Completed" Signal。考虑一下"Completed"发生的时机，例如，当包含这个Button(指上面Clicks on a button"例子中的Button)的Window或者View被关闭时。

通过分别为Value、Error、"Completed"定义事件处理函数，我们将会异步地捕获这些Events。有时可以忽略Error与"Completed"，你只需要定义Value的事件处理函数就行。监听一个Stream也被称作是 **订阅(Subscribing)**，而我们所定义的函数就是观察者(Observer)，Stream则是被观察者(Observable)，其实就是[观察者模式(Observer Design Pattern)](https://en.wikipedia.org/wiki/Observer_pattern)。

上面的示意图也可以使用ASCII重画为下图，在下面的部分教程中我们会使用这幅图：

```
--a---b-c---d---X---|->

a, b, c, d are emitted values
X is an error
| is the 'completed' signal
---> is the timeline
```

既然已经开始对FRP感到熟悉，为了不让你觉得无聊，我们可以尝试做一些新东西：我们将会把一个Click event stream转为新的Click event stream。

首先，让我们做一个能记录一个按钮点击了多少次的计数器Stream。在常见的FRP库中，每个Stream都会有多个方法，`map`、`filter`、`scan`等等。当你调用其中一个方法时，例如`clickStream.map(f)`，它就会基于原来的Click stream返回一个 **新的Stream**。它不会对原来的Click steam作任何修改。这个特性就是 **不可变性(Immutability)**，它之于FRP Stream，就如果汁之于薄煎饼。我们也可以对方法进行链式调用如`clickStream.map(f).scan(g)`：

```
  clickStream: ---c----c--c----c------c-->
               vvvvv map(c becomes 1) vvvv
               ---1----1--1----1------1-->
               vvvvvvvvv scan(+) vvvvvvvvv
counterStream: ---1----2--3----4------5-->
```

`map(f)`会根据你提供的`f`函数把原Stream中的Value分别映射到新的Stream中。在我们的例子中，我们把每一次Click都映射为数字1。`scan(g)`会根据你提供的`g`函数把Stream中的所有Value聚合成一个Value -- `x = g(accumulated, current)`，这个示例中`g`只是一个简单的add函数。然后，每Click一次，`counterStream`就会把点击的总次数发给它的观察者。

为了展示FRP真正的实力，让我们假设你想得到一个包含双击事件的Stream。为了让它更加有趣，假设我们想要的这个Stream要同时考虑三击(Triple clicks)，或者更加宽泛，连击(Multiple clicks)。深呼吸一下，然后想像一下在传统的命令式且带状态的方式中你会怎么实现。我敢打赌代码会像一堆乱麻，并且会使用一些的变量保存状态，同时也有一些计算时间间隔的代码。

而在FRP中，这个功能的实现就非常简单。事实上，这逻辑只有[4行代码](http://jsfiddle.net/staltz/4gGgs/27/)。但现在我们先不管那些代码。用图表的方式思考是理解怎样构建Stream的最好方法，无论你是初学者还是专家。

![Multiple clicks stream](http://i.imgur.com/HMGWNO5.png)

灰色的方框是用来转换Stream的函数。首先，我们把连续250ms内的Click都放进一个列表(原文："First we accumulate clicks in lists, whenever 250 milliseconds of "event silence" has happened." 实在不知道怎么翻译，就按自己的理解写了一下) -- 简单来说就是`buffer(stream.throttle(250ms))`做的事，不要在意这些细节，我们只是展示一下FRP而已。结果是一个列表的Stream，然后我们使用`map()`把每个列表映射为一个整数，即它的长度。最终，我们使用`filter(x >= 2)`把整数`1`给过滤掉。就这样，3个操作就生成了我们想要的Stream。然后我们就可以订阅(监听)这个Stream，并以我们所希望的方式作出反应。

我希望你能感受到这个示例的优美之处。这个示例只是冰山一角：你可以把同样的操作应用到不同种类的Stream上，例如，一个API响应的Stream；另一方面，还有很多其它可用的函数。

## 为什么我要使用FRP

FRP提高了代码的抽象层级，所以你可以只关注定义了业务逻辑的那些相互依赖的事件，而非纠缠于大量的实现细节。FRP的代码往往会更加简明。

特别是在开发现在这些有着大量与Data events相关的UI events的高互动性Webapps、Mobile apps的时候，FRP的优势更加明显。10年前，网页的交互就只是提交一个很长的表单到后端，而前端只有简单的渲染。Apps就表现得更加的实时了：修改一个表单域就能自动地把修改后的值保存到后端，为一些内容"点赞"时，会实时的反应到其它在线用户那里等等。

现在的Apps有着大量各种各样的实时Events，以给用户提供一个交互性较高的体验。我们需要工具去应对这个变化，而FRP就是一个答案。
