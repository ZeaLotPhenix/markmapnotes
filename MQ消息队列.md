# MQ消息队列
## 面试题 <!-- markmap: fold -->
### 相关面试题
#### 面试题顺序
- 90 278 kafka零拷贝原理
- 194 什么是消息队列
- 333 kafka如何保证消息不丢失
- 476 kafka如何保证消息消费的顺序性
- 581 阻塞队列被异步消费怎么保持顺序
- 609 RabbitMQ 如何实现高可用
- 673 如何处理消息队列的积压问题
- 693 kafka消息怎么保证exactlyOnce,怎么保证顺序消费
- 695 kafka中partition分区副本的leader选举算法
- 697 kakfa中一个topic有三个partition,同一个消费组中两个消费者如何消费的
- 949 请讲对RabbitMq工作原理的理解
- 954 如何保证RabbitMQ的高可用
#### 面试题分类
##### kafka
- 90 278 kafka零拷贝原理
- 194 什么是消息队列
- 333 kafka如何保证消息不丢失
- 476 kafka如何保证消息消费的顺序性
- 693 kafka消息怎么保证exactlyOnce,怎么保证顺序消费
- 695 kafka中partition分区副本的leader选举算法
- 697 kakfa中一个topic有三个partition,同一个消费组中两个消费者如何消费的
## 消息中间件
### 常见协议
#### AMQP协议(Advanced Message Queuing Protocol)
- 应用层协议,定义消息队列系统的消息格式/传输方式/处理机制
- 特性
    - 面向消息,异步传输
    - 高可靠性/可拓展型/跨平台
    - 不支持消费者组,每条消息只能被一个消费者消费
- 组成部分
    - 连接(Connection) <!-- markmap: foldAll -->
        - 客户端与消息代理(如RabbitMQ)建立的网络连接,长连接
    - 信道(Channel) <!-- markmap: foldAll -->
        - 在连接内创建的逻辑通信链路,支持多信道复用一个连接,实现多路复用
    - 交换机(Exchange) <!-- markmap: foldAll -->
        - 负责向消息路由到绑定的队列,支持多种路由策略
            - Direct Exchange(默认) 根据精确匹配的路由键路由到队列
            - Fanout Exchange 广播道所有绑定的队列,不考虑路由键
            - Topic Exchange 根据路由键的模式匹配将消息路由到对应队列,支持模糊匹配`order.*`
            - Headers Exchange 根据消息头中的键值匹配路由,不依赖路由键
        - Routing Key 由生产者指定,消息被路由到哪些队列
        - Binding Key 在交换机与队列绑定时指定,用于消息路由
    - 队列(Queue) <!-- markmap: foldAll -->
        - 实际存储传输的消息,消费者从队列中拉取消息处理
    - 绑定(Binding) <!-- markmap: foldAll -->
        - 定义交换机与队列的关系,通过路由键实现消息路由
- 可靠传输机制
    - 消息确认机制(Ackonwledge)
        - 消费者接受消息后需要确认,如果确认失败,消息不会从队列移除
    - 消息持久化机制(Durable)
        - 队列和消息可以被持久化,宕机后不消失
    - 事务支持机制(Transactional)
        - AMQP支持事务机制,生产者可以通过`txSelect`开启事务,`txCommit`和`txRollback`控制消息的提交和回滚
#### MQTT
#### JMS(Java标准消息协议)
### 常见中间件
#### 对比
- 中间件对比
    |消息中间件|模式|支持协议|顺序消息|定时消息|批量消息|消息广播|消息过滤|消息回溯|优先级消息|消息追踪|
    |:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
    |ActiveMQ|Push|AMQP,MQTT,JMS|单消费者或单队列有序|是|否|是|是|是|是|否|
    |Kafka|Pull|TCP|partition内有序|否|是,异步|否|KafkaSteam支持|支持offset|否|否|
    |RocketMQ|Pull|TCP,JMS,OpenMessaging|严格有序,支持扩容|是|是,异步|是|支持|支持时间戳和offset|否|是|
#### Kafka
- 架构概览
    - [官方文档](https://kafka.apache.org/documentation/#design)
    - 架构图 <!-- markmap: foldAll -->
        - ![KafkaMQ架构图](https://developer.qcloudimg.com/http-save/yehe-3335805/7b71e45172c998e4b0dad51af096ae51.png)
    - 基本概念
        - Topic主题
        消息发布的分类单位和数据流分类单位,包含有至少一个分区,消息仅在各自分区内有序
            - Partition分区
            一个有序、不可变且持续追加的提交日志
                - 持久化
                Kafka集群会按照可配置的保留时长，持久化存储所有已发布的消息记录——无论这些记录是否已被消费
                - 容错性
                每个分区可以有零或多个副本,存储在其他的kafka server上
                - 消息Record
                最小数据传输单元
                - Offset消费位点
                由Kafka的内部主题_consumer_offsets来保存所有消费者或消费者组的消费位点
        - [Producer生产者](https://kafka.apache.org/documentation/#theproducer)
        Kafka由生产者负责消息的负载均衡,默认轮询算法,保证相同key的消息始终进入同一分区(有序性)
            - 客户端负载均衡
                - 生产者直接向leader分区的broker发送消息
                - 客户端从服务端拉取对应Topic的可用Broker元信息
                - 支持轮询/随机和自定义语义分区等策略
            - 异步发送
                - 通过可配置的缓冲区大小,将待发送的消息缓存后打包压缩发送
        - Consumer消费者组
        Consumer消费者构成了消费者组,一个消息会只会被所有同名消费者组里的一个消费者消费,但可以广播到所有不同名的消费者组
            - 长轮询优化: 通过可配置的最小抓取字节数或者最大等待时间避免无数据时空转
            - 消费点位追踪[Consumer Offset Tracking](https://kafka.apache.org/documentation/#impl_offsettracking)
                - 组协调器(group coordinator)
                Kafka可为每个消费者组提供一个名为组协调器(group coordinator)的broker用以专门保存和同步该组的offset位点信息,消费者组通过组名与之映射.broker为__consumer_offsets主题每个partition对应的leader分区broker
                - 内部特殊主题(__consumer_offsets)
                组协调器接受到消费者的OffsetCommitRequest请求后,会向一个名为__consumer_offsets的kafka的特殊主题发送消息用以记录消费位点,只有当该主题的所有副本都写入了该位点消息后才会返回成功.超时失败时,consumer会重试发送请求.
                    - 组协调器会也会将offset缓存在内存中来让消费者快速fetch获取已消费位点,当协调器初始化或者被重新选举为Leader时会重新拉取位点刷新缓存
                    - __consumer_offsets主题会定期压缩主题内的消息,仅需维持每个partition最新的消费位点
    - 设计原理
        - [网络层](https://kafka.apache.org/documentation/#networklayer)
            - 线程模型:单线程接受器+多处理器线程
    - 特性
        - [消息批处理](https://kafka.apache.org/documentation/#maximizingefficiency)
        Kafka的生产者会将消息打包压缩成消息集在一次请求中批量发送,服务端会批量将消息块追加到日志中，而消费者也能一次性获取大段连续的消息块
            - 形成更大的网络数据包
            - 实现更大批量的顺序磁盘写入
            - 分配连续的内存区块
            - 将随机写优化为了顺序写
            - 消息集始终都是以压缩包的粒度发送/存储(服务端仅解压校验,存储在分区中仍然是压缩包格式)/消费的
        - 数据零拷贝
        通过自定义的生产者、Broker和消费者三者统一的标准化二进制消息格式,实现了数据块无需修改即可直接传输,避免了序列化的开销
            - 借助现代Linux系统的零拷贝传输机制[sendfiles system call](https://man7.org/linux/man-pages/man2/sendfile.2.html)直接在内核空间完成数据页到Socket的拷贝,绕过了用户空间缓冲区
        - [Push-Pull模式](https://kafka.apache.org/documentation/#design_pull)
        Kafka采取了生产者Push推送消息,消费者Pull拉取消息的模式
            - 允许消费者和生产者以不同的速率处理和生产消息
        - 事务支持
        - [流式处理](https://kafka.apache.org/documentation/#streams)
        Kakfa提供的一个轻量级流处理库,用以处理和分析由kakfa存储的数据
            - 支持特性
                - 状态管理（如聚合、连接操作）
                - 窗口处理（时间窗口、会话窗口）
                - Exactly-once语义
                - 基于DSL（filter、map、join等）和Processor API的低级控制
#### RocketMQ
- 架构概览
    - [官方文档](https://rocketmq.apache.org/zh/docs/quickStart/01quickstart)
    - 架构图 <!-- markmap: foldAll -->
        - ![RocketMQ架构图](https://developer.qcloudimg.com/http-save/yehe-11130581/8a3810d27d4ab3572cf0be800a2f9401.png)
    - 基本概念
        - Topic主题
        消息传输和存储的顶层容器，用于标识同一类业务逻辑的消息
            - 行为约束
                - 消息类型必须一致：发送的消息的类型，必须和目标主题定义的消息类型一致
                - 主题类型必须单一：每个主题只支持一种消息类型，不允许将多种类型的消息发送到同一个主题中
            - 消息类型
            按照消息传输特性的不同而定义的分类，用于类型管理和安全校验
                - 普通消息
                数据传输通道具有可靠传输的能力，且对消息的处理时机、处理顺序没有特别要求
                    - 微服务解耦: 上下游应用拆分
                    - 数据集成传输: 日志收集
                - 顺序消息
                可以有效保证数据传输的顺序性
                    - 股票交易撮合: 先出价先交易
                    - 数据实时增量同步: 基于顺序消息可以实现下游状态和上游操作结果一致
                    - 功能原理
                        - 生产顺序性
                        通过生产者和服务端的协议保障单个生产者串行地发送消息，并按序存储和持久化
                            - 单一生产者：消息生产的顺序性仅支持单一生产者
                            - 串行发送：Apache RocketMQ 生产者客户端支持多线程安全访问，但如果生产者使用多线程并行发送，则不同线程间产生的消息将无法判定其先后顺序
                        - 服务端顺序性
                        保证设置了同一消息组的消息，按照发送顺序存储在同一队列中
                            - 相同消息组的消息按照先后顺序被存储在同一个队列
                            - 不同消息组的消息可以混合在同一个队列中，且不保证连续
                        - 消费顺序性
                        消费严格按照存储的先后顺序来处理
                            - 投递顺序
                                - 为PushConsumer时， Apache RocketMQ 保证消息按照存储顺序一条一条投递给消费者
                                - SimpleConsumer，则消费者有可能一次拉取多条消息。此时，消息消费的顺序性需要由业务方自行保证
                            - 有限重试
                    - 仅支持使用MessageType为FIFO的主题
                - 事务消息
                事务消息保证本地主分支事务和下游消息发送事务的一致性，但不保证消息消费结果和上游事务的一致性,是为最终一致性，即在消息提交到下游消费端处理完成之前，下游分支和上游事务之间的状态会不一致。因此，事务消息仅适合接受异步执行的事务场景
                    - 事务消息交互流程图 <!-- markmap: foldAll -->
                        - ![流程图](https://rocketmq.apache.org/zh/assets/images/transflow-0b07236d124ddb814aeaf5f6b5f3f72c.png)
                    - 关键流程
                        - 事务待提交：半事务消息被发送到服务端，和普通消息不同，并不会直接被服务端持久化，而是会被单独存储到事务存储系统中，等待第二阶段本地事务返回执行结果后再提交。此时消息对下游消费者不可见。
                        - 消息回滚：第二阶段如果事务执行结果明确为回滚，服务端会将半事务消息回滚，该事务消息流程终止。
                        - 提交待消费：第二阶段如果事务执行结果明确为提交，服务端会将半事务消息重新存储到普通存储系统中，此时消息对下游消费者可见，等待被消费者获取并消费。
                        - 消费中：消息被消费者获取，并按照消费者本地的业务逻辑进行处理的过程。 此时服务端会等待消费者完成消费并提交消费结果，如果一定时间后没有收到消费者的响应，Apache RocketMQ会对消息进行重试处理。具体信息，请参见消费重试。
                        - 消费提交：消费者完成消费处理，并向服务端提交消费结果，服务端标记当前消息已经被处理（包括消费成功和失败）。 Apache RocketMQ默认支持保留所有消息，此时消息数据并不会立即被删除，只是逻辑标记已消费。消息在保存时间到期或存储空间不足被删除前，消费者仍然可以回溯消息重新消费。
                    - 仅支持在 MessageType 为 Transaction 的主题
                - 定时消息
                消息被发送至服务端后，在指定时间后才能被消费者消费
                    - 分布式定时调度
                    传统基于数据库的定时调度方案在分布式场景下，性能不高，实现复杂。基于 Apache RocketMQ 的定时消息可以封装出多种类型的定时触发器。
                    - 任务超时处理
                        - 精度高、开发门槛低：基于消息通知方式不存在定时阶梯间隔。可以轻松实现任意精度事件触发，无需业务去重
                        - 高性能可扩展：传统的数据库扫描方式较为复杂，需要频繁调用接口扫描，容易产生性能瓶颈。 Apache RocketMQ 的定时消息具有高并发和水平扩展的能力
                    - 定时时间设置原则
                        - 定时时间必须设置在定时时长范围内，否则立即投递消息
                        - 最大值默认为24小时，不支持自定义修改
                        - 必须设置为当前时间之后，否则立即投递消息
                        - 仅支持在 MessageType为Delay 的主题内使用
                    - 执行流程
                        - 定时中：消息被发送到服务端，和普通消息不同的是，服务端不会直接构建消息索引，而是会将定时消息单独存储在定时存储系统中，等待定时时刻到达
                        - 待消费：定时时刻到达后，服务端将消息重新写入普通存储引擎，对下游消费者可见，等待消费者消费的状态
                - 消息队列
            - MessageQueue消息队列
            消息存储和传输的实际容器，也是消息的最小存储单元
                - 每个主题由多个队列组成,故可支持队列数量的水平拆分和队列内部流式存储
                - 存储顺序性: 队列天然具备顺序性，即消息按照进入队列的顺序写入存储，同一队列间的消息天然存在顺序关系，队列头部为最早写入的消息，队列尾部为最新写入的消息。消息在队列中的位置和消息之间的顺序通过消息位点（Offset）进行标记管理
                - 流式操作语义: Apache RocketMQ 基于队列的存储模型可确保消息从任意位点读取任意数量的消息，以此实现类似聚合读取、回溯读取等特性，这些特性是RabbitMQ、ActiveMQ等非队列存储模型不具备的
                - Message消息
                最小数据传输单元
                    - 消息不可变性
                    - 消息持久化: 默认对消息进行持久化于服务端的存储文件中，保证消息的可回溯性和系统故障场景下的可恢复性
                    - MessageTag消息标签
                    - MessageQueueOffset消息位点: 每条消息在队列中的一个唯一的Long类型坐标
                    - ConsumerOffset消费位点: 基于每个消费者分组维护一份消费记录，该记录指定消费者分组消费某一个队列时，消费过的最新一条消息的位点，即消费位点
                        - 重置消费位点
                            - 初始消费位点不符合需求：因初始消费位点为当前队列的最大消息位点，即客户端会直接从最新消息开始消费。若业务上线时需要消费部分历史消息，您可以通过重置消费位点功能消费到指定时刻前的消息
                            - 消费堆积快速清理：当下游消费系统性能不足或消费速度小于生产速度时，会产生大量堆积消息。若这部分堆积消息可以丢弃，您可以通过重置消费位点快速将消费位点更新到指定位置，绕过这部分堆积的消息，减少下游处理压力
                    - MessageKey消息索引
        - Producer生产者
            - 发送方式
                - 同步发送: 调用线程会一直阻塞，直到某次重试成功或最终重试失败，抛出错误码和异常
                - 异步发送: 调用线程不会阻塞，但调用结果会通过异常事件或者成功事件返回
            - 批量发送
            - 发送重试
            - 消息流控:系统容量或水位过高， Apache RocketMQ 服务端会通过快速失败返回流控错误来避免底层资源承受过高压力
                - 存储压力大: 例如业务上新等需要回溯到指定时刻前开始消费，此时队列的存储压力会瞬间飙升，触发消息流控
                - 服务端请求任务排队溢出: 消费能力不足，导致队列中有大量堆积消息，当堆积消息超过一定数量后会触发消息流控，减少下游消费系统压力
                - 解决方案:
                    - 如何避免: 利用可观测性功能监控系统水位容量等，保证底层资源充足，避免触发流控机制
                    - 突发处理: 建议业务方将请求调用临时替换到其他系统进行应急处理
        - 消费者组ConsumerGroup
        承载多个消费行为一致的消费者的负载均衡分组
            - 并不是运行实体，而是一个逻辑资源
            - 消费者分组内初始化多个消费者实现消费性能的水平扩展以及高可用容灾
            - 消费重试
            - 消费者负载均衡策略
            将主题内的消息分配给指定消费者分组中的多个消费者共同分担
                - 容灾策略
                - 顺序性机制
                同一消息组内的多个消息之间的先后顺序,顺序3,4的消息必须先等拿到顺序1,2的消息的消费者消费完成后才对于同一组内其他的消费者可见
                - 水平拆分策略
                - 消费组内负载均衡
                    - 消息粒度负载均衡
                    同一个队列中的消息，可被平均分配给多个消费者共同消费
                    基于内部的单条消息确认语义实现的。消费者获取某条消息后，服务端会将该消息加锁，保证这条消息对其他消费者不可见，直到该消息消费成功或消费超时
                        - 流程原理图<!-- markmap: foldAll -->
                            - ![原理图](https://rocketmq.apache.org/zh/assets/images/fifoinclustermode-60b2f917ab49333f93029cee178b13f0.png)
                        - PushConsumer默认负载策略
                        - SimpleConsumer默认负载策略
                    - 队列粒度负载均衡：
                    更适合对于流式处理、聚合计算场景，需要明确地对消息进行聚合、批处理
                        - 流程原理图<!-- markmap: foldAll -->
                            - ![原理图](https://rocketmq.apache.org/zh/assets/images/clusterqueuemode-ce4f88dc594c1237ba95db2fa9146b8c.png)
                        - PullConsumer默认负载策略
        - 消费者
        处理消息时主要经过以下阶段：消息获取--->消息处理--->消费状态提交/投递死信队列
            - 必须关联一个消费者组
            - 类型
                - PushConsumer
                消费消息仅通过消费监听器处理业务并返回消费结果。消息的获取、消费状态提交以及消费重试都通过 Apache RocketMQ 的客户端SDK完成
                - SimpleConsumer
                接口原子型的消费者类型，消息的获取、消费状态提交以及消费重试都是通过消费者业务逻辑主动发起调用完成
                    - 处理顺序消息时，会按照消息存储的先后顺序获取消息。即需要保持顺序的一组消息中，如果前面的消息未处理完成，则无法获取到后面的消息
                - PullConsumer(流处理场景)         
        - 订阅关系
        系统中消费者获取消息、处理消息的规则和状态配置,指定某个消费者分组对于某个主题的订阅
            - 消息过滤
            按照过滤规则匹配主题中的消息，只将符合条件的消息投递给消费者消费，实现消息的再次分类
                - TAG过滤：按照Tag字符串进行全文过滤匹配
                    - Tag标签设置
                        - Tag由生产者发送消息时设置，每条消息允许设置一个Tag标签
                        - Tag使用可见字符，建议长度不超过128字符
                    - Tag标签过滤规则: 精准字符串匹配,支持单,多,全匹配
                - SQL92过滤：按照SQL语法对消息属性进行过滤匹配
                    - 通过生产者为消息设置的属性（Key）及属性值（Value）进行匹配。生产者在发送消息时可设置多个属性，消费者订阅时可设置SQL语法的过滤表达式过滤多个属性
        - NameServer路由中心
        负责RocketMQ的路由管理、 服务注册及服务发现及元数据存储
            - 工作流程
                - 负责接受Broker启动时的注册信息
                - 接受生产者拉取Broker地址列表
                - 接受消费者拉取注册表
#### QMQ
- 架构盖伦
    - 架构图 <!-- markmap: foldAll -->
        - ![Qmq架构图](https://img2024.cnblogs.com/blog/3556556/202504/3556556-20250422005827488-448676790.png)
            - 交互流程
                1. delay server 向meta server注册
                2. 实时server 向meta server注册
                3. producer在发送消息前需要询问meta server获取server list
                4. meta server返回server list给producer(根据producer请求的消息类型返回不同的server list)
                5. producer发送延时/定时消息
                6. 延时时间已到，delay server将消息投递给实时server
                7. producer发送实时消息
                8. consumer需要拉取消息，在拉取之前向meta server获取server list(只会获取实时server的list)
                9. meta server返回server list给consumer
                10. consumer向实时server发起pull请求
                11. 实时server将消息返回给consumer
    - 基本概念
        - meta server
        提供集群管理和集群发现的作用
        - server
        提供实时消息服务
        - delay server
        提供延时/定时消息服务，延时消息先在delay server排队，时间到之后再发送给server
        - producer消息生产者
        - consumer消息消费者
        - log消息存储模型
        Qmq使用消息记录log来取代partition,实现了Consumer的弹性扩展和类似RocketMQ的消息合并精简设计
            - 架构图 <!-- markmap: foldAll -->
                - ![Qmq存储模型](https://img2024.cnblogs.com/blog/3556556/202504/3556556-20250422011142699-503857730.png)
                    - 图片解释
                        - message log方框上方的数字:3,6,9几表示这几条消息在message log中的偏移
                        - consume log中方框内的数字3,6,9,20正对应着message log的偏移，表示这几个位置上的消息都是subject1的消息，consume log方框上方的1,2,3,4表示这几个方框在consume log中的逻辑偏移
                        - pull log方框内的内容对应着consume log的逻辑偏移，而pull log方框外的数字表示pull log的逻辑偏移
            - message log
            所有subject的消息进入该log，消息的主存储
            - consume log(类似Topic)
            consume log存储的是message log的索引信息
            - pull log(类似_consumer_offsets)
            每个consumer拉取消息的时候会产生pull log，pull log记录的是拉取的消息在consume log中的sequence,代表消费位点记录
    - 特性
        - 延迟/定时消息
        QMQ提供任意时间的延时/定时消息，你可以指定消息在未来两年内(可配置)任意时间内投递,比RocketMQ固定延迟level更灵活
            - 设计原理
                - 两层hash wheel
                    - 磁盘轮
                    第一层位于磁盘上，默认每个小时为一个刻度(可配置)，每个刻度会生成一个日志文件(schedule log),刻度总数=总时长/刻度时长(默认最大两年为 2 * 366 * 24 = 17568 个文件)
                    - 内存轮
                    第二层在内存中，内存中的hash wheel则是以500ms为一个刻度,当消息的投递时间即将到来的时候，会将这个小时的消息索引(索引包括消息在schedule log中的offset和size)从磁盘文件加载到内存中的hash wheel上
                - 三级log
                在延时/定时消息里也存在三种log
                    - message log
                    和实时消息里的message log类似，收到消息后append到该log就返回给producer，相当于WAL,消息内容从message log同步到了schedule log，所以历史message log都可以删除,只需要占用极小的存储空间(可用SSD优化性能)
                    - schedule log
                    包含完整的消息内容,按照投递时间组织，每个小时一个,可删除过期消息。该log是回放message log后根据延时时间放置对应的log上，这是上面描述的两层hash wheel的第一层，位于磁盘上
                    - dispatch log
                    延时/定时消息投递成功后写入，主要用于在应用重启后能够确定哪些消息已经投递，dispatch log里写入的是消息的offset，不包含消息内容
        - 消息过滤
        - 事务消息
        QMQ在Producer端利用数据库事务来实现事务消息,有自己独立的qmq_produce的库或者表
        - 极致扩缩容
        通过移除partition的设计,避免了partition与consumer的映射关系,实现了极致的消费者扩缩容
        - 幂等消费ExactlyOnce
        通过维护一张消息消费表来实现,收到消息后先校验是否重复消费,是则忽略,否则消费并新增记录,消费失败时删除记录
        - 高可用
            - 数据分片
            因为移除了Partition设计,所以类似实现了RedisCluster的数据分片的高可用
            - 主从复制
            Qmq使用了异步刷盘策略,当配置了从节点时,接受请求后会确保所有从节点刷盘成功后才会返回成功.

#### RabbitMQ