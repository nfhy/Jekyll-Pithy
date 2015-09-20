---
layout: post
title: Java并发工具类初探
excerpt: java中各种并发工具类介绍
category: JAVA
---

同步容器：

* Vector/Hashtable:jdk1.0就已经存在，jdk1.2改写实现List/Map接口。作为ArrayList/HashMap在并发场景中的替代类出现。注意：Hashtable的API中已经建议用ConcurrentHashMap替代Hashtable。
* SynchronizedSet/SynchronizedMap/SynchronizedList:Collections类生成的实现Set/Map/List接口的线程安全类，通过声明互斥锁，为所有线程不安全的操作加锁。注意：三个类的iterator仍然不是线程安全的，使用时需要额外加锁。
* ConcurrentHashMap:线程安全的HashMap，所有线程安全实现都被封装，为了提高并发性能，内部实现非常精彩，之后会单独开一个帖子专门说一下。
* CopyOnWriteArrayList:线程安全的ArrayList，所有写操作都通过“加锁-拷贝数据-修改-覆盖数据-解锁”的方式进行，操作非常消耗资源，但保证了读操作不需要加锁，因此适合读操作远多于写操作的场景。
* ConcurrentLinkedQueue:线程安全的队列(FIFO)。
* LinkedBlockingQueue:阻塞队列，提供可阻塞的take和put方法，当队列空时，take方法阻塞至队列非空；当队列满时，put方法阻塞至队列不满。
* ConcurrentLinkedDeque:线程安全的双端队列。
* LinkedBlockingQueue:阻塞双端队列。提供可阻塞的takeFirst/takeLast/putFirst/putLast方法。

同步器：

* CountDownLatch:闭锁，类似发令枪，所有线程在起跑线等待闭锁发出起跑信号，一起出发。
* Semaphore:计数信号量，类似许可证，线程执行需要获得许可证，执行结束需要返还许可证，当没有许可证可以发放时，线程将被阻塞直到别的线程返还了许可证。
* CyclicBarrier:关卡，类似龙珠，每个线程成功执行都会获得一颗龙珠，所有线程全部成功执行就可以召唤神龙，如果有一个线程执行失败，所有线程都将失败。

