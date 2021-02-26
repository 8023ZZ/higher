#概述
***
### 并发和并行

### 互斥共享和同时共享

### 虚拟化
时分复用和空分复用，把一个物理实体转换成多个逻辑实体

### 异步

### 基本功能
包含进程管理，内存管理，文件管理和设备管理

### 系统调用
如果一个进程在用户态需要使用内核态的功能（进程控制、进程通讯、文件管理等），就进行系统调用从而陷入内核态，由操作系统完成
***
### 宏内核和微内核
#### 1.宏内核
宏内核是将操作系统的功能作为一个整体放到内核。
由于各模块共享信息，因此性能较高
#### 2.微内核
微内核结构下，操作系统被划分成小的、定义良好的模块，只有微内核这一个模块运行在内核态，其他模块运行在用户态。
因为频繁地在用户态和内核态之间切换，所以会有一定的性能损失。
***
### 中断分类
#### 1.外中断
由CPU执行指令以外的事件中断，如 I/O 完成中断，表示设备输入输出处理已完成，处理器能够发送下一个输入输出请求，此外还有时钟中断、控制台中断等。
#### 2.异常
由CPU执行指令的内部事件引起，如非法操作码、地址越界、算数溢出等。
#### 3.陷入
在用户程序中使用系统调用。
***
# 进程管理
***
### 进程与线程
#### 1.进程
进程是资源分配的基本单位。
#### 2.线程
线程是独立调度的基本单位。
#### 3.区别
1.拥有资源
进程是资源分配的基本单位，但是线程不拥有资源，线程可以访问隶属于进程的资源
2.调度
线程是独立调度的基本单位，同一进程中，线程的切换不会引起进程切换，从一个进程的线程切换到另一个进程的线程时，会引起进程切换
3.系统开销
由于创建或撤销进程时，系统都要为之分配或回收资源，如内存空间、I/O设备等，开销远大于创建或撤销线程。再进行进程切换时，涉及当前进程CPU环境的保存及新调度进程CPU环境的设置，而线程切换只需要保存和设置少量寄存器内容，开销很小
4.通信方面
线程间可以直接通过读写进程内的数据来进行通信，但是进程通信需要借助IPC
***
### 进程状态切换
就绪状态（ready）：等待被切换
运行状态（running）
阻塞状态（waiting）：等待资源

只有就绪态和运行态可以互相转换，其他的都是单向转换。就绪状态通过调度算法获得CPU时间，而运行状态的进程，在分配给他的CPU时间片用完后会转入就绪状态，等待下一次调用
阻塞状态是缺少相关资源从而由运行状态转换而来，但是该资源不包含CPU时间，缺少CPU时间会转为就绪态
***
### 进程调度算法
不同环境的调度算法目标不同，因此需要根据不同环境来讨论调度算法
#### 1.批处理系统
调度算法的目标是保证吞吐量和周转时间（从提交到终止的时间）
##### 1.1 先到先来 fiest come first servered（FCFS）
非抢占式，按请求的顺序进行调度
有利于长作业，但是会导致短作业前面有长作业时等待时间过长
##### 1.2 短作业优先 shortest job first (SJF)
非抢占式，按估计运行时间最短的顺序进行调度
长作业可能会饿死，永远得不到调度
##### 1.3 最短剩余时间优先 shortest remaining time next （SRTN）
抢占式，按剩余运行时间顺序排序

#### 2.交互式系统
有大量用户交互，调度目标是快速的进行响应
##### 2.1 时间片轮转
将所有就绪的进程按FCFS的顺序排成一个队列，每次调度时把CPU时间片分配给队首进程，该进程可以执行一个时间片，时间片到期后将进程送到队尾，同时将CPU时间重新分配
如果时间片太小，会导致进程切换的太频繁，如果时间片太长，实时性就不能得到保证
##### 2.2 优先级调度
为每个进程分配一个优先级，按优先级进行响应
为防止低优先级进程永远等不到调度，可以随时间推移增加等待进程的优先级
##### 2.3 多级反馈队列
时间片轮转和优先级调度的结合
设置多个队列，每个队列时间片大小不一致，进程在第一个队列没执行完则移入下一个队列，每个队列优先级不同，上一个队列没有进程在排队时才能进行当前队列的进程

#### 3.实时系统
硬实时：必须满足绝对的截至时间
软实时：容许一定的超时
***

# 内存
***
### 虚拟内存
虚拟内存的目的是为了让物理内存扩展成更大的逻辑内存，从而让程序有更多的可用内存

操作系统将内存抽象为地址空间，每个程序拥有自己的地址空间，地址空间被分割为多个块，每一块称为一页。这些页被映射到物理内存，但又不需要映射到连续的物理内存，也不需要所有页都必须在物理内存中

分页主要用于实现虚拟内存，从而获得更大的地址空间；分段主要是为了使程序和数据可以被划分为逻辑上独立的地址空间并有助于共享和保护

-------------------
# Linux
#### 孤儿进程
一个父进程退出，而它的一个或多个子进程还在运行，那么这些子进程将成为孤儿进程。

孤儿进程将被 init 进程（进程号为 1）所收养，并由 init 进程对它们完成状态收集工作。

由于孤儿进程会被 init 进程收养，所以孤儿进程不会对系统造成危害。

#### 僵尸进程
一个子进程的进程描述符在子进程退出时不会释放，只有当父进程通过 wait() 或 waitpid() 获取了子进程信息后才会释放。如果子进程退出，而父进程并没有调用 wait() 或 waitpid()，那么子进程的进程描述符仍然保存在系统中，这种进程称之为僵尸进程。

僵尸进程通过 ps 命令显示出来的状态为 Z（zombie）。

系统所能使用的进程号是有限的，如果产生大量僵尸进程，将因为没有可用的进程号而导致系统不能产生新的进程。

要消灭系统中大量的僵尸进程，只需要将其父进程杀死，此时僵尸进程就会变成孤儿进程，从而被 init 进程所收养，这样 init 进程就会释放所有的僵尸进程所占有的资源，从而结束僵尸进程。

