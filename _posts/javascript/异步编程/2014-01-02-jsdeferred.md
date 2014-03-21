---
layout: post
title: JavaScript 异步编程 之 jsDeferred库 源码分析
description:
category: javascript
tag:  [deferred,Promise,jsDeferred]
---

转载请注明出处-[本文链接](/2014/01/jsdeferred/)

[jsDeferred][jsDeferred] 一个构思非常惊人的异步流程控制库，发出来分享下
整个代码非常简洁，易用，不过呢是小日本写的东西…

加载jsdeferred定义延迟对象。为方便起见，我们用Deferred.define()方法把接口导出到全局作用于中


```js
Deferred.define(); //全局暴露形式

//此时可全局使用 Deferred 中的 methed方法
//有["parallel", "wait", "next", "call", "loop", "repeat", "chain"]; 这些方法
next(function () {
    console.log("Hello!");
    return wait(5);
}).
next(function () {
    console.log("World!");
});


```

<!--more-->

这个流程中，开始会打印出 “Hello”,然后过5秒接着会打印出 “world”


通过这样做，你就能使用如  parallel() ,wait(), next(),  call(), loop(), repeat(),and chain(),这样的全局函数方法，让我们试用一下这些异步的流程

```js
//直接使用变量Deferred 来执行 效果与上面的一样 //但没有污染全局环境

Deferred.next(function () {
    console.log("Hello!");
    return Deferred.wait(5);
}).
next(function () {
    console.log("World!");
});
```


所以用`Deferred.define()`方法，无疑污染了全局作用域，入侵性太强了，跟mootools,prototype一样, 不过好处嘛，很明显，简单易用了。

不过JSDeferred也提供了无侵入的写法 来保持全局的干净即传入一个变量 来做到将所有的方法 都挂到此变量上

看一个其实现的源码 便很容易发现 见源码第`768`行


```js
/**
 * Export functions to obj.
 * @param {Object} obj A object which this method should export to
 * @param {Array.<string>=} list List of function names (default Deferred.methods)
 * @return {function():Deferred} The Deferred constructor function
 */
Deferred.define = function (obj, list) {
    //如果不传list参数 则将deferred中methods的所有方法挂在obj上
    if (!list) list = Deferred.methods;

    //如果不传参数obj就会当成全局变量
    if (!obj)  obj  = (function getGlobal () { return this })();
    for (var i = 0; i < list.length; i++) {
        //将deferred中methods 全部挂在obj下
        var n = list[i];//
        obj[n] = Deferred[n];//
    }
    return Deferred;
};


```

所以使用的时候 可以传入一个变量 用作上下文

```js
var o = {}; //定义一个对象
Deferred.define(o);//把Deferred的方法加持到它上面，让o成为一个Deferred子类。
o.next(function(){
  /* 处理 */
})

```

另再来看一个`next`方法

1. `next`方法是可以进行链式操作，链式的原理很简单就是要返回当前的的this上下文才可以
2. 所以很明显第一个next我们必须要建一个上下文对象提供给后面next引用，其实跟`jquery`链式一个道理

根据源码看来`Deferred.next`其实就是一个静态方法

```js
Deferred.next = Deferred.next_faster_way_readystatechange
    || Deferred.next_faster_way_Image
    || Deferred.next_tick
    || Deferred.next_default;
```
显而易见，next方法是有四种选择优先级，为什么要这样呢？

1. 目的是用于提供了一个JSDeferred实例与实现第一个异步操作。
2. 异步操作在JSDeferred中有许多实现方法，如setTimeout，img.onerror或 script.onreadystatechange ，
3. 它会视浏览器选择最快的异步方式


我是基于webkit的游览器，所以我们就直接看Deferred.next_faster_way_Image

```js

// Modern Browsers
    var d = new Deferred();
    var img = new Image();
    var handler = function () {
        d.canceller();
        d.call();
    };
    img.addEventListener("load", handler, false);
    img.addEventListener("error", handler, false);
    d.canceller = function () {
        img.removeEventListener("load", handler, false);
        img.removeEventListener("error", handler, false);
    };
    img.src = "data:image/png," + Math.random();
    if (fun) d.callback.ok = fun;
    return d;


```
流程：

1. 创建一个Deferred对象
1. 创建Image对象，实现异步
1. 监听事件
1. if (fun) d.callback.ok = fun;  放置回调处理对象
1. 返回当前deferred对象

由此可见

    next(fn).next(fn).next(fn)
其实就是

    第一个 next()    Deferred.next_faster_way_Image () 返回 d

    第二个 next()   d.next()

第二个next(),其实就是实例Deferred类的原型方法了

具体我们看

```js
next: function (fun) {
        return this._post("ok", fun)
    },

_post: function (okng, fun) {
    this._next = new Deferred();
    this._next.callback[okng] = fun;
    return this._next;
},
```

看到_post方法，就有拨开云雾见月明的感觉了

其实内部会有重新生成一个 Deferred对象挂到父实例的 next上，在绑定回调..

如此依次循环处理

所以初始化的时候其实内部就生成了这么一个队列


_next 上都挂着下一个队列处理

此时都是在初始化准备好的了，在执行的时候 Image 在成功回调中，我们调用了 d.call();

执行实例的call方法


```js
call: function (val) {
        return this._fire("ok", val)
    },
```


```js

_fire: function (okng, value) {
    var next = "ok";
    try {
        value = this.callback[okng].call(this, value);
    } catch (e) {
        next = "ng";
        value = e;
        if (Deferred.onerror) Deferred.onerror(e);
    }
    if (Deferred.isDeferred(value)) {
        value._next = this._next;
    } else {
        if (this._next) this._next._fire(next, value);
    }
    return this;
}

```
1. 取出当前实例的回调方法，传入参数
1. 返回的value 是否还是一个isDeferred对象
1. 如果没有返回值，继续调用内部的_next上的Deferred实例，依次循环


总结：

构思很特别，把每次链式的回调挂到内部的next子属性中，在处理上也保持了一条线的引用关系，而不是常规的用数组的方式存起来

当然上面仅仅只是同步方法的处理分析，也不是标准的遵循CommonJS Promises规范去抒写的

转载请注明出处-[本文链接](/2014/01/jsdeferred/)

==========

参考文章:

1. [JSDeferred 源码分析-by Aaron][JSDeferred 源码分析-by Aaron]
2. [JSDeferred][jsDeferred]


[JSDeferred 源码分析-by Aaron]: http://www.cnblogs.com/aaronjs/p/3247302.html

[jsDeferred]:http://cho45.stfuawsc.com/jsdeferred/


