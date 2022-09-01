# 1. sql优化方法

###### 1. 选择最合适的字段属性

- 表中字段宽度尽量小 - 减少不必要空间损耗

###### 2. 尽量将字段设置为NOT NULL （避免在where子句对字段进行null值判断，） -  减少NULL值比较

- null值比较会使引擎放弃使用索引而使用全表查询
- 可对null值赋默认值确保列中无null值 - default

###### 3. 合理使用ENUM枚举类型

- 省份/性别等可列举项最好使用ENUM - ENUM类型被当作数值型数据处理，速度比文本类快得多

###### 4. 使用连接代替子查询

- 如此MYSQL不需在内存中创建临时表来完成这个逻辑上需要两个步骤的查询工作

- 应确认JOIN查询表中字段被建立索引 - 以优化JOIN的sql语句。(STRING类型相同字符集、DECIMAL与INT不可使用索引)

###### 5.  UNION联合查询代替OR （UNION ALL代替UNION）

- union可将需要使用临时表的两条或多条select整合于一个查询中。
- 当查询结果不出现重复结果集或不在乎重复结果集时可使用union all代替union -  *union需将两个或多个结果集合并后再进行唯一性过滤操作，损耗较大。*

###### 6. 尽量避免在where子句使用!= <>，否则将放弃索引而使用全表查询

###### 7. 使用索引

# 2. 索引

- 索引用于排序、过滤数据以加快搜索和排序操作的速度（尤其含MAX、MIN、ORDERBY等命令时）
- 索引应建立在那些将用于join、where和order by排序的字段上（尽量不要对含有大量重复值的字段建立索引 eg.ENUM）
- 索引改善了检索select的性能，但降低了插入、修改和删除的性能 - *执行此类操作时DBMS必须动态更新索引*



eg. 对admin表的id属性添加索引"id_ind"

```mysql
CREATE INDEX id_ind
ON admin(id)
```



# 3. 触发器 trigger

- 触发器是特殊的存储过程，**在特定的数据库活动发生时自动执行**，可绑定INSERT、UPDATE\DELETE操作
- 触发器内代码由如下数据访问权：
  - INSERT操作所有新数据
  - UPDATE操作所有新数据和旧数据
  - DELETE操作中删除的数据



- 用途举例：
  1. 保证数据一致：INSERT/UPDATE时字母大写
  2. 基于某个表的变动在其他表上执行活动：日志功能
  3. 进行额外验证并根据需要回退数据



## 3.1 触发器语法

```mysql
delimiter $
create trigger triggerName  
after/before insert/update/delete 
on 表名 for each row   #这句话在mysql是固定的  
begin  
    sql语句; 
end $;
```

1. triggerName: 触发器名称
2. after/before: 触发器触发时机
3. tinsert/update/delete: 触发事件， 可处理的触发事件主要有如下：
   - BEFORE INSERT,BEFORE DELETE,BEFORE UPDATE
     AFTER INSERT,AFTER DELETE,AFTER UPDATE
4. delimiter $ 与 end中的 $ 根据具体情况选用



针对不同的触发事件，选用new/old操作不同的数据

![img](https://images2017.cnblogs.com/blog/1147480/201709/1147480-20170922213509837-496822531.png)



## 3.2 触发器实际举例



eg1. 设置表boy插入的boyName均为大写：

```mysql
DROP TRIGGER IF EXISTS `upper`;
DELIMITER $
CREATE TRIGGER `upper`
BEFORE INSERT ON boys FOR EACH ROW
BEGIN
	SET new.boyName = UPPER(new.boyName);
END $
```



eg2. 设置表boy每插入一行语句就在admin中插入记录

```mysql
DROP TRIGGER IF EXISTS `upper`;
DELIMITER $
CREATE TRIGGER `upper`
BEFORE INSERT ON boys FOR EACH ROW
BEGIN
	INSERT INTO admin
	VALUES(new.id,new.boyName,new.userCP);
END $
```

## 3.3 触发器使用注意

- 触发器按照before - 行操作- after的顺序执行，任何一部错误都不会执行剩余操作
- 不能在触发器中以任何形式开始或结束事务语句(start, commit, rollback...)



# 4. 分库分表

## 4.1 分库

解决问题

1. 磁盘存储 - 数据库内容较多MySQL单击存储容量达上限
2. 并发连接 - 避免高并发场景大量请求访问单机数据库



分库策略：

1. 垂直分库：将系统中不同业务类型将数据库进行拆分，使得一个数据库具有其特有应用属性 (用户、订单、商品)
2. 水平分库：**将库中表的数据量切分到不同的数据库服务器**，每个服务器都具有所有的表类型，不过存储的信息有差异。



## 4.2 分表

解决问题：

1. 数据量较大，SQL查询变慢，没命中索引更甚。



分表策略：

1. 垂直分表：将单表一些列拆分出去另立表。（聚簇索引叶子节点单页存储信息更多，B+树高度降低）
   - eg. 用户表分立成用户基本信息与用户详细信息表。
2. 水平分表：按照某些规则（hash取模，range）将数据库中数据切分到多表。
   - eg.将订单表按照时间（年、月、日）切分成多表。



### 水平分库分表策略

1. range - 根据范围划分
   - 缺点：可能会造成热点问题 (某个月增长量较大)
2. hash取模 - 指定的key (id主键) 对分表总数进行取模，把数据分散到多个表中，由取模数决定分表数量。
   - 缺点：rehash时数据迁移。
3. range + hash - 对单表id使用range（range可指定一个较大的范围），对一个range采用hash分表。





## 4.3 分库分表时机

分表：表中数据量急剧增加时，根据当前索引B+树高度以2~3层为宜 (500w数据量)

分库：数据库称为性能瓶颈（单击磁盘容量、并发访问）



## 4.4 分库分表问题解决

1. 本地事务失效 - 需使用**分布式事务**
2. 跨库关联 - 分两次查询联立
3. 聚合函数 - 在各个节点上得到结果后在应用程序端合并处理
4. 分页问题 - 应用端汇聚再分页
5. 分布式ID - 保证分表全局ID唯一 (UUID or 雪花算法)



### 分布式事务

解决问题：非单机环境(分库分表)下事务操作的正确性，尽力保证事务的ACID特性。

- AD原子性持久性：严格遵循
- I隔离性：并行事务之间不可影响；事务中间结果可见性允许适当放宽
- C一致性：事务过程中的一致性适当放宽；事务完成后的一致性严格遵守



参与角色：

- 应用程序： AP - 事务发起者
- 资源管理器：RM
- 事务管理器：TM

- **事务发起者，资源、资源管理器和事务协调者分别位于分布式系统不同节点，保障分布式场景下数据操作的正确执行。**



**两阶段提交/XA**

1. prepare: 所有RM锁住需要的资源并准备执行事务，参与者Ready时向TM报告准备就绪。
2. commit/rollback: TM确认所有RM ready后向所有参与者发送commit命令。

- 若任何一个RM prepare失败，则TM通知本轮参与所有prepare的RM回滚。

![v2-90e76ee46f1be7c1aaa8643a425683b0_720w](D:\Huawei Share\Screenshot\v2-90e76ee46f1be7c1aaa8643a425683b0_720w.jpg)