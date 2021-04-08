### Hystrix
* 创建 command；
* 执行这个 command；
* 配置这个 command 对应的 group 和线程池

1. 创建 command
    一个 HystrixCommand 或 HystrixObservableCommand 对象，代表了对某个依赖服务发起的一次请求或者调用。创建的时候，可以在构造函数中传入任何需要的参数。

    HystrixCommand 主要用于仅仅会返回一个结果的调用。
    HystrixObservableCommand 主要用于可能会返回多条结果的调用。
2. 调用 command 执行方法
   要执行 command，可以在 4 个方法中选择其中的一个：execute()、queue()、observe()、toObservable()。

    其中 execute() 和 queue() 方法仅仅对 HystrixCommand 适用。

    execute()：调用后直接 block 住，属于同步调用，直到依赖服务返回单条结果，或者抛出异常。
    queue()：返回一个 Future，属于异步调用，后面可以通过 Future 获取单条结果。
    observe()：订阅一个 Observable 对象，Observable 代表的是依赖服务返回的结果，获取到一个那个代表结果的 Observable 对象的拷贝对象。
    toObservable()：返回一个 Observable 对象，如果我们订阅这个对象，就会执行 command 并且获取返回结果。
3. 检查是否开启缓存
   如果这个 command 开启了请求缓存 Request Cache，而且这个调用的结果在缓存中存在，那么直接从缓存中返回结果。否则，继续往后的步骤
4. 检查是否开启了断路器
   检查这个 command 对应的依赖服务是否开启了断路器。如果断路器被打开了，那么 Hystrix 就不会执行这个 command，而是直接去执行 fallback 降级机制，返回降级结果。
5. 检查线程池/队列/信号量是否已满
   如果这个 command 线程池和队列已满，或者 semaphore 信号量已满，那么也不会执行 command，而是直接去调用 fallback 降级机制，同时发送 reject 信息给断路器统计。
6. 执行 command
   调用 HystrixObservableCommand 对象的 construct() 方法，或者 HystrixCommand 的 run() 方法来实际执行这个 command
7. 断路健康检查
   Hystrix 会把每一个依赖服务的调用成功、失败、Reject、Timeout 等事件发送给 circuit breaker 断路器。断路器就会对这些事件的次数进行统计，根据异常事件发生的比例来决定是否要进行断路（熔断）。如果打开了断路器，那么在接下来一段时间内，会直接断路，返回降级结果。

    如果在之后，断路器尝试执行 command，调用没有出错，返回了正常结果，那么 Hystrix 就会把断路器关闭。
8. 调用 fallback 降级机制
    在以下几种情况中，Hystrix 会调用 fallback 降级机制。

    断路器处于打开状态；
    线程池/队列/semaphore满了；
    command 执行超时；
    run() 或者 construct() 抛出异常。