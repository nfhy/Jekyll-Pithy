---
layout: post
title: Java ConcurrentHashMap.Segment
excerpt: java ConcurrentHashMap segment部分源码解析
category: JAVA
---

ConcurrentHashMap通过将完整的表分成若干个segment的方式实现锁分离，每个segment都是一个独立的线程安全的Hash表，当需要操作数据时，HashMap通过Key的hash值和segment数量来路由到某个segment，剩下的操作交给segment来完成。
下面来看下segment的实现：

**segment是一类特殊的hash表，继承了ReentrantLock类实现锁功能**

~~~java     
    static final class Segment<K,V> extends ReentrantLock implements Serializable {
    /*
    1.segment的读操作不需要加锁，但需要volatile读
    2.当进行扩容时(调用reHash方法)，需要拷贝原始数据，在拷贝数据上操作，保证在扩容完成前读操作仍可以在原始数据上进行。
    3.只有引起数据变化的操作需要加锁。
    4.scanAndLock(删除、替换)/scanAndLockForPut(新增)两个方法提供了获取锁的途径，是通过自旋锁实现的。
    5.在等待获取锁的过程中，两个方法都会对目标数据进行查找，每次查找都会与上次查找的结果对比，虽然查找结果不会被调用它的方法使用，但是这样做可以减少后续操作可能的cache miss。
         */

        private static final long serialVersionUID = 2249069246763182397L;
		 
		 /*
		 自旋锁的等待次数上限，多处理器时64次，单处理器时1次。
		 每次等待都会进行查询操作，当等待次数超过上限时，不再自旋，调用lock方法等待获取锁。
		 */
        static final int MAX_SCAN_RETRIES =
            Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;
		 /*
		 segment中的hash表，与hashMap结构相同，表中每个元素都是一个链表。
		 */
        transient volatile HashEntry<K,V>[] table;

        /*
         表中元素个数
         */
        transient int count;

        /*
        记录数据变化操作的次数。
        这一数值主要为Map的isEmpty和size方法提供同步操作检查，这两个方法没有为全表加锁。
        在统计segment.count前后，都会统计segment.modCount，如果前后两次值发生变化，可以判断在统计count期间有segment发生了其它操作。
         */
        transient int modCount;

        /*
        容量阈值，超过这一数值后segment将进行扩容，容量变为原来的两倍。
        threshold = loadFactor*table.length
         */
        transient int threshold;

        final float loadFactor;

        Segment(float lf, int threshold, HashEntry<K,V>[] tab) {
            this.loadFactor = lf;
            this.threshold = threshold;
            this.table = tab;
        }
		 /*
		 onlyIfAbsent:若为true，当key已经有对应的value时，不进行替换；
		 若为false，即使key已经有对应的value，仍进行替换。
		 
		 关于put方法，很重要的一点是segment最大长度的问题：
		 代码 c > threshold && tab.length < MAXIMUM_CAPACITY 作为是否需要扩容的判断条件。
		 扩容条件是node总数超过阈值且table长度小于MAXIMUM_CAPACITY也就是2的30次幂。
		 由于扩容都是容量翻倍，所以tab.length最大值就是2的30次幂。此后，即使node总数超过了阈值，也不会扩容了。
		 由于table[n]对应的是一个链表，链表内元素个数理论上是无限的，所以segment的node总数理论上也是无上限的。
		 ConcurrentHashMap的size()方法考虑到了这个问题，当计算结果超过Integer.MAX_VALUE时，直接返回Integer.MAX_VALUE.
		 
		 */
        final V put(K key, int hash, V value, boolean onlyIfAbsent) {
            //tryLock判断是否已经获得锁.
            //如果没有获得，调用scanAndLockForPut方法自旋等待获得锁。
            HashEntry<K,V> node = tryLock() ? null :
                scanAndLockForPut(key, hash, value);
            V oldValue;
            try {
                HashEntry<K,V>[] tab = table;
                //计算key在表中的下标
                int index = (tab.length - 1) & hash;
                //获取链表的第一个node
                HashEntry<K,V> first = entryAt(tab, index);
                for (HashEntry<K,V> e = first;;) {
                //链表下一个node不为空，比较key值是否相同。
                //相同的，根据onlyIfAbsent决定是否替换已有的值
                    if (e != null) {
                        K k;
                        if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                            oldValue = e.value;
                            if (!onlyIfAbsent) {
                                e.value = value;
                                ++modCount;
                            }
                            break;
                        }
                        e = e.next;
                    }
                    else {
                    //链表遍历到最后一个node，仍没有找到key值相同的.
                    //此时应当生成新的node，将node的next指向链表表头，这样新的node将处于链表的【表头】位置
                        if (node != null)
                        //scanAndLockForPut当且仅当hash表中没有该key值时
                        //才会返回新的node，此时node不为null
                            node.setNext(first);
                        else
                        //node为null，表明scanAndLockForPut过程中找到了key值相同的node
                        //可以断定在等待获取锁的过程中，这个node被删除了，此时需要新建一个node
                            node = new HashEntry<K,V>(hash, key, value, first);
                        //添加新的node涉及到扩容，当node数量超过阈值时，调用rehash方法进行扩容，并将新的node加入对应链表表头；
                        //没有超过阈值，直接加入链表表头。
                        int c = count + 1;
                        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                            rehash(node);
                        else
                            setEntryAt(tab, index, node);
                        ++modCount;
                        count = c;
                        oldValue = null;
                        break;
                    }
                }
            } finally {
                unlock();
            }
            return oldValue;
        }

        /*
         hash表容量翻倍，将需要添加的node添加到扩容后的表中。
         hash表默认初始长度为16，实际长度总是2的n次幂。
         设当前table长度为S，根据key的hash值计算table中下标index的公式：
         扩容前:oldIndex = (S-1)&hash
         扩容后:newIndex = (S<<1-1)&hash
         扩容前后下标变化:newIndex-oldIndex = S&hash
         所以，扩容前后node所在链表在table中的下标要么不变，要么右移S。
         根据本方法官方注释说明，大约六分之一的node需要移动。
         
         对于每个链表，处理方法如下：
         
         步骤一：对于链表中的每个node，计算node和node.next的新下标，如果它们不相等，记录最后一次出现这种情况时的node.next，记为nodeSpecial。
         这一部分什么意思呢，假设table[n]所在的链表共有6个node，计算它们的新下标：
         情况1：若计算结果为0:n,1:n+S,2:n,3:n+S,4:n,5:n，那么我们记录的特殊node编号为4；
         情况2：若计算结果为0:n,1:n+S,2:n,3:n+S,4:n+S,5:n+S，那么我们记录的特殊node编号为3；
         情况3：若计算结果为0:n,1:n,2:n,3:n,4:n,5:n,特殊node为0;
         情况4：若计算结果为0:n+S,1:n+S,2:n+S,3:n+S,4:n+S,5:n+S,特殊node为0。
         很重要的一点，由于新下标只可能是n或n+S，因此这两个位置的链表中不会出现来自其它链表的node。
         对于情况3，令table[n]=node0，进入步骤三；
         对于情况4，令table[n+S]=node0，进入步骤三；
         对于情况1，令table[n]=node4，进入步骤二；
         对于情况2，令table[n+S]=node3,进入步骤二。
         
         步骤二：从node0遍历至nodeSpecial的前一个node，对于每一个node，计算它的index值，令node.next=table[index],table[index]=node,也就是说，将node放于对应链表的表头。
         
         步骤三：计算需要插入的node的下标index，同样令node.next=table[index],table[index]=node,将node插入链表表头。
         
         通过三步完成了链表的扩容和新node的插入。
         
         在理解这一部分代码的过程中，牢记三点：
         1.调用rehash方法的前提是已经获得了锁，所以扩容过程中不存在其他线程修改数据；
         2.新的下标只有两种情况，原始下标n或者新下标n+S；
         3.通过2可以推出，原表中不在同一链表的node，在新表中仍不会出现在同一链表中。
         */
        @SuppressWarnings("unchecked")
        private void rehash(HashEntry<K,V> node) {
            //拷贝table，所有操作都在oldTable上进行，不会影响无需获得锁的读操作
            HashEntry<K,V>[] oldTable = table;
            int oldCapacity = oldTable.length;
            int newCapacity = oldCapacity << 1;//容量翻倍
            threshold = (int)(newCapacity * loadFactor);//更新阈值
            HashEntry<K,V>[] newTable =
                (HashEntry<K,V>[]) new HashEntry[newCapacity];
            int sizeMask = newCapacity - 1;
            for (int i = 0; i < oldCapacity ; i++) {
                HashEntry<K,V> e = oldTable[i];
                if (e != null) {
                    HashEntry<K,V> next = e.next;
                    int idx = e.hash & sizeMask;//新的table下标，定位链表
                    if (next == null)
                    	//链表只有一个node，直接赋值
                        newTable[idx] = e;
                    else {
                        HashEntry<K,V> lastRun = e;
                        int lastIdx = idx;
                        //这里获取特殊node
                        for (HashEntry<K,V> last = next;
                             last != null;
                             last = last.next) {
                            int k = last.hash & sizeMask;
                            if (k != lastIdx) {
                                lastIdx = k;
                                lastRun = last;
                            }
                        }
                        //步骤一中的table[n]赋值过程
                        newTable[lastIdx] = lastRun;
                        // 步骤二，遍历剩余node，插入对应表头
                        for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                            V v = p.value;
                            int h = p.hash;
                            int k = h & sizeMask;
                            HashEntry<K,V> n = newTable[k];
                            newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                        }
                    }
                }
            }
            //步骤三，处理需要插入的node
            int nodeIndex = node.hash & sizeMask;
            node.setNext(newTable[nodeIndex]);
            newTable[nodeIndex] = node;
            //将扩容后的hashTable赋予table
            table = newTable;
        }

        /*
        put方法调用本方法获取锁，通过自旋锁等待其他线程释放锁。
        变量retries记录自旋锁循环次数，当retries超过MAX_SCAN_RETRIES时，不再自旋，调用lock方法等待锁释放。
        变量first记录hash计算出的所在链表的表头node，每次循环结束，重新获取表头node，与first比较，如果发生变化，说明在自旋期间，有新的node插入了链表，retries计数重置。
        自旋过程中，会遍历链表，如果发现不存在对应key值的node，创建一个，这个新node可以作为返回值返回。
         根据官方注释，自旋过程中遍历链表是为了缓存预热，减少hash表经常出现的cache miss
         */
        private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
            HashEntry<K,V> first = entryForHash(this, hash);
            HashEntry<K,V> e = first;
            HashEntry<K,V> node = null;
            int retries = -1; //自旋次数计数器
            while (!tryLock()) {
                HashEntry<K,V> f;
                if (retries < 0) {
                    if (e == null) {
                    //链表为空或者遍历至链表最后一个node仍没有找到匹配
                        if (node == null)
                            node = new HashEntry<K,V>(hash, key, value, null);
                        retries = 0;
                    }
                    else if (key.equals(e.key))
                        retries = 0;
                    else
                        e = e.next;
                }
                else if (++retries > MAX_SCAN_RETRIES) {
                    lock();
                    break;
                }
                else if ((retries & 1) == 0 &&
                         (f = entryForHash(this, hash)) != first) {
                //比较first与新获得的链表表头node是否一致，如果不一致，说明该链表别修改过，自旋计数重置
                    e = first = f;
                    retries = -1;
                }
            }
            return node;
        }

        /*
        remove,replace方法会调用本方法获取锁，通过自旋锁等待其他线程释放锁。
        与scanAndLockForPut机制相似。
         */
        private void scanAndLock(Object key, int hash) {
            // similar to but simpler than scanAndLockForPut
            HashEntry<K,V> first = entryForHash(this, hash);
            HashEntry<K,V> e = first;
            int retries = -1;
            while (!tryLock()) {
                HashEntry<K,V> f;
                if (retries < 0) {
                    if (e == null || key.equals(e.key))
                        retries = 0;
                    else
                        e = e.next;
                }
                else if (++retries > MAX_SCAN_RETRIES) {
                    lock();
                    break;
                }
                else if ((retries & 1) == 0 &&
                         (f = entryForHash(this, hash)) != first) {
                    e = first = f;
                    retries = -1;
                }
            }
        }

        /*
        删除key-value都匹配的node，删除过程很简单：
        1.根据hash计算table下标index。
        2.根据index定位链表，遍历链表node，如果存在node的key值和value值都匹配，删除该node。
        3.令node的前一个节点pred的pred.next = node.next。
         */
        final V remove(Object key, int hash, Object value) {
            //获得锁
            if (!tryLock())
                scanAndLock(key, hash);
            V oldValue = null;
            try {
                HashEntry<K,V>[] tab = table;
                int index = (tab.length - 1) & hash;
                HashEntry<K,V> e = entryAt(tab, index);
                HashEntry<K,V> pred = null;
                while (e != null) {
                    K k;
                    HashEntry<K,V> next = e.next;
                    if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {
                        V v = e.value;
                        if (value == null || value == v || value.equals(v)) {
                            if (pred == null)
                                setEntryAt(tab, index, next);
                            else
                                pred.setNext(next);
                            ++modCount;
                            --count;
                            oldValue = v;
                        }
                        break;
                    }
                    pred = e;
                    e = next;
                }
            } finally {
                unlock();
            }
            return oldValue;
        }
		/*
		找到hash表中key-oldValue匹配的node，替换为newValue,替换过程与replace方法类似，不再赘述了。
		*/
        final boolean replace(K key, int hash, V oldValue, V newValue) {
            if (!tryLock())
                scanAndLock(key, hash);
            boolean replaced = false;
            try {
                HashEntry<K,V> e;
                for (e = entryForHash(this, hash); e != null; e = e.next) {
                    K k;
                    if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {
                        if (oldValue.equals(e.value)) {
                            e.value = newValue;
                            ++modCount;
                            replaced = true;
                        }
                        break;
                    }
                }
            } finally {
                unlock();
            }
            return replaced;
        }

        final V replace(K key, int hash, V value) {
            if (!tryLock())
                scanAndLock(key, hash);
            V oldValue = null;
            try {
                HashEntry<K,V> e;
                for (e = entryForHash(this, hash); e != null; e = e.next) {
                    K k;
                    if ((k = e.key) == key ||
                        (e.hash == hash && key.equals(k))) {
                        oldValue = e.value;
                        e.value = value;
                        ++modCount;
                        break;
                    }
                }
            } finally {
                unlock();
            }
            return oldValue;
        }
		 /*
		 清空segment，将每个链表置为空，count置为0，剩下的工作交给GC。
		 */
        final void clear() {
            lock();
            try {
                HashEntry<K,V>[] tab = table;
                for (int i = 0; i < tab.length ; i++)
                    setEntryAt(tab, i, null);
                ++modCount;
                count = 0;
            } finally {
                unlock();
            }
        }
    }


~~~

以上就是ConcurrentHashMap中关于segment部分的源码。可以看出，关于修改操作的线程安全已经被封装在segment中，甚至在ConcurrentHashMap中都不需要关心锁的问题。可以说，segment是整个ConcurrentHashMap的核心和关键。