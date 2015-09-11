---
layout: post
title: 二叉堆与Java中的优先队列
category: JAVA
excerpt: 关于Java优先队列源码的分析
---
之前在A\*算法演示程序的编码过程中，发现javaScript并没有原生的优先队列，于是去Java中找到了PriorityQueue类，研究了一下源码。

Java中的优先队列基于最小二叉堆实现。最小二叉堆具有两个性质：

* 结构性：必须是一颗完全二叉树，树的插入从左到右，一棵完全二叉树的高度为小于log(N)的最大整数。

* 堆序性：二叉树的父节点必须小于两个子节点。

父节点下标为n，两个子节点下标为2n+1和2(n+1)。父节点的值总是小于子节点，两个子节点间大小关系不确定。

Java PriorityQueue特点：

* 不接受null值
* 不接受不可排序的值
* 线程不安全
* 容量没有上限
* 插入和删除元素的时间复杂度为O(log(n))

首先来看下最关键的两个方法，siftDown(int k, E x)和siftUp(int k, E x)。这两个方法是插入、删除、排序和初始化操作的基础。两个方法的目的都是让节点k所在的子树符合二叉堆的性质，堆序性。区别在于，siftDown将节点k作为子树的父节点处理，siftUp则将节点k作为子树的子节点处理。

####siftDown

~~~java
private void siftDownComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>)x;
        int half = size >>> 1;
        //k<half时，节点k有子节点，需要与子节点比较大小
        while (k < half) {
        	  //2k+1,左侧子节点
            int child = (k << 1) + 1;
            Object c = queue[child];
            //2k+2,右侧子节点
            int right = child + 1;
            //判断右侧子节点是否存在且小于左侧子节点
            if (right < size &&
                ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
                c = queue[child = right];
            //如果父节点小于子节点，当前子树已经满足条件
            //不需要继续
            if (key.compareTo((E) c) <= 0)
                break;
            //否则，父节点与子节点对调，继续上述步骤
            queue[k] = c;
            k = child;
        }
        queue[k] = key;
    }

~~~
关于这部分代码，比较有意思的是迭代条件`k<half`。
`half = size>>>1`,我们分size为奇数和偶数两种情况来看：

* 奇数：size为奇数，`half=(size-1)/2`，最后一个节点下标为`size-1`，父节点下标为`(size-1)/2-1`，所以`k<half`保证所有下标为k的节点都有子节点。

* 偶数：size为偶数，`half=size/2`，最后一个节点下标为`size-1`,父节点下标为`((size-1)-1)/2`，也即`size/2-1`，所以`k<half`保证所有下标为k的节点都有子节点。

所有迭代条件`k<half`是合理的。

####siftUp

~~~java
private void siftUpComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>) x;
        while (k > 0) {
            int parent = (k - 1) >>> 1;
            Object e = queue[parent];
            if (key.compareTo((E) e) >= 0)
                break;
            queue[k] = e;
            k = parent;
        }
        queue[k] = key;
    }
~~~

siftUp的代码很容易理解，递归的与父节点比较大小，然后根据结果决定是否需要对调位置。

####heapify

接下来是PriorityQueue的初始化方法中最复杂的一个，从一个无需集合(Collection)生成一个优先队列。

~~~java
private void initFromCollection(Collection<? extends E> c) {
        initElementsFromCollection(c);
        heapify();
    }

~~~

initElementsFromCollection(c)方法很简单，将集合c中的数据放入队列，将集合c的大小赋予size属性。而heapify()方法就是将集合c转换为二叉堆的关键。

~~~java
private void heapify() {
        for (int i = (size >>> 1) - 1; i >= 0; i--)
            siftDown(i, (E) queue[i]);
    }
~~~

`size>>>1`的特殊性我们刚才已经讨论过了，这里看下heapify()是怎么做的，从`(size>>>1)-1`开始向上遍历，对每一个父节点，都要保证它所在的子树符合条件，那么整棵树都将符合条件。

####add offer

向队列中添加元素，两者代码相同，之所以并存是因为PriorityQueue继承自Queue。

~~~java
public boolean offer(E e) {
        if (e == null)
            throw new NullPointerException();
        modCount++;
        int i = size;
        if (i >= queue.length)
        //容量不够时，扩容
            grow(i + 1);
        size = i + 1;
        if (i == 0)
            queue[0] = e;
        else
            siftUp(i, e);
        return true;
    }

~~~

这里将元素放于队列尾部，调用siftUp方法进行调整。最坏的情况，新插入的元素比根节点的元素还要小，那么siftUp要执行到根节点所在的子树。

####removeAt

删除队列中下标为i的元素。由于要保持完整二叉树的结构，删除元素后仍需要进行重新排序。

~~~java
private E removeAt(int i) {
        modCount++;
        int s = --size;
        //如果i就是最后一个节点的下标，直接删除
        if (s == i) 
            queue[i] = null;
        else {
            E moved = (E) queue[s];
            queue[s] = null;
            siftDown(i, moved);
            if (queue[i] == moved) {
                siftUp(i, moved);
                if (queue[i] != moved)
                    return moved;
            }
        }
        return null;
    }
~~~

