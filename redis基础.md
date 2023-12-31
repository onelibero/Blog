# 一、基本介绍

单线程模型，基于c语言实现的内存数据库，处理高性能和高并发问题的缓存数据库

- 高性能：用户读取数据时从redis中读比从数据库中读取更快，性能更高
  - redis将数据存在内存中，内存中数据的访问速度比从磁盘上访问数据快上千倍
  - redis有更好的，优化的数据机构
  - redis基于Reactor模式设计了自己的网络事件处理器，主要是单线程事件循环和IO多路复用
- 高并发：redis的qbs（每秒钟访问的次数）是10w+（单机模式，集群下更高），mysql的qbs是1w（4核8g）

## 三种缓存读写策略

### （1）Cache Aside Pattern（旁路缓存读写策略）

平时常使用的一个缓存读写模式，适合读请求比较多的场景

#### 写：

- 先更新db
- 然后直接删除cache

#### 读：

- 从cache中读取数据，读取到直接返回
- 如果cache中没有，从db中读取数据返回
- 在吧数据更新到cache中

### （2）Read/Write Through Pattern（读写穿透）



### （3）Write Behind Pattern（异步缓存读写）

# 二、底层数据结构

## （1）SDS（动态字符串）

主要用于实现String

SDS有五种实现方式

| 类型     | 字节 | 位   |
| -------- | ---- | ---- |
| sdshdr5  | <1   | <8   |
| sdshdr8  | 1    | 8    |
| sdshdr16 | 2    | 16   |
| sdshdr32 | 4    | 32   |
| sdshdr34 | 8    | 64   |



第一种很少使用，主要使用后四种，他们都包含下列属性

- len：字符串的长度（已经使用的字节数）
  - 可以避免缓冲区溢出：SDS进行修改时先修改len来请求空间大小是否满足要求，如果不满足则先扩展
  - 获取字符串长度复杂度变低：SDS直接读取len就可以获得长度，不需要像c语言一样遍历
  - 二进制安全：c语言以字符\0作为字符串结束符，一些二进制文件（图片，视频，音频等）就可能包含空字符，但是有了len的话就是通过len来判断结束，不存在这个问题
- alloc：总共可用的字符空间的大小，alloc-len就是SDS剩余的空间大小
- buf[]：实际存储字符串的数组
- flags：低三维保存类型标志

SDS相比较于c语言的String，可以减少内存分配次数：为了避免修改字符串时需要重新分配内存，SDS实现了空间预分配和惰性空间释放两种优化策略。

- 当SDS需要增加字符串时Redis会为SDS分配好内存，并且根据特定的算法分配多余的内存，这样可以减少连续执行字符串增长操作所需的内存重新分配次数
- 当SDS需要减少字符串时，这部分内存不会立即被回收，会被记录下来，等待后序使用（也有api支持手动释放）

> 这样也会造成内存碎片：（1）redis存储数据的时候会向操作系统申请更多的内存空间（2）频繁修改redis中的数据（数据删除时redis不会轻易释放内存给操作系统）
>
> 内存碎片也就是不可用的空闲内存，内存碎片不会影响redis的性能，只是会占用内存空间
>
> 解决：（1）通过自身的内存整理（可能对redis性能产生影响）（2）重启对应节点的redis

## （2）LinkedList（双向链表）

## （3）SkipList（跳跃表）

## （4）ZipList（压缩列表）

## （5）QuickList（快速列表）

## （6）Dict（哈希表/字典）

## （7）Intset（整数集合）

## （8）bitmap

# 三、五种基本数据机构

## 1.String

#### （1）实现：SDS

#### （2）介绍：二进制安全的数据机构，用于存储任何类型的数据（字符串，整数，浮点数，图片url或者编码，序列化后的对象）

#### （3）相关命令

key可以看做是对象的名字，value是具体的值

| 命令                             | 作用                           |
| -------------------------------- | ------------------------------ |
| set key value                    | 指定key的值                    |
| get key                          | 获取ket                        |
| mset key1 value1，key2 value2... | 指定多个key的值                |
| mget key1，key2....              | 获取多个key的值                |
| setnx key value                  | 只有key不存在的时候指定key的值 |
| incr key                         | 将key存储的数字值+1            |
| decr key                         | 将key存储的数字值-1            |
| exists key                       | 判断key是否存在                |
| del key（通用）                  | 删除key                        |
| expire key time（通用）          | 给key设置过期时间              |



#### （4）场景

- 存储session，token，图片地址，序列化后的对象
- 用户单位时间的请求数（限流），页面单位时间的访问数
- 分布式锁：通过setnx命令简单实现（存在缺陷）

> 限流：将数字类型的数据存入进去，通过incr进行增加，达到一定量的时候进行具体操作即可

> 为什么要使用分布式锁：像解决秒杀活动，超卖问题一样，我们一般用悲观锁来解决这种多线程之间的问题，但是当部署了分布式的项目，那么就变成了进程之间进行操作，本地锁没办法实现互斥，要对每一个JVM进程进行加锁就是使用了分布式锁

分布式锁产生条件：

- 互斥：任何一个时刻锁只能被一个线程持有
- 高可用：锁的服务是高可用的，当一个锁服务出现问题，能够自动切换到另一个锁服务，即使客户端的释放锁代码出现问题还是会释放锁（超时机制），不会影响其他线程的访问
- 可重入：一个节点获取锁之后还可以再次获取锁
- 高性能（非必须）：获取锁和释放锁的操作应该快速完成，并且不应该对操作系统造成过大的影响
- 非阻塞（非必须）：如果获取不到锁，不能无期限等待避免对系统正常运行造成阻塞

使用setnx实现场景：

- 获取锁：setnx命令可以帮我们实现互斥，获取锁就直接通过setnx进行加锁，只有当锁没存在时才能加锁，如果已经存在那么在调用这个命令无效（返回另外的值），这样就实现了互斥
- 释放锁：del这个命令删除对应的key即可，为了防止互删使用Lua脚本来对锁的唯一值进行判断（Lua脚本能一定程度确保锁的原子性），同时要给锁设定一个过期时间`set 加锁的锁名 能够唯一标识锁的随机字符串 EX 3(3秒) NX(只有当锁对应的key值不存在的时候才能set成功) `，避免客户端挂掉造成未释放锁的操作，**同时还要保证设置指定key的值和过期时间是一个原子操作**
- 问题：如果操作共享资源的时间大于过期时间，就会导致锁提前过期进而导致分布式锁直接失效，设置过长又会影响性能，所以自己进行续期，通过Redisson里面的watch dog 来实现锁的续期，也可以通过Redisson来实现可重入锁（一个线程可以多次获取同一把锁，synchronized和ReentrantLock都属于可重入锁）

## 2.Hash

#### （1）实现：Dict、zipList

#### （2）介绍：像hashmap一样存储类似String类型的键值对，常用于存储很多个字段值的对象

#### （3）命令

key表示对象，field是字段名，value是字段值

| 命令                                       | 作用                             |
| ------------------------------------------ | -------------------------------- |
| hset key filed value                       | 指定哈希表中指定字段的值         |
| hget key filed                             | 获取指定哈希表指定字段的值       |
| hmset key filed1 value1，filed2 value2 ... | 指定多个字段的值                 |
| hmget key filed1，filed2...                | 获取多个字段的值                 |
| hmsetnx key filed value                    | 指定字段不存在时设置指定字段的值 |
| hgetall key                                | 获取指定哈希表的所有字段         |
| hexists key filed                          | 判断指定字段是否存在             |
| hdel key filed1，filed2..                  | 删除指定字段                     |
| hlen key                                   | 获取哈希表中的字段数量           |
| hincrby key filed increment                | 对指定字段进行运算操作（正+负-） |

#### （4）场景：存储用户信息，购物车信息，文章信息，商品信息等

## 3.List

#### （1）实现：LinkedList/ZipList/QuickList

Redis 3.2 之前，List 底层实现是 LinkedList 或者 ZipList。 Redis 3.2 之后，引入了 LinkedList 和 ZipList 的结合 QuickList，List 的底层实现变为 QuickList。从 Redis 7.0 开始， ZipList 被 ListPack 取代

#### （2）介绍：视为list实现的双向链表

#### （3）命令

| 命令                     | 作用                         |
| ------------------------ | ---------------------------- |
| rpush key value1,value2  | 在列表尾部开始插入元素       |
| lpush key value1，value2 | 在列表头部开始插入元素       |
| lset key index value     | 指定列表索引的index处的值    |
| lpop key                 | 移除最左边元素并返回         |
| rpop key                 | 移除最右边元素并返回         |
| llen key                 | 获取列表元素的数量           |
| lrange key start end     | 获取列表start和end之间的元素 |



可以实现队列或者栈

#### （4）场景

- 最新文章、最新动态
- 消息队列

redis实现消息队列的三种方式

- list集合：通过rpush/lpop或者lpush/rpop’的方式实现简易的消息队列
  - 缺陷：性能低，不断轮询集合，没有确认消息机制，也没有广播机制，消息还只能被消费一次
- 发布订阅功能：pub/sub引入的Channel（频道），解决了广播的问题
  - 发布者通过publish投递消息给指定的channel
  - 消费者通过subscribe订阅对应的频道
- Redis5.0新增的Stream数据机构来做消息队列，Stream支持（存在消息丢失和堆积的问题）
  - 发布/订阅模式
  - 按照消费组进行消费
  - 消息持久化（RDB和AOF）

## 4.Set

#### （1）实现：Dict，Intset

#### （2）介绍：不重复的集合，存储不重复的元素，常用于并集，交集，差集

#### （3）命令

| 命令                          | 作用                                   |
| ----------------------------- | -------------------------------------- |
| sadd key member1 member2      | 向指定集合添加一个或多个元素           |
| smembers key                  | 获取指定集合中的所有元素               |
| scard key                     | 获取指定集合的元素数量                 |
| sismembers key member         | 判断该集合是否存在该元素               |
| sinter key1 key2...           | 获取给定集合的交集（返回元素）         |
| sinterstore key3 key1 key2..  | 将获取的交集存储在key3中               |
| sunion key1 key2...           | 获取给定集合的并集（返回元素）         |
| sunionstore key3 key1，key2.. | 将获取的并集存储在key3中               |
| sdiff key1 key2....           | 获取给定集合的差集（返回元素）         |
| sdiffstore key3 key1 key2..   | 将获取的差集存储在key3中               |
| spop key count                | 随机移除并获取指定集合中一个或多个元素 |
| srandmember key count         | 随机获取指定集合中指定数量的元素       |



#### （4）场景

- 存放的数据不能重复的场景：网站UV统计（数据量大还是用HyperLogLog），文章点赞，动态点赞等（scrad）
- 需要获取差集，交集，并集的场景：共同好友，共同粉丝，共同关注，好友推荐等

## 5.ZSet（Sorted Set）

#### （1）实现：ZipList，SkipList

#### （2）介绍：相比较于set每个值多了个权重参数score，让他们变得有序

#### （3）命令

| 命令                                     | 作用                                                         |
| ---------------------------------------- | ------------------------------------------------------------ |
| zadd key member1 score1 member2 score2.. | 向指定有序集合添加一个或多个元素                             |
| zcard key                                | 获取指定有序集合的元素数量                                   |
| zscore key member                        | 获取指定有序集合指定元素的score                              |
| zinterstore key3 numkeys key1 key2...    | 将给定有序集合的交集存储在key3中，对相应元素对应的score进行sum操作，numkeys为集合数量 |
| zunionstore key3 numkeys key1 key2..     | 求并集                                                       |
| zdiffstore key3 numkeys key1 key2...     | 求差集                                                       |
| zrange key start end                     | 获取有序集合start和end之间的元素（从低到高）                 |
| zrevrange key start end                  | 获取有序集合start和end之间的元素（从高到低）                 |
| zrevrank key member                      | 获取指定有序集合中指定元素的排名（score从大到小排序）        |



#### （4）场景：礼物排行榜，步数排行榜，热度排行榜等

# 四、三种特殊数据结构

## 1.Bitmap

#### （1）介绍：存储连续的二进制数字，每个下标叫做offset（偏移量）

#### （2）命令

| 命令                                | 作用                                                  |
| ----------------------------------- | ----------------------------------------------------- |
| setbit key offset value             | 设置指定offset位置的值                                |
| getbit key offset                   | 获取指定offset位置的值                                |
| bitcount key start end              | 获取start和end之前值为1的元素个数                     |
| bitop operation destkey key1 key2.. | 对一个或多个bitmap进行计算，计算福：and，or，xor，not |

#### （3）场景

用户签到情况、活跃用户情况、用户行为统计（点赞视频）

## 2.HyperLogLog

#### （1）介绍：Redis实现了这种算法并且提供了相关api

HyperLogLog是一种基数计算概率算法，redis提供的占用空间非常小，12k的空间就能存储接2^64个不同元素，主要有两种计数方式：稀疏矩阵和稠密矩阵

#### （2）命令

| 命令                          | 介绍                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| pfadd key element1 element2.. | 添加一个或多个元素到HyperLogLog中                            |
| pfcount key1 key2             | 获取一个或多个HyperLogLog的唯一计数                          |
| pfmegre destkey key1 key2     | 将多个HyperLogLog合并到destkey中，destkey会结合多个源算出唯一计数 |

#### （3）场景

热门网站每日/每周/每月访问ip数统计、热门帖子uv统计

## 3.Geospatial index

#### （1）介绍：主要存储地理位置信息，基于sorted set实现

#### （2）命令

| 命令                                             | 介绍                                           |
| ------------------------------------------------ | ---------------------------------------------- |
| geoadd key longitude1 latitude1 member1 ...      | 添加一个或多个元素对应的经纬度信息到GEO中      |
| geopos key member1 member2..                     | 返回给定元素的经纬度信息                       |
| geodist key member1 member2 M/KM/FT/MI           | 返回两个元素之间的距离                         |
| georadius key longitude latitude radius distance | 获取指定位置distance范围内的其他元素           |
| georadiusbymember key member radius distance     | 类似于georadius，只是参照的中心点是geo中的元素 |



#### （3）场景

附件的人

## 五、持久化

Redis实现了三种持久化的方式

### 1.快照（RDB）

#### 概念

redis可以通过创建快照的方式来获取**某个时间点**上内存上的数据，创建之后可以对快照进行备份和复制，快照持久化是 Redis 默认采用的持久化方式

#### 使用

`save time(秒) num(个数) `在time秒之后如果有num个key发生了变化，redis会自动触发bsgave命令创建快照

save：同步保存操作，会阻塞redis主进程

bgsave：fork出一个子进程，子进程执行，不会阻塞主进程

### 2.只追加文件（AOF）

#### 概念

AOF的实时性更好，要通过`appendly yes`开启，开启AOF持久化之后每执行一条redis命令，redis就会将数据写到AOF缓存区中，然后在写入AOF文件中，最后根据相关的fysnc策略来实时持久化

#### 基本流程

1. 将所有命令追加到缓冲区
2. 将AOF缓冲区的数据写入到AOF文件中，这一步需要调用write函数（系统调用），write将数据写入到系统内核缓冲区后直接返回
3. AOF缓冲区根据对应的持久化方式（fsync策略）向硬盘同步操作，这一步需要调用fsync函数（系统调用），fysnc针对单个文件对其进行强制硬盘同步，会阻塞线程直到写入磁盘后返回
4. AOF文件会越来越大，要定期对AOF文件进行重写以达到压缩的目的
5. redis重启的时候，可以加载AOF文件进行数据恢复

> AOF持久化策略
>
> - appendfsync always：主线程调用write函数写入后，后台线程立即调用fsync函数同步AOF文件，fsync完成后线程返回
> - appendfsync everysec：主线程调用write函数写入到缓冲区后直接返回，由后台线程每秒钟调用一次fsync函数同步AOF文件
> - appendfsync no：主线程调用write函数写入到缓冲区后直接返回，由操作系统决定多久调用一次fsync函数，Linux一般是30秒一次

#### 相关概念

系统调用：Linux 系统直接提供了一些函数用于对文件和设备进行访问和控制

为什么先执行命令在记录日志：（都是为了性能）

- 避免额外的语法检查（如果刚执行完命令redis就宕机会导致对应修改记录丢失）
- 不会阻塞当前命令的执行（会阻塞后续命令的执行）

怎么重写AOF文件：

- 当AOF文件太大的时候，redis在后台自动重写AOF产生一个新的AOF文件，这个新的AOF文件和原有的AOF文件所保存的数据库状态一样，但体积更小
- 重写时会有大量的写入操作，避免阻塞进程会在子进程里面执行，重写期间redis会维护一个AOF重写缓冲区，该缓冲区会在子进程创建AOF文件期间记录服务器执行的所有写命令，当子进程完成创建新的AOF文件后会把重写缓冲区的所有内容追加到新的AOF文件的末尾，最后服务器会用新的AOF文件来替换旧的AOF文件
  - 因为在重写的时候会维护一个AOF重写缓冲区，但是此时执行的写命令还是会记录到AOF缓冲池里面，所以会导致这些命令被记录两次，导致使用大量内存，redis7.0版本之后AOF重写做了优化优化了这一个问题

AOF效验机制：在redis启动时对AOF文件进行检查，以判断文件是否完整，是否有丢失的数据

- 这个机制通过使用效验和的数字来验证AOF文件
- 这个效验和是通过对整个AOF文件内容进行CRC64算法计算得出的数字，记录在AOF文件末尾，redis在启动的时候会比较计算出的效验和与文件末尾报错的效验和，从而判断AOF文件是否完整，如果不完整就会拒绝启动

### 3.RDB和AOF混合持久化

- redis保存的数据丢失一些也没什么关系可以使用RDB持久化
- 不建议单独使用AOF，时不时的创建一个RDB快照可以进行数据备份、更快的重启以及解决AOF的引擎错误
- 如果保存的数据要求安全性比较高的话，可以混合使用

## 六、线程模型

对于读命令，Redis一直是单线程模型。Redis4.0版本后引入了多线程来处理一些大键值对的异步操作，Redis6.0版本引入多线程来处理网络请求（提高IO读写性能）

### 单线程模型

Redis基于Reactor模式设计了一套高效的事件处理模型，这套处理模型对应的是Redis中的文件事件处理器，以单线程模型运行，所以一般交Redis是单线程模型

> 这个文件事件处理器：多个socket连接、IO多路复用程序、文件事件分派器、事件处理器
>
> - 使用IO多路复用同时监听多个套接字，并根据套接字目前执行的任务来为套接字关联不同的事件处理器
> - 当被监听的套接字准备好执行连接，读取，写入，关闭等操作的时候，与操作对应的文件事件就会产生，对应的事件处理器就会进行处理

如何监听大量客户端的连接的：

Redis通过IO多路复用程序来监听大量客户端的连接（或者说监听多个socket），会将感兴趣的事件以及类型注册到内核中并监听每个事件是否发生

好处：IO多路复用技术的使用让redis不需要创建多余的线程来监听客户端的大量连接，降低了资源的消耗

### 多线程

Redis在4.0版本之后就引入了多线程的支持，但是主要是用于异步处理请求避免阻塞的

为什么不使用多线程：

- 单线程编程更容易维护
- redis的性能瓶颈不在CPU，主要在网络和内存
- 多线程存在死锁、线程上下文切换等问题，甚至会影响性能

使用多线程（6.0之后）：

- 提高网络IO读写性能，主要用于网络数据的读写

### 后台线程

虽然一直说redis是单线程模型，但实际上还有一些后台线程用于执行一些比较耗时的操作

- 通过 `bio_close_file` 后台线程来释放 AOF / RDB 等过程中产生的临时文件资源。
- 通过 `bio_aof_fsync` 后台线程调用 `fsync` 函数将系统内核缓冲区还未同步到到磁盘的数据强制刷到磁盘（ AOF 文件）。
- 通过 `bio_lazy_free`后台线程释放大对象（已删除）占用的内存空间.

## 七、内存管理

### 1.给数据设置过期时间

为什么设置：因为内存有限，如果所有的数据都永久保存那么很快就会outofmemory

```lua
expire key time # 数据在time秒后过期
setex key 60 value # 数据在60s后过期 （仅字符串类型能这样设置）
ttl key # 查看数据还有多久过期
persist key # 移除一个建的过期时间
```

作用：

- 有助于缓解内存消耗
- 针对验证码和用户登录token这种数据只存在于某个时间段的业务常见，能提高性能不用自己维护判断

### 2.如何判断过期

Redis通过一个过期词典（hash表）来保存过期时间，过期字典将键指向redis数据库中的某个key（键），过期字典的值可以是一个long long类型的整数，这个整数就是保存的过期时间（毫秒精度）

### 3.过期数据的删除

- 惰性删除：只会在使用key的时候判断key是否过期，这样对CPU友好，但是可能导致太多过期key没有删除
- 定期删除：每隔一段时间抽取一批过期key进行删除，redis底层会通过限制删除操作的时长和频率来减少删除操作对CPU时间的影响，对内存更加优化
- redis采用的是定期删除+惰性删除

### 4.redis内存淘汰策略

因为光靠设值过期时间还是会有大量key堆积在内存中，所以还要采用淘汰策略

- volatile-lru：从已经设置过期时间的数据集中挑选最近最少使用的数据淘汰
- volatile-ttl：从已经设置过期时间的数据集中挑选将要淘汰的数据淘汰
- volatile-random：从已经设置过期时间的数据集中任意选择数据淘汰
- volatile-lfu（4.0后新增）：从已经设置过期时间的数据集中挑选最不经常使用的数据淘汰
- allkeys-lru：当内存不足以容纳新的数据的时候，在键空间中移除最近最少使用的key（常用）
- allkeys-random：从数据集中挑选任意数据进行淘汰
- allkeys-lfu（4.0后新增）：当内存不足以容纳新数据的时候，在键空间中移除最不经常使用的key
- no-eviction：禁止驱逐数据，当内存不足的时候写入操作会报错

> lfu算法也是查找hotkey的一种方案，使用--hotkeys参数来查找hotkey时需要把`maxmemory-policy` 参数设置为 LFU 算法