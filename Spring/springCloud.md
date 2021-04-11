### SpringCloud 底层架构
#### Eureka
两大功能，服务注册和发现 + 故障发现

1. 发起服务注册请求，注册到服务注册表
2. client 每隔30s拉取服务注册表，完成服务发现过程。Eureka 使用二级缓存，第一级是 ReadOnly 缓存（ConcurrentHashMap，其缓存更新，依赖于定时器的更新，通过和 readWriteCacheMap 的值做对比，如果数据不一致，则以 readWriteCacheMap 的数据为准），第二级是 ReadWrite 缓存（Guava 缓存，缓存过期时间，默认为 180 秒，当服务下线、过期、注册、状态变更，都会来清除此缓存中的数据。），优化并发冲突。修改服务注册表会立刻同步到 ReadWrite 缓存，另外会有一个线程每隔 30s 定时读取 ReadWrite 缓存然后同步到 ReadOnly 缓存，client 会先去读取 ReadOnly 缓存。Eureka Client 获取全量或者增量的数据时，会先从一级缓存中获取；如果一级缓存中不存在，再从二级缓存中获取；如果二级缓存也不存在，这时候先将存储层的数据同步到缓存中，再从缓存中获取
3. client 每隔 30s 向 server 发送心跳包
4. server 每隔 60s 检查服务注册表中哪些节点超过时间（默认90s）没有发送心跳包，发现是否有故障，如果有故障就清空二级缓存

集群同步时Eureka没有直接使用版本号的属性，而是采用一个叫做lastDirtyTimestamp的字段来对比。

如果开启SyncWhenTimestampDiffers配置（默认开启），当lastDirtyTimestamp不为空时，进行以下处理：

如果请求的lastDirtyTimestamp值大于Server本地该实例的lastDirtyTimestamp值，则表示Eureka Server之间的数据冲突，这个时候返回404，要求应用实例重新进行register操作；
如果请求的lastDirtyTimestamp值小于Server本地该实例的lastDirtyTimestamp值，如果是peer节点的复制请求，则表示数据出现冲突，返回409给peer节点，要求其同步本地实例最新的数据。

peer节点的相互复制并不能保证所有操作都能成功，因此Eureka还通过应用实例与Server之间的heartbeat(心跳)也就是renewLease（租约）操作来进行数据的最终恢复，即如果发现应用实例数据与某个server的数据出现不一致，则Server返回404，应用实例重新进行register操作

feign，他是对一个接口打了一个注解，他一定会针对这个注解标注的接口生成动态代理，然后你针对feign的动态代理去调用他的方法的时候，此时会在底层生成http协议格式的请求，/order/create?productId=1

底层的话，使用HTTP通信的框架组件，HttpClient，先得使用Ribbon去从本地的Eureka注册表的缓存里获取出来对方机器的列表，然后进行负载均衡，选择一台机器出来，接着针对那台机器发送Http请求过去即可

配置一下不同的请求路径和服务的对应关系，你的请求到了网关，他直接查找到匹配的服务，然后就直接把请求转发给那个服务的某台机器，Ribbon从Eureka本地的缓存列表里获取一台机器，负载均衡，把请求直接用HTTP通信框架发送到指定机器上去

参数优化：服务上线，注册表多级缓存同步1秒，注册表拉取频率降低为1秒 服务心跳，1秒上报1次 故障发现，1秒钟检查一次1心跳，如果发现2秒内服务没上报心跳，认为故障了。服务注册中心都没任何压力，最多就是每秒钟几十次请求而已，服务注册中部署个2台机器，每台机器就是4核8G，高可用冗余，任何一台机器死掉，不会影响系统的运行。

数据库，MySQL，16核32G，物理机最佳，32核64G，最多抗个每秒钟几千请求问题不大，平时抗个每秒钟几十或者几百请求，三四千请求，但是只不过此时会导致MySQL机器的负载很高，CPU使用率很高，磁盘IO负载很高，网络负载很高