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

```