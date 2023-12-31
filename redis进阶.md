# 一、事务

Redis提供了**一种将多个命令请求打包的功能，然后按顺序执行打包的所有命令，并且不会被中途打断，**注意区别其与关系型数据库如mysql事务的区别

## 1.使用

- `MULTI:开启事务`：这个命令之后可以输入多个命令，Redis不会立即执行这些命令，而是会放到队列中
- `EXEC:提交事务`：调用EXEC命令后，会执行MULTI后输入的所有命令（FIFO顺序，即先进先出）
- `WATCH:监听key`：当watch命令监视了一个key，如果有其他客户端/session修改这个key，事务不会执行
- `DISCARD:取消事务`:会清空事务队列中保存的所有队列

## 2.Redis的事务特性

### （1）不支持原子性

Redis事务在运行错的情况下，除了出现错误指令的命令外，其余指令都会正常执行，并且Redis不支持回滚（mysql里面有undo log日志所以支持回滚），所以Redis不支持原子性

#### 解决其缺陷

Redis在2.6版本之后开始使用Lua脚本，与事务非常类似，Lua脚本可以批量执行多条redis命令，这些命令会被提交到服务器一次性执行完成，减小了网络开销

Lua脚本执行时不会有其他脚本或Redis命令执行，保证了操作不会被其他指令干扰

但是Lua脚本不会执行出错后的redis语句，且前面执行成功的redis语句也不会被回滚，所以还是不能完全解决原子性的问题

### （2）不支持持久性

前面说了Redis有三种持久化方式

- 快照（RDB）
- 只追加文件（AOF），三种fsync策略
  - always：可以基本满足持久化但是太耗性能
  - no和everysec：存在数据丢失情况
- RDB与AOF混合

所以redis不支持事务的持久性

# 二、性能优化

## 1.减少网络传输

使用批量操作减少网络传输

一个Redis命令执行：

1. 发送命令
2. 命令排队
3. 执行命令
4. 返回结果

第一步到第四步的执行时间称为一次RTT（往返时间），也就是数据在网络上传输的时间

使用批量操作可以减少网络传输次数，进而有效减少网络开销，大幅减少RTT（同时还能减少socket I/O成本）

### （1）原生批操作命令

- MGET、MSET
- HMGET、HMSET
- SADD

### （2）pipeline

对于不支持批量操作的命令，我们利用pipeline（流水线）将一批redis命令封装成一组，一次性提交到服务器（只需要一次网络传输）

- 原生批量操作命令是原子操作，pipeline 是非原子操作。
- pipeline 可以打包不同的命令，原生批量操作命令不可以。
- 原生批量操作命令是 Redis 服务端支持实现的，而 pipeline 需要服务端和客户端的共同实现。

## 2.大量key集中过期问题

对于过期的key采用定期删除+惰性删除策略

因为这个定期任务线程是在redis主线程中执行的，如果突然遇到大量过期的key会导致客户端请求没办法被及时处理，响应速度比较慢

### 解决

- 给key设置随机过期时间
- 开启lazy-free（惰性删除/延迟释放），让redis采用异步方式延迟释放key使用的内存，将该操作交给单独的子线程处理，避免阻塞主线程

## 3.bigkey（大key）

什么是bigkey，简单来说就是value值比较大，占用内存比较多（像String类型的value超过10kb，符合类型的value包含元素超过5000个）

### 危害

会消耗更多的内存空间和带宽，还会对性能造成比较大的影响，所以应该尽量避免redis中存在bigkey

### 查找

- 使用Redis自带的`--bigkeys`参数来查找
- 借助开源工具分析RDB文件（持久化方式采用的RDB）
- 借助公有云的Redis分析服务

### 解决

- 分割bigkey：将一个bigkey分成多个小key
- 手动清理：4.0+可以使用`UNLINK`命令来异步删除一个或多个指令，4.0以下可以考虑使用`SCAN`命令结合`DEL`命令批次删除
  - unlink：用来删除指定的key，与del的区别在于他执行后会执行命令之外的线程中执行实际的回收内存，所以他不是线程阻塞的，而del是阻塞的，所以他相当于是吧键和键空间断开连接
  - scan：基于游标的遍历器
- 采用合适的数据结构：比如使用HyperLogLog来统计页面UV
- 开启lazy-free（惰性删除/延迟释放）：让redis采用异步的方式延迟释放key使用的内存，将该操作交给单独的子进程避免阻塞线程

## 4.hotkey（热key）

如果一个key的访问次数明显多余其他key的话，这个key就可以看做是hotkey，如redis每秒访问5000次，他访问超过2000次，这个key就可以看做热key

### 危害

处理hotkey会占用大量的CPU和带宽，可能会影响redis实例对其他请求的正常处理。如果突然访问hotkey的请求超出了redis的处理能力，redis会宕机，这种情况下，大量请求会落到后面的数据库上造成数据库崩溃

### 查找

- 使用Redis自带的`--hotkeys`参数来查找（需要开启lfu算法）
- 使用`monitor`命令（可以监听redis所有操作的方式，该命令对redis性能影响较大，因此禁止长时间开启monitor）
- 借助开源项目
- 根据业务情况提取预估
- 业务代码分析记录
- 借助公有云redis的分析服务

### 解决

- 读写分离：主节点处理写请求，从节点处理读请求
- 使用redis cluster：将热点数据分散存储在多个redis节点上
- 二级缓存：hotkey采用二级缓存的方式进行处理，将hotkey存放一份到JVM本地内存中使用
- 使用公有云的redis服务可能自带的有相关优化解决功能

## 5.慢查询

执行时间比较长的命令（一个命令分四步：发送命令，命令排队，命令执行，返回结果）

如一些O（n）复杂度的命令

- keys *：查找所有符合规则的key
- hgetall：返回一个hash中所有的键值对
- lrange：会返回List中指定范围内的元素
- smembers：返回set中的所有元素
- sinter/sunion/sdiff：查找交集，并集，差集
- .....

### 查找

在 `redis.conf` 文件中，我们可以使用 `slowlog-log-slower-than` 参数设置耗时命令的阈值，并使用 `slowlog-max-len` 参数设置耗时命令的最大记录条数。

当 Redis 服务器检测到执行时间超过 `slowlog-log-slower-than`阈值的命令时，就会将该命令记录在慢查询日志(slow log) 中，这点和 MySQL 记录慢查询语句类似。当慢查询日志超过设定的最大记录条数之后，Redis 会把最早的执行命令依次舍弃

> 因为慢查询日志也会占用空间，记录条数过大也会占用内存过高

组成（通过slowlog get获取慢日志）

- 唯一渐进的日志标识符
- 执行命令的时间戳
- 执行所需的时间（微妙）
- 组成命令参数的数组
- 客户端ip地址和端口
- 客户端名称

## 6.内存碎片

# 三、生产问题

## 1.缓存穿透

大量不合理的key（即不存在内存，也不存在数据库中）进行请求，这样会导致对数据库造成巨大压力可能会宕机

### 解决

- 参数效验：一些不合法的参数请求直接返回异常给客户端，如查询的数据id<0,传入邮箱格式等
- 缓存无效key：如果缓存和数据库都查不到某个key的数据就写一个到redis中去设置过期时间
- 布隆过滤器：一种数据结构，通过他可以判断这个key是否合法
  - 吧所有可能存在的请求值会存放在布隆过滤器中（通过对值进行hash进行保存的），当用户请求发过来之后会判断用户请求的值是否存在布隆过滤器中，不存在的话会直接返回请求参数错误信息个客户端，存在才会继续操作
  - 加入
    - 使用布隆过滤器里面的哈希函数对元素值进行计算得到哈希值
    - 根据得到的哈希值在位数组中吧对应下标设置为1
  - 判断
    - 对给定元素再次进行相同的哈希计算
    - 得到值后判断位数字中的每个元素是否都为1，如果都为1，说明这个值在布隆过滤器中，如果存在一个值不为1，那么该元素不在其中
    - 因为不同字符串可能哈希出来的位置相同，所以可能造成这个元素存在的误判（也就是显示存在但是可能实际不存在），所以可以适当增加位数组大小或者调整哈希函数来降低概率

## 2.缓存击穿

缓存击穿中对应的数据是热点数据，该数据存在于数据库中不存在于缓存中（由于缓存中的那份数据已经过期），可能导致大量的请求直接到达数据库，对数据库造成压力可能导致宕机（如秒杀）

### 解决

- 设置热点数据永不过期
- 针对热点数据提前预热，将其存入缓存中并设置合理的过期时间（如秒杀就设置秒杀结束前该数据不会过期）
- 请求数据库写数据到缓存前，先获取互斥锁，保证只有一个请求到数据库上减少数据库压力

## 3.缓存雪崩

缓存在同一时间大量失效，导致大量的请求都直接请求到数据库上，对数据库造成巨大压力

### 解决

#### （1）针对redis服务不可用

- 采用集群，避免单机出现问题整个缓存服务都没办法使用
- 限流，避免同时处理大量的请求

#### （2）针对热点缓存失效的情况

- 设置不同的失效时间比如随机设置缓存的失效时间
- 缓存永不失效
- 设置二级缓存

## 4.保证缓存和数据库数据一致的问题

首先一个业务刚开始都直接使用数据库来管理数据，随着业务量的增加开始引入redis，此时业务量需求不大，redis专门拿来读，mysql专门拿来写（缓存利用率低并且数据不一致），当业务量到达一定需求之后，为了保证性能，要对redis中的数据进行读写，对mysql进行读写，那么分为四种

1. 更新缓存
   1. 先更新缓存，在更新数据库
   2. 先更新数据库，在更新缓存
2. 删除缓存
   1. 先删除缓存，在更新数据库
   2. 先更新数据库，在删除缓存

导致不一致的问题：并发和异步（异步的处理是重试，因为第二部操作挂掉那么这四种方式都不能保证数据一致性）

### A.并发问题

#### （1）先更新缓存，在更新数据库

假如有线程a

1. a修改了这个数据的值
2. 在同步到数据库的时候还没到达同步的时间
3. 但是此时这个值快过期了，当这个值过期之后，会从数据库中读取这个值
4. 会造成数据重新覆盖

##### 并发问题（未解决）

假设有两个线程a和b

1. a更新了数据的值
2. b更新了数据的值
3. 此时a的值先更新到数据库，b随后更新到数据库
4. 会造成a或者b修改的值被覆盖掉

#### （2）先更新数据库，在更新缓存

假如有线程a

1. a修改了这个数据的值
2. 此时数据库中修改了这个数据，但是此时这个数据还没过期
3. 那么用户此时查找出来的数据依旧是之前的值，会导致用户感觉没有修改成功

##### 并发问题（未解决）

假如有2个线程a和b

1. a修改了数据的值
2. b修改了数据的值
3. 会导致某一个数据被覆盖，与之前一样

#### （3）先删除缓存，在更新数据库

##### 并发问题（未解决）

假如有两个线程a和b

1. a先删除缓存，在更新这个数据的值
2. 在a更新数据库的同时，b发现读不到这个数据，所以先从数据库中读，此时还没被修改导致读到了之前的值
3. a更新完之后，因为b读到的是之前的值
4. b会吧旧值写入到缓存中

#### （4）先更新数据库，在删除缓存

##### 并发问题（能大概解决）

假如有两个线程a和b

1. 缓存中数据不存在，a从数据库中读取
2. b更新数据，先更新数据库，在删除缓存
3. a将旧值写入到缓存中

但是一般来说这种情况不容易发生，因为更新数据库数据库是会加锁的，而且还要保证

1. 数据刚好过期
2. a读了这个值之后b刚好更新数据
3. **更新数据和删除缓存的速度**要**小于**a读取旧值写入缓存中的速度（因为写要加锁的原因很难满足）

所以这种方式大概率的解决了并发问题

### B.异步问题

解决异步问题那么就是保证都执行成功

解决方式：异步重试

#### （1）使用消息队列

也就是写操作是针对mysql和mq了，写入mq之后的消息被拿来消费的时候在针对redis进行删除操作

消息队列特性：

- 写到队列中的消息未被成功消费前不会消失（重启项目也不会消失）
- 下游从队列拉取消息，成功消费后才会消失，否则还会继续投递消息给消费者

#### （2）订阅数据库变更日志，在操作缓存

当一条数据修改时，mysql会产生一条变更日志（Binlog），我们可以订阅这个日志拿到具体的数据操作，目前有比较开源的成熟中间件：canal

- 只有mysql成功，binlog肯定会有
- canal会自动吧数据库变更日志投递给下游的消息队列

也就是当更新数据库时，canal将binlog日志投递给mq，消费的时候在针对redis进行删除操作

### C.主从库延迟和延迟双删的问题

主从库延迟可能导致缓存被种回旧值，所以要删除缓存，但是此时不能立即删除，要用延迟删除，所以也叫延迟双删

- 延迟时间大于主从复制的时间
- 延迟时间大于线程b读取数据库+写入缓存的时间

所以一般这个时间不好评估（高并发和分布式环境下）

### D.强一制

通过一些协议将缓存和数据库中的数据保持强一致性，这样性能全无，一般不用

# 四、阻塞原因

## 1.O（n）命令

因为O（n）复杂度的命令有时候是会全表扫描的，n越大执行耗时越大，可能造成客户端阻塞

## 2.save创建快照

手动使用save命令生成快照的话可能会阻塞主线程（save操作是同步保存操作，会阻塞主线程），一般默认情况下redis默认配置会使用bgsave

## 3.AOF

### （1）AOF记录时

然后记录日志的途中也会阻塞后续命令的执行（AOF记录日志是在redis主线程中进行的）

### （2）AOF刷盘时

在同步到数据库中时，fsync函数的调用会阻塞线程直到将数据全部保存到数据库中（会阻塞后序命令）

### （3）AOF重写

前面已知AOF重写的过程，当将缓冲区的新数据写到新文件的过程中会产生阻塞

## 4.bigKey

### （1）本身

因为占用内存大，所以它本身存在就会造成阻塞：

- 客户端超时阻塞：操作大key比较耗时，从客户端看就是请求很久都没有响应
- 网络阻塞：如果一个大key是1mb，那么如果每秒访问量是1000，那么对于流量带宽小的服务器是运行不了的
- 阻塞工作线程：如果使用del删除大key时，会阻塞工作线程，没办法处理后续命令

### （2）查找大key时

当我们在使用 Redis 自带的 `--bigkeys` 参数查找大 key 时，最好选择在从节点上执行该命令，因为主节点上执行时，会**阻塞**主节点，所以一般：

- 使用scan命令查找
- 分析RDB文件

### （3）删除大key时

删除操作本质是是否键值对占用的内存空间，但是在具体操作时，在应用释放内存时，**操作系统会将需要释放掉的内存块插入一个空闲内存块的链表**，方便后续管理和分配，这个过程会阻塞当前释放内存的应用程序

## 4.清空数据库

与删除bigkey一样的道理

## 5.集群扩容

## 6.Swap（内存交换）

## 7.CPU竞争

## 8.网络问题