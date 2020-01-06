# Redis

## redis运维日志

### 安装

```sh
$ wget http://download.redis.io/releases/redis-3.2.11.tar.gz
$ tar -zxvf redis-3.2.11.tar.gz
$ ln -s redis-3.2.11 redis
$ cd redis
$ make & make install
```

### 启动方式

- 直接启动：redis-server
- 动态参数启动：redis-server -p 6380
- 指定配置文件启动：redis-server /path/to/redis-**.conf

推荐基础配置：
```
# 是否以守护进程方式启动
daemonize yes
# redis对外端口
port 6380
# 工作目录
dir ./
# 日志文件
logfile "redis-6380.log"
```

### redis客户端

```
redis-cli -h ip -p port
```

### redis API

#### 通用命令

- keys [re pattern]:显示满足通配符匹配的key
- dbsize：查看key总数
- exist key：判断key是否存在
- del key [key...]：删除指定key-value
- expire key seconds：设置key过期时间
- ttl key：查看key剩余过期时间
- persist key：删除key的过期设置

```
127.0.0.1:6379> set hello world
OK
127.0.0.1:6379> expire hello 20
(integer) 1
127.0.0.1:6379> ttl hello
(integer) 16
127.0.0.1:6379> get hello
"world"
127.0.0.1:6379> ttl hello
(integer) 7
127.0.0.1:6379> ttl hello
(integer) -2 (-2代表key已经不存在了)
127.0.0.1:6379> get hello
(nil)
 127.0.0.1:6379> set hello world
OK
127.0.0.1:6379> expire hello 20
(integer) 1
127.0.0.1:6379> ttl hello
(integer) 16 (还有16秒过期)
127.0.0.1:6379> persist hello
(integer) 1
127.0.0.1:6379> ttl hello
(integer) -1 (-1代表key存在，并且没有设置过期时间)
127.0.0.1:6379> get hello
"wordl"
```

- type key:查看key对应value的数据类型（string hash list set zset）

#### 通用命令的时间复杂度

![](https://i.loli.net/2019/09/09/oOukZ7i3l9QmMrB.png)

### 数据结构和内部编码

![](https://i.loli.net/2019/09/09/7fPFYTer3v98HWd.png)

### 单线程

redis是单线程设计的，使用时应该注意以下几点：

1. 一次只运行一条命令
2. 不要执行长（慢）命令

长（慢）命令：keys, flushall, flushdb, slow lua script, mutil/exec, operate big value(collection)都是>=O（n）复杂度的命令

其实redis也不全是单线程，比如异步生成rdb文件等

## Redis基本数据类型

### 字符串

#### [字符串常用命令](http://redisdoc.com/string/index.html)

#### 时间复杂度

![](https://i.loli.net/2019/09/09/79Q3mjdMZWt4Eap.png)

#### 使用场景

- 页面动态缓存
  比如生成一个动态页面，首次可以将后台数据生成页面，并且存储到redis 字符串中。再次访问，不再进行数据库请求，直接从redis中读取该页面。特点是：首次访问比较慢，后续访问快速。
- 数据缓存
  在前后分离式开发中，有些数据虽然存储在数据库，但是更改特别少。比如有个全国地区表。当前端发起请求后，后台如果每次都从关系型数据库读取，会影响网站整体性能。
  我们可以在第一次访问的时候，将所有地区信息存储到redis字符串中，再次请求，直接从数据库中读取地区的json字符串，返回给前端。
- 数据统计
  redis整型可以用来记录网站访问量，某个文件的下载量。（自增自减）
- 时间内限制请求次数
  比如已登录用户请求短信验证码，验证码在5分钟内有效的场景。
  当用户首次请求了短信接口，将用户id存储到redis 已经发送短信的字符串中，并且设置过期时间为5分钟。当该用户再次请求短信接口，发现已经存在该用户发送短信记录，则不再发送短信。
- 分布式session
  当我们用nginx做负载均衡的时候，如果我们每个从服务器上都各自存储自己的session，那么当切换了服务器后，session信息会由于不共享而会丢失，我们不得不考虑第三应用来存储session。通过我们用关系型数据库或者redis等非关系型数据库。关系型数据库存储和读取性能远远无法跟redis等非关系型数据库。
- [Redis字符串常用命令以及应用场景](https://blog.csdn.net/nuomizhende45/article/details/82118035)
- [深入解析Redis中常见的应用场景](https://www.jb51.net/article/124665.htm)


### 哈希

#### [哈希常用命令](http://redisdoc.com/hash/index.html)

#### 时间复杂度

![](https://i.loli.net/2019/09/09/AK4HTYiOxLa6bfZ.png)

#### 使用场景

满足key-field-value的数据结构类型的，且value变动频繁，例如：

- [购物车](https://blog.csdn.net/ahjxhy2010/article/details/79892894)
- 缓存视频的基本信息
- 等等

### 列表

#### [列表常用命令](http://redisdoc.com/list/index.html)

#### 时间复杂度

#### 使用场景

Redis list的应用场景非常多，也是Redis最重要的数据结构之一，比如twitter的关注列表，粉丝列表等都可以用Redis的list结构来实现。

List 就是链表，相信略有数据结构知识的人都应该能理解其结构。使用List结构，我们可以轻松地实现最新消息排行等功能。List的另一个应用就是消息队列，
可以利用List的PUSH操作，将任务存在List中，然后工作线程再用POP操作将任务取出进行执行。Redis还提供了操作List中某一段的api，你可以直接查询，删除List中某一段的元素。

- 栈：LPUSH + LPOP
- 队列： LPUSH + RPOP
- 定长集合：LPUSH + LTRIM
- [消息队列](https://blog.csdn.net/qq_20042935/article/details/89964660)：LPUSH + BRPOP

### 集合

#### [集合常用命令](http://redisdoc.com/set/index.html)

#### 时间复杂度

#### 使用场景

集合有取交集、并集、差集等操作，因此可以求共同好友、共同兴趣、分类标签等。

1、标签：比如我们博客网站常常使用到的兴趣标签，把一个个有着相同爱好，关注类似内容的用户利用一个标签把他们进行归并。

2、共同好友功能，共同喜好，或者可以引申到二度好友之类的扩展应用。

3、统计网站的独立IP。利用set集合当中元素不唯一性，可以快速实时统计访问网站的独立IP。

### 有序集合

#### [有序集合常用命令](http://redisdoc.com/sorted_set/index.html)

#### 时间复杂度

#### 使用场景

- [排行榜系统](https://js.aizhan.com/data/redis/1199.html)
  有序集合比较典型的使用场景就是排行榜系统，例如视频网站需要对用户上传的视频做排行榜，榜单的维度可能是多个方面的：按照时间、按照播放数量、按照获得的赞数。

- 用Sorted Sets来做带权重的队列，比如普通消息的score为1，重要消息的score为2，然后工作线程可以选择按score的倒序来获取工作任务。让重要的任务优先执行。

## Redis其他功能

### 慢查询

#### 生命周期

![](https://i.loli.net/2019/09/09/joNX7RhnsGSt5Vy.png)

#### 配置和命令

- 配置

  - slowlog-max-len：慢查询队列长度
  - slowlog-log-slower-than：慢查询阈值（单位：微秒）
  - slowlog-log-slower-than=0, 记录所有命令
  - slowlog-log-slower-than<0, 不记录任何命令

- 配置方法
  - 默认值
    - config get slowlog-max-len = 128
    - config get slowlog-log-slower-than = 10000
  - 动态配置
    - config set slowlog-max-len 1000
    - config set slowlog-log-slower-than 1000
- 命令
  - slowlog get [n]：获取慢查询队列
	- slowlog len：获取慢查询队列长度
	- slowlog reset：清空慢查询队列

#### 运维经验

1. slowlog-max-len不要设置过大，默认10ms，通常设置1ms Redis的QPS是万级的，也就是一条命令平均0.1ms就执行结束了。通常我们翻10倍，设置1ms。
2. slowlog-log-slower-than不要设置过小，通常设置1000左右 慢查询队列默认长度128，超过的话，按先进先出，之前的慢查询会丢失。通常我们设置1000。
3. 理解命令生命周期 慢查询发生在第三阶段
4. 定期持久化慢查询 slowlog get或其他第三方开源工具


### pipeline

#### 什么是流水线

未使用流水线的网络通信模型：
![](https://i.loli.net/2019/09/09/jxHErwNBmUVAfeY.png)

假设客户端在上海，Redis服务器在北京。相距1300公里。假设光纤速度≈光速2/3，即30000公里/秒2/3。那么一次命令的执行时间就是(13002)/(300002/3)=13毫秒。Redis万级QPS，一次命令执行时间只有0.1毫秒，因此网络传输消耗13毫秒是不能接受的。在N次命令操作下，Redis的使用效率就不是很高了。

使用pipeline的网络通信模型：
![](https://i.loli.net/2019/09/09/usig4aLURqt2D8B.png)

#### 客户端实现

引入maven依赖：

```xml
<dependency>
  <groupId>redis.clients</groupId>
  <artifacId>jedis</artifacId>
  <version>2.9.0</version>
  <type>jar</type>
  <scope>compile</scope>
</dependency>
```

客户端：

```java
 // 不用pipeline
Jedis jedis = new Jedis("127.0.0.1", 6379);
for (int i = 0; i < 10000; i++) {
    jedis.hset("hashkey" + i, "field" + i, "value=" + i);
}

不用pipeline，10000次hset，总共耗时50s，不同网络环境可能有所不同
// 使用pipeline, 我们将10000条命令分100次pipeline，每次100条命令
Jedis jedis = new Jedis("127.0.0.1", 6379);
for (int i = 0; i < 100; i++) {
    Pipeline pipeline = jedis.pipeline();
    for (int j = i * 100; j < (i + 1) * 100; j++) {
        pipeline.hset("hashkey:" + j, "field" + j, "value" + j);
    }
    pipeline.syncAndReturnAll();
}
```

使用pipelne，10000次hset，总共耗时0.7s，不同网络环境可能有所不同。
可见在执行批量命令时，使用pipeline对Redis的使用效率提升是非常明显的。

#### 与M原生操作对比

mset、mget等操作是**原子性操作**，一次m操作只返回一次结果。pipeline非原子性操作，只是**将N次命令打个包**传输，最终命令会被逐条执行，客户端接收N次返回结果。

#### pipeline使用建议

1. 注意每次pipeline携带数据量,pipeline主要就是压缩N次操作的网络时间。但是pipeline的命令条数也不建议过大，避免单次传输数据量过大，客户端等待过长。
2. Redis集群中，pipeline每次只作用在一个Reids节点上。

### [发布订阅](http://redisdoc.com/pubsub/index.html)

- 角色

  - 发布者（publisher）
  - 订阅者（subscriber）
  - 频道（channel）
- 模型
![](https://i.loli.net/2019/09/09/7TjzIhYwJrFpku8.png)

- API

发布
```
API：publish channel message
redis> publish sohu:tv "hello world"
(integer) 3 #订阅者个数
redis> publish sohu:auto "taxi"
(integer) #没有订阅者
```

订阅
```
API：subscribe [channel] #一个或多个
redis> subscribe sohu:tv
1) "subscribe"
2) "sohu:tv"
3) (integer) 1
1) "message"
2) "sohu:tv"
3) "hello world"
```

取消订阅
```
API：unsubscribe [channel] #一个或多个
redis> unsubscribe sohu:tv
1) "unsubscribe"
2) "sohu:tv"
3) (integer) 0
```

其他API
```
psubscribe [pattern…] #订阅指定模式
punsubscribe [pattern…] #退订指定模式
pubsub channels #列出至少有一个订阅者的频道
pubsub numsubs [channel…] #列出给定频道的订阅者数量
```

- 对比消息队列

![](https://i.loli.net/2019/09/09/tSsjmzRM1c6rVJ8.png)

发布订阅模型，订阅者均能收到消息。消息队列，只有一个订阅者能收到消息。因此使用发布订阅还是消息队列，要搞清楚使用场景。

### GEO

#### geo是什么

GEO：存储经纬度，计算两地距离，范围计算等

#### 相关命令

```
API：geoadd key longitude latitude member # 增加地理位置信息
redis> geoadd cities:locations 116.28 39.55 bejing
(integer) 1
redis> geoadd cities:locations 117.12 39.08 tianjin 114.29 38.02 shijiazhuang 118.01 39.38 tangshan 115.29 38.51 baoding
(integer) 4
```

```
API：geopos key member [member…] # 获取地理位置信息
redis> geopos cities:locations tianjin
1)1) "117.12000042200088501"
  2) "39.0800000535766543"
```

```
API：geodist key member1 member2 [unit] # 获取两位置距离,unit:m(米)、km(千米)、mi(英里)、ft(尺)
reids> geodist cities:locations tianjin beijing km
"89.2061"
```

```
API： 获取指定位置范围内的地理位置信息集合
georadius key longitude latitude radius m|km|ft|mi [withcoord] [withdist] [withhash] [COUNT count] [asc|desc] [store key] [storedist key]
georadiusbymember key member radius m|km|ft|mi [withcoord] [withdist] [withhash] [COUNT count] [asc|desc] [store key] [storedist key]
withcoord: 返回结果中包含经纬度 withdist: 返回结果中包含距离中心节点的距离 withhash: 返回结果中包含geohash COUNT count：指定返回结果的数量 asc|desc：返回结果按照距离中心节点的距离做升序/降序 store key：将返回结果的地理位置信息保存到指定键 storedist key：将返回结果距离中心点的距离保存到指定键
redis> georadiusbymember cities:locations beijing 150 km
1）"beijing"
2) "tianjin"
3) "tangshan"
4) "baoding"
```


#### 相关说明

1. since redis3.2+
2. redis geo是用zset实现的，type geokey = zset
3. 没有删除API，可以使用zset的API：zrem key member

## Redis持久化

什么是持久化？Redis的数据操作都在内存中，redis崩掉的话，会丢失。Redis持久化就是对数据的更新异步的保存在磁盘上，以便数据恢复。

### 实现方式

- 快照方式（RDB）
- 写日志方式（AOF）

### RDB

#### 什么是RDB

将Redis内存中的数据，完整的生成一个快照，以**二进制**格式文件（**后缀RDB**）保存在硬盘当中。当需要进行恢复时，再从硬盘加载到内存中。
Redis**主从复制，用的也是基于RDB方式**，做一个复制文件的传输。

#### 触发方式

- save命令触发方式（同步）

  ```
  redis> save
  OK
  ```

  save执行时，会造成Redis的阻塞。所有数据操作命令都要排队等待它完成。
  **文件策略**：新生成一个新的临时文件，当`save`执行完后，用新的替换老的。

- bgsave命令触发方式（异步）

  ```
  redis> bgsave
  Background saving started
  ```

  客户端对Redis服务器下达`bgsave`命令时，Redis会`fork`出一个子进程进行RDB文件的生成。当RDB生成完毕后，子进程再反馈给主进程。fork子进程时也会阻塞，不过正常情况下fork过程都非常快的。
  文件策略：与`save`命令相同。

  ![Snipaste_2020-01-06_16-17-59.png](https://i.loli.net/2020/01/06/F3p8qR1JV5Y6bhM.png)

- 配置触发方式(不建议，RDB生成会很频繁)

  ![ff](https://i.loli.net/2020/01/06/Tt2rRwNFdj9WlmJ.png)

  修改配置文件：
  ```
  # 配置自动生成规则。一般不建议配置自动生成RDB文件
  save 900 1
  save 300 10
  save 60 10000
  # 指定rdb文件名
  dbfilename dump-${port}.rdb
  # 指定rdb文件目录
  dir /opt/redis/data
  # bgsave发生错误，停止写入
  stop-writes-on-bgsave-error yes
  # rdb文件采用压缩格式
  rdbcompression yes
  # 对rdb文件进行校验
  rdbchecksum yes
  ```

- 不容忽略的触发方式

	1. 全量复制 **主从复制**时，主会自动生成RDB文件。
	2. debug reload Redis中的**debug reload**提供debug级别的重启，不清空内存的一种重启，这种方式也会触发RDB文件的生成。
	3. **shutdown** 会触发RDB文件的生成。

### AOF

就是写日志，每次执行`Redis`写命令，让命令同时记录日志（以`AOF`日志格式）。`Redis`宕机时，只要进行日志回放就可以恢复数据。

#### RDB存在的问题

- 耗时、耗内存、耗IO性能
  将内存中的数据全部`dump`到硬盘当中，耗时。`bgsave`的方式`fork()`子进程耗额外内存。大量的硬盘读写耗费`IO性能`。(**IO密集型**)
- 不可控、丢失数据
  宕机时，上次快照之后写入的内存数据，将会丢失。

#### AOF三种策略

首先Redis执行写命令，将命令刷新到硬盘缓冲区当中。

- `always`
  `always`策略让缓冲区中的数据即时刷新到硬盘。
- `everysec`
  `everysec`策略让缓冲区中的数据**每秒刷新**到硬盘。相比`always`，在高写入量的情况下，可以保护硬盘。出现故障可能会丢失一秒数据。
- `no`
  刷新策略让操作系统来决定。

  ![image.png](https://i.loli.net/2020/01/06/HY8QCiRy3Ij6mkD.png)

**通常使用`everysec`策略，这也是AOF的默认策略。**

#### AOF重写

随着时间的推移，命令的逐步写入。`AOF`文件也会逐渐变大。当我们用`AOF`来恢复时会很慢，而且当文件无限增大时，对硬盘的管理，对写入的速度也会有产生影响。`Redis`当然考虑到这个问题，所以就有了AOF重写。

AOF重写就是把**过期的、没用的、重复的以及可优化的命令，进行化简**。只取最终有价值的结果。虽然写入操作很频繁，但系统定义的key的量是相对有限的。
AOF重写可以大大压缩最终日志文件的大小。从而**减少磁盘占用量，加快数据恢复速度**。比如我们有个计数的服务，有很多自增的操作，比如有一个key自增到1个亿，对AOF文件来说就是一亿次incr。AOF重写就只用记1条记录。

- AOF重写两种方式
	- bgrewriteaof命令触发AOF重写 redis客户端向Redis发bgrewriteaof命令，redis服务端fork一个子进程去完成AOF重写。这里的AOF重写，是将Redis内存中的数据进行一次回溯，回溯成AOF文件。而不是重写AOF文件生成新的AOF文件去替换。
	- AOF重写配置 auto-aof-rewrite-min-size：AOF文件重写需要的尺寸 auto-aof-rewrite-percentage：AOF文件增长率 redis提供了aof_current_size和aof_base_size，分别用来统计AOF当前尺寸（单位：字节）和AOF上次启动和重写的尺寸（单位：字节）。 AOF自动重写的触发时机，同时满足以下两点）：
	- aof_current_size > auto-aof-rewrite-min-size
	- aof_current_size – aof_base_size/aof_base_size > auto-aof-rewrite-percentage

- AOF重写配置

  修改配置文件：

  ```
  # 开启正常AOF的append刷盘操作
  appendonly yes
  # AOF文件名
  appendfilename "appendonly-6379.aof"
  # 每秒刷盘
  appendfsync everysec
  # 文件目录
  dir /opt/soft/redis/data
  # AOF重写增长率
  auto-aof-rewrite-percentage 100
  # AOF重写最小尺寸
  auto-aof-rewrite-min-size 64mb
  # AOF重写期间是否暂停append操作。AOF重写非常消耗磁盘性能，而正常的AOF过程中也会往磁盘刷数据。
  # 通常偏向考虑性能，设为yes。万一重写失败了，这期间正常AOF的数据会丢失，因为我们选择了重写期间放弃了正常AOF刷盘。
  no-appendfsync-on-rewrite yes
  ```

### RDB和AOF的比较

![image.png](https://i.loli.net/2020/01/06/gGmjTL1ZShVtK5n.png)

### RDB最佳策略

1. 建议关闭RDB
   无论是Redis主节点，还是从节点，都建议关掉RDB。但是关掉不是绝对的，**主从复制**时还是会借助RDB。
2. 用作数据备份
   RDB虽然是很重的操作，但是对**数据备份**很有作用。文件大小比较小，可以按天或按小时进行数据备份。
3. 主从，从开？
   在极个别的场景下，需要在从节点开RDB，可以再本地保存这样子的一个历史的RDB文件。虽然从节点不进行读写，但是Redis往往单机多部署，由于RDB是个很重的操作，所以还是会对CPU、硬盘和内存造成一定影响。根据实际需求进行设定。

### AOF最佳策略

1. 建议开启AOF
   如果Redis数据只是用作数据源的缓存，并且缓存丢失后从数据源重新加载不会对数据源造成太大压力，这种情况下。AOF可以关。
2. AOF重写集中管理
   单机多部署情况下，发生大量fork可能会内存爆满。
3. everysec
   建议采用每秒刷盘策略

### 最佳策略

1. 小分片
   使用maxmemary对Redis最大内存进行规划。
2. 缓存和存储
   根据缓存和存储的特性来决定使用哪种策略
3. 监控（硬盘、内存、负载、网络）
4. 足够的内存
   不要把就机器全部的内存规划给Redis。不然会出很多问题。像客户端缓冲区等，不受maxmemary限制。规划不当可能会产生SWAP、OOM等问题。

## Redis和memcached

两者都是非关系型内存键值数据库，主要有以下不同：

### 数据类型

Memcached 仅支持**字符串**类型，而 Redis 支持五种不同的数据类型，可以更灵活地解决问题。

### 数据持久化

Redis 支持两种持久化策略：RDB 快照和 AOF 日志，而 Memcached 不支持持久化。

### 分布式

- Memcached 不支持分布式，只能通过在客户端使用**一致性哈希**来实现分布式存储，这种方式在存储和查询时都需要先在客户端计算一次数据所在的节点。
- Redis Cluster 实现了分布式的支持。

### 内存管理机制

- 在 Redis 中，并不是所有数据都一直存储在内存中，**可以将一些很久没用的 value 交换到磁盘**，而
Memcached 的数据则会一直在内存中。
- Memcached 将内存分割成**特定长度的块**来存储数据，以完全解决内存碎片的问题。但是这种方式会使得内存的利用率不高，例如块的大小为 128 bytes，只存储 100 bytes 的数据，那么剩下的 28 bytes 就**浪费**掉了。

## 缓存优化

### 缓存收益与成本的问题

关于缓存收益与成本主要分为三个方面的讲解，第一个是什么是收益；第二个是什么是成本；第三个是有哪些使用场景。

- 收益
  主要有以下两大收益。
	1. **加速读写**：通过缓存加速读写，如 CPU L1/L2/L3 的缓存、Linux Page Cache 的读写、游览器缓存、Ehchache 缓存数据库结果。
	2. **降低后端负载**：后端服务器通过前端缓存来降低负载，业务端使用 Redis 来降低后端 MySQL 等数据库的负载。
- 成本
  产生的成本主要有以下三项。
	1. **数据不一致**：这是因为缓存层和数据层有时间窗口是不一致的，这和**更新策略**有关的。
	2. **代码维护成本**：这里多了一层缓存逻辑，就会增加成本。
	3. **运维费用的成本**：如 Redis Cluster，甚至是现在最流行的各种云，都是成本。
- 使用场景
  使用场景主要有以下三种。
	1. **降低后端负载**：这是对高消耗的 SQL，join 结果集和分组统计结果缓存。
	2. **加速请求响应**：这是利用 Redis 或者 Memcache 优化 IO 响应时间。
	3. **大量写合并为批量写**：比如一些计数器先 Redis 累加以后再批量写入数据库。

### 数据淘汰策略

#### 键的过期时间

- Redis 可以为每个键设置过期时间，当键过期时，会自动删除该键。
- 对于散列表这种容器，只能为整个键设置过期时间（**整个散列表**），而不能为键里面的单个元素设置过期时间。

Redis中有个设置时间过期的功能，即对存储在 redis 数据库中的值可以设置一个过期时间。作为一个缓存数据库，这是非常实用的。如我们一般项目中的 token 或者一些登录信息，尤其是短信验证码都是有时间限制的，按照传统的数据库处理方式，一般都是自己判断过期，这样无疑会严重影响项目性能。

我们 set key 的时候，都可以给一个 expire time，就是过期时间，通过过期时间我们可以指定这个 key 可以存活的时间。

如果假设你设置了一批 key 只能存活1个小时，那么接下来1小时后，redis是怎么对这批key进行删除的？

定期删除+惰性删除。

通过名字大概就能猜出这两个删除方式的意思了。

- 定期删除：redis默认是每隔 100ms 就随机抽取一些设置了过期时间的key，检查其是否过期，如果过期就删除。注意这里是随机抽取的。为什么要随机呢？你想一想假如 redis 存了几十万个 key ，每隔100ms就遍历所有的设置过期时间的 key 的话，就会给 CPU 带来很大的负载！
- 惰性删除 ：定期删除可能会导致很多过期 key 到了时间并没有被删除掉。所以就有了惰性删除。假如你的过期 key，靠定期删除没有被删除掉，还停留在内存里，除非你的系统去查一下那个 key，才会被redis给删除掉。这就是所谓的惰性删除，也是够懒的哈！

但是仅仅通过设置过期时间还是有问题的。我们想一下：如果定期删除漏掉了很多过期 key，然后你也没及时去查，也就没走惰性删除，此时会怎么样？如果大量过期key堆积在内存里，导致redis内存块耗尽了。怎么解决这个问题呢？


#### 淘汰策略

可以设置内存最大使用量，当内存使用量超出时，会施行数据淘汰策略。
Redis 具体有 7 种淘汰策略：

- volatile-lru：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
- volatile-ttl：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
- volatile-random：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
- allkeys-lru：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key（这个是最常用的）.
- allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰
- no-eviction：禁止驱逐数据，也就是说当内存不足以容纳新写入数据时，新写入操作会报错。这个应该没人使用吧！
- 主动更新

一致性最好的就是主动更新。能够根据代码实时的更新数据，但是维护成本也是最高的；算法剔除和超时剔除一致性都做的不够好，但是维护成本却非常低。

根据需求：

1. 低一致性：最大内存和淘汰策略，数据库有些数据是不需要马上更新的，这个时候就可以用低一致性来操作。
2. 高一致性：超时剔除和主动更新的结合，最大内存和淘汰策略兜底。你没办法确保内存不会增加，从而使服务不可用了。

### 遇到的问题

#### 缓存穿透问题

![image.png](https://i.loli.net/2020/01/06/gHmEPeD29UICoQK.png)

如果cache和storage都没有id，依然不断查询，每次查询的cache都会穿透。当请求发送给服务器的时候，缓存找不到，然后都堆到数据库里。这个时候，缓存相当于穿透了，不起作用了。

原因有两点：

1. 业务代码自身的问题。很多实际开发的时候，如果是一个不熟练的程序员，由于缺乏必要的大数据的意识，很多代码在第一次写的时候是 OK 的，但是当需要修改业务代码的时候，常常会出现问题。
2. 恶意攻击和爬虫问题。网络上充斥着各种攻击和各种爬虫模仿着人为请求来访问你的数据。如果恶意访问穿透你的数据库，将会导致你的服务器瞬间产生大量的请求导致服务中止。

那我们去如何发现这些问题呢？
1. 业务的**响应时间**：一般请求的时间都是稳定的，但是如果出现类似穿透现象，必然在短时间内有一个体现。
2. 业务本身的问题。**产品的功能**出现问题。
3. **总调用数，对缓存层命中数、存储层的命中数**这些值的采集。

- 解决方案1：**缓存空对象**
  当缓存中不存在，访问数据库的时候，又找不到数据，需要设置给 **cache 的值为 null**，这样下次再次访问该 id 的时候，就会直接访问缓存中的 null 了。
  但是可能存在的两个问题。首先是**需要更多的键**，但是如果这个量非常大的话，对业务也是有影响的，所以需要**设置过期时间**；其次是缓存层和存储层数据“短期”不一致。当缓存层过期时间到了以后，可能会产生和存储层数据不一致的情况。这个时候需要**使用一些消息队列等方式**，来确保这个值的一致性。
  下面的代码用 Java 来实现简单的缓存空对象
  ```java
  public String getCacheThrough(String key){
      String cacheValue = cache.get(key);
      if(StringUtils.isBlank(cacheValue)){ // 如存储数据为空
          String storageValue = storage.get(key);
          cache.set(key,storageValue);
          if(StringUtils.isBlank(strageValue){
              cache.expire(key.60*10);//需要设置一个过期时间
          }
          return storageValue;
      }else{
        return cacheValue;
    }
  }
  ```

- 解决方案2：布隆过滤器拦截
  布隆过滤器，实际上是一个很长的二进制向量和一系列随机映射函数。布隆过滤器可以用于检索一个元素是否在一个集合中。它的优点是空间效率和查询时间都远远超过一般的算法，缺点是有一定的误识别率和删除困难。
  类似于一个字典，你查词典的时候不需要把所有单词都翻一遍，而是通过目录的方式，用户通过检索的形式在极小内存中可以找到对应的内容。
  虽然布隆过滤器可以通过极小的内存来存储，但是免不了需要一部分代码来维护这个布隆过滤器，并且经常需要根据规则来调整，在选取是否使用布隆过滤器，还需要通过场景来选取。

#### 缓存雪崩

缓存雪崩，是指在某一个时间段，缓存集中过期失效，所有的查询都会落到数据库上。
比如在写本文的时候，马上就要到双十二零点，这波商品比较集中的放入了缓存，假设缓存一个小时。那么到了凌晨一点钟的时候，这批商品的缓存就都过期了。而对这批商品的访问查询，都落到了数据库上，对于数据库而言，就会产生周期性的压力波峰。

解决方案：

解决方案分为三步：

1. 事前（预防）：
     - 选择合适的内存淘汰策略。
     - 对不同分类商品，缓存不同周期。同时，对同一分类中的商品，加上一个随机因子。这样能尽可能**分散缓存过期时间**。
2. 事中（已经发生）：**做二级缓存**，A1为原始缓存，A2为拷贝缓存，A1失效时，可以访问A2，A1缓存失效时间设置为短期，A2设置为长期+限流（实例：本地ehcache缓存 + hystrix限流&降级，避免MySQL崩掉）
3. 事后（已经导致redis宕机）：利用 redis**持久化机制**保存的数据尽快恢复缓存。

![](https://img-blog.csdnimg.cn/20191012210110996.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)

#### 热点 Key问题

我们知道，使用缓存，如果获取不到，才会去数据库里获取。但是如果是热点 key，访问量非常的大，数据库在重建缓存的时候，会出现很多线程同时重建的情况。

![image.png](https://i.loli.net/2020/01/06/eFu54gXdGRkrPB3.png)

解决方法：

- 互斥锁

由下图所示，第一次获取缓存的时候，加一个锁，然后查询数据库，接着是重建缓存。这个时候，另外一个请求又过来获取缓存，发现有个锁，这个时候就去等待，之后都是一次等待的过程，直到重建完成以后，锁解除后再次获取缓存命中。

![image.png](https://i.loli.net/2020/01/06/wr5E6XKRiWufyLY.png)

但是互斥锁也有一定的问题，就是大量线程在等待的问题。下面我们就来讲一下永远不过期。

- 永远不过期

首先在缓存层面，并没有设置过期时间（过期时间使用 expire 命令）。但是功能层面，我们为每个 value 添加逻辑过期时间，当发现超过逻辑过期时间后，会使用单独的线程去构建缓存。这样的好处就是不需要线程的等待过程。见下图。

![](https://i.loli.net/2020/01/06/LVxA9dvcBM45HX2.png)

```java
public String getKey(final String key){
    V v = redis.get(key);
    String value = v.getValue();
    long logicTimeout = v.getLogicTimeout();
    if(logicTimeout < System.currentTimeMillis()){
      String mutexKey = "mutex:key:"+key; //设置互斥锁的key
      if(redis.set(mutexKey,"1","ex 180","nx")){ //给这个key上一把锁，ex表示只有一个线程能执行，过期时间为180秒
        threadPool.execute(new Runable(){
            public void run(){
            String dbValue = db.getKey(key);
            redis.set(key,(dbValue,newLogicTimeout));//缓存重建，需要一个新的过期时间
            redis.delete(keyMutex); //删除互斥锁
     }
   };
  }
 }
return value;
}

```

互斥锁的优点是思路非常简单，具有一致性，其缺点是代码复杂度高，存在死锁的可能性。
永不过期的优点是基本杜绝 key 的重建问题，但缺点是不保证一致性，逻辑过期时间增加了维护成本和内存成本。

### redis单线程

redis集群的每个节点里只有一个线程负责接受和执行所有客户端发送的请求。技术上使用多路复用I/O，使用Linux的epoll函数，这样一个线程就可以管理很多socket连接。

除此之外，选择单线程还有以下这些原因：

1、redis都是对内存的操作，速度极快（10W+QPS）

2、整体的时间主要都是消耗在了网络的传输上

3、如果使用了多线程，则需要多线程同步，这样实现起来会变的复杂

4、线程的加锁时间甚至都超过了对内存操作的时间

5、多线程上下文频繁的切换需要消耗更多的CPU时间

6、还有就是单线程天然支持原子操作，而且单线程的代码写起来更简单

### redis事务

事务大家都知道，就是把多个操作捆绑在一起，要么都执行（成功了），要么一个也不执行（回滚了）。redis也是支持事务的，但可能和你想要的不太一样，一起来看看吧。

redis的事务可以分为两步，定义事务和执行事务。使用multi命令开启一个事务，然后把要执行的所有命令都依次排上去。这就定义好了一个事务。此时使用exec命令来执行这个事务，或使用discard命令来放弃这个事务。

你可能希望在你的事务开始前，你关心的key不想被别人操作，那么可以使用watch命令来监视这些key，如果开始执行前这些key被其它命令操作了则会取消事务的。也可以使用unwatch命令来取消对这些key的监视。

redis事务具有以下特点：

1、如果开始执行事务前出错，则所有命令都不执行

2、一旦开始，则保证所有命令一次性按顺序执行完而不被打断

3、如果执行过程中遇到错误，会继续执行下去，不会停止的

4、对于执行过程中遇到错误，是不会进行回滚的

看完这些，真想问一句话，你这能叫事务吗？很显然，这并不是我们通常认为的事务，因为它连原子性都保证不了。保证不了原子性是因为redis不支持回滚，不过它也给出了不支持的理由。

不支持回滚的理由：

1、redis认为，失败都是由命令使用不当造成

2、redis这样做，是为了保持内部实现简单快速

3、redis还认为，回滚并不能解决所有问题

哈哈，这就是霸王条款，因此，好像使用redis事务的不太多

### 如何解决 Redis 的并发竞争 Key 问题

所谓 Redis 的并发竞争 Key 的问题也就是多个系统同时对一个 key 进行操作，但是最后执行的顺序和我们期望的顺序不同，这样也就导致了结果的不同！

推荐一种方案：分布式锁（zookeeper 和 redis 都可以实现分布式锁）。（如果不存在 Redis 的并发竞争 Key 问题，不要使用分布式锁，这样会影响性能）