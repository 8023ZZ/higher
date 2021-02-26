### synchronized 底层原理是什么
synchronized底层的原理是和jvm指令和monitor有关系的
使用synchronized关键字，底层编译后的jvm指令，会有monitorenter和monitorexit两个指令

monitorenter:
每个对象都有一个关联的monitor，如果要对这个对象加锁，那么必须获取这个对象关联的monitor的lock锁
monitor里面有一个计数器，从0开始，如果一个线程要获取monitor的锁，就看他的计数器是不是0，如果是0，那么线程就可以获取锁，然后对计数器加1
synchronized是可重入锁，同一线程多次加锁会使计数器的值累加


monitorexit：
会对monitor计数器减一

方法加锁不通过monitor指令，而是通过ACC_SYNCHRONIZED关键字，判断方法是否同步成功
monitor指令时synchronized在代码块里的原理

类锁和对象锁不是互斥的

