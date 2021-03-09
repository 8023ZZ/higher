# JDK1.7
![avatar](/static/concurrenthashmap1.7.png)
Jdk1.7中，ConcurrentHashMap由很多Segment组合，而每一个Segment是一个类似于HashMap的结构，每一个HashMap内部可以扩容，但是Segment的个数一旦初始化就不能改变，默认个数为16个，也就是ConcurrentHashMap默认支持16线程并发。

初始化逻辑：
1. 必要参数检查；
2. 校验并发级别concurrencyLevel大小，如果大于最大值，则重置为最大值。无参构造默认值为16；
3. 寻找并发级别concurrencyLevel之上最近的2的幂次方，作为初始化容量大小，默认为16；
4. 记录segmentShift偏移量，这个值为【容量=2的n次方】中的n，后面在put时计算位置时会用到。默认是32 - segmentShift = 28；
5. 记录segmentMask，默认是segmentSize - 1 = 16 - 1 = 15；
6. 初始化segments[0]，默认大小为2，负载因子0.75，扩容阈值是1.5，插入第二个值时才会进行扩容。

``` java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null) 
        throw new NullPointerException();
    int hash = hash(key);
    // hash值无符号右移28位，然后与segmentMask=15做与运算
    // 其实就是把高四位与segmentMask做与运算
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
         (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
        // 如果查找到的 Segment 为空，初始化
        s = ensureSegment(j);
    return s.put(key, hash, value, false);
}

@SuppressWarnings("unchecked")
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // raw offset
    Segment<K,V> seg;
    // 判断 u 位置的 Segment 是否为null
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        Segment<K,V> proto = ss[0]; // use segment 0 as prototype
        // 获取0号 segment 里的 HashEntry<K,V> 初始化长度
        int cap = proto.table.length;
        // 获取0号 segment 里的 hash 表里的扩容负载因子，所有的 segment 的 loadFactor 是相同的
        float lf = proto.loadFactor;
        // 计算扩容阀值
        int threshold = (int)(cap * lf);
        // 创建一个 cap 容量的 HashEntry 数组
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) { // recheck
            // 再次检查 u 位置的 Segment 是否为null，因为这时可能有其他线程进行了操作
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            // 自旋检查 u 位置的 Segment 是否为null
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                   == null) {
                // 使用CAS 赋值，只会成功一次
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}
```

1. 计算put时的key的位置，获取指定位置的segment
2. 如果指定位置的segment为空，则初始化这个segment:
    2.1 检查计算得到位置的Segment是否为null；
    2.2 为null继续初始化，使用Segment[0]的容量和负载因子创建一个HashEntry数组
    2.3 再次检查计算得到的指定位置的Segment是否为null
    2.4 使用创建的HashEntry数组初始化这个Segment
    2.5 自旋判断计算得到的指定位置的Segment是否为null，使用CAS算法在这个位置赋值为Segment
3. Segment.put插入key，value值

``` java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 获取ReentrantLock独占锁，获取不到，scanAndLockForPut获取
    HashEntry<K, V> node = tryLock() ? null : scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        HashEntry<K,V>[] tab = table;
        // 计算要put的数据位置
        int index = (tab.length - 1) & hash;
        // CAS获取index坐标的值
        HashEntry<K, V> first = entryAt(tab, index);
        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                //检查key是否已存在，如果存在则遍历链表寻找位置，找到后替换 value
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
                // first 有值没说明 index 位置已经有值了，有冲突，链表头插法。
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                // 容量大于扩容阀值，小于最大容量，进行扩容
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    // index 位置赋值 node，node 可能是一个元素，也可能是一个链表的表头
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
```

Segment继承了ReentrantLock，所以Segment内部可以方便的获取锁。
1. tryLock()获取锁，获取不到使用scanAndLockForPut方法继续获取；
2. 计算put的数据要放入的index位置，然后获取这个位置上的HashEntry；
3. 遍历put新元素，遍历的原因是因为这里获取到的HashEntry可能是一个空元素，也可能是链表已存在；
    3.1 如果这个位置上的HashEntry不存在：如果当前容量大于扩容阈值，小于最大容量，进行扩容，然后直接头插法插入
    3.2 如果这个位置上的HashEntry存在：判断链表当前元素Key和Hash值是否和要put的key和hash值一致，一致则替换值；不一致则获取链表的下一节点，直到发现相同的值替换，或者链表遍历完毕没有相同的，此时如果当前容量大于扩容阈值，小于最大容量，进行扩容，然后直接头插法插入
4. 如果要插入的位置之前就已经存在，替换后返回旧值，否则返回null；

其中scanAndLockForPut的操作就是不断的自选tryLock()获取锁，当自旋次数大于指定次数时，使用lock()阻塞直到获取锁。

## 扩容rehash
ConcurrentHashMap扩容只会扩到原来的两倍，老数组里的数据迁移到新数组中，位置要么不变，要么变为index + oldSize，参数里的node会在扩容后使用头插法插到指定位置。

# JDK 1.8
## 存储结构
![avatar](/static/concurrenthashmap1.8.png)
不再是Segment数组 + HashEntry数组 + 链表，而是转变为Node数组 + 链表/红黑树。当冲突链表达到一定长度时，链表会转换为红黑树，并发控制使用Synchronized和CAS来操作

## 初始化initTable
``` java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab;int sc;
    while ((tab = table) == null || tab.length == 0) {
        // 如果sizeCtl < 0，说明另外的线程执行CAS成功，正在进行初始化
        if ((sc = sizeCtl) < 0)
            // 让出CPU时间片
            Thread.yield();
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}   
```
ConcurrentHashMap初始化是通过自旋和CAS操作实现的，其中sizeCtl的值决定着当前的初始化状态
-1 说明正在初始化
-N 说明有n-1个线程正在进行扩容
如果table没有初始化则表示table的初始化大小，如果已经初始化则表示table的容量的3/4


## put
``` java
public V put(K key, V value){
    return putVal(key, value, false);
}

// 添加一对键值对的时候，先判断保存这些键值对的数组是否已经初始化
// 如果没有的话就初始化数组
// 然后计算hash，确认放置的数组的位置
// 如果数组对应位置为空则直接添加，如果不为空则取出这个节点
// 如果取出的节点的hash值是MOVED(-1)的话，则表示当前正在对这个数组扩容，复制到新的数组，则当前线程也去帮助复制
// 最后一种情况是，如果这个节点不为空，也不在扩容，则通过synchronized来加锁，进行添加
//      然后判断当前去除的节点位置存放的是链表还是树
//      如果是链表的话则遍历整个链表，直到取出到相同的key，否则尾插法插入链表
//      如果是树的话调用putTreeVal把这个方法添加到树中
// 添加完成后，判断该节点处共有多少个节点（添加前的个数），达到8个则转成红黑树
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null)  
        throw new NullPointerException();
    int hash = spread(key.hashCode());    //取得key的hash值
    int binCount = 0;    //用来计算在这个节点总共有多少个元素，用来控制扩容或者转移为树
    for (Node<K,V>[] tab = table;;) {    //            Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)    
            tab = initTable();    //第一次put的时候table没有初始化，则初始化table
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {    //通过哈希计算出一个表中的位置因为n是数组的长度，所以(n-1)&hash肯定不会出现数组越界
            if (casTabAt(tab, i, null,        //如果这个位置没有元素的话，则通过cas的方式尝试添加，注意这个时候是没有加锁的
                new Node<K,V>(hash, key, value, null)))        //创建一个Node添加到数组中区，null表示的是下一个节点为空
                    break;                   // no lock when adding to empty bin
        }
        /*
         * 如果检测到某个节点的hash值是MOVED，则表示正在进行数组扩张的数据复制阶段，
         * 则当前线程也会参与去复制，通过允许多线程复制的功能，一次来减少数组的复制所带来的性能损失
          */
        else if ((fh = f.hash) == MOVED)    
            tab = helpTransfer(tab, f);
        else {
            /*
             * 如果在这个位置有元素的话，就采用synchronized的方式加锁，
             *     如果是链表的话(hash大于0)，就对这个链表的所有元素进行遍历，
             *         如果找到了key和key的hash值都一样的节点，则把它的值替换到
             *         如果没找到的话，则添加在链表的最后面
             *  否则，是树的话，则调用putTreeVal方法添加到树中去
             *  
             *  在添加完之后，会对该节点上关联的的数目进行判断，
             *  如果在8个以上的话，则会调用treeifyBin方法，来尝试转化为树，或者是扩容
             */
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {        //再次取出要存储的位置的元素，跟前面取出来的比较
                    if (fh >= 0) {                //取出来的元素的hash值大于0，当转换为树之后，hash值为-2
                        binCount = 1;            
                        for (Node<K,V> e = f;; ++binCount) {    //遍历这个链表
                            K ek;
                            if (e.hash == hash &&        //要存的元素的hash，key跟要存储的位置的节点的相同的时候，替换掉该节点的value即可
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)        //当使用putIfAbsent的时候，只有在这个key没有设置值得时候才设置
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {    //如果不是同样的hash，同样的key的时候，则判断该节点的下一个节点是否为空，
                                    pred.next = new Node<K,V>(hash, key,        //为空的话把这个要加入的节点设置为当前节点的下一个节点
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {    //表示已经转化成红黑树类型了
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,    //调用putTreeVal方法，将该元素添加到树中去
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)    //当在同一个节点的数目达到8个的时候，则扩张数组或将给节点的数据转为tree
                        treeifyBin(tab, i);    
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
    addCount(1L, binCount);    //计数
    return null;
}
```
1. 根据key计算出hashcode
2. 判断是否需要进行初始化
3. 即为当前key定位出的Node，如果为空表示当前位置可以写入数据，利用CAS尝试写入，失败则自旋保证成功
4. 如果当前位置的hashcode == MOVED == -1，则需要进行扩容
5. 如果都不满足，则利用synchronized锁写入数据
6. 如果数量大于阈值则转红黑树

## get
1. 根据hash值计算位置
2. 查找到指定位置，如果头节点就是要找的，直接返回他的value
3. 如果头节点hash值小于0，说明正在扩容或者是红黑树，查找之
4. 如果是链表则遍历查找

# 总结
JDK1.7中ConcurrentHashMap使用的分段锁，也就是每一个Segment上同时只有一个线程可以操作，每一个Segment都是类似于HashMap的结构，可以扩容或者将冲突转链表。但是Segment的个数一旦初始化就不能改变。

JDK1.8中ConcurrentHashMap使用的是Synchronized锁加CAS机制，结构进化成了Node数组+链表/红黑树，Node是一个类似于HashEntry的结构，冲突到达一定大小后会转红黑树，小于一定大小后退回链表。

## ConcurrentHashMap实现线程安全的底层原理是什么
JDK1.8以前是分段锁，多个数组分段加锁，一个数组一个锁ReentrantLock

初始化数据结构时：第一次put 时才会初始化 node 数组，通过 CAS 决定哪个线程进行初始化，CAS操作保证了设置sizeCtl标记位的原子性，保证了只有一个线程能设置成，同时用 volatile 修饰的 sizeCtl 表示线程竞争的情况

put时：JDK1.8以后，细化锁粒度，一个数组，put的时候如果为null则进行CAS，如果失败说明有人了，此时synchronized对数组元素加锁，然后基于链表或者红黑树插入自己的数据。如果不为null（因为此时数组位置存放的是链表或红黑树的引用，且出现概率很小）则直接synchronized锁然后基于链表或者红黑树插入自己的数据。只有对数组里同一个位置的元素进行操作时，才会加锁串行化处理；如果是对数组里不同的位置元素进行操作，那么可以正常并发处理。

扩容：多线程之间，以volatile的方式读取sizeCtl属性，来判断ConcurrentHashMap当前所处的状态。通过cas设置sizeCtl属性，告知其他线程ConcurrentHashMap的状态变更。
不同状态，sizeCtl所代表的含义也有所不同。
* 未初始化：
    sizeCtl=0：表示没有指定初始容量。
    sizeCtl>0：表示初始容量。
* 初始化中：
    sizeCtl=-1,标记作用，告知其他线程，正在初始化
* 正常状态：
    sizeCtl=0.75n ,扩容阈值
* 扩容中:
    sizeCtl < 0 : 表示有其他线程正在执行扩容
    sizeCtl = (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2 :表示此时只有一个线程在执行扩容

1. 在扩容之前，transferIndex 在数组的最右边 。此时有一个线程发现已经到达扩容阈值，准备开始扩容。
2. 扩容线程，在迁移数据之前，首先要将transferIndex右移（以cas的方式修改 transferIndex=transferIndex-stride(要迁移hash桶的个数)），获取迁移任务。每个扩容线程都会通过for循环+CAS的方式设置transferIndex，因此可以确保多线程扩容的并发安全。

ForwardingNode:
1. 标记作用，表示其他线程正在扩容，并且此节点已经扩容完毕
2. 关联了nextTable,扩容期间可以通过find方法，访问已经迁移到了nextTable中的数据

1. 线程执行put操作，发现容量已经达到扩容阈值，需要进行扩容操作，此时transferindex=tab.length=32
2. 扩容线程A 以cas的方式修改transferindex=31-16=16 ,然后按照降序迁移table[31]--table[16]这个区间的hash桶
3. 迁移hash桶时，会将桶内的链表或者红黑树，按照一定算法，拆分成2份，将其插入nextTable[i]和nextTable[i+n]（n是table数组的长度）。 迁移完毕的hash桶,会被设置成ForwardingNode节点，以此告知访问此桶的其他线程，此节点已经迁移完毕。
4. 此时线程2访问到了ForwardingNode节点，如果线程2执行的put或remove等写操作，那么就会先帮其扩容。如果线程2执行的是get等读方法，则会调用ForwardingNode的find方法，去nextTable里面查找相关元素。
5. 如果准备加入扩容的线程，发现以下情况，放弃扩容，直接返回。发现transferIndex=0,即所有node均已分配或者发现扩容线程已经达到最大扩容线程数

多线程无锁扩容的关键就是通过CAS设置sizeCtl与transferIndex变量，协调多个线程对table数组中的node进行迁移。

计数：利用CAS递增baseCount值来感知是否存在线程竞争，若竞争不大直接CAS递增baseCount值即可，性能与直接baseCount++差别不大
若存在线程竞争，则初始化计数桶，若此时初始化计数桶的过程中也存在竞争，多个线程同时初始化计数桶，则没有抢到初始化资格的线程直接尝试CAS递增baseCount值的方式完成计数，最大化利用了线程的并行。此时使用计数桶计数，分而治之的方式来计数，此时两个计数桶最大可提供两个线程同时计数，同时使用CAS操作来感知线程竞争，若两个桶情况下CAS操作还是频繁失败（失败3次），则直接扩容计数桶，变为4个计数桶，支持最大同时4个线程并发计数，以此类推…同时使用位运算和随机数的方式"负载均衡"一样的将线程计数请求接近均匀的落在各个计数桶中。