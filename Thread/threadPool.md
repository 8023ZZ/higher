### 线程池底层工作原理
核心参数：corePoolSize，maximumPoolSize，keepAliveTime，queue，拒绝策略
提交线程后，先判断线程池中线程数量是否小于核心线程数 corePoolSize，如果小于的话会直接创建一个线程（基于CAS从ThreadFactory中创建）出来执行任务，线程执行完成后会尝试从任务队列中获取新的任务，如果没有新的任务就会阻塞。
如果大于 corePoolSize，任务会被直接放入任务队列中。如果队列已满，且线程数量小于 maximumPoolSize，此时会创建额外的线程放入线程池中，然后超过corePoolSize数量的非核心线程如果处理完任务也会尝试从队列中获取任务来执行。如果线程数量等于maximumPoolSize 且 队列已满，则新任务会被拒绝。非核心线程是一个一个创建的。
后续队列任务处理完成后，线程空闲时，超过 corePoolSize 的非核心线程在 keepAliveTime 之后就会被释放掉
线程数 = CPU核数 * （CPU计算时间 + IO 等待时间）/ CPU 计算时间
通常如果 maximumPoolSize 为 Integer.MAX_VALUE 的话，corePoolSize 通常设置为0，来一个任务创建一个线程，通常 IO 密集型任务才会用到这个

core：核心线程数量，线程池会一直维持这个数量的线程，哪怕没有任务
queue：如果core用完了会先将任务放到 queue 中
max：如果 queue 也满了才会再次创建新的线程直到达到 max 的值
reject：如果 max 满了，那就执行对应的拒绝策略
keepalive：超过core数量时，而 queue 又为空，这个时候多余的线程会等待空闲时间后就会被回收掉

### 如果机器突然宕机，线程池的阻塞队列中的请求怎么办
线程池中积压的任务必然会宕机，需要在提交线程池之前先记录任务信息及状态
而保证提交和修改成已提交状态一致性就需要使用事务或者保证提交的过程接口时幂等的，即重复提交数据也是唯一的