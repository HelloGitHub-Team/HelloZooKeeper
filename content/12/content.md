# 如何吊打面试官



## 一、模拟面试

终于来到重头戏了，本小节我会从网上找到一些关于 ZK 的面试题进行剖析讲解，并且站在面试官的基础上分析考点，相信看完这一节，出去面试再碰到 ZK 相关的问题你便能披荆斩棘、所向披靡！

我先给大家模拟一个面试的场景：

面试官：我看你简历上用过 ZK，能给我介绍下吗？你是怎么理解 ZK 的作用呢？

（如果你把百度百科中的定义背给他听，我只能说 666，千万别这样，会被别人当成傻子。）

我：我的理解 ZK 是一个脱离于应用的第三方进程，类似的数据库，消息队列，Redis 等都是扮演这个角色，拥有一定的数据存储和查询能力，可以让我们在现在都是分布式部署的应用之间“传递”数据，其次 ZK 支持的回调通知，让应用可以在一些业务场景中感知到数据的变化并及时作出相应的反应。最后，ZK 本身也支持集群部署具有高可用的特点，是一个可靠的第三方中间件。

面试官：嗯，你刚刚提到了回调通知，能仔细跟我聊聊 ZK 是怎么去实现的吗？

我：各种编程语言的客户端都会对这个回调通知进行抽象，通常需要开发者声明一个 callback 的对象，在 Java 的客户端中这个接口是 `Watcher`，ZK 服务端提供了一些方法，比如 `getData`、`exists` 或者最新版本中的 `addWatch` 都可以用来向 ZK 注册回调通知，而向服务端发送的回调通知，只会告诉服务端我当前的这个路径需要被通知，服务端得知后，会在内存中记录下来，路径和客户端之间的关系，客户端自己也需要记录下来，路径和具体回调的关系。当被订阅的路径发生事件的时候，各种增删改吧，服务端就会从内存中的记录去查看有没有需要通知的客户端，有的话会发送一个通知的请求给客户端，客户端收到通知后，就会从本地的记录中取出对应的回调对象去执行 callback 方法！

（实际情况，我觉得面试官可能不会让你一直说下去，应该是互相聊的一个状态）

面试官：嗯，说得挺详细，那你刚刚提到的 `getData`、`exists`、 `addWatch` 三种注册有什么区别吗？

我：`getData`、`exists` 以及 `getChildren` 注册的通知都是一次性的，当服务端通知过一次后，就会删除内存中的记录，之后如果仍然需要通知的话，客户端就要去继续注册，而 `addWatch` 注册的回调通知是永久性的，只需要注册一次可以一直被通知。

面试官：嗯好，你刚刚还提到了 ZK 有一定的数据存储能力，你能说说 ZK 是怎么保存和整理数据的吗？

我：ZK 的数据体现在两部分。

面试官：哦？哪两部分？

我：内存中和磁盘上。

面试官：那你先说说内存里 ZK 是怎么存储数据

我：从逻辑上来讲，ZK 内存中的数据其实是一个树形结构，从 `/` 根节点开始，逐级向下用 `/` 分割，每一个节点下面还可以有多个子节点，就类似于 Unix 中的目录结构，但在实际中，ZK 是使用一个 HashMap 去存储整个树形结构的数据的，key 是对应的全路径字符串，value 则是一个节点对象，包含了节点的各种信息。

面试官：能说说你觉得为什么要这么设计吗？

（其实我觉得一般面试官不会这么问，以下回答也是我个人的猜想）

我：首先 HashMap 查询速度很快，是 Java 标准库中一个非常重要的数据结构，在许多地方都有用到。ZK 本身并不需要排序或者是范围求值的操作，所以 HashMap 完全可以满足查询的需求。至于为什么逻辑上要设计成树形结构，父子节点，这个可能是因为这个结构和 Unix 文件系统很像，非常便于理解以及基于路径进行数据的分类，而且最新的 ZK 中有一些功能是依赖了父子递归这个特性的（比如 `addWatch`），如果是普通的 key-value 是无法满足的。

面试官：嗯好，那你再说说磁盘上 ZK 是怎么存储数据的呢？

我：ZK 在磁盘上规定了两种文件类型，一种是 log 文件，一种是 snapshot。log 文件是增量记录，负责对每一个写请求进行保存，snapshot 文件是全量记录，是对内存的快照。

面试官：ZK 是怎么保证内存中的数据和磁盘中的数据的一致性呢？

我：真正的强一致性，ZK 无法保证。对于每一次的写请求，ZK 是采取先记录磁盘再修改内存的，所以保证了如果出现意外的话，优先记录磁盘可以尽可能的保证数据的完整。如果 ZK 是正常退出的话，也会强制刷磁盘文件和生成 snapshot，保证了一致性，但如果是非正常退出的话，极端情况下的一部分数据是会丢失的。

面试官：你刚刚也提到了 ZK 本身也可以集群部署的？能多聊一点吗？

我：ZK 的配置文件 `zoo.cfg` 中可以配置其他节点的信息，各个节点通过 `dataDir` 目录下的 `myid` 文件进行区分，不同节点之间可以相互通信，客户端连上集群中的任意一个节点都可以进行通信。

面试官：ZK 集群中有几种不同的角色？你知道吗？

我：有 Leader、Follower、Observer 三种角色。

面试官：说说他们之间的区别吧

我：集群中有且只能有一个 Leader，Leader 负责对整个集群的写请求事务进行提交，在一个集群选出 Leader 之前是无法对外提供服务的。Follower 和 Observer 都只能处理读请求，区别是 Follower 有投票权可以参与 Leader 的竞选，Observer 无法参与 Leader 的竞选。

面试官：那你可以跟我讲讲，选举 Leader 依靠哪些信息吗？

我：每一个节点都会维护三个最重要的信息：epoch、zxid、myid。epoch 代表选举的轮次，优先比较，如果相同则继续比较下一级。zxid 代表本节点处理过的最大事务 ID，越大代表当前节点经手的写请求越多，知道的也就越多，第二优先级比较，如果还相同则比较 myid，myid 整个集群中不能重复，所以最终一定能分出胜负。胜利的节点当选 Leader。

（准确的说，epoch 和 zxid 是一个字段，一个记录在高 32 位，一个记录在低 32 位）

面试官：不同节点之间怎么通信呢？怎么去进行选举？

我：每一个 ZK 节点在启动的时候，会通过读取配置文件中的集群信息，与其他节点建立 Socket 连接，集群间的通信就是通过这个 Socket。每个节点选举的时候都把自己认为的候选人信息广播出去，同时也接收来自其他节点的候选人信息，通过比较后，失败的一方会更改自己的候选人信息并重新进行广播，反复直到某一个节点得到半数以上投票，选举就完成了。

面试官：不同的节点角色，在处理读写请求上有什么不同吗？你先聊聊 Leader 吧

我：好滴，Leader 作为集群中的老大，负责对收到的写请求发起**提案** PROPOSAL，告诉其他节点当前收到一个写请求，其他节点收到后，会在本地进行归档，其实就是写入文件输出流，完毕后会发送一个 ACK 给 Leader，Leader 统计到半数以上的 ACK 之后会再次发送给其他节点一个 COMMIT，其他节点收到 COMMIT 之后就可以修改内存数据了。读请求的话不需要提案直接查询内存中的数据返回即可。

面试官：那 Follower 或 Observer 呢？

我：他们收到读请求是一样的，直接返回本地的内存数据即可。但是写请求的话，会将当前请求转发给 Leader，然后由 Leader 去处理，就和之前的流程是一样的。

面试官：不同的请求 ZK 是如何保证顺序呢？

我：这个顺序的保证最终是落实在一个先进先出的队列，优先进该队列的请求会被先处理，所以能保证顺序。

面试官：不同的客户端的请求怎么保证顺序呢？A 先发送了一个创建节点，在该请求返回之前，B 发送了一个查询该节点，B 会阻塞到 A 执行完毕再查询吗？还是直接返回查询不到节点？

我：B 会直接返回查不到。不同的客户端之间的顺序 ZK 不保证，原因是在底层 ZK 是通过一个 Map 去分别放置不同的客户端的请求的，不同的客户端的 key 是不一样的，而这个 Map 的 value 则是我刚刚提到的先进先出的队列。所以只有同一个客户端的请求能被顺序执行，不同的客户端是无法保证的。

面试官：能说说不同的客户端的 key 是什么吗？怎么保证不同。

我：每一个客户端在连接至 ZK 后会被分配一个 sessionId，这个 sessionId 是通过当前时间戳、节点的 myid 和一个递增特性生成的一个 long 类型字段，可以保证不会重复。

面试官：说到 session，你知道 ZK 的会话是怎么维持的吗？

我：你问的是客户端和服务端之间的会话吗？

面试官：是的，你能跟我说说吗？

我：每一个客户端在连接 ZK 的时候会同时上报自己的超时时间，加上刚刚的 sessionId，ZK 的服务端会在本地维护一个映射关系，通过计算可以计算出该 sessionId 的超时时间，并且 ZK 自己也有一个 `tickTime` 的配置，通过一个算法可以将不同客户端不同超时间都映射到相同间隔的时间点上，再将这个超时时间和 sessionId 关系存起来。

面试官：映射到相同的时间点上有什么好处吗？

我：这样服务端在启动后，后台会有一个线程，通过这个统一的时间间隔，取出 session 过期的客户端，向他们发送会话过期的消息，极大的节约了性能。

面试官：客户端是怎么去更新会话的超时时间呢？

我：首先客户端的每次操作都会刷新这个超时时间，其次客户端必须设计一个 PING 的操作，用于在客户端空闲的时候主动去刷新会话超时时间，防止过期。

面试官：除了客户端和服务端之间的会话，还有别的吗？

我：服务端和服务端之间也有心跳，而且服务端的心跳是由 Leader 主动发起的，向其他节点发送 PING 请求，而其他节点收到 PING 后，需要把本地的会话信息一并发送给 Leader。

---

编不下去了，上面一些题具有我强烈的主观偏好性，我觉得如果面试官是个菜鸡的话，这些问题大部分都问不出来，所以重点是不在于我怎么回答，而是当你对背后的原理了然于胸时，自然是神挡杀神，佛挡杀佛。

我说说我认为比较重要的几个特性：

- 回调通知，ZK 其他原理可以不懂，但是怎么用回调是肯定要知道的。
- 选举，ZK 最具特色的一个属性，基本都会问一下。
- 持久化，说清楚两种文件的区别。
- 会话，会话的概念，以及怎么维持。

最后通过一个模拟面试回答了一下我认为 ZK 中比较有特点的面试问题，如果大家对面试问题还有什么疑问记得留言给我噢～必须给你们安排上！

## 二、网上真题

我大部分题目是网上直接搜的，网址在这里 [ZooKeeper面试题（2020最新版）](https://blog.csdn.net/ThinkWon/article/details/104397719)但是过滤了一些太 low 的题目。

[4. ZooKeeper 怎么保证主从节点的状态同步？](https://blog.csdn.net/ThinkWon/article/details/104397719#4_Zookeeper__78)

我上面说了 Leader 在接受到写请求后，会发起提案，然后等待其他节点的 ACK，这个 ACK 是要求半数以上通过才能继续下去的，所以能收到半数以上的 ACK 说明集群中的一半以上都已经完成了本地磁盘的归档，自然是保证了主从之间的数据同步。

---

[5. 四种类型的数据节点 Znode](https://blog.csdn.net/ThinkWon/article/details/104397719#5__Znode_92)

我之前的文章中有介绍现在 ZK 中有 7 种节点类型，关于新节点的原理我还没来得及讲，所以他如果这么问了你可以很官方的回答他：

- 持久节点
- 持久顺序节点
- 临时节点
- 临时顺序节点

他一般后面会接着问两者的区别，临时节点会随着客户端的会话断开而自动删除，原理就是在创建临时节点的时候，服务端会维护一个 sessionId 和它对应的临时节点路径列表，当关闭会话时，把这个列表里的路径都拿出来一一删除即可。而顺序节点的区别就在于 ZK 会自动为路径加上数字的后缀，仅此而已。

---

并发创建时，顺序节点怎么保证后缀数字唯一呢？

ZK 的请求是放入队列里一个个处理的，所以其实并没有所谓的并发，前一个请求处理完再处理下一个请求，自然就能保证后缀数字的唯一性了。

---

[10. ACL 权限控制机制](https://blog.csdn.net/ThinkWon/article/details/104397719#10_ACL__200)

ZK 讲权限分为两大类，两大类又能继续细分：

- 客户端的角色权限
  - IP
  - 用户名密码
  - world，最宽泛的权限，也就是没有权限
  - super，特殊的用户名密码，相当于管理员权限
- 节点的数据权限
  - Create，创建
  - Delete，删除
  - Read，读
  - Write，写
  - ACL，读写权限

---

[11. Chroot 特性](https://blog.csdn.net/ThinkWon/article/details/104397719#11_Chroot__238)

chroot 是 ZK 设计给客户端的命名空间隔离，作为不同客户端的根节点，由客户端去维护，总的来说就是发送请求之前把 chroot 的路径拼接上，再去请求服务端。chroot 对于服务端是透明的，完全不知道的。

---

[15. 数据同步](https://blog.csdn.net/ThinkWon/article/details/104397719#15__302)

Learner 和 Leader 之间同步数据是一个比较漫长和复杂的过程，总的来说可以大致分为以下步骤：

- Learner 上报自己的信息给 Leader
- Leader 根据 Learner 信息决定使用何种同步方法
  - DIFF，直接从最近的 500 个提案中恢复数据，直接发送提案即可
  - TRUNC，通常出现于 Learner 是前 Leader，需要降级自己的数据达到和 Leader 一致
  - SNAP，Leader 直接发送整个内存快照给 Follower
- Leader 和 Learner 开始同步
- 同步完成后开始对外提供服务

![](./images/1.png)

## 三、配置大全

托大家的福，我把 ZK 的源码全部（爆肝）浏览了一遍，找到了至少 99% 的配置选项，ZK 的配置大致可以分为 3 种：

- 启动命令行传入的参数
- `zoo.cfg` 配置文件中的参数
- 当前环境变量中的参数

![](./images/2.gif)

### 3.1 命令行参数

命令行参数很少，而且没有对应的配置名称，这里我简单介绍下：

单机版只支持两种形式的命令行传参

- 客户端监听端口加 data 目录，上一节源码调试中用的就是这一个形式，例如: `2181 /your/zk/data/path`
- 或者只传一个参数，`zoo.cfg` 的路径，例如：`/your/zoocfg/path`

集群版更简单只支持 `zoo.cfg` 的路径一个参数

### 3.2 zoo.cfg 文件中的配置

我仔细查看源码的时候发现有些配置实际作用时需要计算又或者是一鱼两吃，被多个地方使用，所以很难一步到位的讲清楚，所以下面的介绍仅供参考，配置项加星号（*）的是我未来打算开篇讲解的。

| 配置项                                  | 默认值（单位）                   | 介绍                                                       |
| --------------------------------------- | -------------------------------- | ---------------------------------------------------------- |
| dataDir                                 | /tmp/zookeeper                   | 存放 snapshot、myid 文件路径                               |
| clientPort                              | 2181                             | 监听客户端请求端口                                         |
| tickTime                                | 2000（毫秒）                     | 影响客户端会话检查间隔、服务端之间心跳间隔                 |
| syncLimit                               | 5                                | tickTime * syncLimit 决定了服务端心跳超时时间              |
| initLimit                               | 10                               | tickTime * initLimit 决定了 ACK 的超时时间                 |
| dataLogDir                              | 和 dataDir 一致                  | 存放 log 文件路径                                          |
| minSessionTimeout                       | tickTime * 2                     | 客户端的超时时间最小值                                     |
| maxSessionTimeout                       | tickTime * 20                    | 客户端的超时时间最大值                                     |
| electionAlg                             | 3                                | 选举算法（1，2 已被废弃）                                  |
| localSessionsEnabled*                   | false                            | 启用本地会话                                               |
| localSessionsUpgradingEnabled*          | false                            | 本地会话可以升级成全局会话                                 |
| clientPortAddress                       | -                                | 客户端的 host 要求，不配置的话可以接受任意发向 2181 的请求 |
| secureClientPort                        | -                                | SSL 安全端口号                                             |
| secureClientPortAddress                 | -                                | SSL 安全 host                                              |
| observerMasterPort*                     | -                                | 使 Observer 通过 Follower 去了解集群中的选举情况           |
| clientPortListenBacklog                 | 50                               | TCP 服务端用于临时存放已完成三次握手的请求的队列的最大长度 |
| maxClientCnxns                          | 60                               | 客户端最大连接数                                           |
| connectToLearnerMasterLimit             | 0                                | 决定了 Follower 连接 Leader 的超时时间                     |
| quorumListenOnAllIPs                    | false                            | 服务端是否接受来自任意 IP 地址的请求                       |
| peerType                                | -                                | 选项 observer / participant，决定节点角色                  |
| syncEnabled                             | true                             | Learner 是否需要本地持久化文件                             |
| dynamicConfigFile*                      | -                                | 动态配置路径                                               |
| autopurge.snapRetainCount               | 3                                | 保留多少个最新的 snapshot 文件                             |
| autopurge.purgeInterval                 | 0（小时）                        | 间隔多久进行一次 snapshot 的清理                           |
| standaloneEnabled                       | true                             | 是否允许单机模式启动                                       |
| reconfigEnabled*                        | false                            | 是否允许动态配置                                           |
| sslQuorum                               | false                            | 集群间是否使用 SSL 通信                                    |
| portUnification                         | false                            | 是否允许不安全连接                                         |
| sslQuorumReloadCertFiles                | false                            | 启用密钥更新时自动加载                                     |
| quorum.auth.enableSasl                  | false                            | 启用集群间 SASL 鉴权                                       |
| quorum.auth.serverRequireSasl           | false                            |                                                            |
| quorum.auth.learnerRequireSasl          | false                            |                                                            |
| quorum.auth.learner.saslLoginContext    | QuorumLearner                    |                                                            |
| quorum.auth.server.saslLoginContext     | QuorumServer                     |                                                            |
| quorum.auth.kerberos.servicePrincipal   | zkquorum/localhost               |                                                            |
| quorum.cnxn.threads.size                | 20                               | 集群间异步建立连接线程池最大线程数                         |
| jvm.pause.info-threshold.ms             | 1000（毫秒）                     | INFO 输出暂停统计阈值                                      |
| jvm.pause.warn-threshold.ms             | 10000（毫秒）                    | WARN 输出暂停统计阈值                                      |
| jvm.pause.sleep.time.ms                 | 500（毫秒）                      | JVM 暂停统计线程 sleep 间隔                                |
| jvm.pause.monitor*                      | false                            | 是否启用 JVM 暂停统计                                      |
| metricsProvider.className*              | DefaultMetricsProvider（全路径） | 统计实现类路径                                             |
| multiAddress.enabled                    | false                            |                                                            |
| multiAddress.reachabilityCheckTimeoutMs | 1000（毫秒）                     |                                                            |
| multiAddress.reachabilityCheckEnabled   | true                             |                                                            |
| （以开头）server.*                      |                                  | 集群配置                                                   |
| （以开头）group*                        |                                  | 分组配置                                                   |
| （以开头）weight*                       |                                  | 权重                                                       |
| （以开头）metricsProvider.*             |                                  | 自定义的统计配置                                           |

以上就是 3.6.2 中 `zoo.cfg` 所有的官方配置选项了

### 3.3 环境变量配置

Java 程序想要指定环境变量有两种方法：

- 只需要在启动的时候在后面加上 `-DpropertyKey=propertyValue` 即可
- ZK 还支持一种简单的方式就是在 `zoo.cfg` 中直接指定（指定时不需要写 `zookeeper.` 的前缀）。只要不是在上面 2.2 中 ZK 自己定义的配置项里，ZK 启动的时候读取这些配置会自动帮他们添加 `zookeeper.` 前缀并加入当前环境变量中

如果该配置是 `follower.nodelay`，就只能用第一种方式添加环境变量了。

让我们也来看看 ZK 自己定义了哪些环境变量配置吧

| 配置项                                                 | 默认值                         | 配置                                         |
| ------------------------------------------------------ | ------------------------------ | -------------------------------------------- |
| zookeeper.server.realm*                                | -                              | 客户端配置                                   |
| zookeeper.clientCnxnSocket                             | ClientCnxnSocketNIO（全路径）  | 客户端配置，通信的实现类                     |
| zookeeper.client.secure                                | true                           | 客户端配置                                   |
| zookeeper.request.timeout                              | 0                              | 客户端配置，异步 API 超时时间                |
| zookeeper.server.principal                             | -                              | 客户端配置                                   |
| zookeeper.sasl.client.username                         | zookeeper                      | 客户端配置                                   |
| zookeeper.sasl.client.canonicalize.hostname            | true                           | 客户端配置                                   |
| zookeeper.disableAutoWatchReset                        | false                          | 客户端配置，会话超时自动清空 watcher         |
| zookeeper.sasl.clientconfig                            | -                              |                                              |
| zookeeper.sasl.client                                  | true                           | 启用 SASL                                    |
| zookeeper.ssl(.quorum).authProvider                    | x509                           | SSL 实现类，加 quorum 的是服务端的配置，下同 |
| zookeeper.ssl(.quorum).protocol                        | TLSv1.2                        |                                              |
| zookeeper.ssl(.quorum).enabledProtocols                | -                              |                                              |
| zookeeper.ssl(.quorum).ciphersuites                    | 根据不同的 jvm 版本            |                                              |
| zookeeper.ssl(.quorum).keyStore.location               | -                              |                                              |
| zookeeper.ssl(.quorum).keyStore.password               | -                              |                                              |
| zookeeper.ssl(.quorum).keyStore.type                   | -                              |                                              |
| zookeeper.ssl(.quorum).trustStore.location             | -                              |                                              |
| zookeeper.ssl(.quorum).trustStore.password             | -                              |                                              |
| zookeeper.ssl(.quorum).trustStore.type                 | -                              |                                              |
| zookeeper.ssl(.quorum).context.supplier.class          | -                              |                                              |
| zookeeper.ssl(.quorum).hostnameVerification            | true                           |                                              |
| zookeeper.ssl(.quorum).crl                             | false                          |                                              |
| zookeeper.ssl(.quorum).ocsp                            | false                          |                                              |
| zookeeper.ssl(.quorum).clientAuth                      | -                              |                                              |
| zookeeper.ssl(.quorum).handshakeDetectionTimeoutMillis | 5000（毫秒）                   |                                              |
| zookeeper.kinit                                        | /usr/bin/kinit                 |                                              |
| zookeeper.jmx.log4j.disable                            | false                          | 禁用 jmx log4j                               |
| zookeeper.admin.enableServer*                          | true                           | 是否启用 Admin Server                        |
| zookeeper.admin.serverAddress*                         | 0.0.0.0                        |                                              |
| zookeeper.admin.serverPort*                            | 8080                           |                                              |
| zookeeper.admin.idleTimeout*                           | 30000                          |                                              |
| zookeeper.admin.commandURL*                            | /commands                      |                                              |
| zookeeper.admin.httpVersion*                           | 11                             |                                              |
| zookeeper.admin.portUnification                        | false                          |                                              |
| zookeeper.DigestAuthenticationProvider.superDigest     | -                              | 管理员账号密码                               |
| zookeeper.ensembleAuthName                             | -                              |                                              |
| zookeeper.requireKerberosConfig                        | -                              |                                              |
| zookeeper.security.auth_to_local                       | DEFAULT                        |                                              |
| （以开头）zookeeper.authProvider.*                     | -                              | 自定义 scheme 校验规则                       |
| zookeeper.letAnySaslUserDoX                            | -                              |                                              |
| zookeeper.SASLAuthenticationProvider.superPassword     | -                              |                                              |
| zookeeper.kerberos.removeHostFromPrincipal             | -                              |                                              |
| zookeeper.kerberos.removeRealmFromPrincipal            | -                              |                                              |
| zookeeper.X509AuthenticationProvider.superUser         | -                              |                                              |
| zookeeper.4lw.commands.whitelist*                      | -                              | 四字命令白名单                               |
| zookeeper.preAllocSize                                 | 65536 * 1024                   |                                              |
| zookeeper.forceSync                                    | yes                            |                                              |
| zookeeper.fsync.warningthresholdms                     | 1000（毫秒）                   | fsync 告警阈值                               |
| zookeeper.txnLogSizeLimitInKb                          | -1（KB）                       | log 文件大小                                 |
| zookeeper.datadir.autocreate                           | true                           | data 目录自动创建                            |
| zookeeper.db.autocreate                                | true                           |                                              |
| zookeeper.snapshot.trust.empty                         | false                          | 不信任空的 snapshot 文件                     |
| zookeeper.snapshot.compression.method                  | 空字符串                       | snapshot 文件压缩实现                        |
| zookeeper.commitProcessor.numWorkerThreads             | CPU 核心数                     |                                              |
| zookeeper.commitProcessor.shutdownTimeout              | 5000（毫秒）                   |                                              |
| zookeeper.commitProcessor.maxReadBatchSize             | -1                             |                                              |
| zookeeper.commitProcessor.maxCommitBatchSize           | 1                              |                                              |
| zookeeper.fastleader.minNotificationInterval           | 200（毫秒）                    | 收集选票超时时间（初始）                     |
| zookeeper.fastleader.maxNotificationInterval           | 60000（毫秒）                  | 收集选票超时时间（最大）                     |
| zookeeper.leader.maxTimeToWaitForEpoch                 | -1                             |                                              |
| zookeeper.leader.ackLoggingFrequency                   | 1000                           |                                              |
| zookeeper.testingonly.initialZxid                      | -                              | 初始化 zxid，仅供测试！                      |
| zookeeper.leaderConnectDelayDuringRetryMs              | 100                            | Leaner 连接 Leader 超时时间                  |
| follower.nodelay                                       | true                           | 设置 TCP no delay                            |
| zookeeper.forceSnapshotSync                            | false                          | Learner 强制使用 snapshot 和 Leader 进行同步 |
| zookeeper.leader.maxConcurrentSnapSyncs                | 10                             |                                              |
| zookeeper.leader.maxConcurrentDiffSyncs                | 100                            |                                              |
| zookeeper.observer.reconnectDelayMs                    | 0（毫秒）                      | Observer 延迟重连至 Leader                   |
| zookeeper.observer.election.DelayMs                    | 200（毫秒）                    | Observer 延迟开始选举                        |
| zookeeper.observerMaster.sizeLimit                     | 32 * 1024 * 1024               |                                              |
| zookeeper.electionPortBindRetry                        | 3                              | 选举端口连接重试次数                         |
| zookeeper.tcpKeepAlive                                 | false                          | Socket keep alive 设置                       |
| zookeeper.cnxTimeout                                   | 5000（毫秒）                   | Socket 超时时间                              |
| zookeeper.multiAddress.enabled                         | false                          |                                              |
| zookeeper.multiAddress.reachabilityCheckTimeoutMs      | 1000（毫秒）                   |                                              |
| zookeeper.multiAddress.reachabilityCheckEnabled        | true                           |                                              |
| zookeeper.quorumCnxnTimeoutMs                          | -1                             |                                              |
| zookeeper.observer.syncEnabled                         | true                           | Observer 是否需要本地归档                    |
| zookeeper.bitHashCacheSize                             | 10                             | 位图初始缓存大小                             |
| zookeeper.messageTracker.BufferSize                    | 10                             |                                              |
| zookeeper.messageTracker.Enabled                       | false                          |                                              |
| zookeeper.pathStats.slotCapacity                       | 60                             |                                              |
| zookeeper.pathStats.slotDuration                       | 15                             |                                              |
| zookeeper.pathStats.maxDepth                           | 6                              |                                              |
| zookeeper.pathStats.sampleRate                         | 0.1                            |                                              |
| zookeeper.pathStats.initialDelay                       | 5                              |                                              |
| zookeeper.pathStats.delay                              | 5                              |                                              |
| zookeeper.pathStats.topPathMax                         | 20                             |                                              |
| zookeeper.pathStats.enabled                            | false                          |                                              |
| zookeeper.watcherCleanThreshold                        | 1000                           |                                              |
| zookeeper.watcherCleanIntervalInSeconds                | 600                            |                                              |
| zookeeper.watcherCleanThreadsNum                       | 2                              |                                              |
| zookeeper.maxInProcessingDeadWatchers                  | -1                             |                                              |
| zookeeper.watchManagerName                             | WatchManager（全路径）         |                                              |
| zookeeper.connection_throttle_tokens                   | 0                              |                                              |
| zookeeper.connection_throttle_fill_time                | 1                              |                                              |
| zookeeper.connection_throttle_fill_count               | 1                              |                                              |
| zookeeper.connection_throttle_freeze_time              | -1                             |                                              |
| zookeeper.connection_throttle_drop_increase            | 0.02                           |                                              |
| zookeeper.connection_throttle_drop_decrease            | 0.002                          |                                              |
| zookeeper.connection_throttle_decrease_ratio           | 0                              |                                              |
| zookeeper.connection_throttle_weight_enabled           | false                          |                                              |
| zookeeper.connection_throttle_global_session_weight    | 3                              |                                              |
| zookeeper.connection_throttle_local_session_weight     | 1                              |                                              |
| zookeeper.connection_throttle_renew_session_weight     | 2                              |                                              |
| zookeeper.extendedTypesEnabled*                        | false                          | 是否启用 TTL 节点类型                        |
| zookeeper.emulate353TTLNodes*                          | false                          | 是否兼容 3.5.3 的 TTL                        |
| zookeeper.client.portUnification                       | false                          |                                              |
| zookeeper.netty.server.outstandingHandshake.limit      | -1                             |                                              |
| zookeeper.netty.advancedFlowControl.enabled            | false                          |                                              |
| zookeeper.nio.sessionlessCnxnTimeout                   | 10000（毫秒）                  |                                              |
| zookeeper.nio.numSelectorThreads                       | CPU 核心数 / 2 再开方          |                                              |
| zookeeper.nio.numWorkerThreads                         | CPU 核心数 * 2                 |                                              |
| zookeeper.nio.directBufferBytes                        | 64 * 1024（字节）              |                                              |
| zookeeper.nio.shutdownTimeout                          | 5000（毫秒）                   |                                              |
| zookeeper.request_stale_connection_check               | true                           |                                              |
| zookeeper.request_stale_latency_check                  | false                          |                                              |
| zookeeper.request_throttler.shutdownTimeout            | 10000（毫秒）                  |                                              |
| zookeeper.request_throttle_max_requests                | 0                              |                                              |
| zookeeper.request_throttle_stall_time                  | 100                            |                                              |
| zookeeper.request_throttle_drop_stale                  | true                           |                                              |
| zookeeper.serverCnxnFactory                            | NIOServerCnxnFactory（全路径） |                                              |
| zookeeper.maxCnxns                                     | 0                              |                                              |
| zookeeper.snapshotSizeFactor                           | 0.33                           |                                              |
| zookeeper.commitLogCount                               | 500                            |                                              |
| zookeeper.sasl.serverconfig                            | Server                         |                                              |
| zookeeper.globalOutstandingLimit                       | 1000                           |                                              |
| zookeeper.enableEagerACLCheck                          | false                          |                                              |
| zookeeper.skipACL                                      | no                             |                                              |
| zookeeper.allowSaslFailedClients                       | false                          |                                              |
| zookeeper.sessionRequireClientSASLAuth                 | false                          |                                              |
| zookeeper.digest.enabled                               | true                           |                                              |
| zookeeper.closeSessionTxn.enabled                      | true                           |                                              |
| zookeeper.flushDelay                                   | 0                              |                                              |
| zookeeper.maxWriteQueuePollTime                        | zookeeper.flushDelay / 3       |                                              |
| zookeeper.maxBatchSize                                 | 1000                           |                                              |
| zookeeper.intBufferStartingSizeBytes                   | 1024                           |                                              |
| zookeeper.maxResponseCacheSize                         | 400                            |                                              |
| zookeeper.maxGetChildrenResponseCacheSize              | 400                            |                                              |
| zookeeper.snapCount                                    | 100000                         |                                              |
| zookeeper.snapSizeLimitInKb                            | 4194304（千字节）              |                                              |
| zookeeper.largeRequestMaxBytes                         | 100 * 1024 * 1024              |                                              |
| zookeeper.largeRequestThreshold                        | -1                             |                                              |
| zookeeper.superUser                                    | -                              |                                              |
| zookeeper.audit.enable                                 | false                          | 是否启用 audit 日志                          |
| zookeeper.audit.impl.class                             | Log4jAuditLogger（全路径）     | audit 日志功能实现类                         |

<img src="./images/3.jpeg" style="zoom:67%;" />

ZK 的配置还是很多的，有些我这里 TODO 了，以后有机会和大家详细介绍下～而且相当一部分的配置 ZK 官方的文档中已经给出了解释，可以查看 [ZK 3.6.2 配置文档](https://zookeeper.apache.org/doc/r3.6.2/zookeeperAdmin.html)。

我这里还要吐槽下，ZK 中有些配置是用 true 或者 false，有些使用 yes 或者 no，明显是两个（波）人开发的，这种不应该做一个统一吗？yes 或 no 真的很多余...

## 四、系列结语

感谢你们能看到这里，陪伴这个系列从开始到现在！这个项目从有想法立项到之后跟蛋蛋沟通，再到正式开始编写，到最后我写下这段结语，大概经历了三个多月（你们看到的时候应该是更晚），现在回头再看之前写的东西，感慨颇深。

（截图来自-**B 站何同学**的视频）

<img src="./images/4.png" style="zoom:67%;" />

如果以我自己的自控能力，这玩意自己搞，搞着搞着可能就凉了，在此感谢**蛋蛋**给予我的帮助和鼓励。关于 ZK 我的确之前有研究过一段时间，但是以现在的眼光看，当时的研究其实也就是皮毛而已（可能现在也还是），很多东西是我这次整理时现学的，收获非常多，最直观的感受就是，我以后出去面试不会再害怕 ZK 相关的问题了。

限制于篇幅，本篇是 HelloZooKeeper 系列的最后一篇，但是也只是暂告一段落，因为我的目标是努力拼凑出完整的 ZK 知识版图，而之后的知识点会以单篇的形式进行更新，期待一下吧～ 

最后再来个赞吧！

![](./images/5.png)