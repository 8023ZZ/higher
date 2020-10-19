# 简介
JDK1.8之前的HashMap由数组+链表组成，数组是HashMap的主体，链表主要是为了解决哈希冲突而存在的。
JDK1.8及以后在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为8）时，链表会转换为红黑树（转换前会进行判断，如果当前数组长度小于64，那么会先进行数组扩容，而不是转换为红黑树），以减少搜索时间。
HashMap核心为扩容、拉链法和红黑树转换

# 源码分析
## JDK1.8之前
HashMap通过key的hashCode经过扰动函数处理后得到hash值，然后通过(n-1)&hash 判断当前元素存放的位置（n为数组的长度），如果当前位置存在元素的话，就判断当前元素的hash以及key是否相同，如果相同则覆盖，不相同则通过拉链法解决冲突。

扰动函数是指HashMap的hash方法，使用hash方法是为了防止一些实现比较差的hashCode()方法，也就是减少碰撞。

``` java
static final int hash(Object key) {
    int h;
    // hashCode()返回散列值，也就是hashCode
    // ^：按位异或
    // >>>：无符号右移，忽略符号位，空位以0补齐
    return (key == null) ? 0 : (h == key.hashCode()) ^ (h >>> 16);
}
```

HashMap的初始容量为2^4(1 << 4)，最大容量为2^30(1 << 30)，默认填充因子为0.75，桶节点数大于8时会转红黑树，小于6时转链表。

loadFactor加载因子是控制数组存放数据的疏密程度，越趋近于1越密，链表长度越大。loadFactor太大会导致查找元素效率低，太小会导致数组的利用率低，默认值为0.75。

两个内部类，Node和TreeNode，一个是链表节点，另一个是树节点。

## put方法
HashMap只提供了put()用于添加元素，putVal()只是put()调用的一个方法。

1. 如果定位到的数组位置没有元素，则直接插入
2. 如果定位到的数组位置有元素，则和要插入的key进行比较，如果key相同就直接覆盖，如果不相同，就判断p是否是一个树节点，如果是则调用e = ((TreeNode<K,V>)p ).putTreeVal(this, tab, hash, key, value)将元素添加进入，如果不是就遍历链表插入（尾插法）。
![avatar](../static/Hashmap的put方法流程图.png)

``` java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // table未初始化或者长度为0，进行扩容
    if ((tab = table) == null || (n = tab.length) == 0) {
        n = (tab = resize()).length;
    } 
    // (n-1)&hash 确定元素存放在哪个桶中，桶为空，则新生节点放入桶中（此时这个节点是放在数组中的）
    if ((p = tab[i = (n-1)&hash]) == null) 
        tab[i] = newNode(hash, key, value, null);
    // 桶中已存在元素
    else {
        Node<K, V> e; K k;
        // 比较桶中第一个元素（数组中的节点）的hash值相等，key相等
        if (p.hash == hash &&
            ((k = p.key) == key ||(key !=null && key.equals(k))))
            // 将第一个元素赋值给e，用e来记录
            e = p;
        // hash值不相等，即key不相等且p为红黑树节点
        else if (p instanceof TreeNode)
            // 放入树中
            e = ((TreeNode<K, V>)p).putTreeVal(this,tab,hash,key,value);
        // hash值不相等，即key不相等且p为链表节点
        else {
            // 在链表末尾插入节点
            for (int binCount = 0; ; ++binCount) {
                // 到达链表的尾部
                if ((e = p.next) == null) {
                    //在尾部插入新节点
                    p.next = newNode(hash,key,value,null);
                    //节点数量达到阈值，转为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    // 跳出循环
                    break;
                }
            //判断链表中节点的key值与插入元素的key值是否相等
            if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                // 相等，跳出循环
                break;
            p = e;
            }
        }
    // 表示在桶中找到key值、hash值与插入元素相等的节点
    if (e != null) {
        // 记录e的value
        V oldValue = e.value;
        // onlyIfAbsent为false或者旧值为null
            if (!onlyIfAbsent || oldValue == null)
                //用新值替换旧值
                e.value = value;
            // 访问后回调
            afterNodeAccess(e);
            // 返回旧值
            return oldValue;
        }
    }
    // 结构性修改
    ++ modCount;
    // 实际大小大于阈值则扩容
    if (++size > threshold)
        resize();
    // 插入后回调
    afterNodeInsertion(evict);
    return null;
}
```

## resize方法
进行扩容，会伴随一次重新hash分配，并且会遍历hash表中所有的元素，编码过程中需要尽量避免

``` java
final Node<K, V>[] resize() {
    Node<K, V> oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 超过最大值则不进行扩容，只能进行碰撞
        if (old >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 没超过最大值，则扩充为原来的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1;
    }
    // 初始化容量
    else if (oldThr > 0)
        newCap = oldThr;
    else {
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 重新计算resize上限
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 把每个bucket都移动到新的buckets中
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { 
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 原索引
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 原索引+oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 原索引放到bucket里
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 原索引+oldCap放到bucket里
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

## HashMap的长度为什么是2的幂次方
数组下标的计算方式为 (n-1)&hash，而取余(%)操作中如果除数是2的幂次则等价于与其除数减一的与(&)操作，也就是hash%length == hash&(length-1) 成立的前提是length是2的n次方。并且采用二进制操作&，相对于%能有效提升效率，所以HashMap的长度是2的幂次方。

## HashMap多线程操作死循环问题
主要原因在于并发下的Rehash会造成元素之间形成一个循环链表。1.8后解决了这个问题，但还会存在比如数据丢失等问题。

两线程并发执行transfer()方法时：

state 1:
thread1 resize前半段: e -> next  cup时间片到
state 2:
thread2 完成rehash，next -> e
state 3:
thread1 resize后半段： step1.next -> e step2.e -> next -> e

循环链表出现的原因：1.rehash是头插法，造成了逆序；2.第一个线程先保存next节点后挂起，第二个线程执行结果导致next后面追加了e，第一个线程再执行，先头插e，在头插next，由于第二线程导致next后面有e，所以继续头插，e插到next前面，出现循环链表死循环。
jdk1.8修改为尾插法后修复了这个问题。