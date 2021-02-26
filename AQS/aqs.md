## <a href="/AQS/synchronized.md">Synchronized</a>
## <a href="/AQS/CAS.md">CAS</a>
## AQS实现原理
Abstract Queue Synchronizer，抽象线程同步器，核心变量 volatile 修饰的 state 和 双向链表 Node 以及 Condition 单向队列，线程加锁时通过 CAS 来更新 state，失败后进入队列等待，成功后记录加锁线程变量。一个线程释放锁后会判断双向队列头节点是否为空，不为空则唤醒头节点的next节点出队

ReentrantLock 默认是非公平锁

## Synchronized 和 ReentrantLock 区别
* Synchronized是 JVM 层面的锁，ReentrantLock 是利用 CAS 自旋来保证线程操作的原子性和 volatile 关键字来保证数据可见性以及锁的功能
* Synchronized 不需要手动释放，但是 ReentrantLock 需要手动释放
* Synchronized 是不可中断类型的锁，但是 ReentrantLock 可以通过设置超时方法或者调用 interrupt() 方法进行中断
* Synchronized 是非公平锁，ReentrantLock可以自行选择是否公平
* Synchronized 不能绑定 Condition，但是 ReentrantLock 可以通过绑定 Condition 结合 await()/signal() 方法来实现精准唤醒，而不是像 Synchronized 通过 Object 类的 wait()/notify()/notifyAll() 方法要么随机唤醒要么全部唤醒
* Synchronized 是对象锁，锁保存在对象头中，ReentrantLock锁的是线程，根据进入的线程和 int 类型的 state 标识锁的获取和争抢