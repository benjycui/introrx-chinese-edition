# FRP介绍

作者：[@andrestaltz](https://twitter.com/andrestaltz)

翻译：[@benjycui](https://github.com/benjycui)

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
