# RP入门

作者：[@andrestaltz](https://twitter.com/andrestaltz)

翻译：[@benjycui](https://github.com/benjycui)、[@jsenjoy](https://github.com/jsenjoy)

> 作者在[原文](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)后回答了不少人的疑惑，推荐一看。

> 在翻译时，术语我尽量不翻译，就算翻译了也会给出原文以作对照。因为就个人观察的情况而言，术语翻译难以统一，不同的译者会把同一个概念翻译成不同的版本，最终只会让读者困惑。而且术语往往就几个单词，记起来也不难。

> 作者在回复中提了一下FRP与RP是不同的，同时建议把这份教程中的FRP都替换为RP，所以译文就把FRP都替换为RP了。

很明显你有兴趣学习这种被称作RP(Reactive Programming)的新技术，特别是它对应的实现如：Rx、Bacon.js、RAC等。

学习RP是很困难的一个过程，特别是在缺乏优秀资料的前提下。刚开始学习时，我试过去找一些教程，并找到了为数不多的实用教程，但是它们都流于表面，从没有围绕RP构建起一个完整的知识体系。库的文档往往也无法帮助你去了解它的函数。不信的话可以看一下这个：

> **Rx.Observable.prototype.flatMapLatest(selector, [thisArg])**

> ！@#￥%……&*

尼玛。

我看过两本书，一本只是讲述了一些概念，而另一本则纠结于如何使用RP库。我最终放弃了这种痛苦的学习方式，决定在开发中一边使用RP，一边理解它。在[Futurice](https://www.futurice.com)工作期间，我尝试在真实项目中使用RP，并且当我遇到困难时，得到了[同事们的帮助](http://blog.futurice.com/top-7-tips-for-rxjava-on-android)。

在学习过程中最困难的一部分是 **以RP的方式思考**。这意味着要放弃命令式且带状态的(imperative and stateful)编程习惯，并且要强迫你的大脑以一种不同的方式去工作。在互联网上我找不到任何关于这方面的教程，而我觉得这世界需要一份关于怎么以RP的方式思考的实用教程，这样你就有足够的资料去起步。库的文档无法为你的学习提供指引，而我希望这篇文章可以。

## 什么是RP?

在互联网上有着一大堆糟糕的解释与定义。[维基百科](https://en.wikipedia.org/wiki/Functional_reactive_programming)一如既往的空泛与理论化。[Stackoverflow](http://stackoverflow.com/questions/1028250/what-is-functional-reactive-programming)的权威答案明显不适合初学者。[Reactive Manifesto](http://www.reactivemanifesto.org/)看起来是你展示给你公司的项目经理或者老板们看的东西。微软的[Rx terminology](https://rx.codeplex.com/) "Rx = Observables + LINQ + Schedulers" 过于重量级且微软味十足，只会让大部分人困惑。相对于你所使用的MV*框架以及钟爱的编程语言，"Reactive"和"propagation of change"这些术语并没有传达任何有意义的概念。框架的Views层当然要对Models层作出反应，数据的改变当然会传播开来(propagate)。如果没有这些，就没有东西会被渲染了。

所以不要再扯这些废话了。

### RP是使用异步数据流进行编程

一方面，这并不是什么新东西。Event buses或者click events本质上就是异步事件流(asynchronous event stream)，你可以监听并处理这些事件。RP的思路大概如下：你使用任何东西创建data stream，而不仅仅是click或hover事件(原文："FRP is that idea on steroids. You are able to create data streams of anything, not just from click and hover events.")。Stream廉价且常见，任何东西都可以是一个stream：变量、用户输入、属性、Cache、数据结构等等。举个例子，想像一下你的Twitter feed就像是click events那样的data stream，你可以监听它并相应的作出响应。

**在这个基础上，你还有令人惊艳的函数去combine、create、filter这些Stream。**这就是函数式(Functional)魔法的用武之地。Stream能接受一个，甚至多个stream为输入。你可以_merge_两个stream，也可以从一个stream中_filter_出你感兴趣的events以生成一个新的stream，还可以把一个stream中的value_map_到一个新的stream中。

既然stream在RP中如此重要，那么我们就应该好好的了解它们，就从我们熟悉的"点击一个按钮"这个event stream开始。

![Click event stream](http://i.imgur.com/cL4MOsS.png)

Stream就是一个 **按时间排序的Events序列** ，它可以emit三种不同的events：(某种类型的)value、error或者一个"completed" 的信号。举一个"completed"的例子，当包含这个按钮(指上面点击一个按钮例子中的按钮)的window或者view被关闭时，"completed"便会发生。

通过分别为value、error、"completed"定义事件处理函数，我们将会异步地捕获这些events。有时可以忽略error与"completed"，你只需要定义value的事件处理函数就行。监听一个stream也被称作是 **订阅(Subscribing)**，而我们所定义的函数就是观察者(Observer)，stream则是被观察者(Observable)，其实就是[观察者模式(Observer Design Pattern)](https://en.wikipedia.org/wiki/Observer_pattern)。

上面的示意图也可以使用ASCII重画为下图，在下面的部分教程中我们会使用这幅图：

```
--a---b-c---d---X---|->

a, b, c, d are emitted values
X is an error
| is the 'completed' signal
---> is the timeline
```

既然已经开始对RP感到熟悉，为了不让你觉得无聊，我们可以尝试做一些新东西：我们将会把一个click event stream转为另一个的click event stream。

首先，让我们做一个能记录一个按钮被点击了多少次的计数器stream。在常见的RP库中，每个stream都会有多个方法，`map`、`filter`、`scan`等等。当你调用其中一个方法时，例如`clickStream.map(f)`，它就会基于原来的click stream返回一个 **新的stream**。它不会对原来的click steam作任何修改。这个特性就是 **不可变性(Immutability)**，它之于RP stream，就如果酱之于薄饼。我们也可以对方法进行链式调用如`clickStream.map(f).scan(g)`：

```
  clickStream: ---c----c--c----c------c-->
               vvvvv map(c becomes 1) vvvv
               ---1----1--1----1------1-->
               vvvvvvvvv scan(+) vvvvvvvvv
counterStream: ---1----2--3----4------5-->
```

`map(f)`会根据你提供的`f`函数把原stream中的value分别映射到新的stream中。在我们的例子中，我们把每一次点击都映射为数字1。`scan(g)`会根据你提供的`g`函数把stream中的所有value聚合成一个新的value -- `x = g(accumulated, current)`，这个示例中`g`只是一个简单的add函数。然后，每点击一次，`counterStream`就会把点击的总次数发给它的观察者。

为了展示RP真正的实力，让我们假设你想得到一个包含双击事件的stream。如果我们想要的这个stream要同时考虑三击(triple clicks)，或者更加宽泛，连击(multiple clicks)的话，那便更加有趣了。深呼吸一下，然后想像一下在传统的命令式且带状态的方式中你会怎么实现。我敢打赌代码会像一堆乱麻，并且会使用一些的变量保存状态，同时也有一些计算时间间隔的代码。

而在RP中，这个功能的实现就非常简单。事实上，这逻辑只有[4行代码](http://jsfiddle.net/staltz/4gGgs/27/)。但现在我们先不管那些代码。用图表的方式思考是理解怎样构建Stream的最好方法，无论你是初学者还是专家。

![Multiple clicks stream](http://i.imgur.com/HMGWNO5.png)

灰色的方框是用来转换stream的函数。首先，我们把时间间隔大于250ms的Click都放进一个列表(原文："First we accumulate clicks in lists, whenever 250 milliseconds of "event silence" has happened." ) -- 简单来说就是`buffer(stream.throttle(250ms))`做的事，不要在意这些细节，我们只是展示一下RP而已。结果是一个列表的stream，然后我们使用`map()`把每个列表映射为一个整数，即它的长度。最终，我们使用`filter(x >= 2)`把整数`1`给过滤掉。就这样，3个操作就生成了我们想要的stream。然后我们就可以订阅(监听)这个stream，并以我们所希望的方式作出反应。

我希望你能感受到这个示例的优美之处。这个示例只是冰山一角：你可以把同样的操作应用到不同种类的stream上，例如，一个API响应的stream；另外，RP还有很多其它可用的函数。

## 为什么我要使用RP

RP提高了代码的抽象层级，所以你可以只关注定义了业务逻辑的那些相互依赖的事件，而非纠缠于大量的实现细节。RP的代码往往会更加简明。

特别是在开发现在这些有着大量与Data events相关的UI events的高互动性Webapps、mobile apps的时候，RP的优势将更加明显。10年前，网页的交互就只是提交一个很长的表单到后端，而前端只有简单的渲染。Apps就表现得更加的实时了：修改一个表单域就能自动地把修改后的值保存到后端，为一些内容"点赞"时，会实时的反应到其它在线用户那里等等。

现在的Apps有着大量各种各样的实时events，以给用户提供一个交互性较高的体验。我们需要工具去应对这个变化，而RP就是一个答案。

## 以RP方式思考的例子

下面让我们接触实实在在的代码。它是一个可运行的，非虚构的例子，也没有只解释了一半的概念。它将一步一步地指导我们以RP的方式思考。学完教程之后，我们将写出真实可用的代码，并做到知其然，知其所以然。

在这个教程中，我将会使用 **JavaScript** 和 **[RxJS](https://github.com/Reactive-Extensions/RxJS)**，因为JavaScript是现在最多人会的语言，而[Rx* 库](https://rx.codeplex.com/)有多种语言版本，并支持多种平台([.NET](https://rx.codeplex.com/), [Java](https://github.com/Netflix/RxJava), [Scala](https://github.com/Netflix/RxJava/tree/master/language-adaptors/rxjava-scala), [Clojure](https://github.com/Netflix/RxJava/tree/master/language-adaptors/rxjava-clojure),  [JavaScript](https://github.com/Reactive-Extensions/RxJS), [Ruby](https://github.com/Reactive-Extensions/Rx.rb), [Python](https://github.com/Reactive-Extensions/RxPy), [C++](https://github.com/Reactive-Extensions/RxCpp), [Objective-C/Cocoa](https://github.com/ReactiveCocoa/ReactiveCocoa), [Groovy](https://github.com/Netflix/RxJava/tree/master/language-adaptors/rxjava-groovy), 等等)。所以，无论你用的是什么语言、库，你都能从下面这个教程中学到东西。

## 实现"Who to follow"推荐界面

在Twitter上，这个界面看起来是这样的：

![Twitter Who to follow suggestions box](http://i.imgur.com/eAlNb0j.png)

我们将会重点模拟它的核心功能，如下：

* 启动时从API那里加载帐户数据，并显示3个推荐
* 点击"Refresh"时，加载另外3个推荐用户到这三行中
* 点击帐号所在行的'x'按钮时，清除那个推荐然后显示一个新的推荐
* 每行都会显示帐号的头像，以及他们主页的链接

我们可以忽略其它的特性和按钮，因为它们是次要的。同时，因为Twitter最近关闭了对非授权用户的API，我们将会为Github实现这个推荐界面，而非Twitter。这是[Github获取用户的API](https://developer.github.com/v3/users/#get-all-users)。

如果你想先看一下最终效果，这里有完成后的代码 http://jsfiddle.net/staltz/8jFJH/48/ 。

## Request与Response

**在Rx中你该怎么处理这个问题呢？** 好吧，首先，(几乎)_所有的东西都可以转为一个stream_。这就是Rx的咒语。让我们先从最简单的特性开始："在启动时，从API加载3个帐号的数据"。这并没有什么特别，就只是简单的3个步骤：    
1. 发出一个请求，    
2. 收到一个响应，    
3. 渲染这个响应。  
让我们继续，用stream模型作为我们的请求。一开始可能会觉得杀鸡用牛刀，但我们应当从最基本的开始，是吧？

在启动的时候，我们只需要发出一个请求，所以如果我们把它转为一个data stream的话，那就是一个只有一个value的stream。稍后，我们知道将会有多个请求发生，但现在，就只有一个请求。

```
--a------|->

a是一个String 'https://api.github.com/users'
```

这是一个包含了我们想向其发出请求的URL的stream。每当一个请求事件发生时，它会告诉我们两件事："什么时候"与"什么东西"。"什么时候"这个请求会被执行，就是什么时候这个event会被emit。"什么东西"会被请求，就是这个emit出来的value：一个包含URL的字符串。

在RX*中，创建只有一个value的stream是非常简单的。官方把一个stream称作Observable，因为它可以被观察(can be observed => observable)，但是我发现那是个很傻逼的名子，所以我把它叫做_Stream_。

```javascript
var requestStream = Rx.Observable.just('https://api.github.com/users');
```

但是现在，那只是一个由字符串组成的stream，并不会引起其他的操作。当这个value被emit的时候，我们需要以某种方式引起相关的操作。通过[订阅(Subscribing)](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md#rxobservableprototypesubscribeobserver--onnext-onerror-oncompleted)这个stream，便可以完成这个目的。

```javascript
requestStream.subscribe(function(requestUrl) {
  // execute the request
  jQuery.getJSON(requestUrl, function(responseData) {
    // ...
  });
}
```

留意一下我们使用了jQuery的Ajax函数(我们假设你已经知道[它的用途](http://devdocs.io/jquery/jquery.getjson))去发出异步请求。但是等一等，Rx可以用来处理 **异步** data stream，那这个请求的响应就不能当作一个包含了将会到达的数据的stream么？当然，从理论上来讲，应该是可以的，所以我们尝试一下。

```javascript
requestStream.subscribe(function(requestUrl) {
  // execute the request
  var responseStream = Rx.Observable.create(function (observer) {
    jQuery.getJSON(requestUrl)
    .done(function(response) { observer.onNext(response); })
    .fail(function(jqXHR, status, error) { observer.onError(error); })
    .always(function() { observer.onCompleted(); });
  });

  responseStream.subscribe(function(response) {
    // do something with the response
  });
}
```

[`Rx.Observable.create()`](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md#rxobservablecreatesubscribe)所做的事就是通过显式的通知每一个Observer(或者说是Subscriber) data events(`onNext()`)或者errors (`onError()`)来创建你自己的stream。而我们所做的就只是把jQuery Ajax Promise包装起来而已。**打扰一下，这意味者Promise本质上就是一个Observable？**

&nbsp;
&nbsp;
&nbsp;
&nbsp;
&nbsp;

![Amazed](http://www.myfacewhen.net/uploads/3324-amazed-face.gif)

Yes.

Observable就是Promise++。在Rx中，你可以用`var stream = Rx.Observable.fromPromise(promise)`轻易的把一个Promise转为Observable，所以我们就这样做吧。唯一的不同就是Observable并不遵循[Promises/A+](http://promises-aplus.github.io/promises-spec/)，但概念上没有冲突。Promise就是只有一个Value的Observable。Rx Stream比Promise更进一步的是允许返回多个value。

这样非常不错，并展现了Observables至少有Promise那么强大。所以如果你相信Promise宣传的那些东西，那么也请留意一下Rx Observables能胜任些什么。

现在回到我们的例子，如果你已经注意到了我们在`subscribe()`内又调用了另外一个`subscribe()`，这类似于Callback hell。同样，你应该也注意到`responseStream`是建立在`requestStream`之上的。就像你之前了解到的那样，在Rx内有简单的机制可以从其它Stream中转换并创建出新的stream，所以我们也应该这样做。

你现在需要知道的一个基本的函数是[`map(f)`](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md#rxobservableprototypemapselector-thisarg)，它分别把`f()`应用到stream A中的每一个value，并把返回的value放进stream B里。如果我们也对request stream与response stream进行同样的处理，我们可以把request URL映射为response Promise(而Promise可以转为Streams)。

```javascript
var responseMetastream = requestStream
  .map(function(requestUrl) {
    return Rx.Observable.fromPromise(jQuery.getJSON(requestUrl));
  });
```

然后，我们将会创造一个叫做"_Metastream_"的怪物：包含stream的stream。暂时不需要害怕。Metastream就是emit的每个value都是stream的stream。你可以把它想像为[指针(Pointer)](https://en.wikipedia.org/wiki/Pointer_(computer_programming))：每个value都是一个指向其它stream的指针。在我们的例子里，每个request URL都会被映射为一个指向包含响应Promise stream的指针。

![Response metastream](http://i.imgur.com/HHnmlac.png)

Response的Metastream看起来会让人困惑，看起来也没有什么帮助。我们只想要一个简单的response stream，它返回的Value应该是JSON而不是一个JSON对象的"Promise"。是时候介绍[Mr. Flatmap](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md#rxobservableprototypeflatmapselector-resultselector)了：它是`map()`的一个版本，用来"flatten" Metastream。就是说，通过把作用在"trunk" stream上的所有操作都应用到"branch" stream上。Flatmap不是用来"fix" Metastream的，因为Metastream并不是一个bug，这只是一些用来处理Rx中的异步响应(asynchronous response)的工具。

```javascript
var responseStream = requestStream
  .flatMap(function(requestUrl) {
    return Rx.Observable.fromPromise(jQuery.getJSON(requestUrl));
  });
```

![Response stream](http://i.imgur.com/Hi3zNzJ.png)

很好。因为response stream是根据request stream定义的，所以**如果**我们后面在request stream上发起更多的请求的话，在response stream上我们将会得到相应的response event，就像预期的那样：

```
requestStream:  --a-----b--c------------|->
responseStream: -----A--------B-----C---|->

(小写字母是一个request，大写字母是对应的response)
```

现在，我们终于有了一个response stream，所以可以把收到的数据渲染出来了：

```javascript
responseStream.subscribe(function(response) {
  // render `response` to the DOM however you wish
});
```

把目前为止所有的代码放到一起就是这样：

```javascript
var requestStream = Rx.Observable.just('https://api.github.com/users');

var responseStream = requestStream
  .flatMap(function(requestUrl) {
    return Rx.Observable.fromPromise(jQuery.getJSON(requestUrl));
  });

responseStream.subscribe(function(response) {
  // render `response` to the DOM however you wish
});
```

## Refresh按钮

我之前并没有提到返回的JSON是一个有着100个用户数据的列表。因为这个API只允许我们设置偏移量，而无法设置返回的用户数，所以我们现在是只用了3个用户的数据而浪费了另外97个的数据。这个问题暂时可以忽略，稍后我们会学习怎么缓存这些数据。

每点击一次refresh按钮，request stream便应emit一个新的URL，这样我们就能得到一个新的response。我们需要两样东西来达成这项需求：    
1. 由refresh按钮上click events组成的stream(咒语：一切皆stream)；    
2. Request stream也应该改为由refresh click stream所生成的stream。幸运的是，RxJS提供了从event listener生成Observable的函数。

```javascript
var refreshButton = document.querySelector('.refresh');
var refreshClickStream = Rx.Observable.fromEvent(refreshButton, 'click');
```

既然refresh click event本身并没有提供任何要请求的API URL，我们需要把每一次的点击都映射为一个URL。现在，我们把refresh click stream映射为新的request stream，其中每一次点击都分别映射为对API endpoint请求一个随机的偏移量。

```javascript
var requestStream = refreshClickStream
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  });
```

因为我比较笨并且也没有使用自动化测试，所以我刚把之前做好的一个特性搞烂了。现在在启动时不会再发出任何的request，而只有在点击refresh按钮时才会。额...但是这两个需求我都必须满足：无论是点击refresh按钮时还是刚打开页面时都该发出一个request。

我们知道怎么分别为这两种情况生成stream：

```javascript
var requestOnRefreshStream = refreshClickStream
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  });

var startupRequestStream = Rx.Observable.just('https://api.github.com/users');
```

但怎样才能把这两个"合成(merge)"一个呢？好吧，我们有[`merge()`](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md#rxobservableprototypemergemaxconcurrent--other)这个函数，参考下面这个图解：

```
stream A: ---a--------e-----o----->
stream B: -----B---C-----D-------->
          vvvvvvvvv merge vvvvvvvvv
          ---a-B---C--e--D--o----->
```

这样就简单了：

```javascript
var requestOnRefreshStream = refreshClickStream
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  });

var startupRequestStream = Rx.Observable.just('https://api.github.com/users');

var requestStream = Rx.Observable.merge(
  requestOnRefreshStream, startupRequestStream
);
```

还有一个更加干净的可选方案，不需要使用中间变量。

```javascript
var requestStream = refreshClickStream
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  })
  .merge(Rx.Observable.just('https://api.github.com/users'));
```

甚至可以更短，更具有可读性：

```javascript
var requestStream = refreshClickStream
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  })
  .startWith('https://api.github.com/users');
```

[`startWith()`](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md#rxobservableprototypestartwithscheduler-args)函数做的事和你预期的完全一样。无论你输入的怎样的stream，`startWith(x)`输出的stream一开始都是`x`。但是还不够[DRY](https://en.wikipedia.org/wiki/Don't_repeat_yourself)，我重复了API URL。一个改进的方法是移掉`refreshClickStream`最后的`startWith()`，并在一开始的时候模拟点击一次refresh按钮。

```javascript
var requestStream = refreshClickStream.startWith('startup click')
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  });
```

很好。如果你把之前我"搞烂了的版本"的代码和现在的相比，就会发现唯一的不同是加了`startWith()`函数。

## 用stream构建三个推荐

到现在为止，我们只是谈及了这个 _推荐_ UI元素在responeStream的`subscribe()`内执行的渲染步骤。对于refresh按钮，我们还有一个问题：当你点击`refresh`时，当前存在的三个推荐并不会被清除。新的推荐会在response到达后出现，为了让UI看起来舒服一些，当点击刷新时，我们需要清理掉当前的推荐。

```javascript
refreshClickStream.subscribe(function() {
  // clear the 3 suggestion DOM elements
});
```

不，别那么快，朋友。这样不好，我们现在有 **两个** subscriber会影响到推荐的DOM元素(另外一个是`responseStream.subscribe()`)，而且这样完全不符合[separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns)。还记得RP的咒语么？

&nbsp;
&nbsp;
&nbsp;
&nbsp;

![Mantra](http://i.imgur.com/AIimQ8C.jpg)

所以让我们把一条推荐模型设计成emit的value为一条推荐内容的JSON对象的stream。其余的两条推荐，我们也以同样的方法处理。第一条推荐现在看起来是这样的：

```javascript
var suggestion1Stream = responseStream
  .map(function(listUsers) {
    // get one random user from the list
    return listUsers[Math.floor(Math.random()*listUsers.length)];
  });
```

对于`suggestion2Stream`和`suggestion3Stream`，我们可以简单地拷贝`suggestion·Stream`的代码来使用。这不是DRY，但是它会让我们的例子变得更加简单一些。而且我觉得这是一个良好实践，它让我们思考如何去减少重复代码。

我们不在responseStream的subscribe()中处理渲染了，我们这么处理：

```javascript
suggestion1Stream.subscribe(function(suggestion) {
  // render the 1st suggestion to the DOM
});
```
回到"当刷新时，清理掉当前的推荐"，我们可以很简单的把刷新点击映射为`null`(即没有推荐数据)，并且在`suggestion1Stream`中包含进来，如下：

```javascript
var suggestion1Stream = responseStream
  .map(function(listUsers) {
    // get one random user from the list
    return listUsers[Math.floor(Math.random()*listUsers.length)];
  })
  .merge(
    refreshClickStream.map(function(){ return null; })
  );
```

当渲染时，`null`解释为"没有数据"，所以把UI元素隐藏起来。

```javascript
suggestion1Stream.subscribe(function(suggestion) {
  if (suggestion === null) {
    // hide the first suggestion DOM element
  }
  else {
    // show the first suggestion DOM element
    // and render the data
  }
});
```

现在的示意图：

```
refreshClickStream: ----------o--------o---->
     requestStream: -r--------r--------r---->
    responseStream: ----R---------R------R-->
 suggestion1Stream: ----s-----N---s----N-s-->
 suggestion2Stream: ----q-----N---q----N-q-->
 suggestion3Stream: ----t-----N---t----N-t-->
```

`N`即代表了`null`

作为一种补充，我们也可以在一开始的时候就渲染空的推荐内容。这通过把`startWith(null)`添加到Suggestion stream就完成了：

```javascript
var suggestion1Stream = responseStream
  .map(function(listUsers) {
    // get one random user from the list
    return listUsers[Math.floor(Math.random()*listUsers.length)];
  })
  .merge(
    refreshClickStream.map(function(){ return null; })
  )
  .startWith(null);
```

现在结果是：

```
refreshClickStream: ----------o---------o---->
     requestStream: -r--------r---------r---->
    responseStream: ----R----------R------R-->
 suggestion1Stream: -N--s-----N----s----N-s-->
 suggestion2Stream: -N--q-----N----q----N-q-->
 suggestion3Stream: -N--t-----N----t----N-t-->
```

## 关闭推荐并缓存Response

还有一个功能需要实现。每一个推荐，都该有自己的"X"按钮以关闭它，然后在该位置加载另一个推荐。最初的想法，是点击任何关闭按钮时都需要发起一个新的请求：

```javascript
var close1Button = document.querySelector('.close1');
var close1ClickStream = Rx.Observable.fromEvent(close1Button, 'click');
// and the same for close2Button and close3Button

var requestStream = refreshClickStream.startWith('startup click')
  .merge(close1ClickStream) // we added this
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  });
```

以上这段代码并不能起作用。这将会关闭并且重新加载_所有_的推荐，而不是仅仅处理我们点击的那一个。有多种不同的方法可以解决这个问题。为了让这个问题有趣起来，我们可以通过复用之前的请求来解决它。API返回的response中有100个用户，而我们仅仅使用了其中的3个，所以还有很多的新数据可以使用，无须重新发起请求。

同样的，我们用stream的方式来思考。当点击'close1'时，我们想要从responseStream _最近emit的_ response列表中获取一个随机的用户，如：

```
    requestStream: --r--------------->
   responseStream: ------R----------->
close1ClickStream: ------------c----->
suggestion1Stream: ------s-----s----->
```

在Rx*中，[`combineLatest`](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md#rxobservableprototypecombinelatestargs-resultselector)似乎实现了我们想要的功能。它接受两个stream作为输入（stream A和stream B），当其中一个stream emit一个value时，`combineLatest`便把最近两个emit的value`a`和`b`从各自的Stream中取出并且返回一个`c = f(a,b)`，`f`为你定义的函数。用图来表示便更加清晰了：

```
stream A: --a-----------e--------i-------->
stream B: -----b----c--------d-------q---->
          vvvvvvvv combineLatest(f) vvvvvvv
          ----AB---AC--EC---ED--ID--IQ---->

f是把输入字符转化成大写字母的函数
```

我们可以在`close1ClickStream`和`responseStream`上使用combineLatest()。无论什么时候当一个按钮被点击时，我们可以拿到response最新emit的value，并且在`suggestion1Stream`上产生一个新的value。另一方面，combineLatest()是对称的，当一个新的response 在`responseStream` emit时，它将会把最后的'close1'的点击事件一起合并来产生一个新的推荐。这是有趣的，因为它允许我们把之前的`suggestion1Stream`代码简化成下边这个样子：

```javascript
var suggestion1Stream = close1ClickStream
  .combineLatest(responseStream,
    function(click, listUsers) {
      return listUsers[Math.floor(Math.random()*listUsers.length)];
    }
  )
  .merge(
    refreshClickStream.map(function(){ return null; })
  )
  .startWith(null);
```

还有一个问题需要解决。combineLatest()使用最近的两个数据源，但是当其中一个来源没发起任何事件时，combineLatest()无法在output stream中产生一个data event。从上边的ASCII图中，你可以看到，在第一个stream emit `a`这个值时并没有任何输出产生，只有当第二个stream emit `b`时才有值输出。

有多种方法可以解决这个问题，我们选择最简单的一种，一开始在'close 1'按钮上模拟一个点击事件：

```javascript
var suggestion1Stream = close1ClickStream.startWith('startup click') // we added this
  .combineLatest(responseStream,
    function(click, listUsers) {
      return listUsers[Math.floor(Math.random()*listUsers.length)];
    }
  )
  .merge(
    refreshClickStream.map(function(){ return null; })
  )
  .startWith(null);
```

## 总结

终于完成了，所有的代码合在一起是这样子：

```javascript
var refreshButton = document.querySelector('.refresh');
var refreshClickStream = Rx.Observable.fromEvent(refreshButton, 'click');

var closeButton1 = document.querySelector('.close1');
var close1ClickStream = Rx.Observable.fromEvent(closeButton1, 'click');
// and the same logic for close2 and close3

var requestStream = refreshClickStream.startWith('startup click')
  .map(function() {
    var randomOffset = Math.floor(Math.random()*500);
    return 'https://api.github.com/users?since=' + randomOffset;
  });

var responseStream = requestStream
  .flatMap(function (requestUrl) {
    return Rx.Observable.fromPromise($.ajax({url: requestUrl}));
  });

var suggestion1Stream = close1ClickStream.startWith('startup click')
  .combineLatest(responseStream,
    function(click, listUsers) {
      return listUsers[Math.floor(Math.random()*listUsers.length)];
    }
  )
  .merge(
    refreshClickStream.map(function(){ return null; })
  )
  .startWith(null);
// and the same logic for suggestion2Stream and suggestion3Stream

suggestion1Stream.subscribe(function(suggestion) {
  if (suggestion === null) {
    // hide the first suggestion DOM element
  }
  else {
    // show the first suggestion DOM element
    // and render the data
  }
});
```

**你可以查看这个最终效果 http://jsfiddle.net/staltz/8jFJH/48/**

这段代码虽然短小，但实现了不少功能：它适当的使用separation of concerns实现了对multiple events的管理，甚至缓存了响应。函数式的风格让代码看起来更加declarative而非imperative：我们并非给出一组指令去执行，而是通过定义stream之间的关系 **定义这是什么**。举个例子，我们使用Rx告诉计算机 _`suggestion1Stream` **是** 由 'close 1' Stream与最新响应中的一个用户合并(combine)而来，在程序刚运行或者刷新时则是`null`_。

留意一下代码中并没有出现如`if`、`for`、`while`这样的控制语句，或者一般JavaScript应用中典型的基于回调的控制流。如果你想使用`filter()`，上面的`subscribe()`中甚至可以不用`if`、`else`(实现细节留给读者作为练习)。在Rx中，我们有着像`map`、`filter`、`scan`、`merge`、`combineLatest`、`startWith`这样的stream函数，甚至更多类似的函数去控制一个事件驱动(event-driven)的程序。这个工具集让你可以用更少的代码实现更多的功能。

## 下一步

如果你觉得Rx*会成为你首选的RP库，花点时间去熟悉这个[函数列表](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md)，包括了如何转换(transform)、合并(combine)、以及创建Observable。如果你想通过图表去理解这些函数，看一下这份[RxJava's very useful documentation with marble diagrams](https://github.com/Netflix/RxJava/wiki/Creating-Observables)。无论什么时候你遇到问题，画一下这些图，思考一下，看一下这一大串函数，然后继续思考。以我个人经验，这样效果很明显。

一旦你开始使用Rx*去编程，很有必要去理解[Cold vs Hot Observables](https://github.com/Reactive-Extensions/RxJS/blob/master/doc/gettingstarted/creating.md#cold-vs-hot-observables)中的概念。如果忽略了这些，你一不小心就会被它坑了。我提醒过你了。通过学习真正的函数式编程(Funational programming)去提升自己的技能，并熟悉那些会影响到Rx*的问题，比如副作用(side effect)。

但是RP不仅仅有Rx*。还有相对容易理解的[Bacon.js](http://baconjs.github.io/)，它没有Rx*那些怪癖。[Elm Language](http://elm-lang.org/)则以它自己的方式支持RP：它是一门会编译成Javascript + HTML + CSS的FRP **语言**，并有一个[Time travelling debugger](http://debug.elm-lang.org/)。非常NB。

Rx在需要处理大量事件的前端和Apps中非常有用。但它不仅仅能用在客户端，在后端或者与数据库交互时也非常有用。事实上，[RxJava是实现Netflix's API服务器端并发的一个重要组件](http://techblog.netflix.com/2013/02/rxjava-netflix-api.html)。Rx并不是一个只能在某种应用或者语言中使用的framework。它本质上是一个在开发任何event-driven软件中都能使用的编程范式(paradigm)。

如果这份教程能帮到你，[请与更多人分享](https://twitter.com/intent/tweet?original_referer=https%3A%2F%2Fgist.github.com%2Fstaltz%2F868e7e9bc2a7b8c1f754%2F&amp;text=The%20introduction%20to%20Reactive%20Programming%20you%27ve%20been%20missing&amp;tw_p=tweetbutton&amp;url=https%3A%2F%2Fgist.github.com%2Fstaltz%2F868e7e9bc2a7b8c1f754&amp;via=andrestaltz)。
