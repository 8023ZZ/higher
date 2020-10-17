# 简介
ArrayList继承于AbstractList，实现了List, RandomAccess, Cloneable, java.io.Serializable这些接口
``` java
public class ArrayList<E> extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```
Cloneable接口代表ArrayList可以被克隆。
Serializable接口意味着ArrayList可以被序列化，能通过序列化去传输。

# 源码解读
``` java
// 默认初始容量大小
private static final int DEFAULT_CAPACITY = 10;
```

## 构造方法

ArrayList有三种构造方法
``` java
// 默认构造函数，使用初始容量10构造一个空列表（无参数构成）
// 实际上初始化赋值的是一个空数组，当真正对数组进行元素添加操作时，
// 才真正分配容量。
// 即向数组中添加第一个元素时，数组容量扩为10。
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

// 带初始容量参数的构造函数（用户自己指定容量）
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        // 初始容量大于0，则创建initialCapacity大小的数组
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
    }
}

// 构造包含指定collection元素的列表，这些元素利用该集合的迭代器按顺序返回
// 如果指定的集合为null，抛出NullPointerException
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

## 扩容机制
以无参函数创建的ArrayList为例分析
``` java
// 将指定的元素追加到队列的末尾
public boolean add(E e) {
   // 添加元素之前，先调用ensureCapacityInternal方法
   ensureCapacityInternal(size + 1);
   // ArrayList添加元素的实质相当于就是为数组赋值
   elementData[size++] = e;
   return true; 
}

// 得到最小扩容量
// 当要add进第一个元素时，minCapacity为1，在Math.max()方法比较过后，minCapacity为10
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        // 获取默认容量和传入参数的较大值
        minCapacity = Math.max(DEFAULT_CAPICITY, minCapacity);
    }
    ensureExplicitCapacity(minCapacity);
}

// 判断是否需要扩容
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    if (minCapacity - elementData.length > 0) {
        grow(minCapacity);
    }
}
```

1. 当我们要add进第一个元素到ArrayList时，elementData.length为0（因为还是一个空的list），因为执行了ensureCapacityInternal()方法，所以minCapacity为10。此时，if条件判断成立，进入grow方法。
2. 当add第2个元素时，minCapacity为2，此时，elementData.length在添加第一个元素后扩容成了10，所以if条件判断不成立，不会进入grow方法。
3. 添加第3.4...10个元素时，依然不会执行grow方法，数组容量为10。
4. 添加第11个元素时，minCapacity为11，比elementData.length要大，会进入grow方法扩容。

``` java
// 要分配的最大数组大小
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

private void grow(int minCapacity) {
    // oldCapacity为旧容量，newCapacity为新容量
    int oldCapacity = elementData.length;
    // 将oldCapacity右移一位，其效果相当于oldCapacity / 2
    // 位运算的速度远快于乘除运算
    // 运算式的结果就是将新容量更新为旧容量的1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 然后检查新容量是否大于最小需要容量，若还是小于最小需要容量，那么就把最小需要容量当作数组的新容量
    if (newCapacity - minCapacity < 0) 
        newCapacity = minCapacity;
    // 如果新容量大于MAX_ARRAY_SIZE，执行hugeCapacity()来比较minCapacity和MAX_ARRAY_SIZE
    // 如果minCapacity大于最大容量，则新容量为Integer.MAX_VALUE，否则新容量大小为MAX_ARRAY_SIZE，即为MAX_VALUE-8
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

奇数的位运算会丢掉小数位，右移n位相当于除以2的n次方

1. 当add第一个元素时，oldCapacity为0，经比较，后一个if判断成立，newCapacity为10，但是第二个if判断不成立，数组容量为10，size增为1
2. 当add第11个元素时，newCapacity为15，比minCapacity(10)要大，第一个if判断不成立，新容量没有大于MAX_ARRAY_SIZE，不会执行hugeCapacity方法，数组容量扩增为15，size增为11

java中的length属性是针对数组，标识数组的长度
java中的length()方法是针对字符串，标识字符串长度
java中的size()方法是针对集合，标识集合有多少元素

ArrayList的最大长度即为Integer.MAX_VALUE。

## System.arraycopy()和Arrays.copyOf()方法
``` java
// 在此列表的指定位置插入指定的元素
// 先调用rangCheckForAdd对index进行界限检查
// 然后调用ensureCapacityInternal保证capacity足够大
// 再将index开始之后的所有成员后移一个位置，将element插入指定位置
// 最后size + 1
public void add(int index, E element) {
    rangeCheckForAdd(index);

    ensureCapacity(size + 1);
    // arraycopy()方法实现数组自己复制自己
    // elementData：源数组；index：源数组中的起始位置；
    // elementData：目标数组；index + 1: 目标数组中的起始位置；
    // size - index: 要复制的数组元素的数量
    System.arraycopy(elementData, index, elementData, index+1, size-index);
    elementData[index+1] = element;
    size ++;
}
```

``` java
// 以正确的顺序返回一个包含此列表中的所有元素的数组（从第一个到最后一个元素）
// 返回数组的运行时类型是指定数组的运行时类型
public Object[] toArray() {
    // elementData：要复制的数组； size：要复制的长度
    return Arrays.copyOf(elementData, size);
}
```
使用Arrays.copyOf()方法主要是为了给原有数组扩容。

Arrays.copyOf()方法内部实际调用了System.arraycopy()方法，copyOf()是系统自动在内部新建了一个数组，并返回该数组

## ensureCapacity方法
提供给外部进行调用。
``` java
// 如有必要，增加此ArrayList实例的容量，以确保它至少可以容纳由minimum capacity参数指定的元素数
public void ensureCapacity(int minCapacity) {
    int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) ?
                    0 : DEFAULT_CAPACITY;
    if (minCapacity > minExpand) {
        ensureExplicitCapacity(minCapacity);
    }
}
```
最好在add大量元素之前用ensureCapacity方法，以减少增量重新分配的次数。