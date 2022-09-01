# 1.redis命令汇总

1. redis-server /.../redis.conf   后台启动redis
2. ps -ef | grep redis 查看后台启动的redis进程
3. redis-cli 进入redis进程
4. select num  进入数据库，下标从0~15
5. 杀死进程  kill -9 pid : pid就是redis进程列表中的进程号
6. 启动redis  **redis-server /etc/redis.conf** -> 通过指定配置文件来启动redis
7. flushdb 清空当前数据库所有key；flushall清空所有数据库所有key
8. dbsize 查看当前库中key的数量



# 2.redis - 底层原理

- 单线程+多路I/O复用
  - 单线程: 黄牛买票，并通知相关顾客买到了票
  - 多路IO复用：顾客买票或者干自己的事/
- 原子操作：不会被线程调度机制打断的操作



## 2.1 字符串SDS - 简单动态字符串

| sdshdr | free     | len      | buf                      |
| ------ | -------- | -------- | ------------------------ |
|        | 空闲长度 | 字节长度 | 字节数组（元素存储位置） |

相较C原生字符串优点：

1. len获取复杂度o(1)
2. 缓冲区杜绝溢出 （提前分配free）
3. 减少修改长度所需内存重分配次数
4. 二进制安全 (buf存储字节数组而非文本，避免编码校验)

## 2.2 字典dict

| dict | type | privdata | ht     | rehashidx          |
| ---- | ---- | -------- | ------ | ------------------ |
|      | 类型 | 私有数据 | 哈希表 | 索引计数器变量(-1) |

| dictht       | table           | size   | sizemask     | used         |
| ------------ | --------------- | ------ | ------------ | ------------ |
| 对应hash的dt | dictEntry[size] | 桶数目 | size-1, hash | 表内元素数量 |

load_factor = ht[].used / ht[].size

扩容操作：

- 扩容发生：
  - 服务器目前未执行BGSAVE/WRITEAOF且load_factor >= 1
  - 服务器目前正执行BGSAVE/WRITEAOF且load_factor >= 5

- 渐进式rehash  （避免集中式rehash影响性能）

  1. 为ht[1]分配空间 size = ht[0].size * 2
  2. rehashidx = 0 - rehash开始
  3. 每次对字典进行crud时除进行指定操作外还会顺带将rehashidx上的键值对rehash到ht[1]，rehashidx++
  4. rehash完成时rehashidx = -1

  - rehash期间新增只在ht[1]，增、删、改在ht[0]与ht[1]同时进行

## 2.3 跳跃表skiplist

有序数据结构，通过在每个节点维持多个指向其他节点的指针达到快速访问节点的目的

| header | tail   | level            | length   |
| ------ | ------ | ---------------- | -------- |
| 头节点 | 尾节点 | 层数最大节点层数 | 节点数目 |



| level （span）                                               | BW后退指针 | 分值   | 成员对象o                |
| ------------------------------------------------------------ | ---------- | ------ | ------------------------ |
| l(1~32), 只连接下一层对应level  （span对应跨度 - 两个level之间节点数目rank） | 前一个节点 | double | 节点保存的成员对象 - sds |

## 2.4 压缩列表ziplist

节约内存，一系列特殊编码的连续内存块组成的顺序性数据结构。适用于key元素少且元素短(int / 小String)



| zlbytes    | zltail           | zllen    | entryX                 | zlend    |
| ---------- | ---------------- | -------- | ---------------------- | -------- |
| 内存字节数 | 尾节点字节偏移量 | 节点数目 | 节点(字节数组or整数值) | 标记尾端 |

节点格式：

| privious_entry_length   | encoding      | context               |
| ----------------------- | ------------- | --------------------- |
| 前一个节点长度(1/5字节) | 数据类型&长度 | 节点值(整数/字节数组) |

连锁更新：

- 每个节点长度介于250~254时privious_entry_length都为1，当表头插入一个>254时privious_entry_length长度都变为5.此时会引发后续连锁更新，空间重分配操作。

# 3.redis - 数据类型

| type   | encoding           | *ptr             |
| ------ | ------------------ | ---------------- |
| 类型   | 编码               | 底层数据结构指针 |
| string | int embstr raw     |                  |
| list   | ziplist linkedlist |                  |
| hash   | ziplist ht         |                  |
| set    | intset ht          |                  |
| zset   | ziplist skiplist   |                  |

| 编码常量   | 底层结构      | 描述                   |
| ---------- | ------------- | ---------------------- |
| INT        | long          |                        |
| EMBSTR     | embset编码SDS | 保存短字符串 <= 32字节 |
| RAW        | SDS           | 长字符串 > 32字节      |
| HT         | 字典          |                        |
| LINKEDLIST | 双端列表      |                        |
| ZIPLIST    | 压缩列表      |                        |
| INSERT     | 整数集合      |                        |
| SKIPLIST   | 跳跃表和字典  |                        |



## String - 字符串

#### 1. 概念

- Redis最基本的类型
- **二进制安全**：Redis的String可以包含任何数据（jpg，序列化对象）
- Redis字符串value最多可以是512M

#### 2. 指令

- set \<key>\<value> 
- get\<key>
- mset \<key1>\<value1>\<key2>\<value2> 批量set
- mget \<key1>\<key2> 批量get
- msetnx \<key1>\<value1>\<key2>\<value2> 批量非空set
  - **原子性：批量删除时有一个添加失败全部失败**
- getrange \<key>\<起始位置>\<结束位置> 获取[]范围的key
- setrange \<key>\<起始位置>\<value> 从起始位置开始覆盖后面value长度个数的值
- **setex \<key><过期时间>\<value>  设置k v的同时指定过期时间**
- getset \<key>\<value> 用value替换key中的旧值

- append \<key>\<value> : 将k1键的值末尾添加值
  - 若key无值则新建并添加
- strlen \<key> 获取键对应值的长度
- setnx \<key>\<value> 键不存在时设置值
-  incr/decr \<key>  key中存储数字值时，对应数字值+/-1
-  incrby/decrby \<key>  num  key中存储数字值时，对应数字值+/-num



#### 3. 数据结构

- int，embstr，raw
- embstr 与 raw都为SDS，区别在于embstr将对象与SDS连续分配内存，只调用一次内存分配函数。 raw则为两次，回收同理。
- int -> raw : append命令转换时
- embstr只读，输入修改命令时转换为raw

## list - 列表

#### 1. 简介

- 单键多值：一个key对应多个值组成一个列表

- Redis列表为简单字符串列表，按照插入顺序排序的双向链表

#### 2. 指令

1. lpush/rpush \<key>\<value1>\<value2>... 左/右插入一或多值
2. lpop/rpop \<key> 从左边或右边取出一个值
   - 值空时键空
3. rpoplpush \<key1>\<key2> 从key1列表右边取值插入key2左边
4. l/rrange \<key>\<x>\<y> : 获取key中从[x,y)的value， 可取负值表示反向获取
5. lindex\<key>\<index> : 获取key下标处的值
6. llen\<key> : 获取key列表长度
7. linsert \<key> before \<value>\<newvalue> : 在value前面添加newvalue
8. lrem \<key>\<n>\<value> : 将前n个value值删除
9. lset\<key>\<index>\<value> : 将列表key下标为index的值替换为value

#### 3. 数据结构

- linkedlist & ziplist
- 列表元素数量<512且保存所有字符串长度<64字节使用ziplist，否则linkedlist

- **快速链表 quickList？** - 结合链表和ziplist，满足快速插入删除又不出现太大空间冗余
  - ziplist：压缩列表，一块连续的内存存储紧挨着存储元素 
  - 数据两较多时将ziplist用链表形式连接形成quickList



## Set - 集合

#### 1. 简介

- string类型的无序集合，**底层是一个value为null的hash表**

#### 2. 命令

1. sadd\<key1>\<value1>\<value2> 将一个或多个member元素加入到集合key中，去重
2. smembers \<key> 将key中的set集合打印
3. sismember \<key>\<value> 判断集合key中是否含有value，1 ? 0
4. scard \<key> 返回集合中元素个数
5. srem \<key>\<value1>\<value2> : 从key中删除元素
6. spop \<key> : 从key中随机弹出元素
7. srandmember \<key>\<n> : key取n个值打印，不删除
8. smove \<k1>\<k2>\<value> ：从k1中取value置入k2
9.  sinter/sunion/sdiff \<k1>\<k2> 返回两集合的交/并/差集 - 差集：k1有k2无的

#### 3. 数据结构

- intset & hash
- intset: 集合所有元素都是int且不超过512个

## hash

#### 1. 简介

- 键值对集合，string类型的field和vaue的映射表，**适合用于存储对象** **类似于 - \<String, Object>**
- key相当于表，filed相当于字段，value相当于值
  1. json存储：user : {id=1, name=jing, age=20}
     - 每次修改用户属性需反序列化获取属性修改后再序列化回去，开销较大
  2. user : id  1   user : name jing    user : age 20
     - 数据过于分散混乱
  3. hash \<String,  Map<key, value>>



#### 2. 命令

- hset \<key>\<field>\<value> 给key集合中的field属性赋值value
- hget \<key>\<field>\<value> 获取value
- hmset \<key>\<field1>\<value1>\<field2>\<value2> 批量删除key中field值  **hset亦可实现此操作**
- hexists \<key>\<field> 查看key中field是否存在
- hkeys \<key> 查询集合中所有fields 
- hvals \<key> 查询集合中所有values
- hincrby \<key>\<field>\<increment> 给key域field值加上增量increment
- hsetnx \<key>\<field>\<value> 当且仅当key或field不存在时设置field属性为value



#### 3. 数据结构

- hash类型对应数据结构类似于set ： ziplist | hash
  - ziplist: 按顺序存储于连续空间，先key后value
  - hashtable: 按键值对存储
  - **ziplist: 键值对的键、值长度小于64字节且保存元素数量数少于512个**



## Zset (sorted set) 有序集合

#### 1. 简介

- 没有重复元素的字符串集合，有序集合的每个成员都关联了一个评分，被用来从低到高的方式排序集合中的成员。
- 成员唯一，评分可重复



#### 2. 常用命令

1. zadd \<key>\<score1>\<value1>\<score2>\<value2> 将一个或多个member元素及其评分置入有序集key中
2. zrange \<key>\<start>\<stop> [withscores] 返回有序集key，下标再start和stop之间， withscores可将评分一起显式
3. z(rev)rangebyscore \<key>\<min>\<max> [withscores] [limit offset count] 返回key中[min, max]之间的集合
4. zincrby \<key>\<increment>\<value> 增加score数
5. zrem \<key>\<value> 删除元素
6. zcount \<key>\<min>\<max> 统计分数区间集合
7. zrank \<key>\<value>返回排名(按分数)

#### 3. 数据结构

- ziplist | dict + skiplist  *128个元素、64字节*

  - ziplist: 每个集合元素使用两个紧挨的压缩列表节点存储，1-成员，2-分数

  - dict + skiplist

    - **字典和跳跃表共享元素的成员和分值。**

    - zsl：按分支从小到大(分值相同则按对象)保存集合元素（Object-元素成员，score-元素分值）
    - dict：键-元素成员，值-分值




- **跳跃表：层、跨度、前进指针L、后退指针BW、分值、成员对象**
  - **层数越高对应跨度越大，根据层数不同Ln对应下一个节点跨度不同**
  - **后退指针BW指向相邻的前一个节点**
  - **查找过程，按层数查找<=给定值的最大元素，然后层数减少继续查找。**





## Redis6新数据类型

### 1. Bitmaps - 位操作的字符串

### 2. HyperLogLog - 基数运算

- 不能取值只能存放的set

### 3. Geospatial - 与经纬度有关的地理位置信息



## key

1. exists keyName 若有对应的key返回1 否则返回0
2. type key 查询key的数据类型
3. object encoding zset1
4. del key 删除对应的key 删除成功返回1
5. unlink key 选择非阻塞删除
   - 仅将keys从keyspace元数据中你删除，真正删除会在后续异步操作； del直接删
6. expire key time 设置key的过期时间 s为单位
7. ttl key 查看key还有多久过期， -1永不过期 -2已经过期



## 过期键删除策略

1. 定时删除：内存友好
2. 惰性删除：CPU友好
3. 定期删除：惰性删除 + 定时删除
   - RDB策略: 写入时不写入过期键；恢复时主服务器不恢复过期键，从服务器恢复（会跟随主服务器同步而删除）
   - AOF策略: 写入-删除时AOF文件追加一条DEL命令来记录删除；恢复时不会恢复过期键命令。
   - 主从复制：主服务器删除会通知从服务器del删除；从服务器不会主动清理过期键。 - 保证主从同步

# 4. 配置文件

- 目录：配置文件/etc/redis.conf; 日志/home/redis_log/redis_log.log

1. bind 127.0.0.1 : 表示只支持本地连接，注掉以使其支持其他连接
2. protected mode yes 开启保护模式，只支持本地访问；改为no使其支持远程访问
3. port 6379
4. tcp-backlog
   - backlog连接队列，未完成的握手队列+已完成的握手队列
5. timeout 0 永不超时
6. tcp-keepalive 检测连接存活机制







# 5. Redis事务

- 作用：串联多个命令防止命令插队
- Redis事务：单独的隔离操作，所有命令被序列化、按顺序执行。执行过程不会被其他客户端命令请求打断。

## 5.1 Redis事务命令

1. Multi：开启事务，**进入组队阶段**
   - 将输入命令按顺序进入命令队列，不会被执行直至输入Exec
2. Exec：将Multi输入的命令依次执行，**执行阶段**
3. discard：放弃组队，命令不会得到执行



- 错误处理（部分原子性）
  - **组队时任何命令失败（命令写错了），则执行时所有命令失败**
  - **执行时任何命令失败（增了一个不是int的string），则该命令不会被执行**



## 5.2 事务冲突的解决

1. 悲观锁: - 做操作之前先上锁：拿数据时上锁，别人想要拿数据时就会被阻塞直至锁开放。
2. 乐观锁：拿数据时不上锁，但更新数据时会判断在此期间别人有没有更新该数据（可使用版本号/时间戳等机制判断）。check-and-set



- 使用方法：multi之前先执行**watch key命令**监视一个或多个key，若在事务执行之前这个key被其他命令改动则改事务不会被执行； **unwatch取消监视**



## 5.3 事务特性

1. 单独的隔离操作
   - 序列化，按顺序执行。
2. 没有隔离级别概念
   - 事务提交前不会被实际执行
3. 不保证原子性
   - 事务中一条命令执行失败其后命令仍会执行



# 6. Redis持久化

## 6.2 Redis内存淘汰机制

volatile-lru 从已设置过期时间的数据集中挑选最近最少使用的数据淘汰

volatile-lfu

volatile-ttl 从已设置过期时间的数据集中挑选将要过期的数据淘汰
volatile-random从已设置过期时间的数据集中任意选择数据淘汰
allkeys-lru从所有数据集中挑选最近最少使用的数据淘汰

allkeys-lfu

allkeys-random从所有数据集中任意选择数据进行淘汰
noeviction禁止驱逐数据

## 6.1 RDB

**指定时间间隔内将内存中的数据集快照写入磁盘**（snapshot）

SAVE() & BGSAVE() RDB相关持久化命令

1. SAVE(): 阻塞服务器进程，服务器不处理命令请求进行持久化
2. BGSAVE(): 派生一个子进程进行持久化，父进程继续处理请求   **fork() -> rdbSave() -> signal()  父进程轮询信号**
   - BGSAVE执行期间会拒绝SAVE与BGSAVE防止竞争条件
   - BGREWRITEAOF会延迟到BGSAVE结束后执行 (BGREWRITEAOF执行期间会拒绝BGSAVE) -> 性能考虑，防止两个进程同时执行大量磁盘写入操作。
   - **BGSAVE可通过配置save选项来自动执行 save a b即在a秒内进行了至少b次修改则会触发bgsave** 通过dirty计数器与lastsave时间戳

恢复：REDIS服务器启动时检查本地rdb文件，有则恢复（优先AOF）

REDIS文件；databases；key_value_pairs

| REDIS     | db_version | databases          | EOF         | check_sum                               |
| --------- | ---------- | ------------------ | ----------- | --------------------------------------- |
| RDB标志位 | 版本号     | 数据库及键值对数据 | RDB正文结束 | 前四个字段计算得出，检查RDB文件出错与否 |

| SELECTDB | db_number  | key_value_pairs |
| -------- | ---------- | --------------- |
| 标识位   | 数据库号码 | 键值对          |

**k_v_p:   type key value ;   expiretime_ms ms type key value.**



- 原理: 单独创建fork一个子进程来持久化，会将数据写入一个临时文件中，待持久化过程结束，再用该临时文件替换上次持久化好的文件(默认-dump.reb)。 
  - *优点：*
    1. 主进程不进行I/O操作，确保高性能
    2. 节省磁盘空间、恢复速度快
  - 缺点：
    1. 最后一次持久化可能数据丢失（服务器挂掉）。
    2. 写时拷贝技术，若数据庞大则较消耗内存。



- 备份：



## 6.2 AOF（默认不开启）

appendonly yes开启aof（aof、rdb同时开启时默认取aof数据）

以日志形式记录每个写操作，记录redis执行过的所有写指令（不记录读），**只许追加文件不可以改写文件**。保存为aof文件。（*Redis重启时根据日志文件内容将写指令从前到后执行一次已完成数据恢复工作*）



持久化

1. 命令追加：执行写命令后以协议格式将命令追加到服务器aof_buf缓冲区

2. 文件写入&同步：取决于appendfsync选项

   | always                           | everysec           | no                   |
   | -------------------------------- | ------------------ | -------------------- |
   | aof_buf缓冲区写入并同步到aof文件 | 每隔一s同步aof文件 | 操作系统决定何时同步 |



恢复：

1. 创建fake client以执行AOF文件 (REDIS命令只可在客户端上下文执行)
2. 从AOF文件分析并读取写命令
3. fake client执行被读出的写命令
4. 2,3循环直至读完



AOF重写：避免AOF文件体积膨胀 -> 根据服务器当前数据库状态实现，不依赖于旧AOF文件  **BGREWRITEAOF**

- 子进程重写，aof重写缓冲区避免主进程与子进程数据不同步
- 服务器执行写命令后会同时将写命令发送给AOF缓冲区与AOF重写缓冲区
- 子进程执行AOF重写后向父进程发送信号，父进程
  1. **将AOF重写缓冲区所有内容写入新AOF文件**
  2. 对新AOF文件改名，原子地覆盖原有AOF文件，完成替换



## 6.3 summary

RDB:

- 优：文件紧凑，体积小，网络传输快，恢复速度块
- 缺：实时持久化效率低，需满足特定版本格式。

AOF:

- 优点：秒级持久化，兼容性好
- 缺点：文件大，恢复速度慢

- 官方推荐二者都用
- 若对数据不敏感，可以单独选用rdb
- 单独使用aof可能出现bug - 不建议单用
- 若只做内存缓存，可以都不用



# U7 - 主从复制

- 主机数据更新后根据配置和策略，自动同步到备机的机制。 一般一主多从
  - 主机-master以写为主；从机-slave以读为主
- 优势:
  1. 读写分离 - 分担服务器压力
  2. 容灾快速恢复 - 从机挂掉时切换到其他从机



## 1. 设置步骤

1. 复制conf文件到新文件夹
2. 添加redis*.conf 设置内容如下：
   - include /myredis/redis.conf
   - pidfile /var/run/redis_6379.pid
   - port 6379 - 端口号
   - dbfilename dump6379.rdb - dump文件名
   - **salveof \<ip> \<port>  从服务器需配置指定主机**
   - **masterauth password 若主服务设置密码，从服务器还需配置**
3. 以redis6379为主，再配置两从6380、6381
4. 启动三个redis服务,配置主从关系（在1中配置）
   - 查看主机运行状况 - info replication



- **主从复制原理:**

  - 全量复制：slave服务接受数据库文件数据后将其存盘加载至内存
  - 增量复制：master将修改命令传递给slave同步

  1. 从服务器连接主服务器后，*从服务器向主服务器发送数据同步消息，主服务器执行rdb持久化将快照文件发送给从服务器，主服务器同时缓存持久化期间执行的写命令再快照文件发送完毕后发送写命令* - **全量复制**
  3. 每次主服务器写操作时，主动与从服务器进行数据同步操作向服务器发送相同的写命令。 **- 增量复制**



## 2. 主从关系

1. 一主二仆
   1. 从头开始复制
   2. 主机shutdown后从机仍听从于主机
2. 薪火相传
   - 从服务器仍可挂载从服务器，从成暂主 -> *从服务器用slaveof ip port指定其他从服务器作为自己主服务器*
   - 缺点：中间slave宕机，后面的slave均无法备份
3. 反客为主
   - 基于薪火相传，中间从服务器使用**slaveof no one**将变成主服务器
   - **自动化反客为主 - 哨兵模式**

## 3. 哨兵模式

- 自动化反客为主：后台监控主机股账，若主机故障则根据投票数自动将从机转换为主机（此时故障主机为从机）
- 方法：配置sentinel.conf文件，设置*sentinel monitor mymaster 127.0.0.1 6379 1x   - 哨兵监控master
  - x：当有x个sentinel认为一个master失效时，master才算真正失效

- 启动哨兵：redis-sentinel  哨兵配置文件位置



- 新主登基规则：从下线的主服务的所有从服务中挑选一个成为主服务：
  1. 优先级靠前 config中的replica-priority num关键字
  2. 偏移量最大
  3. runid最小 （redis启动时随机生成的40位runid）

- 群仆称臣：挑选出新主机后sentinel向原主机的从机发送slaveof新主机命令，复制新master
- 旧主俯首：下线的服务重新上线时，sentinel向其发送slaveof命令让其成为新主的仆从







- 复制延时: master同步到slave有一定延迟



# U8 - 集群

## 8.1 集群简介

- 引入：服务器容量不够、并发操作分摊问题、<u>ip地址/端口号变化时对应修改主机地址、端口信息。</u>等的解决方案。
- **概念：Redis集群实现了对Redis水平扩容，即启动N个redis节点，将整个数据库分布存储在N个节点中，每个节点存储总数据的1/N.**
  - **通过分区来提供一定程度可用性：即使集群中一部分节点失效或无法通讯，集群也可以继续处理命令请求。**

- 搭建方式
  1. 代理模式：客户端通过代理服务器找寻合适的主机进行连接。
  2. 无中心化集群：客户端可以访问任何一台主机，主机之间可以相互连接访问，最终寻找合适的主机进行连接。
- 优缺点:
  - 优点：扩容、分摊压力、无中心化配置简单
  - 缺点：不支持多键操作(设置user)，多键Redis、lua脚本不支持、迁移麻烦(代理->五中心化)

## 8.2 集群有关配置

```redis
cluster-enabbled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 15000
```



- cluster-enabled yes 开启集群
- cluster-config-file nodes-6379.conf - 设置当前节点配置文件名
- cluster-node-timeout 15000 设置失联时间 - 减少网络分区造成数据丢失时间





## 8.3 分配规则&slots&故障恢复

- 数量：至少要有三个主机。

- 分配原则：尽量保证每个主数据库运行在不同IP地址，每个主库和从库不在一个IP地址上。（高可用）



- 插槽：集群使用公式**CRC(key)%16384**计算键key属于哪个槽（对应哪个主机），其中CRC16(key)语句用户计算键key的CRC16校验和。
  - **作用：将值分配到不同主机内存中。**

- 相关命令：
  - cluster keyslot k1: 查询key对应的插槽位置
  - cluster countkeysinslot num: **查询num槽位对应键数** （需切换到插槽对应主机进行查询）
  - cluster getkeysinslot num: 查询对应槽位中的键



- 故障恢复：
  - 主机宕机：从机成为主机提供服务，若主机启动时主机变成新主机的从机
  - 某一段插槽主从全挂：根据cluster-require-full-coverage yes/no决定
    - yes: 集群全挂。
    - no：该段插槽数据全不能使用，也无法存储。



# *U9 - redis应用问题*

## 1. 缓存穿透

- 现象：
  1. 应用服务器压力变大
  2. redis命中率降低 -> 数据库查询率提升
- 原因：redis查询不到数据库内容(因此一直查询)；出现很多非正常url访问
- 解决方案：
  1. 空值缓存：将查询空值也存储在redis中，设置空值过期时间很短(<5min)
  2. 设置可访问名单(白名单)：使用bitmaps定义一个可以访问的名单，名单id作为bitmaps偏移量，每次访问时与bitmaps的id比较，若id不在其中则进行拦截，不允许访问。
  3. 布隆过滤器(Bloom Filter)：底层bitmaps；检索一个元素是否在一个集合中。*优点：时空效率很高；缺点：有一定误识别率和删除困难。*
  4. 始时监控：redis命中率急剧降低时，排场访问对象和访问数据，与运维配合设置黑名单限制服务。



## 2. 缓存击穿

- 现象：
  1. 数据库访问压力瞬时增大
  2. redis中没有出现大量key过期且正常运行
- 原因: redis某个(少数)被大量访问的key过期
- 解决方案：
  1. 预先设置热门数据：redis高峰访问前将一些热门数据提前存入redis中，加大key的存活时长
  2. 适时调整：现场监控热门数据，实时调整key过期时长
  3. 使用锁

## 3. 缓存雪崩

- 现象：数据库压力变大 - 响应变慢 - 服务器崩溃
- 原因：缓存在短时间内大面积失效，所以后面请求落在数据库上，造成数据库短时间内承受大量请求而崩溃。
- 解决方案：
  1. 构建多级缓存架构：nginx缓存+redis缓存+其他缓存。。。
  2. 使用锁/队列：加锁/队列保证不会有大量线程对数据库进行一次读写，避免失效时大量并发请求落在底层存储系统 - 不适用高并发。
  3. 设置过期标志更新缓存：记录缓存过期提前量，若(快)过期则会触发通知另外线程在后台更新实际key的缓存。
  4. 将缓存失效时间分散：使缓存过期时间重复率降低，降低集体失效的概率。

## 4.分布式锁

- 针对分布式集群系统（多线程、多系统、分布于不同机器），单机部署情况下并发控制锁策略失效 - **引入跨JVM的互斥机制控制共享资源访问。**
  - 实现方式：数据库、缓存(Redis)、Zookeeper (**Redis性能最高**) 
    - redis实现方式：为某一key设置锁setnx，通过对该key获取判断是否有锁
- 确保分布式锁可用的满足条件：
  1. **互斥性** - 任意时刻只有一个客户端能持有锁
  2. **不会发生死锁** - 设置过期时间
  3. **手动只能解自身的锁** - 设置uuid
  4. **加锁解锁具有原子性** - lua脚本设置



- *setnx \<key1>\<value1> ：给key上锁； del释放锁*
  - 解决锁长时间占用: 给锁设置过期时间 **expire key time**
  - 上锁的同时设置过期时间：**set key value nx ex time**



- Java中设置分布式锁
  - redisTemplate.opsForValue().setIfAbsent(key,v,time,timeFormat):  **对key上锁，上锁成功返回true；否则返回false。可设置锁过期时间time以timeFormat为单位。**
  - *问题：可能占用锁期间服务器卡顿，超时造成锁释放；但服务器卡顿结束之后继续操作，操作结束之后又手动释放了别人抢占的锁。* (手动释放：操作结束时释放锁；自动释放：锁过期自动释放)
  - **解决方案：为锁value设置uuid，每次手动释放前都判断当前uuid和要释放锁的uuid是否一样，相同时才能手动释放。**



# U10 - redis单线程模型

I/O多路复用程序，文件事件分派器与事件处理器。

- 多个socket并发产生不同的事件，I/O多路复用程序监听多个socket将**事件就绪的socket**置于一个就绪队列排队；每次从队列中有序、同步取出一个socket分派事件分派其使其分派给对应的事件处理器；待事件处理完毕再从队列中取出下一个socket交给事件分派器。

#  Redis6新特性

1. ACL - 访问控制列表 - 根据可执行命令和可访问键来限制某些连接。(授予权限 - 用户名&密码；可执行的命令；可操作的key)
2. IO多线程 - <u>客户端交互部分的网络IO交互处理多线程</u>(Redis执行命令仍单线程)