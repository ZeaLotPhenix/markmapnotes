---
title: 缓存中间件
markmap:
  colorFreezeLevel: 2
---

## Redis
### 相关面试题
- 面试题顺序
  - 157 Zookeeper和Redis哪个好
  - 169 谈一谈你对Redis的理解
  - 204 Redis为什么这么快
  - 224 316 Redis和Mysql如何保持数据一致性
  - 215 布隆过滤器及其作用 
  - 330 Redis内存淘汰算法和原理是什么
  - 584 Redis缓存淘汰策略
  - 493 Redis的持久化策略
  - 527 Redis多线程模型怎么理解?会有线程安全问题吗?
  - 559 怎么防止缓存击穿的问题
- 面试题侧重
  - 综合理解
    - 157 Zookeeper和Redis哪个好
        分布式锁 
    - 169 谈一谈你对Redis的理解
    - 204 Redis为什么这么快
  - 原理
    - 330 Redis内存淘汰算法和原理是什么
    - 584 Redis缓存淘汰策略
    - 670 Redis过期策略
    - 493 Redis的持久化策略
    - 527 Redis多线程模型怎么理解?会有线程安全问题吗?
    - 493 Redis的持久化策略
  - 运维
    - 641 Redis哨兵机制和集群有什么区别
    - 654 Redis主从复制原理
    - 715 为什么Redis最大槽数是16384个
  - 应用
    - 224 316 Redis和Mysql如何保持数据一致性
    - 559 怎么防止缓存击穿的问题
    - 215 布隆过滤器及其作用
    - 671 热key问题
    - 672 Redis遇到hash冲突怎么办
    - 大Key问题
### 技术原理
- I/O buffer成本问题
- 4K操作系统,从硬盘中每一次读取最小4k
### 设计原理
#### 数据结构 *Key-Value Pair*
##### Redis的数据结构<!-- markmap: fold -->
- 简单动态字符串*SimpleDynamicString*(SDS)
  ==Redis自定义的字符串==
  - sds.h
  - |len|alloc|flags|buf[]|
    |:-:|:-:|:-:|:-:|
    |4|4|1|n,a,m,e,\0|
  - *优点*
      获取字符串长度时间复杂度未O(1)
      支持动态扩容
      减少内存分配次数
      二进制安全
- IntSet
  ==数据量不多时使用==
  - intset.h
  - |encoding|length|contents|
    |:-:|:-:|:-:|
    |INTSET_ENC_INT32|4|5,10,20,50000|
  - *优点*
      元素唯一且有序
      类型升级机制(节约内存空间)
      二分查找算法提升效率
- Dict
  ==不连续的内存空间,浪费严重==
  - 哈希表*DictHashTable*
  - 哈希节点*HashEntity*
  - 字典*Dict*
  - *Dict的扩容*`static int _dictExpandIfNeeded(dict *d)`
      结构类似Java的HashTable(数组+链表),但有2个DictHashTable
      扩缩容时渐进式rehash(一次只迁移部分数据)
      渐进式rehash时新数据只添加到新表上
- ZipList(特殊的双向链表)
  ==连续的内存空间,申请效率低==
  - 使用prevrawlen(前一个节点的长度)来替代prev指针,len替代next指针
  - ~~缺点~~
    级联更新问题
- QuickList
  ==使用ZipList作为节点的双向链表==
  - *优点*
    解决了传统链表的内存占用问题
    控制了ZipList大小,解决了连续内存空间申请效率问题
    中间节点可压缩,进一步节约了空间
- SkipList
  ==增删改查效率与红黑树基本一致,实现更简洁==
  - sever.h `typedef struct zskiplist` `typedef struct zskiplistNode`
  - 元素按score升序排列,相同score再按ele字典排序
  - 每个节点都可能包含多层指针,指针层数越高,跨度越大
- RedisObject
  ==Redis的所有5种数据类型的键和值都会被封装成一个RedisObject==
  - server.h `typedef struct redisObject`
##### Redis的5种数据类型
- String
  - encoding
    - int
    - raw(基于SDS)
    - embstr(也是基于SDS,大小<44B时>)
      ==连续内存空间,申请效率更高==
  - bitmaps
    - 1.有用户系统,统计用户登录天数,且窗口随机
    - 2.统计活跃用户
  - geo
  - hyperloglog
- hash
  - encoding
    - ZipList
    - HashTable
- list
  - encoding
    - QuickList
    - command
      - lpush `t_list.c void pushGenericCommand()`
- set
  - encoding
    - intset
    - HashTable
  - command
    - SADD,SREM
    - SMEMBERS,SISMEMBER
    - SINTER,SDIFF,SUNION,SMOVE 
- z_set(sorted sets)
  - encoding
    - ZipList
    - HashTable
#### 网络模型
##### 5种IO模型*UNIX网络编程一书*
- 5种IO模型对比
  |IO模型|等待数据就绪|拷贝内核数据|
  |:-:|:-:|:-:|
  |阻塞IO|阻塞等待(recvfrom)|阻塞等待|
  |非阻塞IO|轮询等待(recvfrom)|阻塞等待|
  |IO多路复用|监听多个FD等待(select/poll/epoll+recvfrom)|阻塞等待|
  |信号驱动IO|注册信号异步等待(sigaction)|阻塞等待|
  |异步IO(aio)|指定数据写入地址(aio_read)|递交(aio_read)数据|

- IO多路复用
  - select函数
  - poll函数
  - epoll函数
    - epoll_create函数
    - epoll_ctl函数
    - epoll_wait函数(内核回调)
      LT模式/ET模式
    - Web流程
- 信号驱动IO
- 异步IO
##### Redis中用到的网络模型
- Redis是单线程还是多线程?
  命令单线程
  部分执行逻辑多线程
- 为什么要选择单线程?
  纯内存操作
  减少多线程导致的上下文切换
  避免多线程线程安全问题
- 对应源码
  ae_epoll.c
  ae_evport.c
  ae_kqueue.c
  ae_select.c
  ae.c
- redis启动入口
  server.c `int main(int argc, char **argv)`
#### 通信模型
- RESP协议(Redis Serialization Protocol)
  - 单行字符串 `+OK\r\n`
  - 错误Errors `-Error message\r\n`
  - 数值 `:10\r\n`
  - 多行字符串 `$5\r\nhello\r\n`
  - 数组`*3\r\n$3\r\nset\r\n$4\r\nname\r\n$6\r\n虎哥\r\n`
#### 内存模型
- 内存策略
  - 过期策略`EXPIRE key TTL: expire oid 10`
    - 数据结构
    - 惰性删除
      访问该key时才检查存活时间,若已过期才删除
    - 周期删除
      SLOW模式与FAST模式
      定时任务,周期性抽样部分过期key,再执行删除
  - 淘汰策略
    - 时机:执行任何客户端命令之前
    - 8种淘汰策略
      - noeviction
      - volatile-ttl
      - allkeys-random
      - volatile-random
      - allkeys-lru
      - volatile-lru
      - allkeys-lfu
      - volatile-lfu
#### 高可用原理
- 三种集群模式
  - 主从模式MasterSlave
  - 哨兵模式Sentinel单主多从
    - 主观下线/客观下线
    - Redis故障转移/选举master
      1. 排除主观下线/已断联/回复ping大于5秒的slave
      2. 排除与失效主服务器链接端口时长超过down-after十倍的slave
      3. 按数据同步偏移量offset选出最完整的Slave
      4. offset相同选id最小的slave
      5. sentinel哨兵leader向选举出的slave发送slave of no one通知其升级为master,
      并通知其他哨兵去新slave订阅更新master信息
      6. sentinel哨兵leader向其他slave发送slave of通知其订阅新的master节点建立链接
      7. 新master将其rdb信息同步给其他slave恢复数据,并更新配置文件
      8. 如果原下线的master恢复,会降级为slave,与新master全量同步
    - Sentinel哨兵故障转移/选择sentinel哨兵leader
    *Raft算法*
      1. 超过半数投票选出leader,leader用于下达redis故障转移命令
      2. leader挂了继续用raft选举出剩余followers中的leader
      3. 所有的sentinel哨兵都会再redis master中注册自己的信息,所以可以感知到其他的哨兵的信息
    - 选举期间访问瞬断问题
  - 集群模式Cluster多主多从
  *一致性哈希算法* *gossipn协议通信*
    - 集群节点的初始化`cluster.c`
    - 执行命令时会先判定是否在本节点,不在的话会下发redirect命令让客户端转发到具体节点
    - 故障转移
      1. 主节点挂掉,从节点上线,并gossipn通知其他节点信息
      2. 主节点恢复,会降级为从节点,并向新主节点全量同步
      3. 主从都挂了,整个集群会fail状态,不可用
#### 持久化机制
- RDB模式
  周期性记录
- AOF模式
  命令级别记录/每秒(默认)/不持久化
  - aof buf
  - aof 重写=buf
### 使用
#### Redis命令
##### 管理命令
- `redis-cli -h host -p port -a password`
##### 基本命令
- set
- get
- del
- expire
- exists
- geo相关
    - geodist
    - georadius
    - georadiusbymember
    - geoadd
    - geopos
    - geohash
##### 使用场景
- 数据一致性
  - 思考点
    - 原子性:读操作/写操作对于数据库和缓存的读写都不是原子的,读写操作对于数据库和Reids的读写指令可能交叉
    - 容错性:网络请求失败,特别时对于Redis的写失败不会删除旧数据
    - 判断场景是读多写多/读多写少/读少写少/读少写多4种中的哪种
  - [常见的7种方案](https://zhuanlan.zhihu.com/p/10924246278)
    - CacheAside旁路缓存模式
      - 读操作
      查询缓存：若缓存中存在数据，则直接返回；
      如果缓存中没有数据，从数据库加载最新数据，写入缓存后返回。
      - 写操作
      首先更新数据库；
      删除缓存，使后续读操作触发缓存重建。
    - ReadThrough直读缓存模式
      - 读操作
      若缓存命中，直接返回数据；
      若缓存未命中，缓存系统从数据库加载数据并返回，同时更新缓存。
      - 写操作
      同时更新缓存和数据库，确保两者一致。
    - WriteThrough直写缓存模式
      - 读操作
      数据直接从缓存返回，不查询数据库
      - 写操作
      更新缓存；
      缓存同步更新数据库
    - 双写
      - 读操作
      数据直接从缓存读取
      - 写操作
      同时更新数据库和缓存
    - DoubleDeletion延迟双删
      - 读操作
      查询缓存：若缓存中存在数据，则直接返回；
      如果缓存中没有数据，从数据库加载最新数据，写入缓存后返回
      - 写操作
      先删除缓存
      再更新数据库
      更新数据库后，等待一定时间，再次删除缓存，确保一致性
    - 异步消息队列
      - 读操作
      查询缓存：若缓存中存在数据，则直接返回；
      如果缓存中没有数据，从数据库加载最新数据，写入缓存后返回
      - 写操作
      首先更新数据库；
      然后将更新操作通过消息队列通知缓存服务，刷新或删除缓存
    - Binlog同步
      - 读操作
      查询缓存：若缓存中存在数据，则直接返回；
      如果缓存中没有数据，从数据库加载最新数据，写入缓存后返回
      - 写操作
      只写数据库
      数据库更新时产生 Binlog；
      同步工具（如 Canal）实时解析 Binlog，将变更事件通知缓存服务；
      缓存服务根据 Binlog 内容更新或删除缓存
  - 7种方案对比
    |方案|一致性保障|适用场景|复杂度|
    |:-:|:-:|:-:|:-:|
    |Cache Aside|最终一致性|读多写少场景，容忍短暂不一致|中|
    |Read-Through|最终一致性|读请求频繁、缓存命中率高的场景|中|
    |Write-Through|强一致性|写多读少、对实时一致性要求高的场景|中|
    |双写|弱一致性|更新频率低、简单业务逻辑的场景|低|
    |延时双删策略|最终一致性|并发写敏感的高一致性场景|中|
    |异步消息队列同步|最终一致性|高并发、高可用要求的复杂场景|高|
    |Binlog 同步|强一致性|数据一致性高要求、复杂系统|高|

## memcached
### Key-Value Pair