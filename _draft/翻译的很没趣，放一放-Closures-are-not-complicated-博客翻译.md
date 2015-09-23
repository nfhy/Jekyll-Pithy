---
layout: post
title: Closures are not complicated节选翻译
category: javaScript
excerpt: 关于闭包原理的说明
---

原文地址：[closures-are-not-complicated](http://blog.niftysnippets.org/2008/02/closures-are-not-complicated.html)

作者很幽默，文章很有趣，但是网站被墙了，所以要用代理。这里把比较重要的部分节选并翻译出来。

#1：javaScript中所有东西都是一个对象

调用一个javaScript方法的上下文-参数、变量都是称为“call object”的对象的一部分。解释器为这一方法的执行专门生成一个“call object”并为它赋值：

1. 参数名为arguments的Array，包含我们调用方法时的传入的参数以及名为callee的指向调用方法的引用。
2. 方法声明时的每个参数，以及调用方法时传入对应的值，如果某个参数在调用时没有被赋值，它会被设为undefined。
3. 方法内每个被声明的变量。

让我们看下下面的方法：

~~~javascript
function updateDisplay(panel, contentId)
{
    var url;

    // ...
}

updateDisplay(leftPanel, 123);

~~~

对foo方法的调用会生成一个call object对象：
~~~javascript

call.arguments = [leftPanel, 123]
call.arguments.callee = updateDisplay
call.panel = leftPanel
call.contentId = 123
call.url = undefined

~~~

这一对象在执行方法中的代码时会用到，如传入的参数，声明的变量等。

#2：变量名通过作用域链(scope chain)解析

作用域链是一个有序的对象链，当javaScript解释器遇到一个未知变量时，解释器会去检查作用域链的顶层对象是否有这个变量名，如果有，变量就被找到了；否则，解释器会去查找作用域链的下一层。全局对象(global object)在链的最底层。在浏览器中，全局对象称为“window”，因为它有一个名为“window”的属性指向它自己。

当我们调用一个方法时，这个方法的call object对象会被放入作用域链的顶端，最底层是全局对象，如下图所示：

![alt text](https://raw.githubusercontent.com/nfhy/medias/master/images/clauses/scope_chain1.png "scope chain")

scop



