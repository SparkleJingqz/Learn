# U12 - MySQL成本计算与优化 （暂pass）

## 1. 成本概述

- I/O成本：将页从磁盘加载到内存的耗时
- CPU成本：读取、检测、排序记录的耗时
- **计算方法：**
  - **读取页面成本-1.0；读取/检测记录成本-0.2**
  - 二级索引+回表: 范围区间数量-1.0；需要回表的记录数

## 2. 单表查询成本

- 根据搜索条件分析可能用到的索引，将之与全表扫描进行比较选取最合适





# U14 - MySQL规则优化

## 1. 条件化简

1. 移出不必要括号
2. 常量传递
   - *eg. a=5 && b>a  ===  a=b&b>5*
3. 等值传递 
   -  *eg. a=b&b=2 === a=2&b=2*
4. 移出无用条件：移出true/false表达式
5. 表达式计算：表达式中包含常量时现计算出来
   - **注：优化器不会化简表达式，只会预先计算常量。**
6. HAVING&WHERE：没有出现聚集函数与group by子句时
7. 常量表检测：先执行常量表查询，再将涉及该表条件替换为常数，最后分析其余表查询成本。
   - **常量表：查询表记录数<2或者使用主键/唯一二级等值匹配**
8. 外连接消除：reject-NULL **查询条件指定where被驱动表关联列值不为null**，与内连接无异。
   - 优化器通过评估表不同连接顺序成本选出成本最低连接顺序进行查询。



## 2. 子查询优化

- 按返回结果集区分：
  - 标量子查询：只返回一个单一值的子查询
  - 行子查询：返回一条记录的子查询
  - 列子查询：查询一个列的数据
  - 表子查询：查询多行多列
- 按与外层查询关系区分：
  - 不相关子查询：子查询单独运行出结果且不依赖于外层查询值。
  - 相关子查询：子查询执行需依赖于外层查询



- 布尔表达式与子查询
  - in/= any (in 与 =any含义相同)
  - any/some(子查询): 查询条件满足子查询任意语句即可
    -  **SELECT * FROM t1 WHERE m1 > ANY(SELECT m2 FROM t2)**
  - all(子查询)：查询条件满足子查询所有结果。 
    - **SELECT * FROM t1 WHERE m1 > (SELECT MIN(m2) FROM t2);**
  - exists子查询：**判断子查询结果集是否有记录，有即为true**



- **注意事项：**
  1. *子查询必须用小括号括起来*
  2. *select子句的子查询必须是标量子查询*
  3. *in/any/all子查询注意:*
     1. *不能有limit*
     2. *order by/distinct/无聚集函数的group by,having/ (查询优化器进行优化)*



### 2.1 子查询执行方式

#### 1. 标量/行子查询

- 不相关：**对于包含不相关的标量/行子查询查询语句。mysql会分别独立执行外层查询和子查询（两个单表查询）**
- 相关：先从外层查询获取一条记录，依次作为子查询条件，然后再将结果用作外层查询的条件。





#### 2. IN子查询

- 不相关子查询 - *物化*: **不直接将不相关子查询结果集当作外层查询参数，而是将该结果集写入一个临时表。**（物化表）
  - 物化表：**列相同、记录去重(表中所有列建立主键/唯一索引)、建立基于内存的Memory存储引擎的hash索引临时表/磁盘的B+树索引**

- 物化表转连接：相当于原表与物化表之间的内连接-等值匹配，优化器评估成本选取驱动表（物化表每列建立了索引）
- <u>半连接semi-join</u>：对于s1表某条记录，**只关心在s2表中是否存在与之匹配的记录，而不关心具体有多少记录匹配，结果集只保留s1表中的记录**
  - **相关子查询不是独立查询，不能转化为物化表进行查询。**
  - 实现方法之一: **重复值去除-建立临时表记录建立主键/唯一索引**
  - 使用条件：IN语句且在外层的where或on；子查询不包含聚集函数与排序/分组



#### 3. 派生表优化

1. 派生表物化-延迟物化：真正使用到派生表时才去尝试物化派生表。
2. 尝试将派生表和外表合并（查询重写为没有派生表形式）



# U15 Explain详解

| id            | ;结尾的select语句每个select都关联要给id          |
| ------------- | ------------------------------------------------ |
| select_type   | select关键字对应的查询类型                       |
| table         | 表名                                             |
| partitions    | 匹配的分区信息                                   |
| type          | 针对单表的访问方法                               |
| possible_keys | 可能使用的索引                                   |
| key           | 实际使用的索引                                   |
| key_len       | 实际使用索引长度                                 |
| ref           | 使用索引列等值查询时，与索引列等值匹配的对象信息 |
| rows          | 预估需读取的记录条数                             |
| filtered      | 某表经搜索条件过滤后剩余记录条数百分比           |
| Extra         | 额外信息                                         |

查询优化器可能对涉及子查询的语句进行重写为连接查询 (id值相同)

## 1. select_type : 查询语句涉及select关键字的查询类型

| SIMPLE             | 不涉及UNION与子查询                                          |
| ------------------ | ------------------------------------------------------------ |
| PRIMARY            | 涉及UNION( ALL), 子查询的大查询而言其由多个小查询构成，最左侧小查询即为PRIMARY |
| UNION              | UNION除PRIMARY外                                             |
| UNION RESULT       | 临时表完成UNION查询去重 (id为null)                           |
| SUBQUERY           | 包含子查询的查询语句不能转换为semi-join且为**不相关子查询，决定物化方案执行**时对应的子查询第一个select类型  *SUBQUERY子查询物化 ->只会执行一次* |
| DEPENDENT SUBQUERY | 不能转换为semi-join且为相关子查询的子查询第一个select关键字 *会被执行多次* |
| DEPENDENT UNION    | 包含UNION (ALL)的大查询中的小查询都依赖于外层查询时，第二个开始小查询类型即为该 |
| DERIVED            | 物化方式执行的包含派生表的查询，该派生表对应子查询即为DERIVED |
| MATERIALIZED       | 子查询物化后与外层查询进行连接查询时，该子查询类型即为该     |
| 。。。             |                                                              |

## 2. type MySQL对某个表查询时的访问方法

| type              | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| const             | 根据主键/唯一键与常数进行等值匹配                            |
| eq_ref            | 连接查询中被驱动表通过主键/唯一键等值匹配访问                |
| ref               | 通过普通二级索引与常量等值匹配                               |
| ref_or_null       | 对普通二级索引进行等值匹配查询且值也可以为NULL               |
| *index_merge*     | 对某个表的查询使用了**索引合并？** (Intersection, Union, Sort-Union) |
| *unique_subquery* | 使用EXISTS子查询且子查询可以使用主键/唯一键进行等值匹配      |
| index_subquery    | 基于unique_subquery的普通索引                                |
| range             | 使用索引获取某些范围区间记录                                 |
| index             | 索引覆盖且需扫描全部索引记录时                               |
| ALL               | 全表扫描                                                     |
|                   |                                                              |



# U18 Buffer Pool缓存

策略: InnoDB处理客户端请求访问某个页的数据时将该页数据从磁盘加载到内存中再进行访问并缓存与内存；再次访问该页面数据时不需要进行磁盘I/O

## 1. Buffer Pool

*作用*：为缓存磁盘页，在MySQL服务器启动时便向操作系统申请一块连续内存Buffer Pool。 默认128M

*存储*：控制块包含对页的控制信息（页表空间编号、页号、地址等），控制块与缓存页一一对应。控制块在Buffer Pool前端；缓存也在后端。

*free链表* - 分配空闲缓存页：所有空闲的缓存页对应的控制块作为一个节点加入free链表，当需空闲缓存页来存储页面信息时从free链表摘取控制块获取对应空闲缓存页地址信息来使用。

*获取已缓存页* - 哈希处理： 表空间号+页号为key获取对应缓存页。

*flush链表* - 脏页处理：修改过的缓存页对应的控制块添加到flush链表中，不立即同步磁盘。

*LRU链表* - 缓存处理：划分区域的LRU链表  <u>预读处理会造成可能不会读取的页面加载到Buffer Poll中；全表扫描可能造成Buffer Pool大换血</u>

- young区域：存储使用频率非常高的缓存页 - 热数据
- old区域：存储使用频率较低的缓存页 - 冷数据

1. **磁盘上某个页面在初次加载到Buffer Pool中某个缓存页时该页对应控制块置于old区域头部**  LRU策略
2. **对某个处在old区域的缓存页第一次访问时记录其访问时间，若后续访问时间与第一次访问时间在某个时间间隔内时就不讲该页移动到young区域中。**  避免临时访问误差



# U19 事务

## 1. ACID属性

1. 原子性：事务是不可分割的最小单位，只有执行/不执行两种情况
2. 隔离性：事务之间不相互影响
3. 一致性：事务处理前后**数据库保持某些既定约束**
4. 持久性：事务操作对数据库的改变是永久的



## 2. 事务语句

```mysql
start transaction;
select ...;
update ...;
savepoint a;
delete ...;
commit / rollback (to a);
```



# U20 redo日志

功能：**让已经提交的事务对数据库数据所做修改永久生效，即使系统崩溃重启后仍能将修改恢复 - 记录这部分修改。**

- 数据库首先将数据从磁盘读取到内存中的Buffer Pool中再访问，防止事务提交后对Buffer Pool修改后系统崩溃未能刷进磁盘中。
- 记录修改，系统恢复时根据所记录的修改更新数据页。
- 记录内容：*在某个页面的某个偏移量处修改了几个字节的值，以及具体的修改内容。*

通用结构： 

- type：该条redo日志的类型
- space ID：表空间ID
- page number：页号
- offset, len：可选字段，偏移量与长度
- data：该条redo日志具体内容



对数据进行操作所产生的redo日志可能有很多，会涉及除数据信息之外的如Page Header、Page Tailer(页内数据总量)，还包括数据本身。 根据操作类型可将redo日志分为delete/insert等，所修改的信息也分为逻辑信息与物理信息。（以insert举例）

- insert型redo日志保存插入信息，使用redo日志恢复insert记录时需先调用与insert有关的函数使得逻辑信息（Header记录数目...、Trailer检验和）得到相应的修改，然后才执行insert记录修改物理信息。



## 1. Mini-Transaction

一个sql语句 (insert, delete)可能对应多个页面的修改，需保证一条语句对多个页面的修改是不可分割的。这可能会产生多条redo日志。

- eg.悲观插入：涉及页面分裂、数据复制操作，需修改多个页面数据。
- 解决方案：只有一条redo日志 - type首位为1； 否则将该组redo日志最后加入一个标记日志 - 只有type字段值为31.



Mini-Transaction (mtr): 对底层页面的一次原子访问过程。 一个插入操作会涉及一个mtr，包含了多个对底层页面的操作，他们整体是原子不可分割的。

- mtr生成的redo日志记录于512KB的页中，对应的保存redo日志的内存空间也存在redo log buffer(redo日志缓冲区)，一个log buffer包含多个redo日志记录页。
- 通过buf_free的全局变量指明redo记录应写于log buffer的具体位置。
- **写入时将mtr结束时生成的一组redo日志统一全部复制到log buffer中。**
- 刷新log buffer的时记： 空间不足、后台线程刷、关闭服务器时。



## 2. redo本地日志文件组

MySQL数据目录下默认存在ib_logfile0-1文件，log buffer中的日志默认情况刷新到这两个文件中，可以设置。循环写入。



## 3. LSN日志序列号 log sequence Number

记录已经写入的redo日志量，以mtr为单位，初始值**8704**，记录每个mtr在log buffer中存储的偏移量(包含header和trailer)。



## 4. flushed_to_disk_lsn

全局变量，记录当前log buffer中哪些日志已经被刷新到磁盘中的日志文件。

- 8704 ~ buf_next_to_write 已刷新到磁盘文件； buf_next_to_write ~ buf_free (lsn) 写入到log buffer但未刷新到磁盘的redo日志； buf_free (lsn)~ 未使用的log buffer



有新的mtr对应redo日志写入log buffer时lsn值会增长，flushed_to_disk_lsn会随着log buffer刷盘而增长。 当flushed_to_disk_lsn == lsn时标志着log buffer中的所有redo日志已经写入磁盘。



## 5. checkpoint 

### - flush链表

mtr对底层的原子访问除将redolog写入log buffer中，还会将mtr执行过程可能修改的页面加入到Buffer Pool的flush链表。

结构: 对应o_m为写前lsn， n_m为写后lsn。 每个控制块只对应一个修改页面。  且按照页面修改记录新-头插法； 多次更新的页面不会改变控制块顺序，但会修改对应的n_m值。![image-20220407231425430](C:\Users\86137\AppData\Roaming\Typora\typora-user-images\image-20220407231425430.png)

判断某redo日志占用的磁盘空间是否可被覆盖的依据：其对应的flush页面是否已经刷新到磁盘中。



**checkpoint_lsn**： 记录当前系统可被覆盖的redo日志总量（初始8704）

- redo日志在磁盘中可被覆盖：<u>只有flush链表中的控制块对应页面被修改到磁盘上时才可。</u>，此时flush链表对应控制块会被移除。
- 措施：读取最早o_m值（链表尾部），则当前系统lsn<o_m的redo日志都是可以被覆盖， **checkpoint_lsn = 尾部o_m**
- 持久化：将checkpoint_lsn和对应redo文件偏移量以及checkpoint次数编号写到文件checkpoint1/2中



<img src="C:\Users\86137\AppData\Roaming\Typora\typora-user-images\image-20220409211114413.png" alt="image-20220409211114413" style="zoom:67%;" />

- Log Sequenc Number: lsn 记录已写入log_buffer中的mtr偏移量，8704开始
- Log Flushed Up To : flushed_to_disk_lsn，记录log_buffer中哪些redo日志已被写入磁盘
- Pages flushed up to : flush链表尾部o_m ，小于该o_m的可被覆写 (同步到本地文件的checkpoint_lsn)
- checkpoint_lsn : 本地化到日志文件的尾部o_m，小于该checkpoint_lsn的可被覆写

## 6. innodb_flush_log_at_trx_commit 持久化保障

持久性：事务对数据库的改变是永久性的。 （需保障事务提交时对应的redo日志立即写入磁盘中，会影响性能）

0：不立即像磁盘同步redo日志，交给后台线程处理。（数据库/操作系统挂了g）

1：持久化操作，默认值。

2：事务提交时将redo日志写到log_buffer中，但不保证写入磁盘。  （数据库挂了可，操作系统挂了不行）



## 7. 崩溃恢复

- 起点：从日志文件两个checkpoint中选取最大的checkpoint_lsn作为恢复起点
- 终点：redo日志log block(page)中的log block header代表该日志写入的大小，写满则为512，若不为512则该block未被写满，终点。
- 恢复操作：**读取redo日志文件checkpoint_lsn起点，根据space_id(表空间id)与page_number创建哈希映射，将槽相同的日志以链表按时间顺序相连。执行redo日志时以哈希表为顺序执行 - 减少磁盘IO操作**
- 避免重复：checkpoint_lsn至尾部o_m可不重新刷：根据页面头文件file_header中FIL_PAGE_LSN属性记录最近一次修改对应lsn值。 若checkpoint_lsn < 该值则不需要执行恢复操作。



# U21 undo日志 - 保证回滚

undo日志被分配到类型为FIL_PAGE_UNDO_LOG类型的页面中

roll_pointer: 指向记录对应undo日志的一个指针

## 1. insert类型undo日志

| end_of_record                    | undo_type  | undo_no   | table_id     | 主键列信息<len,value> | start of record                    |
| -------------------------------- | ---------- | --------- | ------------ | --------------------- | ---------------------------------- |
| 标志本条undo结束，下一条页面地址 | INSERT_REC | UNDO_NO++ | 修改对应表id | 存储空间大小与真实值  | 上一条undo结束，本条在页面开始地址 |

## 2. delete类型undo日志

删除过程：

1. 将正常记录对应的delete_mask标识位设置为1，状态为中间状态
2. 事务提交后由专门线程将真正记录删除，将中间状态记录加入垃圾链表中，更改页面头字节状态。

PAGE_FREE: 指向被删除记录组成的垃圾链表的头节点

PAGE_GARBAGE: 当前页面可重用存储空间总字节数。  -> **每次尝试插入新纪录时首先判断PAGE_FREE对应头节点删除记录存储空间是否可容纳新纪录，若可直接重用(可能造成内存碎片)，若不可则申请新空间。  页面快满时判断PAGE_GARBAGE与内存碎片是否足够容纳新纪录，若可InnoDB会尝试重新组织页面记录消除碎片。**

| e_o_r | u_t  | u_n  | t_i  | info bits | old trx_id   | old roll_pointer | <l,v> | index_col_info len | 索引列各列信息<pos,len,value> | s_o_r |
| ----- | ---- | ---- | ---- | --------- | ------------ | ---------------- | ----- | ------------------ | ----------------------------- | ----- |
|       |      |      |      |           | 旧的trx_id值 | 旧的roll_pointer |       | <p,l,v>占用大小    | 被索引的列的各列信息          |       |

## 3. update类型undo日志

### 1. 不更新主键列

就地更新：若更新列更新前与更新后每列占用存储空间大小一样，可就地更新。

删旧加新：有任何一个列更新前后占用存储空间大小不同则删旧加新。 若新纪录存储空间<=旧记录则可直接重用垃圾链表；否则新申请。

### 2. 更新主键

1.将旧记录delete mask设为1，事务提交后由专门线程做purge操作加入垃圾链表。 (MVCC访问)

2.根据更新后各列的值创建一条新记录插入聚簇索引中

**产生两条undo日志：TRX_UNDO_DEL_MASK_REC;  TRX_UNDO_INSERT_REC。**

| n_updated    | <pos,old_len,old_value> |                      |
| ------------ | ----------------------- | -------------------- |
| 多少列被更新 | 被更新列更新前信息      | 其余信息与delete相同 |



# 24 - 事务隔离级别与MVCC

## 1. 事务隔离级别

- MySQL基于C/S架构软件，对于一个S可能有多个C与之进行连接，CS连接称之为一个会话。**每个C都可以在会话中向服务器发出请求语句 - S可能会同时处理多个事务**
  - 不保证绝对的隔离性以提供良好的性能。



- 设置隔离级别：

  ```mysql
  SET GLOBAL|SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
  ```



## 2. MVCC-多版本并发控制

- 聚簇索引记录的两个隐藏列：trx_id;   roll_pointer
  - trx_id:  一个事务对某条聚簇索引记录**改动(增、删、改)**时，将该事务id赋值给trx_id隐藏列
  - roll_pointer:  每次对某条聚簇索引记录改动时，将旧的版本写入到undo日志中，该隐藏列指向这条undo日志记录找到修改前的信息
    - 注:insert undo只在回滚时起作用，提交后就没作用了（在这之前没有对应的旧版本）



### 2.1 版本链

- 版本链：**每次对记录进行改动，都会记录一个undo日志，每条undo日志都有一个roll_pointer属性(除insert)将undo日志连接起来构成链表。**
  - **版本链的头节点：当前记录的最新值**



### 2.2 ReadView

- 根据隔离级别判断版本链中哪个版本对当前事务可见
- ReadView:  m_ids; min_trx_id; max_trx_id;  creator_trx_id
  - m_ids: 生成ReadView时当前系统活跃的读写事务的事务id列表
  - min_trx_id: m_ids最小值
  - creator_trx_id:  生成该ReadView的事务id



- **据ReadView判断记录版本可见性**

  - **被访问版本trx_id与creator_trx_id相同：当前事务在访问它自己修改过的记录，可**
  - **被访问版本<min_trx_id:   生成该版本的事务在当前事务生成View之前已提交， 可**
  - **被访问版本>max_trx_id:  生成该版本事务在当前事务生成View后才开启，不可访问**
  - **else:  判断被访问版本是否在m_ids列表，若不在则已提交-可；若在则该版本还活跃-不可**

- 若某个版本数据对当前事务不可见，则顺着版本链继续找下一个版本数据直至找到当前版本可见记录（若均不可见则该条记录对该事务不可见，查询结果不包含）

   

- ReadView之于隔离级别的生成时机：

  - **READ COMMITTED: 每次读取数据前生成一个独立ReadView**
  - **REPEATABLE READ: 第一次读取数据前生成一个ReadView**



# 25 - 锁

- 锁结构：一个事务对记录做改动时，首先查看内存中有无与该记录关联的锁结构。无-生成所结构与之相关联
  - trx：锁结构由哪个事务组成
  - is_waiting: 当前事务是否正在等待(true/false)
    - false：未在等待，加锁成功
    - true：正在等待，获取锁失败



- 隔离级别对应使用措施
  1. 一致性读：使用ReadView - ReadCommitted; RepeatableRead
  2. 锁定读：读、写都加锁
     - 脏读: 写时加锁
     - 不可重复读：读时加锁
     - 幻读：



## 1. 快照/当前读



快照读&当前读

- 快照读：根据隔离级别与MVCC规则生成readView ()
- 当前读：根据情况给记录加next_key_lock，一定程度避免幻读。 （RR隔离级别）

- ```mysql
  # 快照读  读取记录的可见版本 - 根据MVCC规则
  select .. from where;
  # 当前读，加行级锁 读取记录的最新版本
  # 其他的增/删/改操作都属于当前读 - 先获取记录再执行增/删/改操作
  select * from table where xxx lock in share mode; (共享锁)
  select * from table where xxx for update;
  ```

1. 主键/唯一索引，全部命中时退化为记录锁
2. 无索引列，当前读操作时全表加gap锁阻止插入。
3. 非唯一索引列，对命中记录前后加gap锁
   - eg. 6,8,8,10  id=8 : 在(6,10]间加gap锁  **为了阻止插入8造成的前后不一致现象**



## 2. 表锁 - 多粒度锁

**InnoDB表级锁：针对表的DDL语句 drop table; alter table**



1. 加S锁：
   - 别的事务可以继续获取该表以及该表记录的S锁，但不可获取该表与该表记录的X锁
2. 加X锁:
   - 别的事务关于该表与该表记录的X、S锁都不可以获取



- **意向锁：避免为表加锁时逐行检查该表有无行锁**
  - 意向共享锁IS：事务在某条记录加S锁时在表级别加IS
  - 意向独占锁IX：事务在某条记录加X锁时表记别添加IX

```mysql
LOCK TABLES t READ; #获取读锁
LOCK TABLES t WRITE; #获取写锁
```



### AUTO-INC锁 / 轻量级锁

对应列设置AUTO_INCREMENT修饰时使用

1. AUTO_INC锁：执行插入语句时在表级别加一个AUTO_INC锁，为每条AUTO_INCREMENT列分配递增值，语句执行结束后再释放AUTO_INC锁
2. 轻量级AUTO-INC : 在insert语句获取到AUTO_INCREMENT对应值之后就把该锁释放而非插入完成再释放。



## 3. 行锁

Record Locks  *LOCK_REC_NOT_GAP*: 记录锁 S/X

*LOCK_GAP*: **防止插入幻影记录-幻读现象**

- 给当前记录添加gap锁时，当前记录与其相邻记录间不可再插入记录了，直至拥有gap锁的事务提交释放。
  - <u>数据页Supremum记录为该页数据最大记录，给Supremum添加gap锁即可保证最后一条记录之后也不可进行插入操作。</u>

*Next_Key_Locks*: **Record Locks + Gap Locks**

*LOCK_INSERT_INTENTION* 插入意向锁：

- 插入事务在gap锁记录上添加插入意向锁。在gap锁被释放后插入事务获取插入意向锁并执行插入操作，插入意向锁的事务不会相互阻塞可同时获取插入意向锁执行插入操作。

隐式锁：事务插入数据时防止在提交前被别人修改

- 对于聚簇索引，trx_id隐藏列记录最后改动该记录的事务id，插入操作对应的记录trx_id即为插入事务的id。**其他事务想要获取该记录的S/X锁时首先判断trx_id隐藏列代表事务是否为当前活跃事务，若是则帮助插入事务创建一个X锁并使自身进入等待状态。**

聚簇索引：trx_id是否活跃，若不活跃则为当前事务添加X锁，自身也添加X锁

二级索引：Page Header部分PAGE_MAX_TRX_ID - 对该页面做改动的最大事务ID，比较当前活跃的最小事务id，回表重复聚簇索引做法。



### 补充说明 幻读问题解决

1. 串行化隔离级别
2. 当前读for update时，使用next_key_lock解决，新的insert和update阻塞
3. 快照都采用MVCC解决幻读问题？



- InnoDB锁内存结构：对记录加锁时若符合如下条件，可将这些记录的锁放到一个 **锁结构**
  1. 同一个事务中进行的加锁操作
  2. 被枷锁的记录在同一页面中
  3. 加锁的类型相同
  4. 等待状态相同
- ![capture_20211209152655064](D:\Huawei Share\Screenshot\capture_20211209152655064.bmp)





# Question

1. 乐观锁，悲观锁
2. MySQL主从复制

