---
layout: post
title:  jQuery 之deferred 分析［转］
description:
category: deferred,Promise,jquery
---
原文：http://www.cnblogs.com/aaronjs/p/3356505.html
###构建Deferred对象时候的流程图
jQuery Deferred对象的流程图
![Deferred对象的流程图](http://images.cnitblog.com/blog/329084/201310/07212436-77fb87abe03846cd857da70ab1698c32.jpg)


<!--more-->

###源码解析
因为callback被剥离出去后，整个deferred就显得非常的精简

```js

jQuery.extend({

    Deferred : function(){}

    when : function()

})
//将两个方法挂到jquery上
```

看看2个静态方法内部都干了些什么：

Deferred整体结构：

源码精简了部分代码

```js
Deferred: function( func ) {
        //1. 创建三个$.Callbacks对象，分别表示成功，失败，处理中三种状态
        var tuples = [
                // action, add listener, listener list, final state
                [ "resolve", "done", jQuery.Callbacks("once memory"), "resolved" ],
                [ "reject", "fail", jQuery.Callbacks("once memory"), "rejected" ],
                [ "notify", "progress", jQuery.Callbacks("memory") ]
            ],
            state = "pending",

            //   创建了一个promise对象，具有state、always、then、primise方法
            promise = {
                state: function() {},
                always: function() {},
                then: function( /* fnDone, fnFail, fnProgress */ ) { },
                // Get a promise for this deferred
                // If obj is provided, the promise aspect is added to the object
                promise: function( obj ) {}
            },
            deferred = {};
        jQuery.each( tuples, function( i, tuple ) {
            deferred[ tuple[0] + "With" ] = list.fireWith;
        });
        //扩展primise对象生成最终的Deferred对象
        promise.promise( deferred );
        // all done //返回该对象
        return deferred;
    },
```

1. 显而易见Deferred是个工厂类，返回的是内部构建的deferred对象
1. tuples 创建三个$.Callbacks对象，分别表示成功，失败，处理中三种状态
1. 创建了一个promise对象，具有state、always、then、primise方法
1. 扩展primise对象生成最终的Deferred对象，返回该对象

这里其实就是3个处理,但是有个优化代码的地方,就是把共性的代码给抽象出来,通过动态生成了



具体源码分析:

Deferred自身则围绕这`三个对象` *[resolve,reject,notify]*  ????  进行更高层次的抽象

1. 触发回调函数列表执行(函数名)
1. 添加回调函数（函数名）
1. 回调函数列表（jQuery.Callbacks对象）
1. deferred最终状态（第三组数据除外）

```js
var tuples = [
        // action, add listener, listener list, final state
        [ "resolve", "done", jQuery.Callbacks("once memory"), "resolved" ],
        [ "reject", "fail", jQuery.Callbacks("once memory"), "rejected" ],
        [ "notify", "progress", jQuery.Callbacks("memory") ]
    ],
```

这里抽象出2组阵营：

1组：回调方法/事件订阅 - `tuples[*][1]`
    done，fail，progress

2组：通知方法/事件发布 - `tuples[*][0]`
   resolve，reject，notify，resolveWith，rejectWith，notifyWith


tuples 元素集 其实是把相同有共同特性的代码的给合并成一种结构，然后通过一次处理

```js

jQuery.each( tuples, function( i, tuple ) {
    //tuple= [ "resolve", "done", jQuery.Callbacks("once memory"), "resolved" ]
    var list = tuple[ 2 ],
        stateString = tuple[ 3 ];
    promise[ tuple[1] ] = list.add;  //将回调函数存入
    if ( stateString ) {
        list.add(function() {
            state = stateString;

        // [ reject_list | resolve_list ].disable; progress_list.lock
        }, tuples[ i ^ 1 ][ 2 ].disable, tuples[ 2 ][ 2 ].lock );
    }
    deferred[ tuple[0] ] = function() {
        //with方法
        deferred[ tuple[0] + "With" ]( this === deferred ? promise : this, arguments );
        return this;
    };
    deferred[ tuple[0] + "With" ] = list.fireWith;
});
```


对于tuples的3条数据集是分2部分处理的

####第一部分将回调函数存入

    promise[ tuple[1] ] = list.add;

其实就是给promise赋予3个回调函数

    promise.done = $.Callbacks("once memory").add
    promise.fail = $.Callbacks("once memory").add
    promise.progress = $.Callbacks("memory").add

如果存在deferred 最终状态

原文: 默认会预先向doneList,failList中的list添加三个回调函数
    <small style="color:red">这里的是什么呢???</small>

个人理解: 会预先向done,fail中的 list添加三个回调函数


```js
if ( stateString ) {
    //即这里是 List   ----> Done.list,failList
    list.add(function() {
        // state = [ resolved | rejected ]
        state = stateString;

    // [ reject_list | resolve_list ].disable; progress_list.lock
    }, tuples[ i ^ 1 ][ 2 ].disable, tuples[ 2 ][ 2 ].lock );
}

//这里有个小技巧

//i ^ 1 按位异或运算符

//所以实际上第二个传参数是1、0索引对调了，所以取值是failList.disable与doneList.disable
```


`通过stateString有值这个条件，预先向doneList,failList中的list添加三个回调函数`

分别是:

    doneList : [changeState, failList.disable, processList.lock]
    failList : [changeState, doneList.disable, processList.lock]

* changeState `改变状态的匿名函数`，deferred的状态，分为三种：`pending`(初始状态), `resolved`(解决状态), `rejected`(拒绝状态)
* 不论`deferred`对象最终是`resolve`（还是`reject`），在首先改变对象状态之后，都会`disable`另一个函数列表failList(或者doneList)
* 然后lock processList保持其状态，最后执行剩下的之前done（或者fail）进来的回调函数




所以第一步最终都是围绕这add方法

1. done/fail/是list.add也就是callbacks.add，将回调函数存入回调对象中


####第二部分很简单，给deferred对象扩充6个方法

1. resolve/reject/notify 是 callbacks.fireWith，执行回调函数
2. resolveWith/rejectWith/notifyWith 是 callbacks.fireWith 队列方法引用

最后合并promise到deferred

    promise.promise( deferred );
    jQuery.extend( obj, promise )

所以最终通过工厂方法Deferred构建的异步对象带的所有的方法了


return 内部的deferred对象了
![deferred对象的方法](http://images.cnitblog.com/blog/329084/201310/03131510-2375bbda3f2f4524a53e5c5df5eccda0.png)

由此可见我们在

    var defer = $.Deferred(); //构建异步对象

的时候,内部的对象就有了4个属性方法了

* deferred: Object
  1. always: function () {}
  2. done: function () {}
  3. fail: function () {}
  4.  notify: function () {}
  5.  notifyWith: function ( context, args ) {}
  6.  pipe: function ( /* fnDone, fnFail, fnProgress */ ) {}
  7.  progress: function () {}
  8.  promise: function ( obj ) {}
  9.  reject: function () {}
  10.  rejectWith: function ( context, args ) {}
  11.  resolve: function () {}
  12.  resolveWith: function ( context, args ) {}
  13.  state: function () {}
  14.  then: function ( /* fnDone, fnFail, fnProgress */ ) {}

* promise: Object
   1. always: function () {}
   2. done: function () {}
   3. fail: function () {}
   4. pipe: function ( /* fnDone, fnFail, fnProgress */ ) {}
   5. progress: function () {}
   6. promise: function ( obj ) {}
   7. state: function () {}
   8. then: function ( /* fnDone, fnFail, fnProgress */ ) {}

* state: "pending"

* tuples: Array[3]

![构造图](http://images.cnitblog.com/blog/329084/201310/07212451-50defafec7bc468f817adb82476c9bb9.jpg)

以上只是在初始化构建的时候，我们往下看看动态执行时候的处理


###执行期

一个最简单的demo为例子

```js
 var d = $.Deferred();

 setTimeout(function(){
        d.resolve(22)
  },0);

 d.then(function(val){
      console.log(val);
 })
```

当延迟对象被 `resolved` 时，任何通过 `deferred.then`或d`eferred.done` 添加的 `doneCallbacks`，都会被调用。回调函数的执行顺序和它们被添加的顺序是一样的。传递给 `deferred.resolve() `的 args 参数，会传给每个回调函数。当延迟对象进入 resolved 状态后，再添加的任何 `doneCallbacks`，当它们被添加时，就会被立刻执行，并带上传入给 `.resolve()`的参数


换句话说，我们调用d.resolve(22) 就等于是调用

匿名函数并传入参数值 22

```js
function(val){
      console.log(val); //22
 }
```

当前实际的使用中会有各种复杂的组合情况，但是整的外部调用流程就是这样的

###resolve的实现

我们回顾下，其实Deferred对象，内部的实现还是Callbacks对象，只是在外面再封装了一层API，供接口调用


```js
d.resolve(22)

```

实际上调用的就是通过这个代码生成的

```js
//此时deferred[ tuple[0] ] 相当于 deferred.resolve(22)
deferred[ tuple[0] ] = function() {
    //执行 deferred.resolveWidth(promise) //执行了list.fireWith(promise, 22);
    deferred[ tuple[0] + "With" ]( this === deferred ? promise : this, arguments );
    return this;
};
deferred[ tuple[0] + "With" ] = list.fireWith;
```

最终相当于执行

    callbacks.fireWith() // //所以此时的的执行 在callbacks.add()添加的函数

所以最终又回到回调对象`callbacks`中的私有方法`fire()`了


Callbacks会通过`callbacks.add()` 把回调函数给注册到内部的`list = []`上,我们回来过看看
`deferred.then()`

```js
    //
    d.then(function(val){
          console.log(val);
     })
```
###再来看一下then的实现

```js
then: function( /* fnDone, fnFail, fnProgress */ ) {
    var fns = arguments;
    //1.递归jQuery.Deferred
    return jQuery.Deferred(function( newDefer ) {
        jQuery.each( tuples, function( i, tuple ) {
            var action = tuple[ 0 ],
                fn = jQuery.isFunction( fns[ i ] ) && fns[ i ];
            // deferred[ done | fail | progress ] for forwarding actions to newDefer
            deferred[ tuple[1] ](function() {
                    //传递了func 能过作用域找到
                   //省略............
            });
        });
        fns = null;
    }).promise(); //链式调用了promise()
},
```
分三步

1. 递归jQuery.Deferred
1. 传递了func
1. 链式调用了promise()




因为在异步对象的方法都是嵌套找作用域属性方法的

这里我额外的提及一下作用域

    var d = $.Deferred();

这个异步对象d是作用域是如何呢？

第一层：无可争议，浏览器环境下最外层是 window

第二层：jquery本身是一个闭包

第三层: Deferred工厂方法产生的作用域


如果用d.then()方法呢？

很明显then方法又是嵌套在内部的函数，所以执行的时候都默认会包含以上三层作用域+自己本身函数产生的作用域了

我们用个简单图描绘下

![](http://images.cnitblog.com/blog/329084/201310/03210232-7602c75e6dab41b6bff219c0a288f7d3.jpg)
根据规则，在最内部的函数能够访问上层作用域的所有的变


我们先从使用的层面去考虑下结构设计:

demo 1


```js
var defer = $.Deferred();

var filtered = defer.then(function( value ) {
      return value * 2;
    });

defer.resolve( 5 );

filtered.done(function( value ) {
    console.log(value) //10
});

```

demo 2

```js
  var defer = $.Deferred();

  defer.then(function(value) {
    return value * 2;
  }).then(function(value) {
    return value * 2;
  }).done(function(value) {
      alert(value)  //20
  });

  defer.resolve( 5 );

```
其实这里就是涉及到defer.then().then().done()  链式调用了



API是这么定义的:

```js
deferred.then( doneFilter  [, failFilter ] [, progressFilter ] )
```
     //从jQuery 1.8开始, 方法返回一个新的promise（承诺），通过一个函数，可以过滤deferred（延迟）的状态和值。替换现在过时的deferred.pipe()方法。
    //doneFilter 和 failFilter函数过滤原deferred（延迟）的解决/拒绝的状态和值。
    //progressFilter 函数过滤器的任何调用到原有的deferred（延迟）的notify 和 notifyWith的方法。
    //这些过滤器函数可以返回一个新的值传递给的 promise（承诺）的.done() 或 .fail() 回调，
    //或他们可以返回另一个观察的对象（递延，承诺等）传递给它的解决/拒绝的状态和值promise（承诺）的回调。
    //如果过滤函数是空，或没有指定，promise（承诺）将得到与原来值相同解决（resolved）或拒绝（rejected）。



我们抓住几点：

返回的是新的`promise`对象
内部有一个过滤器函数


从`demo 1`中我们就能看到

经过`x.then()`方法处理的代码中返回的`this（filtered ）`,不是原来的`$.Deferred()`所有产生的那个异步对象(defer )了

<p class="red">所以，每经过一个then那么内部处理的this都要被重新设置，那么为什么要这样处理呢？</p>

```js
then: function( /* fnDone, fnFail, fnProgress */ ) {
    var fns = arguments;
    //分别为deferred的三个callbacklist添加回调函数，根据fn的是否是函数，分为两种情况
    return jQuery.Deferred(function( newDefer ) {
        jQuery.each( tuples, function( i, tuple ) {
            var action = tuple[ 0 ],
                fn = jQuery.isFunction( fns[ i ] ) && fns[ i ];
            // deferred[ done | fail | progress ] for forwarding actions to newDefer
            deferred[ tuple[1] ](function() {
                var returned = fn && fn.apply( this, arguments );
                if ( returned && jQuery.isFunction( returned.promise ) ) {
                    returned.promise()
                        .done( newDefer.resolve )
                        .fail( newDefer.reject )
                        .progress( newDefer.notify );
                } else {
                    newDefer[ action + "With" ]( this === promise ? newDefer.promise() : this, fn ? [ returned ] : arguments );
                }
            });
        });
        fns = null;
    }).promise();
},

```



在`Deferred`传递实参的时候，支持一个flag，`jQuery.Deferred(func)`

传递一个回调函数

```js
// Call given func if any
if ( func ) {
    func.call( deferred, deferred );
}
```
所以newDefer可以看作是

newDefer = $.Deferred();
那么func回调的处理的就是过滤函数了



```js
deferred[ tuple[1] ](function() {
    var returned = fn && fn.apply( this, arguments );
    if ( returned && jQuery.isFunction( returned.promise ) ) {
        returned.promise()
            .done( newDefer.resolve )
            .fail( newDefer.reject )
            .progress( newDefer.notify );
    } else {
        newDefer[ action + "With" ]( this === promise ? newDefer.promise() : this, fn ? [ returned ] : arguments );
    }
});
```

这里其实也有编译函数的概念，讲未来要执行的代码，预先通过闭包函数也保存起来，使其访问各自的作用域




第一步

分解tuples元素集

```js
jQuery.each( tuples, function( i, tuple ) {
   //过滤函数第一步处理
})
```

第二步

分别为 deferred[ done | fail | progress ]执行对应的add方法，增加过滤函数给done | fail | progress 方法

```js
deferred[ tuple[1] ]（
传入过滤函数
）//过滤函数 执行的时候在分解代码即


deferred[done] = list.add = callback.add
```

第三步

返回`return jQuery.Deferred().promise()`

此时构建了一个新的`Deferred`对象，但是返回的的是经过`promise()`方法处理后的，返回的是一个受限的`promise`对象



所以整个then方法就处理了2个事情

构建一个新的deferred对象，返回受限的promise对象
给父deferred对象的[ done | fail | progress ]方法都增加一个过滤函数的方法


我们知道`defer.then`方法返回的是一个新的`jQuery.Deferred().promise()`对象

那么我们把defer.then返回的称之为子对象,那么如何与父对象var defer = $.Deferred() 关联的起来的

我看看源码

```js
deferred[ tuple[1] ](//过滤函数//)
```

deferred其实就是根级父对象的引用,所以就嵌套再深,其实都是调用了父对象deferred[ done | fail | progress 执行add罢了


![](http://images.cnitblog.com/blog/329084/201310/07212521-e2749ec11b1c44049e104d9124320667.jpg)


从图中就能很明显的看到 2个不同的deferred对象中 done fail progress分别都保存了不同的处理回调了

```js
deferred.resolve( args )
```

当延迟对象被 `resolved `时，任何通过 `deferred.then`或`deferred.done` 添加的 `doneCallbacks`，都会被调用
回调函数的执行顺序和它们被添加的顺序是一样的
传递给 `deferred.resolve()` 的 args 参数，会传给每个回调函数
当延迟对象进入 `resolved` 状态后，再添加的任何 `doneCallbacks`，当它们被添加时，就会被立刻执行，并带上传入给.resolve()的参数


流程如图

![](http://images.cnitblog.com/blog/329084/201310/07212507-bba369612b92425a908b5bbb2977b586.jpg)


流程解析：

1. 执行fire()方法，递归执行list所有包含的处理方法
2. 执行了默认的 changeState, disable, lock 方法、

3. 执行过滤函数

根据 var returned = fn.apply( this, arguments )的返回值(称作returnReferred)是否是deferred对象

  1. 返回值是deferred对象，那么在returnReferred对象的三个回调函数列表中添加newDeferred的resolve(reject,notify)方法，也就是说newDeferrred的执行依赖returnDeferred的状态
  2. 不是函数的情况（如值为undefined或者null等），直接链接到newDeferred的resolve(reject,notify)方法，也就是说  newDeferrred的执行依赖外层的调用者deferred的状态或者说是执行动作（resolve还是reject或者是notify）  此时deferred.then()相当于将自己的callbacklist和newDeferred的callbacklist连接起来


![](http://images.cnitblog.com/blog/329084/201310/07224128-b173cd377afd4ea9bf6caf1bf449872e.png)


下面就是嵌套deferred对象的划分了

![](http://images.cnitblog.com/blog/329084/201310/08084454-e443324b4f894f809174a49590366a0d.png)




