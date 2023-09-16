# 一、存储引擎

## 1.MySQL体系结构

| 客户端连接器（native c api，jdbc，php，net等）               |
| ------------------------------------------------------------ |
| 连接池（Authorization，Thread Reuse，Caches，Connection limits等） |
| SQL接口（DML,DDL,Triggers,etc等）解析器（Query Transient等）查询优化器，缓存 |
| 可插拔存储引擎（Innodb，myisam，ndb，memory等）              |
| 系统文件 文件和日志                                          |

- 连接层：最上层是一些客户端和链接服务，主要完成一系列连接、授权认证等安全方案，服务器也会为每个连接用户验证其操作权限
- 服务层：这层架构主要完成大多数核心服务，如SQL接口，完成缓存的查询，SQL的分析和优化，部分内置函数的执行，所有跨存储引擎的功能也在这一层实现，如过程、函数
- 引擎层：存储引擎真正的负责了MySQL中数据的存储和提取，服务器通过API和存储引擎进行通信，不同的存储引擎具有不同的功能，这样也能根据对应需求选取合适的存储引擎
- 存储层：将数据存储在文件系统之上，并完成与存储引擎的交互

## 2.存储引擎简介

> 存储引擎：存储数据、建立索引、更新/查询数据等技术的实现方式，存储引擎是基于表的，而不是基于库的，所以存储引擎也可被称为表类型

### 1.创建表时指定存储引擎

```sql
create table 表名{
} engine = 存储引擎名称 [comment 表注释];
```

### 2.查询当前数据库支持的存储引擎

```
show engines
```

## 3.存储引擎特点

### InnoDB

#### 介绍：InnoDB是一种兼顾可靠性和高性能的通用存储引擎，在mysql5.5版本后InnoDB就是默认的MySQL存储引擎

#### 特点

- DML操作遵循ACID模型（事务的四大特性），支持**事务**
- **行级锁**，提高并发访问性能
- 支持**外键**FOREIGN KEY约束，保证数据完整性和正确性

#### 文件

xxx.ibd：xxx代表的是表名，InnoDB引擎的每张表都会对应这样一个表空间文件，存储该表的表结构（frm，sdi）、数据和索引

参数：innodb_file_per_table：决定多张表共享表空间还是每张表对应一个表空间参数（`show variables like '参数'`即可查看）

可以在命令行根据命令：`ibd2sdi ibd文件 `进行查看

#### 逻辑存储结构

- TableSpece：表空间
- Segment：段
- Extent：区
- Page：页
- Row：行

![逻辑存储结构.png](/upload/逻辑存储结构.png)

### MyISAM

#### 介绍：MyISAM是MySQL早期的默认存储引擎

#### 特点

- 不支持事务，不支持外键
- 支持表锁，不支持行锁
- 访问速度快

#### 文件

- xxx.sdi：存储表结构信息（json格式的数据，可以进入json.cn进行格式化展示）
- xxx.MYD：存储信息
- xxx.MYI：存储索引

### Memory

#### 介绍：Memory引擎的表数据是存储在内存中，由于收到硬件问题或断电等影响，这些表只能当作临时表或缓存使用

#### 特点：

- 内存存放
- hash索引（默认）

#### 文件

#### xxx.sdi：存储表结构信息



### 三种存储引擎的区别

| 特点         | InnoDB              | MyISAM | Memory |
| ------------ | ------------------- | ------ | ------ |
| 存储限制     | 64TB                | 有     | 有     |
| 事务安全     | **支持**            | -      | -      |
| 锁机制       | **行锁**            | 表锁   | 表锁   |
| B+tree索引   | 支持                | 支持   | 支持   |
| Hash索引     | -                   | -      | 支持   |
| 全文索引     | 支持（5.6版本之后） | 支持   | -      |
| 空间使用     | 高                  | 低     | N/A    |
| 内存使用     | 高                  | 低     | 中等   |
| 批量插入速度 | 低                  | 高     | 高     |
| 支持外键     | **支持**            | -      | -      |



## 4.存储引擎选择

> 在选择存储引擎时根据系统的特点选择合适的引擎，对于复杂的应用系统还能选择多种存储引擎进行组合



- InnoDB：MySQL的默认存储引擎，支持事务，外键，行级锁，如果对事务的完整性有较高，要求在并发条件下要求数据的一致性，数据操作除了插入和查询之外，还包含很多的更新、删除操作，那么InnoDB存储引擎是比较合适的
- MyISAM：如果应用是以读写操作为主，很少的更新和删除操作，并且对事务的完整性和并发性要求不高，那么这个引擎比较适合 （Mongodb）
- Memory：将所有数据保存在内存中，访问速度快，通常用于临时表及缓存。Memory的缺陷就是对表的大小有限制，太大的表无法缓存在内存中，而且无法保证数据的安全（redis）

# 二、索引

## 1.索引概述

是帮助MySQL高效获取数据的数据结构（有序）。在数据之外，数据库系统还维护着满足特定查找算法的数据结构，这些数据结构以某种方式引用（指向）数据，这样就可以在这些数据上实现高级查找算法，这种数据结构就是索引

### 优缺点

| 优势                                                        | 劣势                                                         |
| ----------------------------------------------------------- | ------------------------------------------------------------ |
| 提供数据检索的效率，降低数据库的IO成本                      | 索引列也是要占用空间的                                       |
| 通过索引列对数据进行排序，降低数据排序的成本，降低CPU的消耗 | 索引大大提高了查询效率，同时也降低更新表的速度，如对表进行插入更新删除时效率降低 |

## 2.索引结构

MySQL的索引在存储引擎层中实现，不同的存储引擎有不同的结构，主要包含以下几种：

| 索引结构              | 描述                                                         |
| --------------------- | ------------------------------------------------------------ |
| B+Tree索引            | 最常见的索引类型，大部分引擎都支持B+树索引                   |
| Hash索引              | 底层数据结构是用哈希表实现的，只有精确匹配索引列的查询才有效，不支持范围查询 |
| R-tree（空间索引）    | 空间索引是MyISAM引擎的一个特殊索引类型，主要用于地理空间数据类型，通常使用较少 |
| Full-text（全文索引） | 是一种通过建立倒排索引，快速匹配文档的方式，类似于Lucene，Solr，ES |

| 索引                  | InnoDB        | MyISAM | Memory |
| --------------------- | ------------- | ------ | ------ |
| B+Tree索引            | 支持          | 支持   | 支持   |
| Hash索引              | 不支持        | 不支持 | 支持   |
| R-tree（空间索引）    | 不支持        | 支持   | 不支持 |
| Full-text（全文索引） | 5.6版本后支持 | 支持   | 不支持 |



### 二叉树

二叉树缺点：顺序插入时，会形成一个链表，查询性能大大降低。数据量大的情况下层级较深，检索速度慢

红黑树缺点：大数据量情况下，层级较深，检索速度慢

### B-Tree（多路平衡查找树）

> 数的度数是指一个节点的子节点个数

以一颗最大度数（max-degree）为5（5阶）的b-tree为例（每个节点最多存储4个key，5个指针）：

数据存储在key中

实例：按此时效果（max-degree=5），则每片树叶只能4个节点

插入 23 234 345 899四个节点之后

| 0023 | 0234 | 0345 | 0899 |
| ---- | ---- | ---- | ---- |
|      |      |      |      |



此时在插入数据1000，数据会从中分裂，中间数据向上，其余数据分开，当前数据插入，形成树

（注意此时 0345是key，他的左边是指针指向左下角的子节点，他的右边是指针指向右下角的子节点）



![B-tree生成.png](/upload/B-tree生成.png)

### B+Tree

- 所有的元素都会出现在叶子节点（叶子节点存储数据）
- 叶子节点形成单向链表

以一颗最大度数（max-degree）为5（5阶）的b+tree为例：

插入 1000 567 234 232 后

| 0232 | 0234 | 0567 | 1000 |
| ---- | ---- | ---- | ---- |
|      |      |      |      |



当插入数据890后，中间数据向上分裂，其余数据分开，当前数据插入，子节点仍然会保留该中间值，左侧指针指向该值

![B+tree生成.png](/upload/B+tree生成.png)MySQL索引数据结构对经典的B+Tree进行了优化，在原有的B+Tree基础上，增加了一个指向相邻叶子节点的链表指针，就形成了带有顺序的B+Tree，提高了区间访问的性能（即不再是单向的链表了）

### Hash

哈希索引就是采用一定的hash算法，将键值换算成新的hash值，映射到对应的槽位上，然后存储在hash表中

如果两个（或多个）键值，映射到一个相同的槽位上，他们就产生了hash冲突（也称hash碰撞），可以通过链表来解决

特点：

- hash索引只能用于对等比较（=，in），不支持范围查询（between，<,>,...）
- 无法利用索引完成排序操作
- 查询效率高，通常只需要一次检索就可以了，效率高于B+Tree索引

存储引擎支持：Memory引擎，而InnoDB中具有自适应的hash功能，hash索引是存储引擎根据B+Tree索引在指定条件下自动构建的

### 为什么InnoDB存储引擎选择使用B+Tree索引结构

1. 相对于二叉树，层级更少，搜索效率更高
2. 对于B-Tree，无论是叶子节点还是非叶子节点，都会保存数据，这样导致一页中存储的键值减少，指针跟着减少，要同样保存大量数据，只能增加树的高度，导致性能较低
3. 相对于hash索引，B+Tree支持范围匹配及排序操作

## 3.索引分类

### 主键索引（PRIMARY）

针对表中主键创建的索引（默认自动创建，只能一个）

### 唯一索引（UNIQUE）

避免同一个表中某数据列中的值重复（可以有多个）

### 常规索引

快速定位特定数据（可以有多个）

### 全文索引（FULLTEXT）

全文索引查找的是文本中的关键词，而不是比较索引中的值（可以有多个）



在InnoDB中，根据索引的存储形式，又可以分为以下两种

### 聚集索引

将数据存储与索引放到了一块，索引结构的叶子节点保存了行数据（必须有而且只有一个）

选取规则：

- 如果存在主键，主键索引就是聚集索引
- 如果不存在主键，将使用第一个唯一（UNIQUE）索引作为聚集索引
- 如果表没有主键，或没有合适的唯一索引，则InnoDB会自动生成一个rowid作为隐藏的聚集索引

### 二级索引

将数据与索引分开存储，索引结构的叶子节点关联的是对应的主键

### 回表查询![回表查询.png](/upload/回表查询.png)

## 4.索引相关

### 判断以下哪个SQL执行效率高？

```
select * from user where id = 10;
select * from user where name = 'Arm';
```

根据id查只需要进行一次聚集索引，根据name查哪怕name有索引也会进行回表查询，所以根据id查效率更高

### InnoDB主键索引的B+Tree高度为多高

假设：一行数据为1k，一页中可以存储16行这样的数据。InnoDB的指针占用6个字节的空间，主键即使为bigint，占用字节数为8。

如上图，一个page就是类似于包含15 20那一块，

高度为2时：n*8+（n+1）*6 = 16 *1024 算出 n 约为**1170 （n为key值）** **存储数据：1171\* 16 = 18736 个**

高度为3时：1171*1171*16 = 21939856（每个指针会在指向一页，所以就是1171*1171）

## 5.索引的操作语法

### 创建索引

```
create [unique | fulltext] index 索引名 on 表名 (字段,...);
```

`如 create index index_user_name on user(name);`为user表的name字段创建一个名为` index_user_name`的索引

`如 create index idx_user_pro_age_sta on user(profession,age,status); `为user表创建联合索引

### 查看索引

```
shwo index from 表名;
```

### 删除索引

```
drop index 索引名 on 表名;
```

## 6.SQL性能分析

### SQL执行频率

MySQL客户端连接成功后，通过show [session | global] status 命令可以查看服务器状态信息。

通过当前指令可以查看当前数据的增删改查的访问频次：`select global status like 'Com_______'(七个下划线)`

### 慢查询日志

慢查询日志记录了所有执行时机超过指定参数（long_query_time,单位：秒，默认10）的所有SQL语句的日志

`show variables like 'slow_query_log'；`查看是否开启

MySQL 的慢查询日志默认没有开启，需要在mysql配置文件(/etc/my.cnf)中配置如下：

开启MySQL慢查询开关：slow_query_log = 1设置慢日志的时间为2秒，SQL语句执行时间超过2秒，就会视为慢查询，记录慢查询日志：long_query_time = 2

重启mysql `systemctl restart mysqld`



### profile详情

show profile 能够在做SQL优化时帮助我们了解时间都耗费到哪里去了。通过have_profile参数，能够看到当前MySQL是否支持profile : `select @@have_profiling;`

默认profiling是关闭的`select @@profiling;`查看，可以通过set语句在session/global级别开启profiling `set profiling = 1;`

```sql
show profiles;  查看每一条SQL耗时的基本情况
show profile for query query_id;   查看指定的query_id的SQL语句各个阶段的耗时情况
show profile cpu for query query_id;   查看指定query_id的SQL语句CPU使用的情况
```

### explain执行计划

explain或者desc命令获取MySQL如何执行select语句的信息，包括在select语句执行过程中表如何连接和链接的顺序。

```
explain/desc select 字段列表 from 表名 where 条件；
```

explain执行计划个字段含义：

- id：select查询的序列号，表示查询中执行select子句或者是操作表的顺序(id相同，执行顺序从上到下，id不同，值越大，越先执行)
- select_type：表示select的类型，常见的取值有simple（简单表，即不用表连接或者子查询）、primary（主查询，即外层的查询）、union（union中的第二个或者后面的查询语句）、subquery（select/where之后包含子查询）等
- type：表示连接类型，性能由好到差的连接类型为Null，system，const（唯一索引），eq_ref、ref（非唯一索引）、range（）、index（对所有索引）、all（全表扫描）
- possible_key：显示可能应用在这张表上的索引，一个或多个
- key：实际使用的索引，如果是null，则没有使用索引
- key_len：表示索引使用的字节数，该值为索引字段最大可能长度，并非实际使用长度，在不损失精确性的前提下，长度越短越好
- rows：MySQL认为必须要执行查询的行数，在InnoDB引擎的表中，是一个估计值，可能并不总是精确的
- filtered：表示返回结果的行数占需读取行数的百分比，filtered的值越大越好

## 7.索引使用原则

### 验证索引效率

在未建立索引前，查看SQL耗时：`select * from 表名 where 条件`;

针对字段创建索引：`create index 索引名 on 表名(字段名);`

再次查看SQL耗时发现大大降低

### 索引失效

#### 最左前缀法则

如果索引了多列（联合索引），要遵循最左前缀法则。最左前缀法则指的是查询从索引的最左列开始，并且不跳过索引中的列

如果跳跃某一列，索引将部分失效（后面的字段索引失效）

> 案例：有profile、age、status三个组成的联合索引，此时如果查询时条件没有最左侧profile存在，则索引失效，或者存在profile不存在age，则只有profile走了索引，status未走索引

#### 范围查询

在联合索引中，出现范围查询（>,<），范围查询右侧的列索引失效

#### 索引列运算

在索引列上进行运算操作，索引将失效

> 如对字段进行了字符串拼接

#### 字符串不加引号

字符串类型字段使用时，不加引号，索引将失效

> 如 status = 0，status是字符串类型

#### 模糊查询

如果仅仅是尾部模糊匹配，索引不会失效，如果头部模糊匹配，索引会失效。

> ‘软件%’不会失效，‘%软件’则会失效

#### or连接的条件

用or分隔开的条件，如果or前的条件中的列有索引，而后面的列中没有索引，那么设计的索引不会被用到

> id = 10 or age = 23；如果id有索引但是age没有索引则索引不会被用到

#### 数据分布影响

如果mysql评估使用索引比全表更慢，则不使用索引

### SQL提示

- use index（指定使用索引，只是推荐）：`explain select * from 表名 use index(索引名) where 条件;`
- ignore index（忽略这个索引）：`explain select * from 表名 ignore index(索引名) where 条件;`
- force index（强制使用索引）：`explain select * from 表名 force index(索引名) where 条件;`

### 覆盖索引

尽量使用覆盖索引（查询使用了索引，并且需要返回的列在该索引中已经全部能够找到），减少select * 的使用

> 在explain执行计划返回的值查看SQL性能发现**extra列：**
> using index condition：查找使用了索引，但是需要回表查询数据
>
> using where；using index：查询使用了索引，需要的数据都在索引列中都能找到，所以不需要回表查询

### 前缀索引

当字段类型为字符串（varchar，text等）时，有时候需要索引很长的字符串，这会让索引变得很大，查询时会浪费大量的磁盘IO，影响查询效率

> 前缀索引：只将字符串的一部分前缀，建立索引，这样可以大大节约检索空间，从而提高索引效率

- 创建：`create index 索引名 on 表名（字段名(n)）;`加n就是前几个作为前缀

- 前缀长度：可以根据索引的选择性来决定，而选择性是指不重复的索引值（基数），和数据表记录总数的比值，索引选择性越高则查询效率越高，唯一索引的选择性是1（最好的索引选择性）

  `select count(distinct substring(email,1,5))/count(*)from 表名;(查看选择性`

  

### 单列索引与联合索引

- 单列索引：一个索引只包含单个列
- 联合索引：一个索引包含多个列
- 在业务场景中如果存在多个查询条件，考虑针对查询字段建立索引时，建立建立联合索引，而非单列索引

## 8.索引使用原则

1. 针对数据量大，且查询比较频繁的表建立索引（数据超过几十万条）
2. 针对于常作为查询条件（where）、排序（order by）、分组（group by）操作的字段建立索引
3. 尽量选择区分度高的列作为索引，尽量建立唯一索引，区分度越高使用索引的效率越高
4. 如果是字符串类型的字段，字段的长度较长，可以针对字段的特点，建立前缀索引
5. 尽量使用联合索引，减少单列索引，查询时，联合索引很多时候可以覆盖索引，节省存储空间，避免回表提高查询效率
6. 要控制索引的数量，索引并不是多多益善，索引越多维护索引结构的代价也就越大，会影响增删改的效率
7. 如果索引列不能存储null值，请在建表时使用not null 约束它，当优化器指定每列是否包含null值时，它可以更好的确定哪个索引最有效的用于查询

# 三、SQL语句优化

## 1.insert优化

### 批量插入

```
insert into 表名 values(字段列表),(字段列表)...;
```

### 手动提交事务

```sql
start transaction;
insert into ........;
insert into ........;
insert into ........;
insert into ........;
insert into ........;
commit;
```

### 主键顺序插入

按照从小到大顺序插入，如 1 2 3 4 5 6；

### 大批量数据插入

如果一次性要插入大批量数据，使用insert语句插入性能较低，此时可以使用MySQL数据库提供的load指令进行插入

```sql
//客户端连接服务端时，加上参数 --local-infile
mysql --local-infile -u root -p
//设置全局参数local_infile为1，开启从本地加载文件导入数据的开关
set global local_infile =1;
//执行load指令准备好的数据，加载到数据表中
load data local infile '/root/***.sql' into table 表名 fields terminated by ',' lines terminated by '\n';
```

## 2.主键优化

数据组织方式：在InnoDB存储引擎中，表数据都是根据主键顺序组织存放的，这种存储方式的表称为表索引组织表（IOT）。

- 页分裂：页可以为空，也可以填充一半，也可以填充100%。每个页包含了2-N行数据（如果一行数据多大，会行溢出），根据主键排列（当主键乱序插入时，保证主键从小到大排列，会在数据按顺序所在页中间进行分裂）
- 页合并：当删除一行记录时，实际上记录并没有被物理删除，只是记录被标记为删除并且它的空间变得允许被其他声明使用，当页中删除的记录达到MERGE_THRESHOLD（可以设置，默认为页的50%），InnoDB就会寻找最近的页看看能否进行合并以优化使用空间
- 主键设计原则：
  - 满足业务需求的情况下，尽量降低主键的长度
  - 插入数据时，尽量选择顺序插入，选择使用AUTO_INCREMENT自增主键
  - 尽量不要使用UUID做主键或者是其他自然主键，如身份证号
  - 业务操作尽量避免对主键的修改

## 3.order by优化

- Using filesort：通过表的索引或全表扫描，读取满足条件的数据行，然后在排序缓冲区sort buffer中完成排序操作，所有不是通过索引直接返回排序结果都叫FileSort排序
- Using index：通过有序索引顺序扫描直接返回有序数据，这种情况即为Using index，不需要额外排序，操作效率高

创建索引时可在字段后面加 asc 或者desc 表示索引存储的是升序还是降序排列

## 4.group by优化

- 分组操作时可以通过索引来提高效率
- 分组操作时索引的使用也是满足最左前缀法则

## 5.limit优化

在大数据量情况下，开始位置越后，查询时间越长

优化思路：一般分页查询时，通过创建覆盖索引能够较好的提高性能（查id），可以通过覆盖索引加子查询形式进行优化

```sql
//将此查询返回的数据当成一个表
select id from 表名 order by id limit 20000000,10
//进行覆盖索引
select * from 表名 t,(select id from 表名 order by id limit 20000000,10) a where t.id = a.id；
```

## 6.count优化

- MyISAM引擎吧一个表的总行数存放在了磁盘上，因此执行count(*)的时候会直接返回这个个数，效率很高

- InnoDB引擎就很麻烦，他执行count(*)的时候，需要把数据一行一行从引擎里面读出来然后累积计算

  优化思路：自己计数

- count的几种用法

  - count()是一个聚合函数，对于返回的结果集，一行行地判断，如果count函数的参数不是null，累计值就+1，否则不加，最后返回累计值
  - 用法：count（*）、count（主键）、count（字段）、count（1）
  - count（主键）：InnoDB遍历整张表吧每一行的id取出返回给服务层，服务层拿到后直接累加（主键不可能为null）
  - count（字段）：
    - 没有not null约束：InnoDB引擎会遍历整张表吧每一行的字段值都取出来，返回给服务层，服务层判断是否为null，不为null则计数累加
    - 有not null约束：InnoDB引擎会遍历整张表吧每一行字段都取出，返回给服务层，服务层直接累加
  - coun（1）：InnoDB引擎遍历整张表但不取值，服务层对于返回的每一行都放1个数字1，直接进行累加
  - count（*）：不会取全部字段，做了优化不取值，服务层直接按行进行累加
  - 效率：count（字段）< count（id）< count（1）约为 count（*）

## 7.update优化

- InnoDB的行锁是针对索引加的锁，不是针对记录加的锁，并且该索引不能失效，否则会从行锁升级为表锁，升级为表锁后并发性会降低
- 尽量根据主键/索引字段进行数据更新

# 四、视图/存储过程/触发器

## 1.视图

视图是一种虚拟存在的表。视图中的数据并不在数据库中实际存在，行和列数据来自定义视图查询中使用的表，并且是在使用视图动态生成的（视图只保存了查询的SQL逻辑，不保存查询结果）

```sql
//创建
create [or replace] view 视图名称[(列名列表)] as select 语句 [with [cascaded | local] check option]
加上 [with [cascaded | local] check option] 避免插入的数据违反条件
如 create or replace view stu as select id,name from student where id <=10;
//查询
查看创建视图的语句：show create view 视图名称;
查询视图数据：select * from 视图名称;
//修改
create or replace view 视图名称[(列名列表)] as select 语句 [with [cascaded | local] check option]
alter view 视图名称[(列名列表)] as select语句 [with [cascaded | local] check option]
//删除
drop view [if exits] 视图名称 ,......;
```

### （1）视图检查选项

```sql
 [with [cascaded | local] check option] 
MySQL会通过视图检查判断正在更改的行，如插入、更新、删除以及其符合视图的定义，MySQL运行基于一个视图1创建视图2，创建的视图的视图检查选项会向下检查，也就是视图1如果有检查选项，则更改的行会在视图1进行检查，如果没有则不会对视图1条件进行检查
如  create or replace view stu as select id,name from student where id <=10 with cascaded check option;   
此时 对stu的字段 id的操作就会判断id是否超过了10，超过10则操作无效
```

### （2）视图的更新和作用

要使视图可更新，视图中的行与基础表中的行之间必须存在一对一关系。如果视图包含以下任何一项，则视图不可更新：

1. 聚合函数或窗口函数(SUM(),MIN(),MAX(),COUNT()等)
2. distinct
3. group by
4. having
5. union 或者union all

作用：

- 操作简单：视图可以简化用户对数据的理解，也可以简化他们的操作。那些被经常使用的查询可以被定义为视图，从而使得用户不必为以后的操作每次指定全部条件
- 安全：数据库可以授权，但不能授权到数据库特定行和特定的列上。通过视图用户只能查询和修改他们所能见到的数据
- 数据独立：视图可以帮助用户屏蔽真实表结构变化带来的影响

如：为了保证数据库安全性，开发人员在操作某表时只能看到用户的基本字段，屏蔽手机号和地址

查询每个学生所选修的课程（三张表），将这其中的关键信息提取出来形成一个视图

## 2.存储过程

> 事先经过编译并存储在数据库中一段SQL语句的集合，调用存储过程可以简化应用开发人员的很多工作，减少数据在数据库和应用服务器之间的传输，对于提高数据处理的效率很好。（就是数据库SQL语言层面的代码封装和重用）

特点：

- 封装，复用
- 可以接收参数，也可以返回数据
- 减少网络交互，效率提升

```sql
//创建
create procedure 存储过程名称([参数列表]) 
begin 
        --SQL语句  
end;
如：
create procedure p1() 
begin 
       select count(*) from student;  
end;
//调用
call 存储过程名称([参数列表]) ;
//查看
select * from information_schema.routines where routine_schema = 'xxx';  指定数据库的
show create procedure 存储过程名称;
//删除
drop procedure if exists 存储过程名称;
```

在创建的时候因为命令行是以；作为结束符的，所以需要使用`delimiter `指定结束符，如`delimiter &&`

### 变量

- 系统变量是由MySQL服务器提供的，分为全局变量（golbal）和会话变量（session）
- 用户定义变量根据用户自己需要定义的变量，作用域为当前连接
- 局部变量根据需要定义在局部生效的变量,访问之前需要declare声明，可用作存储过程中局部变量和输入参数，局部变量的范围是在其内声明的begin .. end模块

```sql
//查看系统变量
show global/session variables;        --查看所有系统变量
show global/session variables like '%....';   --可以通过like模糊匹配方式直接查找变量
show @@global/session 系统变量名;   --可以查看指定变量的值
//设置系统变量
set global/session 系统变量名=值;
set @@global/session 系统变量名=值;

//设置用户自定义变量(无需初始化，默认为null)
set @变量名 = 值;
set @变量名 := 值;
set @变量名 := 值,@变量名2 :=值;
select count(*) into @变量名 from 表名;
//使用
select @变量名;

//局部变量
create procedure 存储过程名称([参数列表]) 
begin
         declare 变量名 int default 0;
         select count(*) into 变量名 from 表名;  （或者 set 变量名 = 值）
         select 变量名;
end;
```

如果没有指定global/session，默认是session。mysql重启服务后所设置的参数会失效，要想不失效，可以在/etc/my.cnf中配置

### if

```sql
例子:
create procedure 存储过程名称([参数列表]) 
begin 
        declare score int default 58;
        declare result varchar(10) ;
        if score >= 85 then
           set result := '优秀';
        elseif score >=60 then
           set result :='及格';
        else
           set result :='不及格';
        end if;
        select result;
end;
```

### 参数

| 类型  | 含义                                     | 备注 |
| ----- | ---------------------------------------- | ---- |
| in    | 该类参数作为输入，也就是需要调用时传入值 |      |
| out   | 该类参数作为输出，也就是该参数作为返回值 |      |
| inout | 既可以作为输入参数，也可以作为输出参数   |      |

```sql
create procedure 存储过程名称([in/out/inout 参数名 参数类型]) 
begin 
        --SQL语句;
end;

例如:
create procedure p1(in score int,out result varchar(10)) 
begin 
        if score >= 85 then
           set result := '优秀';
        elseif score >=60 then
           set result :='及格';
        else
           set result :='不及格';
        end if;
end;
//调用  call p1(55,@result); select @result;
```

### case

```sql
例如:
create procedure p1(in month int) 
begin 
        declare result varchar(10);
        case
            when month>=1 and month <=3 then
                   set result := '第一季度';
            when month>=1 and month <=3 then
                   set result := '第一季度';
            when month>=1 and month <=3 then
                   set result := '第一季度';
            when month>=1 and month <=3 then
                   set result := '第一季度';
            else
                   set result :='非法参数';
        end case;
   select concot('输入月份:',month,'所属季度:',result);
end;

//调用
call p6(11);
```

### 循环

```sql
//while :有条件的时候进行循环
create procedure 存储名称（参数 如 in n int）
begin
     declare total int default 0;
      while n>0 do
            set total :=total+n;
            set n := n-1;
      end while;
      select total;
end;
//repeat :有条件的时候退出循环
create procedure 存储名称（参数 如 in n int）
begin
     declare total int default 0;
     repeat 
            set total :=total+n;
            set n := n-1;
      util n<=0
      end repeat;
      select total;
end;
//loop :leave 退出循环      iterate 跳过当前循环进行下一次循环
create procedure 存储名称（参数 如 in n int）
begin
     declare total int default 0;
      sum:loop  (sum是设置的标记labe)
             if n<=0 then
                  leave sum;
             end if;
             if n%2=1 then
                  set n= n -1;
                  iterate sum;
             end if;

             set total :=total +n;
             set n= n -1;
      end loop sum;
      select total;
end;
```

### 游标

> 用来存储过程查询结果集的数据类型，在存储过程和函数中可以使用游标对结果集进行循环处理。

```sql
//声明游标(要先声明普通变量)
declare 游标名称 cursor for 查询语句;
//打开游标
open 游标名称;
//获取游标记录
fetch 游标名称 into 变量[,变量];
//关闭游标
close 游标名称;

例子
create procedure 存储名称（参数 如 in uage int）
begin
     declare uname varchar(100);
     declare upro varchar(100);
     declare u_cursor cursor for select name,profession from 表名 where age<=uage;
     declare exit handler for sqlstate = '02000' close u_cursor;
     create table if not exits 新表名{
         id int primary key auto_increment;
         name     varchar(100),
         profession varchar(100)   
    };
    open u_cursor;
    while true do
        fetch u_cursor into uname,upro;
        insert into 新表名 values(null,uname,upro);
    end while;
    close u_cursor;
end;
```

### 条件处理程序

> 条件处理程序可以用来定义在流程控制结构执行过程中遇到问题时相应的处理步骤

```sql
declare handler_action handler for condition_value [condition_value]...statement;
handler_action
           continue:继续执行
           exit:终止执行
condition_value
           sqlstate sqlstate_value:状态码，如02000
           sqlwarning :所有以01开头的
           not found :所有以02开头的
           sqlexception :所有没有被sqlwarning或not fount 捕获的sqlstate代码的简写
```

## 3.存储函数

> 存储函数是有返回值的存储过程，存储函数的参数只能是in类型的

characteristic说明：

- deterministic：相同的输入参数总是产生相同的结果
- no sql：不包含sql语句
- reads sql data：包含读取数据的语句，但不包含写入数据的语句

```sql
create function 存储函数名称([参数列表])
returns type [characteristic ....]
begin
      --sql语句
      return ...;
end;

例如:
create function 存储函数名称(n int)
returns int determinstic
begin
     declare total int default 0;
     while n>0 do
        set total := total + 1;
        set n := n - 1;

     end while;
     return total;
end;

select 存储函数名称(n);
```

## 4.触发器

与表有关的数据类型，在增删查改之前或者之后，触发并执行触发器中定义的SQL语句集合，触发器的这种特性可以协助应用在数据库端确保数据的完整性，日志记录，数据效验等操作；

使用别名old和new来引用触发器中发生变化的记录内容，这与其他数据库类似。现在只支持行级触发，不支持语句级触发

```sql
//创建
create trigger trigger_name
before/after insert/update/delete
on 表名 for each row --行级触发器
begin
     trigger_stmt
end;
//查看
show triggers
//删除
drop trigger [数据库名schema_name.]trigger_name; --如果没有指定schema_name 默认为当前数据库

例子:监控那个表将信息存入哪个表
create trigger 触发器名
    after insert on 表名 for each row
on 表名 for each row --行级触发器
begin
     insert into 语句
end;
```

# 五、锁

锁是计算机协调多个进程或线程并发访问某一资源的机制。在数据库中，除传统的计算资源(cpu、ram、i/o)的争用之外，数据也是一种供许多用户共享的资源。如何保证数据并发访问的一致性，有效性是所有数据库必须解决的一个问题，锁冲突也是影响数据库并发访问性能的一个重要因素。

- 全局锁：锁定数据库中的所有表
- 表级锁：每次操作锁定整张表
- 行级锁：每次操作锁定对应的行数据

## 全局锁

对整个数据库的实例加锁，加锁后整个实例就处于只读状态，后序的dml的写语句和ddl语句和更新的事务提交语句都将被阻塞

典型使用：全库的逻辑备份

- 如果在主库上备份，那么在备份期间都不能执行更新，业务基本上就得停摆
- 如果在从库上备份，那么在备份期间从库不能执行主库同步过来的二进制日志（binlog），会导致主从延迟

在InnoDB引擎中，我们可以在备份时加上参数 --single-transaction参数来完成不加锁的一致性数据备份

```sql
mysqldump  --single-transaction -uroot -p1234 itcast > itcast.sql;
//开启全局锁
flush tables with read lock;
//备份
mysqldump -uroot -p1234 itcast > itcast.sql;
//关闭锁
unlock tables;
```

## 表级锁

- 表锁
- 元数据锁
- 意向锁

### （1）表锁

- 表共享读锁（read）
- 表独占写锁（write）

读锁不会阻塞其他客户端的读，但是会阻塞写。写锁既会阻塞其他客户端的读，也会阻塞其他客户端的写

```sql
//加锁
lock tables 表名... read/write
//释放锁
unlock tables 
```

### （2）元数据锁（MDL）

MDL加锁过程是系统自动控制的，无需显示使用，在访问一张表的时候会自动加上，MDL锁主要作用是维护表元数据的数据一致性，在表上有活动事务的时候，不可以对元数据进行写入操作。为了避免DML与DDL冲突，保证读写的正确性

在mysql5.5中引入了MDL,当对一张表进行增删查改时，加MDL读锁（共享），当对表结构进行变更操作时，加MDL写锁（排他）

| 对应sql                                    | 锁类型                                | 说明                                             |
| ------------------------------------------ | ------------------------------------- | ------------------------------------------------ |
| lock tables xxx read/write                 | shared_read_only/shared_no_read_write |                                                  |
| select/select ...lock in share mode        | shared_read                           | 与shared_read/shared_write兼容，与exclusive互斥  |
| insert、update、delete、select..for update | shared_write                          | 与shared_read、shared_write兼容，与exclusive互斥 |
| alter table...                             | exclusive                             | 与其他的MDL都互斥                                |



查询元数据锁：

```
select object_type,object_schema,object_name,lock_type,lock_duration from performance_schema.metadata_locks;
```

### （3）意向锁

为了避免DML在执行时，加的行锁和表锁的冲突，在InnoDB中加入了意向锁，使得表锁不用检查每行数据是否加锁，使用意向锁来减少锁的检查（当加了行锁时，添加意向锁，在行锁未提交事务时，如果想添加表锁，只需要判断意向锁和要添加的表示是否兼容）

- 意向共享锁（IS）：由语句select...lock in share mode添加，与表锁共享锁（read）兼容，与表锁排它锁（write）互斥
- 意向排他锁（IX）：由insert，update，delete，select...for update 添加，与表锁的两种都互斥，意向锁之间不会互斥

查看意向锁：

```
select object_schema,object_name,index_name,lock_type,lock_mode,luck_data from performance_schema.data_locks;
```

## 行级锁

每次操作锁住对应的行数据，锁定粒度小，发生锁冲突的概率最低，并发度最高，应用在InnoDB存储引擎中

InnoDB的数据是基于索引组织的，行锁是通过对索引上的索引项加锁来实现的，而不是对记录加的锁

主要分为：

- 行锁（Record lock）：锁定单个记录的锁，防止其他事务对此进行update和delete，在RC、RR隔离级别下都支持
- 间隙锁（Gap lock）：锁定索引记录间隙（不包含该记录），确保索引记录间隙不变，防止其他事务在这个间隙进行insert，产生幻读。在RR隔离级别下都支持
- 临建锁（Next-Key lock）：行锁和间隙锁组合，同时锁住数据，并在锁住数据前面的间隙Gap，在RR隔离级别下支持

查看行级锁：

```
select object_schema,object_name,index_name,lock_type,lock_mode,luck_data from performance_schema.data_locks;
```

## 行锁

InnoDB实现了两种类型的行锁：

1. 共享锁（S）：允许一个事务去读一行，阻止其他事务获取相同数据集的排他锁（与共享锁兼容）
2. 排他锁（X）：允许获取排他锁的事务更新数据，阻止其他事务获得相同数据集的共享锁和排他锁（都互斥）

| sql                           | 行锁类型   | 说明                               |
| ----------------------------- | ---------- | ---------------------------------- |
| insert ....                   | 排他锁     | 自动加锁                           |
| update ....                   | 排他锁     | 自动加锁                           |
| delete .....                  | 排他锁     | 自动加锁                           |
| select..                      | 不加任何锁 |                                    |
| select ... lock in share mode | 共享锁     | 需要在select后加lock in share mode |
| select.. for update           | 排他锁     | 需要在select后加for updtae         |



默认情况下，InnoDB在pereatable read 事务隔离级别运行，InnoDB使用next-key锁进行搜索和索引扫描，以防止幻读

1. 针对唯一索引进行检索时，对已经存在的记录进行等值匹配时，将会自动优化为行锁
2. InnoDB的行锁是针对索引加的锁，不通过索引条件检索数据，那么InnoDB将对表中的所有记录加锁，此时就会升级为表锁



## 间隙锁/临建锁

默认情况下，InnoDB在pereatable read 事务隔离级别运行，InnoDB使用next-key锁进行搜索和索引扫描，以防止幻读

1. 索引上的等值查询（唯一索引），给不存在的记录加锁时，优化为间隙锁
2. 索引上的等值查询（普通索引），向右遍历时最后一个值不满足查询需求时，next-key lock退化为间隙锁
3. 索引上的范围查询（唯一索引）--会访问到不满足条件的第一个值为止

> 间隙锁唯一的目的是防止其他事务插入间隙，间隙锁可以共存，一个事务采用的间隙锁不会组织另一个只是在同一个间隙上采用间隙锁。



# 六、InnoDB引擎

## 1.逻辑存储结构

表空间，段，区，页，行，存储字段

![逻辑存储结构.png](/upload/逻辑存储结构.png)

- 表空间（ibd文件），一个mysql实例可以对应多个表空间，用于存储记录、索引等数据
- 段：分为数据段、索引段、回滚段，InnoDB是索引组织表，数据段就是B+树的叶子节点，索引段为B+树的非叶子节点，段用来管理多个区
- 区：表空间的单元结构，每个区的大小为1M，默认情况下InnoDB的存储引擎页大小为16k，即一个区中一共有64个连续的页
- 页：是InnoDB存储引擎磁盘管理的最小单元，每个页的大小默认为16kb，为了保证页的连续性，InnoDB存储引擎每次从磁盘申请4-5个区
- 行：InnoDB存储引擎数据是按行进行存放的
  - Trx_id:最后一次操作事务的id，每次对某条记录进行改动时，都会吧对应的事务id赋值给trx_id隐藏列，
  - Roll point：每次对某条记录进行改动时，都会吧旧的版本写入到undo日志中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息

## 2.架构

擅长事务处理，具有崩溃回复特性，在日常开发中使用广泛

### 内存结构（80%）

#### Buffer Pool

缓冲池是主内存中的一个区域，里面可以缓存磁盘上经常操作的真实数据，在执行增删查改操作时，先操作缓冲池中的数据（若缓冲池没有数据，则从磁盘加载并缓存），然后再以一定频率刷新到磁盘，从而减少磁盘I/O，加快处理速度

缓冲池以page页为单位，底层采用链表数据结构管理page，根据状态，将page分为三种类型：

- freepage：空闲page，未被使用
- clean page：被使用page，数据没有被修改过
- dirty page：脏页，被使用page，数据被修改过，也中数据与磁盘的数据产生了不一致

#### Change Buffer

更改缓冲区（针对非唯一二级索引页），在执行DML语句时，如果这些数据page没有在Buffer Pool中，不会直接操作磁盘，而会将数据变更存在更改缓冲区Change Buffer中，在未来数据被读取时，再将数据恢复到Buffer Pool中，在将合并后的数据刷新到磁盘中

存在的意义：

与聚集索引不同，二级索引通常是非唯一的，并且以相对随机的顺序插入二级索引，同样删除和更新可能会影响索引树中不相邻的二级索引页，每一次都操作磁盘，会造成大量的磁盘IO,有了Change Buffer之后，我们可以在缓冲池中进行合并，减少磁盘IO

#### Adaptive Hash Index

自适应hash索引，用于优化对Buffer Pool数据的查询，InnoDB引擎会监控表上各索引页的查询，如果观察到hash索引可以提升速度，则建立hash索引，称之为自适应hash索引（无需人工干预，系统根据情况自动生成）

参数：adaptive_hash_index

#### Log Buffer

日志缓冲区，用来报错要写入磁盘的log日志数据（redo log，undo log），默认大小为16mb，日志缓冲区的日志会定期刷新到磁盘中，如果需要更新、插入或删除多行的事务，增加日志缓冲区的大小可以节省磁盘I/O

参数：

- innodb_log_buffer_size ：缓冲区大小
- innodb_flush_log_at_trx_commit ：日志刷新到磁盘时机
  - 0：每秒将日志写入并刷新到磁盘一次
  - 1：日志在每次事务提交时刷新到磁盘
  - 2：日志在每次事务提交后写入，并每秒刷新到磁盘一次

### 磁盘结构

#### System Tables

系统表空间是更改缓冲区的存储区域，如果表是在系统表空间而不是每个表文件或通用表空间中创建的，他也可能包含表和索引数据（在MySQL5.x版本中还包含InnoDB数据字典，undolog等）

参数：innodb_dara_file_path

#### File-Per-Table Tablespaces

每个表的文件表空间包含单个InnoDB表的数据和索引，并存储在文件系统上的单个数据文件中

参数：innodb_file_per_table

文件名：xxx.ibdibd

#### General Tablespaces

通用表空间，需要通过`create tablespace `语法创建通用表空间，在创建表时，可以指定该表空间

文件名：xxx.ibd

```sql
//创建通用表空间
create tablespace xxx add
datafile 'file_name'(idb文件名)
engine = engine_name(存储引擎名)
//指定表空间
create table xxx{
  id int primary key auto_increment,
  name varchar(10)
} engine=innodb tablespace 通用表空间名;
```

#### Undo Tablespace

撤销表空间，mysql实例在初始化时会自动创建两个默认的undo表空间（初始大小为16MB）用于存储undo log日志

文件名：xxx.ibu

#### Temporary Tablespace

InnoDB使用会话临时表空间和全局临时表空间，存储用户创建的临时表等数据

文件名：xxx.ibt

#### Doublewrite Buffer Files

双写缓冲区，InnoDB引擎将数据页从Buffer Pool刷新到磁盘前，先将数据写入双写缓冲区文件中，便于系统异常时恢复数据

文件名： ib_16384_0.dblwr ib_16384_1.dblwr

#### Redo Log

重做日志，是用来实现事务的持久性，该日志文件由两部分组成

- 重做日志缓冲（redo log buffer）：在内存中
- 重做日志文件（redo log）：在磁盘中

当事务提交之后会把所有修改信息都会存到该日志中，用于刷新脏页到磁盘时，发生错误时进行数据恢复使用

文件名：ib_logfile0 ib_logfile1

### 后台线程

> 负责InnoDB内存数据刷新到磁盘

分为四类

- Master Thread：核心后台线程，负责调度其他线程，还负责将缓冲池中的数据异步刷新到磁盘中，保持数据的一致性，还包括脏页的刷新、合并插入缓存、undo页的回收

- IO Thread：在InnoDB存储引擎中大量使用了AIO来处理IO请求，这样可以极大的提高数据库的性能，而IO Thread主要负责这些IO请求的回调 `show engine innodb status`

  | 线程类型             | 默认个数 | 职责                           |
  | -------------------- | -------- | ------------------------------ |
  | Read thread          | 4        | 负责读操作                     |
  | Write thread         | 4        | 负责写操作                     |
  | Log thread           | 1        | 负责将日志缓冲区刷新到磁盘     |
  | insert buffer thread | 1        | 负责将写缓冲区内容刷新到磁盘区 |

- Purge Thread：主要用于回收事务已经提交了undo log，在事务提交后，undo log可能不用了，就用它来回收

- Page Cleaner Thread：协助Master Thread刷新脏页到磁盘的线程，它可以减轻Master Thread的工作压力，减少阻塞

## 3.事务原理

> 事务是一组操作的集合，它是一个不可分割的工作单位，事务会把所有操作作为一个整体一起向系统提交或撤销操作请求，即这些操作要么同时成功，要么同时失败

### redo log（负责持久性）

重做日志，记录的是事务提交时数据页的物理修改，是用来实现事务的持久性，该日志由两部分组成

- 重做日志缓冲（redo log buffer）：内存中
- 重做日志文件（redo log file）：磁盘中
- 当事务提交之后会把所有修改的信息都存到该日志文件中，用于刷新脏页到磁盘，发生错误时进行数据恢复使用

> 当事务提交后，假如是update语句，首先看内存结构里面是否有缓存，如果没有就从磁盘结构中读取到Buffer Pool中，此时在进行修改，修改后的数据就称为脏页，如果此时脏页直接刷新到磁盘（会造成大量IO浪费性能），就会用到redo log，就是内存结构中Redolog buffer 会从Buffer Pool中获取数据页的编号，在吧这个刷新到磁盘结构中的ib_logfile0/1文件（WAL Write-Aphead Logging）

### undo log（负责原子性）

回滚日志，用于记录数据被修改前的信息，作用包括两个提供回滚和MVCC（多版本并发控制）

undo log是逻辑日志，可以认为当delete一条记录时，undo log会记录一条对应的insert记录，反之亦然。当执行rollback时，就可以从undo log中的逻辑记录读取到相应的内容并进行回滚

undo log销毁：在事务执行时产生，事务提交时，并不会立即删除undo log，因为这些日志可能还用于mvcc

undo log存储：undo log采用段的方式进行管理和记录，存放在前面介绍的rollback segment回滚段中，内部包含1024个undo log segment

## 4.MVCC

### 当前读

读取的是记录的最新版本，读取时保证其他并发事务不能修改当前记录，会对读取的记录进行加锁，对于我们日常的操作：select...lock in share mode（共享锁），select..for update、update、insert、delete（排他锁）都是一种当前读

### 快照读

简单的select（不加锁）就是快照读，快照读，读取的是记录数据的可见版本，有可能是历史数据，不加锁是非阻塞读

- Read Commit：每次select都会生成一个快照读
- Repeatable Read：开启事务后第一个select语句才是快照读的地方
- Serializable：快照度会退化为当前读

### MVCC实现原理

Multi-Version Concurrency Control，多版本并发控制。指维护一个数据的多个版本，使得读写操作没有冲突，快照读为MySQL实现MVCC提供了一个非阻塞读的功能，MVCC的具体实现还需要依赖于数据库记录中的三个隐式字段、undo log日志、readView。

#### （1）记录中的隐藏字段

| 隐藏字段    | 含义                                                         |
| ----------- | ------------------------------------------------------------ |
| DB_TRX_ID   | 最近修改事务id，记录插入这条记录或最后一次修改该记录的事务Id |
| DB_ROLL_PTR | 回滚指针，指向这条记录的上一个版本，用于配合undo log，指向上一个版本 |
| DB_ROW_ID   | 隐藏主键，如果表结构没有指定主键，将会生成该隐藏字段         |

#### （2）undo log日志

回滚日志，在insert，update，delete的时候产生的便于数据回滚的日志

insert的时候，产生的undo log日志只在回滚时需要，在事务提交后，可被立即删除，而update，delete的时候产生的undo log日志不仅在回滚时需要，在快照读时也需要，不会被立即删除

**undo log版本链**

不同事务或相同事务对同一条记录进行修改，会导致该记录的undolog生成一条记录版本链表，链表的头部是最新的旧记录，链表尾部是最早的旧记录

例子：

1. 开启事务2，修改id为30的记录，age改为3，提交事务
2. 开启事务2，修改id为30的记录，name改为A3，提交事务
3. 开启事务4，查询id为30的记录，修改id为30的记录，age为10，查询id为30的记录
4. 开启事务5，查询id为30的记录 ，查询id为30的记录

记录最初数据是日志中的最尾端数据



![undo log版本链.png](/upload/undo%20log版本链.png)

#### （3）readView

ReadView（读视图）是快照读SQL执行时MVCC提取数据的依据，记录并维护系统当前活跃的事务（未提交的）id

- m_ids：当前活跃的事务id的集合
- min_trx_id：最小活跃事务id
- max_trx_id：预分配事务id，当前最大事务id+1（因为事务id是自增的）
- creator_trx_id：readview创建者的事务id

版本链数据访问规则：

trx_id代表当前事务id

1. trx_id == creator_trx_id? 可以访问该版本 **成立**说明数据是当前这个事务更改的
2. trx_id<min_trx_id? 可以访问该版本 **成立**说明数据已经提交了
3. trx_id>max_trx_id?不可以访问该版本 **成立**说明该事务是在readView生成后才开启
4. min_trx_id<=trx_id<=max_trx_id?如果trx_id不在m_ids中是可以访问该版本的 **成立**说明数据已经提交

不同隔离级别生产readView的时机不同

- read committed（RC）：在事务中每一次执行快照读时生成readView
- repeatable read（RR）：仅在事务中第一次执行快照读时生成readView，后续复用该readView

# 七、MySQL管理

## 1.系统数据库

mysql8.0之后

### （1）information_schema

提供了访问数据库元数据的各种表和视图，包含数据库、表、字段类型及访问权限等

### （2）sys

包含了一系列方便DBA和开发人员利用performance_schema性能数据库进行性能优化调优和诊断的视图

### （3）mysql

存储MySQL服务器正常运行所需要的各种信息（时区、注册、用户、权限等）

### （4）performance_schema

为MySQL服务器运行时状态提供了一个底层监控功能，主要用于收集数据库服务器性能参数

## 2.常用工具

### （1）mysql

```sql
语法:mysql [options] [database]
选项:
-u,--user=name  #指定用户名
-p,--password=[name] #指定密码
-h,--host=name  #指定服务器ip或域名
-P,--port=port  #指定连接端口
-e,--execute=name #执行sql语句并退出
如：mysql -uroot -p123456 db01 -e "select * from stu";
```

### （2）mysqladmin

mysqladmin是一个执行管理操作的客户端程序，可以用它来检查服务器的配置和当前状态、创建并删除数据库等

```
mysqladmin --help
```

### （3）mysqlbinlog

由于服务器生成的二进制文件以二进制格式保存，所以如果想要检查这些文本的文本格式，就会使用mysqlbinlog日志管理工具

```sql
语法:mysqlbinlog [options] logs-files1 logs-files2....
选项:
-d,--database=name  #指定数据库名，只列出指定的数据库相关操作
-o,--offset=# #忽略掉日志中的前n行命令
-r,--result-file=name  #将输出的文本格式日志输出到指定文件
-s,--short-form  #显示简单格式，忽略掉一些信息
--start-datatime=data1 --stop-datatime=date2  指定日期间隔内的所有日志
--start-position=pos1 --stop-position=pos2  指定位置间隔内的所有日志
```

### （4）mysqlshow

mysql客户端对象查找工具，用来很快的查找存在那些数据库、数据库中的表、表中的列或者索引

```sql
mysqlshow [options] [db_name [table_name[col_name]]]

--count 显示数据库及表的统计信息(数据库，表均可以不指定)
-i      显示指定数据库及指定表的状态信息
```

### （5）mysqldump

用于备份数据库或在不同数据库之间进行数据迁移，备份内容包含创建表及插入表的sql