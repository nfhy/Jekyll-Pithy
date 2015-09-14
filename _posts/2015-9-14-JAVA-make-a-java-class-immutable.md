---
layout: post
title: Java中的不可变类
excerpt: 不可变类的优点和声明方式
category: JAVA
---

###不可变类的优点

* 易于构造，测试和使用
* 天然线程安全，没有同步问题
* 不需要实现clone方法
* 引用不可变类的实例时，不需要考虑实例的值发生变化的情况

###如何构造不可变类

* 不声明“setter”方法。
* 所有属性设为private final。
* class声明为final，不允许继承。
* 构造方法声明为私有
