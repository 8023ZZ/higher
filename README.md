https://shishan100.gitee.io/docs/#/
# 准备
知识点收集总结
包含Java、Java Web、框架、算法、网络等方面内容

***
## 目录
***
### Java
##### [基础](#base)
##### [集合](#collection)
##### [线程](#thread)
##### [同步器](#aqs)
***
### 框架
##### [Spring](#spring)
##### [SpringBoot](#springboot)
##### [Jfinal](#jfinal)
***
### Java Web
##### [Servlet](#servlet)
##### [Nacos](#nacos)
##### [Sentinel](#sentinel)
##### [Zookeeper](#zookeeper)
##### [Netty](#netty)
***
### DataBase
##### [Mysql](#mysql)
##### [Redis](#redis)
***
### 算法
##### [排序](#sort)
##### 限流
***
### 网络
##### [计算机网络](#net)
##### [Http](#tttp)
##### [Socket](#socket)
***
### [操作系统](#systemGen) 
***
### 设计
##### DDD领域驱动设计
***



#### <span id="base">基础部分</span>
默认构造函数一般会把成员变量的值初始化为默认值，如int -> 0，Integer -> null    
  
java有三种变量类型，分别为类变量、成员变量和局部变量。类变量是静态变量  
  
##### 1.字符型常量和字符串常量的区别
1.字符型常量相当于一个整型值（ASCII值），可以参加表达式计算；字符串常量代表一个地址值（该字符串在内存中存放位置）
2.字符型常量只占两个字节(2 bytes)，字符串常量占若干个字节

##### 2.基础类型的包装类和常量池
装箱：将基本类型用他们对应的引用类型包装起来
拆箱：将包装类型转换为基本类型

除了float和double以外，其他的基本类型的包装类都实现了常量池技术。其中Byte、Short、Integer、Long默认创建了数值[-128,127]的相应类型的缓存数据，Character创建了数值在[0, 127]范围的缓存数据，Boolean直接返回True和False。超出范围仍然会去创建新对象。

Integer a = 40：编译的时候会直接将代码封装成 Integer a = Integer.valueOf(40)，从而使用常量池的对象。
Integer a = new Integer(40)，这种情况下会创建新对象。

##### 3.值传递
按值调用表示方法接收的是调用者提供的值，而按引用调用表示方法接收的是调用者提供的变量地址。一个方法可以修改传递引用所对应的变量值，而不能修改传递值调用所对应的变量值。
Java中总是采用按值传递，也就是说，方法得到的是所有参数值的一个拷贝，也就是说，方法不能修改传递给它的任何参数变量的内容。
方法得到的是对象引用的拷贝，对象引用及其他的拷贝同时引用同一个对象。

总结：1.一个方法不能改变一个基本数据类型的参数（eg.int,boolean）
     2.一个方法可以改变一个对象参数的状态
     3.一个方法不能让对象参数引用一个新的对象

new 关键字创建对象实例（对象实例存放在堆内存中），对象引用指向对象实例（对象引用存放在栈内存中）。一个对象引用可以指向0个或1个对象，一个对象实例可以有n个引用指向它。

调用子类构造方法前，jvm会先调用父类的无参构造方法，帮助子类进行初始化工作。

##### 4.HashCode()和equals()
hash值不同的对象一定不相同，通过hashCode()方法可以大大减少equals的次数，提升效率。
hashCode()在散列表中才有用，其他情况下没有作用。
hashCode()默认行为是对堆上的对象产生独特值，如果没有重写hashCode()，则该class的两个对象无论如何都不会相等

##### 5.transient关键字
阻止实例中那些用此关键字修饰的变量序列化，当对象被反序列化时，被transient修饰的变量值不会被持久化和恢复。transient只能修饰变量，不能修饰类和方法。

##### 6.性能分析
CPU占满：
通过top查询Cpu占满的线程号PID -> 获取线程号对应的16进制数 -> 通过jstack查询线程堆栈的dump信息，查看线程状态

内存泄漏：
jstat查看程序GC情况 -> 查看dump状态

死锁：
jstack

线程频繁切换：
通过vmstat查看r（等待线程数量），cs（每秒线程上下文切换次数），us（用户态占用CPU的时间百分比），sy（内核态占用CPU时间的百分比） -> 通过pidstat查看Java进程内部的线程切换数据

系统上下文切换分为三种情况：
1.多任务：多任务环境中，一个进程被切出CPU，执行另一个进程，会发生上下文切换
2.终端处理：发生中断时，硬件会切换上下文
3.用户和内核模式切换：当操作系统需要在用户模式和内核模式之间进行转换时，需要进行性上下文切换，比如进行系统函数调用

##### 获取class的四种方式
1. 通过.class
Class a = A.class;
2. 通过forName()静态方法传入类路径获取
Class a = Class.forName("cn.com.src.a");
默认会进行初始化
3. 通过对象实例的getClass()方法
Class a = e.getClass();
4. 通过类加载器 xxxClassLoader.loadClass()传入类路径获取
class clazz = ClassLoader.loadClass("cn.com.src.a");
通过类加载器获得Class对象不会进行初始化，静态块和静态对象不会进行执行

##### <a href="/Base/IO.md">NIO、AIO和BIO</a>

#### <span id="collection">集合</span>
Java中除了Map结尾的类以外，都实现了Collection接口，以Map结尾的类都实现了Map接口
![avatar](/static/collection.jpg)

##### ArrayList和LinkedList的区别
1. 都不保证线程安全
2. ArrayList底层是Object[]数组，LinkedList底层是双向链表，1.6之前是循环链表，1.7取消了循环
3. ArrayList采用数组存储，所以插入和删除元素的时间复杂度受元素位置的影响。LinkedList采用链表存储，插入前需获取前置节点的位置，时间复杂度近似为O(n)
4. LinkedList不支持快速随机访问，快速随机访问就是通过元素的序号快速获取元素对象
5. ArrayList的空间浪费主要体现在list列表的结尾会预留一定的容量空间，而LinkedList的空间花费则体现在它的每一个元素都需要消耗比ArrayList更多的空间（因为需要存放前驱后驱以及数据）

##### RandomAccess接口
```java
public interface RandomAccess {
}
```
RandomAccess接口是一个标识，用来标识实现这个接口的类具有随机访问功能。RandomAccess接口只是标识，并不是说ArrayList实现RandomAccess接口才具备随机访问功能，而是ArrayList接口首先具备快速随机访问功能，所以才标记它实现RandomAccess接口。

##### <span><a href="/Collection/ArrayList.md">ArrayList</a>

##### TreeMap和HashMap的区别
TreeMap和HashMap都继承自AbstractMap，但TreeMap还实现了SortedMap和NavigableMap。
![avatar](/static/TreeMap继承结构.png)
NavigableMap接口让TreeMap有了对集合内元素的搜索的能力。
实现SortMap接口让TreeMap有了对集合中的元素根据键排序的能力。默认按Key的升序排序，不过也可以指定排序的比较器。

##### <a href="/Collection/HashMap.md">HashMap</a>

##### <a href="/Collection/ConcurrentHashMap.md">ConcurrentHashMap</a>

#### <span id="thread"><a href="/Thread/thread.md">线程</a></span>
#### <span id="aqs"><a href="/AQS/aqs.md">同步器</a></span>
***
### 框架
***
### JavaWeb
#### <span id="servlet"><a href="/JavaWeb/Servlet.md">Servlet</a></span>
***
### DataBase
***
### 算法
***
### 网络
#### <span id="net"><a href="/Net/net.md">计算机网络</a></span>
#### <span id="net"><a href="/Net/http.md">Http</a></span>
#### <span id="socket"><a href="/Net/socket.md">Socket</a></span>
***
### <span id="systemGen"><a href="/System/system.md">操作系统</a></span>