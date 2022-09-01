## 0.基础知识

### 0.1 关键字基础概念

1. 码：设K为表中一个属性/属性组，若除K外所有属性都**完全函数依赖**于K，则K即为码/候选码
2. 函数依赖：若表中属性x值确定，则y值确定，那么可以成为y函数依赖x：**x -> y | y=f(x)**
   1. 完全函数依赖：x -> y 且 x任意真子集x'不满足 x' -> y  <u>**x F-> y**</u>
   2. 传递函数依赖：x -> y, y -> z : x -> z   <u>**x T-> z**</u>
   3. 部分函数依赖：x -> y且并非完全依赖  **<u>x P-> y</u>**
3. 元组：行
4. 属性：列



### 0.2 三范式

1. 1NF: 数据库属性不可拆分
2. 2NF: 基于1NF，消除非主属性对码的部分函数依赖
3. 3NF: 基于2NF，消除非主属性对码的传递函数依赖

## 1.DML(Data Manipulation Language) 数据操纵语言

### 1.1 select - 查询

#### 1.1.1 基础查询

- select 查询的东西(列表) from 来自的表名



基础查询可进行如下查询：

1. 查询单个字段 select name from employees;
2. 查询多个字段 select name, birth from employees;
3. 查询表中所有字段 select * from employees;
4. 查询常量值 select 100; select 'Jing';
5. 查询表达式 select 100 / 2;
6. 查询函数 select version()



注:

1. distinct关键字：去重，去除查询结果中的重复项

2. +符号：只有运算符的功能，无连接符的功能
1. 若两个字符均为数值型，进行加法运算
   
2. 若一方或均为字符型，尝试转换为数值型再运算
   
3. 只要有一方为null，结果就为null
   
3. concat() 函数：将查询字合并，可为合并项起别名

   - eg. SELECT CONCAT(last_name, first_name) 姓名

#### 1.1.2 条件查询

- select 查询列表 from 表名 where 筛选条件

筛选条件分类：

1. 条件表达式筛选：
   - 条件运算符：<, >, =判断相等, <>(!=) 判断不等, >=, <=
   - between and 相当于 >= and <=
   - is null 与 is not null 查询值为或不为空 **(不可用=  != null)**
   - 安全等于 <=> 除判断值是否相等外还可与null进行比较

2. 逻辑表达式筛选：
   - 逻辑运算符：and-&&, or-||, not - !
   - in 类似于 多个or，in列表中的**值类型必须兼容**且**不可使用通配符**
3. 模糊查询：like关键字,常与通配符搭配使用
   - 常见通配符：% 表示任意多个字符、_表示任意单个字符
   - eg. last_name LIKE '%a%' 表示姓中含字母a的筛选条件

#### 1.1.3 排序查询

- order by 排序列表 [desc/asc]
- desc-降序， asc升序， 默认asc
- 一般置于查询语句最后面(limit除外)

#### 1.1.4 分组查询

- 分组函数做条件置于having子句中
- 优先使用分组前筛选
- 分组后也可添加排序置于分组查询最后 - group by 介于 where 与 order by

|            | 数据源         | 位置           | 关键字 |
| ---------- | -------------- | -------------- | ------ |
| 分组前筛选 | 原始表         | group by子句前 | where  |
| 分组后筛选 | 分组后的结果集 | group by子句后 | having |

eg. 查询有奖金的工种中最高工资大于12000的工种id与最高工资

```mysql
SELECT job_id,MAX(salary)
FROM employees
WHERE commission_pct IS NOT NULL
GROUP BY job_id
HAVING MAX(salary)>12000;
```

#### 1.1.5 *连接查询*

- 又称多表查询，当查询字段来自于多个表时使用连接查询
- 连接查询本质上即为将笛卡尔乘积进行条件筛选



- **on与where的执行顺序及效率**：
  - from a join b 与 from a,b产生的临时表结果集都是执行笛卡尔积(行乘积数)
  - on: 取得结果集的**同时**进行数据刷选、过滤
  - where：获得结果集**之后**，进行数据刷选过滤
  - on先，where后

##### 1- sql92标准 - 仅支持内连接

1. 多表等值连接的结果为多表交集部分
2. n表连接,至少需要n-1个连接条件
3. 多表的顺序无要求
4. 一般需要为表起别名
5. 可以搭配前面介绍的所有子句使用(排序,筛选,分组...)



**代码案例:**

```mysql
#案例1.查询女神名和对应男神名
SELECT NAME,boyName
FROM beauty,boys
WHERE beauty.`boyfriend_id`=boys.`id`;

#案例2.查询员工名和对应部门名

SELECT last_name,department_name
FROM employees,departments
WHERE employees.`department_id`=departments.`department_id`;

#- 为表起别名
#案例:查询员工号,工种号,工种名
SELECT last_name,em.`job_id`,job_title
FROM employees AS em,jobs AS jo
WHERE em.`job_id`=jo.`job_id`;
#当查询的两个连接表含有共同的索引时,添加表名消除歧义
#通过表起别名的方法简化语句,提高语句简洁度
#注,起别名后查询字段只能用别名
#from中两表顺序可以变更

#- 筛选条件
#案例:查询有奖金的员工名,部门名
SELECT last_name,department_name,commission_pct
FROM employees AS em,departments AS de
WHERE em.`commission_pct` IS NOT NULL
	AND em.`department_id`=de.`department_id`;

#案例:查询城市名中第二个字符为o的部门名和城市名
SELECT city,department_name
FROM locations AS l,departments AS d
WHERE l.`location_id`=d.`location_id`
	AND city LIKE '_o%';
	
#- 添加分组
#案例:查询每个城市的部门个数
SELECT city,COUNT(*) AS 部门个数
FROM locations AS l,departments AS d
WHERE l.`location_id`=d.`location_id`
GROUP BY city;

#案例:查询有奖金的每个部门名和部门的领导编号以及该部门饿最低工资
SELECT department_name,d.manager_id,MIN(salary)
FROM departments AS d,employees AS e
WHERE e.`commission_pct` IS NOT NULL
	AND d.`department_id`=e.`department_id`
GROUP BY department_name,d.`manager_id`; #确保二者共组(只以department_name分组不合理,因其可能有多个领导编号)

#- 添加排序
#案例:查询每个工种的工种名和员工个数,按员工个数降序
SELECT job_title,COUNT(*) AS 员工个数
FROM employees AS e, jobs AS j
WHERE e.`job_id` = j.`job_id`
GROUP BY job_title
ORDER BY 员工个数 DESC;

#- 实现三(多)表来连接
#案例:查询员工名,部门名和其所在城市
SELECT last_name,department_name,city
FROM employees AS e,departments AS d,locations AS l
WHERE d.`location_id`=l.`location_id`
	AND e.`department_id`=d.`department_id`;

#2. 非等值连接
#案例1: 查询员工的工资和工资急别
SELECT salary,grade_level
FROM employees AS e,job_grades AS j
#where salary >= lowest_sal and salary <= highest_sal;
WHERE salary BETWEEN lowest_sal AND highest_sal
ORDER BY salary;


#3. 自连接
#自表连接自表(利用别名)
#案例: 查询员工名及其上级名称
#由员工的名称对应的manager_id找到employee_id再找到对应上级名
SELECT e.last_name,e.manager_id,m.employee_id,m.last_name
FROM employees AS e,employees AS m#巧用别名,强👍
WHERE e.`manager_id`=m.`employee_id`;
```



##### 2- sql99标准

语法：

```mysql
select ...
from 表一 别名[连接类型]
join 表二 别名
on 连接条件
[where 筛选条件]
[group by分组]
[having 组内筛选]
[order by排序列表]
```
1. 内连接：inner 等值、非等值、自连接

2. 外连接：

   - 外连接查询结果为主表中的所有记录:
     1. 若从表有主表对应匹配值，则显式匹配值
     2. 若从表无匹配值，则显式nnull
     3. **外连接查询结果=内连接结果+主表有从表无的记录**

   1. 左外：left (outer) 左边为主表
   2. 右外：right(outer) 右边为主表
   3. 全外:  **(mysql不支持)** full(outer) 内连接结果+表1有表2没有+表2有表1没有

3. 交叉连接：cross



内连接代码：

```mysql
#1.等值连接
/*
特点:
1.添加排序,分组,筛选
2.inner可省略
3.筛选条件可置于where后面,连接条件置于on后面,提高分离性便于阅读
4.inner join连接和sql92语法中的等值连接效果一样,都查询多表交集
*/
#案例1:查询员工名,部门名
SELECT last_name,department_name
FROM employees AS e
INNER JOIN departments AS d
ON e.`department_id`=d.`department_id`;
#案例2:查询名字中包含e的员工名和工种名
SELECT last_name,department_name
FROM employees AS e
INNER JOIN departments AS d
ON e.`department_id`=d.`department_id`
WHERE e.last_name LIKE '%e%';
#案例3.查询部门个数大于3的城市名和部门个数
SELECT city,COUNT(*) AS 部门个数
FROM locations AS l
INNER JOIN departments AS d
ON l.`location_id`=d.`location_id`
GROUP BY city
HAVING 部门个数>3;

#案例4:查询哪个部门的员工个数>3的部门名和员工个数,按员工数降序
SELECT department_name,COUNT(*) AS 员工个数
FROM departments AS d
INNER JOIN employees AS e
ON d.`department_id`=e.`department_id`
GROUP BY department_name
HAVING 员工个数>3
ORDER BY 员工个数 DESC;

#案例5:查询员工名,部门名,工种名,并按部门名降序
SELECT last_name,department_name,job_title
FROM jobs AS j
INNER JOIN departments AS d
INNER JOIN employees AS e
ON e.`department_id`=d.`department_id`
AND e.`job_id`=j.`job_id`
ORDER BY department_name DESC;

#2. 非等值连接
#案例:查询员工工资级别
SELECT grade_level,COUNT(*)
FROM employees AS e
INNER JOIN job_grades AS jg
ON salary BETWEEN jg.`lowest_sal` AND jg.`highest_sal`
GROUP BY grade_level
ORDER BY grade_level DESC;

#3. 自连接
#案例:查询员工及其上级姓名
SELECT CONCAT(e.`last_name`,' ',e.`first_name`) AS employee,
	CONCAT(m.`last_name`,' ',m.`first_name`) AS manager
FROM employees AS e
INNER JOIN employees AS m
ON e.`manager_id`=m.`employee_id`
WHERE e.`last_name` LIKE '%k%' OR e.`first_name` LIKE '%k%';

```



外连接代码：

```mysql
#左外连接
SELECT NAME,boyName
FROM beauty g
LEFT OUTER JOIN boys AS b
ON g.`boyfriend_id`=b.`id`
ORDER BY boyName;
#右外
SELECT NAME,boyName
FROM boys AS b
RIGHT OUTER JOIN beauty AS g
ON g.`boyfriend_id`=b.`id`
ORDER BY boyName;

#案例: 查询哪个部门没有员工
SELECT department_name
FROM departments AS d
LEFT OUTER JOIN employees AS e
ON d.`department_id`=e.`department_id`
WHERE e.`department_id` IS NULL;

#全外连接 (mysql不支持)

SELECT NAME,boyName
FROM beauty g
FULL OUTER JOIN boys b
ON g.`boyfriend_id`=b.`id`;

#Ⅲ. 交叉连接(笛卡尔乘积结果)
SELECT NAME,boyName
FROM beauty g
CROSS JOIN boys b
#ON g.`boyfriend_id`=b.`id`;

#案例:查询部门名为SAL或IT的员工信息
SELECT e.*
FROM employees AS e
RIGHT OUTER JOIN departments AS d
ON d.department_id=e.department_id
WHERE d.department_name IN('SAL','IT')
AND d.department_id IS NOT NULL;
```

#### 1.1.6 子查询

- (嵌套查询) 出现在其他语句中的select语句
  1. select后面

按子查询出现位置分类：

1. select后： 仅支持标量子查询
2. from后：仅支持表子查询
3. where或having后：
   1. 标量子查询
   2. 列子查询
   3. 行子查询
4. exists后(相关子查询)：表子查询

按查询结果集行数不同分类：

1. 标量子查询：结果集只有一行一列
2. 列子查询：结果集只有一列多行
3. 行子查询：结果集有一行多列
4. 表子查询：结果集一般为多列多行



子查询解析&代码

```mysql
#1. 标量子查询
#案例1: 谁的工资比Abel高?
SELECT *
FROM employees
WHERE salary>(
	SELECT salary
	FROM employees
	WHERE last_name='Abel'
);

#案例2: 查询job_id与141号员工相同,salary比143号员工多的员工 姓名, job_id和工资
SELECT last_name,job_id,salary
FROM employees
WHERE job_id=(
	SELECT job_id
	FROM employees
	WHERE employee_id=141
)
AND salary>(
	SELECT salary
	FROM employees
	WHERE employee_id=143
)
#案例3: 返回公司工资最少的员工信息
SELECT *
FROM employees
WHERE salary =(
	SELECT MIN(salary)
	FROM employees
);

#案例4:查询最低工资大于50号部门最低工资的部门id和其最低工资
SELECT department_id,MIN(salary)
FROM employees
WHERE department_id IS NOT NULL
GROUP BY department_id
HAVING MIN(salary) > (
	SELECT MIN(salary)
	FROM employees
	WHERE department_id = 50
);

#非法使用标量子查询

#2. 列子查询(多行子查询)
#案例1: 返回location_id是1400或1700的部门中所有的员工姓名

#内连接:
SELECT DISTINCT last_name
FROM employees AS e, departments AS d
WHERE d.`location_id` IN(1400, 1700)
	AND e.`department_id`=d.`department_id`;

#子查询
1.查询location_id是1400/1700的部门号
SELECT DISTINCT department_id
FROM departments
WHERE location_id IN(1400, 1700)

2.查询员工姓名,要求部门号是1列表中的某一个

SELECT last_name
FROM employees
WHERE department_id IN( #IN 与 = ANY类似
	SELECT DISTINCT department_id
	FROM departments
	WHERE location_id IN(1400, 1700)
)

#案例2: 返回其他部门中比job_id为'IT_PROG'部门任意工资低的员工的工号,姓名,job_id,salary
SELECT employee_id,last_name,job_id,salary
FROM employees
WHERE salary < (
	SELECT MAX(salary)
	FROM employees
	WHERE job_id IN('IT_PROG')
)
AND job_id NOT IN('IT_PROG')

#3. 行子查询(一行多列/多行多列)
#案例: 查询员工编号最小中工资最高的员工信息
SELECT *
FROM employees
WHERE employee_id=(
	SELECT MIN(employee_id)
	FROM employees
) AND salary=(
	SELECT MAX(salary)
	FROM employees
)

#行子查询法:
SELECT *
FROM employees
WHERE (employee_id,salary)=(
	SELECT MIN(employee_id), MAX(salary)
	FROM employees
)

#二. select后面
#案例: 查询每个部门的员工个数

#方法1:分组(无法查出没有员工的department)
SELECT department_id, COUNT(*)
FROM employees e
GROUP BY department_id

#方法2:select子查询 只能是一行一列
SELECT *,(
	SELECT COUNT(*)
	FROM employees AS e
	WHERE e.department_id = d.department_id
) AS 个数
FROM departments d;

#案例: 查询员工号=102的部门名
SELECT department_name
FROM departments d, employees e
WHERE e.`employee_id`=102
	AND e.`department_id`=d.`department_id`;
	
#法2:
SELECT (
	SELECT department_name
	FROM departments d
	INNER JOIN employees e
	ON d.department_id=e.department_id
	WHERE e.employee_id=102
)AS 部门

#三. from后面
#将子查询结果充当一张表,要求必须起别名

#案例: 查询每个部门的平均工资的工资等级

SELECT avg_dep.*, grade_level
FROM(
	SELECT department_id, AVG(salary) AS sal
	FROM employees
	GROUP BY department_id
) AS avg_dep
INNER JOIN job_grades AS jo
ON avg_dep.sal BETWEEN lowest_sal AND highest_sal

#四. exists后面(相关子查询)
/*
exists(完整的查询语句),返回1(存在),0(不存在)
一般都可以代替
*/
SELECT EXISTS(SELECT employee_id FROM employees) AS isExit

#案例1:查询有员工的部门名
SELECT department_name
FROM departments d
#法一:exists
WHERE EXISTS(
	SELECT *
	FROM employees e
	WHERE e.department_id = d.department_id
)
#法二:in
WHERE d.department_id IN(
	SELECT department_id
	FROM employees
)

#案例2:查询没有女朋友的男神信息

#法一:in
SELECT boyName
FROM boys b
WHERE b.id NOT IN(
	SELECT boyfriend_id
	FROM beauty g
)

#法二:exists
SELECT boyName
FROM boys b
WHERE NOT EXISTS(
	SELECT boyfriend_id
	FROM beauty g
	WHERE b.id=g.boyfriend_id
)


#1.查询和Zlotkey相同部门的员工姓名和工资
SELECT last_name,salary
FROM employees
WHERE department_id=(
	SELECT department_id
	FROM employees
	WHERE last_name='Zlotkey'
)
#2.查询工资比公司平均工资高的员工的员工号,姓名和工资
SELECT employee_id,last_name,salary
FROM employees
WHERE salary>(
	SELECT AVG(salary)
	FROM employees
)
#3.查询各部门中工资比本部门平均工资高的员工的员工号,姓名,工资
#my step
SELECT department_id,last_name,salary
FROM employees e1
WHERE salary>(
	SELECT AVG(salary)
	FROM employees e2
	WHERE e1.department_id=e2.department_id
)
ORDER BY department_id

#step one:查询各部门平均工资
SELECT department_id,AVG(salary)
FROM employees
GROUP BY department_id

#step two:
SELECT e1.department_id,employee_id,last_name,salary
FROM employees e1
INNER JOIN(
	SELECT department_id,AVG(salary) ag
	FROM employees
	GROUP BY department_id
) a1
WHERE e1.department_id=a1.department_id
AND e1.salary>a1.ag
ORDER BY department_id
#4.查询和姓名中包含字母u的员工在相同部门的员工的员工号和姓名
SELECT employee_id,last_name
FROM employees
WHERE department_id IN(
	SELECT department_id
	FROM employees e
	WHERE e.last_name LIKE('%u%')
)

#5.查询在部门的location_id为1700的部门工作的员工的员工号
SELECT employee_id
FROM employees e
WHERE e.department_id IN(
	SELECT department_id
	FROM departments
	WHERE location_id=1700
)

#6.查询管理者是king的员工姓名和工资
#用in更普适(若结果表中包含超过一个值)
SELECT last_name,salary
FROM employees
WHERE manager_id IN(
	SELECT employee_id
	FROM employees e
	WHERE e.last_name='K_ing'
)

#7.查询工资最高的员工的姓名,要求first_name和last_name显示为一列,列明为姓,名
SELECT CONCAT(last_name,' ',first_name) AS 姓.名
FROM employees e
WHERE e.salary=(
	SELECT MAX(salary)
	FROM employees
)
```

#### 1.1.7 联合查询

- 应用场景：当要查询的结果来自于多个表且表间无直接连接关系但**查询信息一致**时使用。
- 语句：union 将多条查询语句合并为一个结果(同结构)
- 特点：
  1. 要求多条查询语句查询列数一致
  2. 多条查询语句查询的每列类型与顺序最好一致
  3. union关键字**自动去重**(union all不会)

```mysql
#引入案例: 查询部门编号>90 || 邮箱号包含a的员工信息
SELECT *
FROM employees
WHERE department_id>90
UNION#合并查询
SELECT *
FROM employees
WHERE email LIKE '%a%'
```



#### 1.1.8 分页查询

- 应用场景：当要显式的数据一页无法显式完全时，需分页提交sql请求
- 语句：limit offset,size; 限制范围
  - (offset要显示条目的起始索引[从0开始],size要显示的条目个数)
- 特点：
  1. limit语句置于查询语句最后
  2. 表示size的公式：(显式页数为page，每页条目数为size)：
     - limit (page-1)*size, sizew



```mysql
#案例:查询有奖金的,工资较高的前10名员工信息
SELECT *
FROM employees
WHERE commission_pct IS NOT NULL
ORDER BY salary DESC
LIMIT 0,10;
```



#### 1.1.9 查询语句执行顺序总结

```mysql
	select 查询列表      ---7  #显示结果
	from 表             ---1 #先检索表
	[join...]	        ---2 #再检测连接顺序
	on 连接条件          ---3  #检测连接条件
	where ...           ---4  #检测筛选条件
	group by...         ---5  #分组
	having ...          ---6  #分组后筛选
	order by...         ---8  #排序
    limit offset,size;  ---最后 #分页
```



### 1.2 insert into - 插入

- 语法：
  1. `insert into 表名(列名...) values(值1,...);`
  2. `INSERT INTO beauty
     SET id=14,NAME='三上悠亚',phone='13412412414'`
- 要求:
  1. 插入的值的类型与列的数据类型一致/兼容
  2. 可以插入null(直接插null或不填列名)
  3. 列的顺序可以调换
  4. 列数和值个数必须一致
  5. **可以省略列名，此时默认为所有列且复制顺序与表中列顺序一致**
- 方式1与方式2比较：
  1. 方式1支持插入多行，方式2不支持`INSERT INTO beauty(id, NAME, sex, borndate, phone,photo,boyfriend_id)
     VALUES(13,'蔡徐坤','女','1990.4.23','1342412442',NULL,2),
     (13,'蔡徐坤','女','1990.4.23','1342412442',NULL,2),
     (13,'蔡徐坤','女','1990.4.23','1342412442',NULL,2),`
  2. 方式1支持子查询，方式2不支持`INSERT INTO beauty(id,NAME,phone)
     SELECT 15,'宋茜','1412412412'`



方式1示例：

```mysql
INSERT INTO beauty(id, NAME, sex, borndate, phone,photo,boyfriend_id)
VALUES(13,'蔡徐坤','女','1990.4.23','1342412442',NULL,2);
```

方式2示例:

```mysql
INSERT INTO beauty
SET id=14,NAME='三上悠亚',phone='13412412414'
```



### 1.3 update - 修改

- 语法：`update 表名
  set 列=新置,...(类型一致/兼容)
  where 筛选条件;`

- 修改多表条记录：

  1. sql92：

     `update 表名
     set 列=新置,...(类型一致/兼容)
     where 筛选条件;`

  2. sql99:

     `update 表1 别名
     inner|left|right join 表2 别名
     on 连接条件
     set 列=值,...
     where 筛选条件;`



修改示例:

```mysql
#案例1:修改beauty表中蔡徐坤性别改成男
UPDATE beauty
SET sex='男'
WHERE id=13
#案例2:修改boys鹿晗改为张飞,魅力值改为10
UPDATE boys
SET boyName='张飞',userCP=10
WHERE boyName LIKE'%鹿%'

#案例3:修改张无忌的女朋友手机号为114
UPDATE beauty g
INNER JOIN boys b
ON g.boyfriend_id=b.id
SET g.phone='144'
WHERE b.boyName='张无忌'

#案例4:修改没有男朋友的女神的男朋友编号都为张飞的编号(2)
UPDATE beauty g
LEFT JOIN boys b
ON g.boyfriend_id=b.id
SET g.boyfriend_id=(
	SELECT id
	FROM boys
	WHERE boyName='张飞'
)
WHERE b.id IS NULL

```

 

### 1.4 delete - 删除

- **纠正错误 - delete也可以指定删除的表项 - delete *table* from ... **

#### 1.4.1 delete 

1. 单表删除：`delete from 表明 where 筛选条件`

2. 多表删除

   1. sql92:

      `delete (表1/表2)别名
      from 表1 别名,表2 别名
      where 连接条件
      and 筛选条件;`

   2. sql99:

      `delete (表1/表2)别名
      from 表1 别名,表2 别名
      where 连接条件
      and 筛选条件;`

delete语句示例:

```mysql
#1.单表删除
#案例1:删除手机编号最后一位为9的女神
DELETE g
FROM beauty g
WHERE phone LIKE '%9'

#2.多表删除
#案例:删除张无忌的女朋友的信息
DELETE g
FROM beauty g
INNER JOIN boys b
ON g.boyfriend_id =(
	SELECT id
	FROM boys
	WHERE boyName='张无忌'
)

#案例:删除段誉以及其女朋友信息
DELETE g,b
FROM beauty g
INNER JOIN boys b
ON g.boyfriend_id=b.id
WHERE b.boyName='段誉'
```



#### 1.4.2 truncate

- 语法：truncate table 表名；-删除整表



- delete 与 truncate比较
  1. delete可以加where条件,truncate不可
  2. truncate删除效率高(整表)
  3. 若要删除的表中有自增长列,
     若用delete删除后再插入数据,自增长列的值从断点开始
     若用truncate删除后再插入数据,自增长列值从1开始
  4. truncate删除无返回值，delete删除返回影响行数
  5. truncate不可回滚，delete可回滚
  6. 2. truncate删除效率高(整表)
        3. 若要删除的表中有自增长列,
           若用delete删除后再插入数据,自增长列的值从断点开始
           若用truncate删除后再插入数据,自增长列值从1开始
           4.truncate删除无返回值,delete删除有返回值(受影响的行数)
           5.truncate删除不能回滚,delete删除可以回滚



## 2. 一些常见函数

#### 2.1 字符函数

1. length()：查询获取参数值字节个数(汉语字符根据字符集的不同所代表的字节数也有不同)
2. concat()： 字符串拼接函数，一般用于拼接查询列表
3. upper(),lower()： 将给定字符全部转为大/小写
4. substr()： 截取给定位置字符，有四种重载
   - 注：**MySQL中索引从1开始而非0**
5. instr(a, b)： 返回子串(b)在大字符串(a)中的起始索引，若无则返回0
6. trim()： 去除字符前后空格(默认)或其他指定字符或字符串(x from y)
   - eg. SELECT TRIM('a' FROM 'aa我爱你aa非常的aaaa');
7. lpad / rpad (a, str, b) ：用指定字符实现左/右填充至指定长度
8. replace(a, str, b)： 用b替换str中的a

#### 2.2 数学函数

1. round(a, (b))： 四舍五入, -保留指定(b)位数
2. cell / floor()： 向上/向下取整 - 返回结果大于等于 / 小于等于该参数的最小整数
3. truncate()： 截断， 截取小数点后几位 (无取舍)
4. mod(a, b)： 取余 : **a - a / b * b**

#### 2.3 日期函数

1. now()： 返回当前系统日期与实践
2. curdate()： 返回当前系统日期
3. curtime()： 返回当前实践
4. hour(time)： 获取给定time的小时数
5. **str_to_date(将日期格式字符转换成指定格式日期)**
   - %Y 四位年份
     %y 两位年份
     %M 英文月份
     %m 两位月份
     %c 一位(可能)月份
     %d 日
     %H 小时(24)
     %h 小时(12)
     %i 分钟
     %s 秒
   - eg.  SELECT *
         FROM employees
         WHERE hiredate = STR_TO_DATE('4-3 1992','%m-%d %Y');
6. data_format(time, str): 将日期time转换成字符str
   - eg. SELECT DATE_FORMAT(NOW(),'%y年%M月%d日');

#### 2.4 *分组函数*

sum(), avg(), max(), min(), count()

- 分组函数一般用作统计使用，一般用作分组判断依据
  1. sum, avg一般用于处理数值型
  2. max, min, count可处理任何数据类型
  3. 可与distinct搭配实现去重
  4. 与分组函数统一查询字段要求：查询所得字段列数相同

#### 2.5 其他函数

1. VERSION();
2. DATABASE();
3. USER();



## 3.DDL(Data Define Language) 数据库定义语言

### 3.1 create-表的创建

语法：

```mysql
create table 表名(
	列名 列的类型[长度] [列的约束],
	列名 列的类型[长度] [列的约束],
	...
)
```



eg.创建表Book

```mysql
CREATE TABLE Book(
	id INT,#图书编号
	bName VARCHAR(20),#图书名
	authorId INT,
	price DOUBLE,
	publishDate DATETIME
);
```



### 3.2 alter-表的修改

#### 3.2.1 修改列名-change

- change中的COLUMN关键字可加可不加，*最好要加*
- change最后需添加关于本列的类型



eg.修改book列名publishdate -> pubDate

```mysql
ALTER TABLE book #colume可省
CHANGE COLUMN publishdate pubDate DATETIME;
```



#### 3.2.2 修改列的类型/约束-modify



eg.将pubDate改为varchar

```mysql
ALTER TABLE book
MODIFY COLUMN pubDate VARCHAR(20);
```



#### 3.2.3 添加/删除列-add/drop

- 添加的列需跟该列的类型

```mysql
ALTER TABLE author
ADD COLUMN annual DOUBLE;


ALTER TABLE author
DROP COLUMN annual;
```

#### 3.2.4 修改表名-rename to

```mysql
ALTER TABLE author1
RENAME TO author;
```

### 3.3 表的删除

#### 3.3.1 - drop table

- if exists用于判断表是否存在，避免非空情况

```mysql
DROP TABLE IF EXISTS newa;
```

#### 

### 3.4 表的复制

#### 3.4.1 仅复制表的结构- copy like

```mysql
CREATE TABLE copy LIKE author;
```

#### 3.4.2 复制表的结构与数据

- 选中数据进行复制

```mysql
CREATE TABLE copy2
SELECT * FROM author;
```



#### 3.4.3 只复制部分数据

- 类似于3.4.2 选中再复制

```mysql
CREATE TABLE IF NOT EXISTS copy3
SELECT id,aName
FROM author
WHERE country='China';
```

#### 3.4.4 仅复制某些字段

- 判断条件：where false - 只复制字段

```mysql
CREATE TABLE copy4
SELECT id,aName
FROM author
WHERE FALSE;
```

## 4. 常见数据类型

### 4.1 数值型

- 使用原则：选择的类型越简单越好,能保存数值的类型越小越好

#### 4.1.1 整形

- 设置无符号,有符号: 默认有符号,否则在数据类型后添加关键字 unsigned
- 值超出范围则传入临界值(报out of range异常)
- 不设置长度会有默认值(int 11),可以设置zerofill填充多余位为0



1. Tinyint : 1字节
2. Smallint : 2字节
3. Mediumint : 3字节
4. Int， integer : 4字节
5. Bigint : 8字节



#### 4.1.2 浮点型

- M为数字范围,D为小数范围.若超过范围则插入临界值
- 默认情况float,double根据数据调整范围,decimal的M 10;D 0
- 定点型精确度较高,若要求插入数值精度较高(货币运算)使用定点型

1. 定点数
   1. float(M,D) 4字节
   2. double(M,D) 8字节
2. 浮点数
   1. DEC(M,D)
   2. DESC(M,D)



### 4.2 文本型

1. char(M) ：最多字符数M,**固定长度符**,耗费空间,效率较高,适用于固定长度值(sex)
2. varchar(M) : 最多字符数M,**可变长度符**,动态改变空间,效率较低,适用于可变长度值(name)
3. text 较长的文本
4. blob  较长的文本 -多用于存储二进制数据(图片)



### 4.3 日期型

1. date: 4字节 保存日期 (1000-01-01, 9999-12-31)
2. datetime: 8字节 保存日期加时间 范围(1000-9999)
3. timestamp: 4字节 保存日期加时间 范围(1970-2038时区影响)
4. time: 3字节 
5. year: 1字节 (2021)

### 4.4 枚举型 

- 即一列的每项的值仅可能为枚举中的一个(ENUM)或多个(SET)值

1. ENUM: 只可插入一个枚举
2. set：可插入多个枚举集合



```mysql
create table tab_char(
	c1 set('a','b','c'),
	c2 set("aa", "bb")
);
```



## 5. 约束

- 约束：一种限制，**用于限制表中数据，为保证表中的数据的准确和可靠性**
- 查看表的索引: `show index from 表;`



### 5.1 六大约束：

1. not null : 非空约束,保证字段值不为空

2. default : 默认约束,保证字段有默认值

3. primary key : 主键约束,保证该字段值唯一且非空

4. unique : 唯一约束,保证该字段值唯一可空(控制亦唯一)

5. check : 检查约束**(mysql不支持)**

6. foreign key : 外键约束

   - 作用:用于限制两个表的关系,保证该字段值必须来自于主表的关联列的值;
   - **从表的外键列类型与主表的关联列类型一致/兼容**

   - 用法:在从表添加外键约束,用于引用主表中某列的值.(专业编号,员工表部门编号,工种编号)

   - 限制:主表的对应列必须为**主键/唯一**
   - 插入数据时应先插入主表数据,再插从表;删除数据时先删从表再删主表



### 5.2 分类

1. 列级约束: (符合要求的可在一列中添加多个约束,以空格间隔)
   - 六大约束语法上都支持,但外键约束无效果
2. 表级约束: (**除了非空,默认**,其他的都支持)



### 5.3 用法

##### 1. 创建表时添加列级约束:

```mysql
CREATE TABLE IF NOT EXISTS stuifo(
	id INT PRIMARY KEY, #主键约束
	stuName VARCHAR(20) NOT NULL, #非空约束
	gender CHAR(1) CHECK(gender='男' OR gender='女'), #检查约束
	seat INT UNIQUE, #唯一约束 (可以为null,但也只能有一个null)
	age INT DEFAULT 18, #默认约束
	bookId INT REFERENCES book(id) #外键约束 列级约束中其无效果
);
```



##### 2. 创建表时添加表级约束:

- constraint + 约束修饰名 + 约束类型 + [外键: FOREIGN KEY(本表关联列名) + REFERENCES + 外表(外表被关联列名)]
- **非空与默认不支持表级约束**

```mysql
CREATE TABLE IF NOT EXISTS stuinfo1(
	id INT,
	stuName VARCHAR(20),
	gender CHAR(1),
	seat INT,
	age INT,
	bookId INT,
	
	CONSTRAINT pk PRIMARY KEY (id),
	CONSTRAINT uq UNIQUE(seat),
	CONSTRAINT rf_bookid FOREIGN KEY(bookId) REFERENCES author(aid)
	
);
```

##### 3. 修改表时增加/修改约束

- 应该为一次只能修改一行

```mysql
#列级约束 - 列的后面
ALTER TABLE stuinfo2
MODIFY COLUMN seat INT UNIQUE; 

#表级约束 - 表的后面
ALTER TABLE stuinfo2
ADD UNIQUE(seat);

#表级约束- 添加外键
ALTER TABLE beauty
ADD CONSTRAINT girl_boy FOREIGN KEY(boyfriend_id) REFERENCES boys(id)

```



##### 4. 修改表时删除约束

1. 删除主键 - drop primary key; (主键唯一,不需跟列名)
2. 删除唯一 - drop index 列名;
3. 删除外键 - drop foreign key 外键的修饰名
4. 删除非空 - modify column ... (null)
5. 删除默认 - modify column ...(null)



eg.创建表stuinfo2关联表author(tid->aid),删除表中所有约束

```mysql
#创建
DROP TABLE IF EXISTS stuinfo2;
CREATE TABLE IF NOT EXISTS stuinfo2(
	id INT PRIMARY KEY AUTO_INCREMENT,
	sex ENUM("man", "woman") NOT NULL,
	age INT DEFAULT 18,
	seat INT UNIQUE,
	tid INT,
	
	CONSTRAINT stu_tea FOREIGN KEY(tid) REFERENCES author(aid)

);

#删除非空
ALTER TABLE stuinfo2
MODIFY COLUMN sex ENUM("man", "woman") NULL;
#删除默认
ALTER TABLE stuinfo2
MODIFY COLUMN age int;
#删除主键
alter table stuinfo2
drop primary key;
#删除唯一
alter table stuinfo2
drop index seat;
#删除外键
alter table stuinfo2
drop foreign key stu_tea;
```

### 5.4 级联删除/置空

- 若删除主表中的数据有从表的关联值则主表数据删除失败，报错。
- 基于此希望可以绕过从表进行主表的删除



#### 5.4.1 级联删除

- 删除主表记录时同时将从表记录删除
- on delete cascade

```mysql
ALTER TABLE stuinfo2
ADD CONSTRAINT cr_stuinfo2_author FOREIGN KEY(major_id) REFERENCES author(aid) ON DELETE CASCADE;

```

#### 5.4.2 级联置空

- 删除主表记录的同时将从表相关联的值置(外键列)为null
- ON DELETE SET NULL;

### 5.5 标识列/自增长列

- 不手动插入值,系统提供默认序列值(增长)
- 标识列必须和键搭配(主键/唯一/外键)
- 一个表之多有一个标识列
- 标识列只可设置为数值型
- 标识列可以通过**set auto_increment_increment**改步长,也可通过设置更改初始值



1. 创建表时设置标识列
2. 修改表时删除标识列
   - 类似于默认和非空的删除方式



## 6.TCL(Transaction Control Language) 事务控制语言

- 事务：一个或一组sql语句组成的一组逻辑执行单元
  - 这一组逻辑单元要么都执行要么都不执行

### 6.1 ACID属性

1. 原子性Actomicity：事务处理的过程不可分割，要么都发生要么都不发生
2. 一致性Consistency：事务处理前后数据库状态都处于一个一致性状态，操作前和操作后的总量一般保持不变。
   - 数据库系统一致性：数据库系统保障，事务执行后数据库中数据符合各种约束条件。
   - 逻辑一致性：应用层面保障。
3. 隔离性Isolation：多事务操作时事物之间互相不会产生印象
4. 持久性Durability：事务提交后表中数据发生的变化是永久的。



### 6.2 事务读取问题

对于两个事务t1,t2

1. 脏读： t1读取到了t2已修改但未提交的数据
2. 不可重复读：t1读取到了t2已修改已提交的数据
3. 幻读：t1读取到了t2在该表中插入的一些新数据
   - t1在t2插入前后读的数据内容不同-幻读

### 6.3 事务的隔离级别

1. read uncommitted(读未提交)
2. read committed(读已提交)
3. **repeatable read(可重复读) - InnoDB默认**
4. serializable(串行化)



### 6.4 事务的具体操作

1. set autocommit=0; 取消自动提交
2. 可省 start transaction; 开启事务
3. 编写事务sql语句
4. savepoint x设置rollback点
5. commit/rollback(to x) 提交/回滚(完全/x之后)



eg:

```mysql
SET autocommit=0;
START TRANSACTION;
DELETE FROM dept2 WHERE department_id = 50;
SAVEPOINT a; #设置保存点 只搭配rollback使用
DELETE FROM dept2 WHERE department_id = 60;
ROLLBACK TO a;
```



## 7 视图

- 含义:虚拟表,和普通表一样使用,只保存sql逻辑不保存查询结果(封装 )-一般应用于一些需临时调用的数据
  - mysql5.0.1出现新特性,是通过表动态生成的数据



- 优点:
  1. 实现sql语句重用
  2. 简化复杂的sql操作
  3. 保护数据,提高安全性(不使用基表)



|       | 创建语法       | 占用时机物理空间   | 一般操作           |
| ----- | -------------- | ------------------ | ------------------ |
| view  | create view as | 不占用(只保存逻辑) | 增删改查(一般只查) |
| table | create table   | 占用时机空间       | 增删改查           |



### 7.1 视图创建



eg.创建查询视图

```mysql
CREATE VIEW v1
AS
SELECT last_name,employee_id
FROM employees
WHERE last_name LIKE '%z%';

SELECT * FROM v1;
```



### 7.2 视图修改



1. create or replace view 视图名
   as
   查询语句;
2. alter view 视图名
   as
   查询语句;



eg.修改v1

```mysql
ALTER VIEW v1
AS
SELECT * FROM employees;

SELECT * FROM v1;
```



### 7.3 视图删除

```mysql
DROP VIEW v1;
```



### 7.4 查看视图

1. desc v1
2. show create view 视图;



### 7.5 视图更新(增删改)

- 注:**视图更新会影响到对应基表**,一般而言视图会设置为只读
- 具备如下特点的视图不允许更新:
  1. group by, distinct, having, union(all)
  2. 常量视图
  3. select中包含子查询
  4. join
  5. from一个不能更新的视图
  6. where子句子查询引用了from子句的表



## 8. 变量

变量分类如下:

1. 系统变量
   1. 全局变量:	作用域：服务器每次启动将为所有全局变量赋初始值，针对所有的会话（连接）有效，但不能跨重启（修改配置文件可以更改初始值）
   2. 会话变量: 作用域: 仅仅针对于当前会话（连接）有效
2. 自定义变量
   1. 用户变量
   2. 局部变量



### 8.1 系统变量

- 说明:系统变量由系统提供而非用户定义,属于服务器层面
- 注意:若全局变量需添加global,(默认)会话变量需添加session



使用实例

```mysql
#1.查看所有的系统变量
#show global|[session] variables;
#2.查看满足条件的部分系统变量

#show global|[session] variables like '%char%';

#3.查看指定的某个系统变量的值

#select @@global|[session].系统变量名；
SELECT@@transaction_isolation;

#4.为某个系统变量赋值

#1): set global|[session] 系统变量名=值； mysql8.0不支持
SET GLOBAL transaction_isolation='read-committed';

SET transaction_isolation='read-committed';

#2): set @@global|[session].系统变量名=值
SET @@session.transaction_isolation='repeatable-read';
```





### 8.2 自定义变量

使用步骤:声明,赋值,使用(查看,比较,运算)

1. 声明
   1. set @用户变量名=值
   2. set @用户变量名:=值
   3. select @用户变量名:=值
   4. 局部变量:declare 变量名 类型 值;
2. 赋值
   1. 同初始化
   2. `SELECT COUNT(*) INTO @count
      FROM author;`
3. 查看用户变量值
   1. select @count;



应用范围:

1. 用户变量: 可应用至任何地方
2. 局部变量: 只可应用于begin end里



使用示例:

```mysql
#1.用户变量
SET @num1=22;
SET @num2:=23;
SELECT @num3:=@num1+@num2;
SELECT @num3;

#2.局部变量 错误用例（只能置于begin end中）
DECLARE num1 INT DEFAULT 22;
DECLARE num2 INT DEFAULT 23;
DECLARE SUM INT;
SET SUM = num1 + num2;
SELECT SUM;
```



## 9. 存储过程-procedure

- 含义:一组预先编译好的sql语句集合,可理解为批处理语句
- 好处:
  1. 提高代码重用性
  2. 简化操作
  3. 减少编译次数和数据库服务器连接次数,提高效率



### 9.1 创建&调用

```mysql
create procedure 存储过程名(参数列表)
begin
	存储过体(一组合法的SQL语句)
END
```

1. 参数列表包含三部分：参数模式 参数名 参数类型
   - 参数模式：
     - IN：该参数可以作为输入，也就是该参数需要调用方传入值 	 
     - OUT：该参数可以作为输出，也就是该参数可以作为返回
     - INOUT：该参数既可以作为输入又可以作为输出（既需要传入值又可以返回值）	  

2. 若存储过程体仅仅只有一句话,begin end可以省略
3. 存储过程体中的每条SQL语句的结尾要求必须加分号

4. 存储过程的结尾可以使用 delimiter 重新设置(delimiter 结束标记)
5. 调用:call 存储过程名();

eg-五个实例

```mysql
#案例1：插入到admin表中五条记录

#创建存储过程
DELIMITER $
CREATE PROCEDURE myp1()
BEGIN
	INSERT INTO admin(username, PASSWORD)
	VALUES('johna','1111'),
	('johnb','1111'),
	('johnc','1111'),
	('johnd','1111'),
	('johne','1111');
END$

#调用
CALL myp1();

#2.创建带in模式参数的存储过程

#案例2：创建存储过程实现 根据女神名，查询对应的男神信息

DROP PROCEDURE IF EXISTS myp2;
DELIMITER $
CREATE PROCEDURE myp2(IN beautyName VARCHAR(20))
BEGIN
	SELECT b.*
	FROM boys b
	RIGHT JOIN beauty g
	ON b.id=g.boyfriend_id
	WHERE g.name=beautyName;
END$

CALL myp2('岳灵珊');

#案例3：创建存储过程实现用户是否登陆成功

DROP PROCEDURE IF EXISTS myp3;
DELIMITER $
CREATE PROCEDURE myp3(IN username VARCHAR(10),IN PASSWORD VARCHAR(10))
BEGIN
	DECLARE result INT DEFAULT 0;#声明并初始化变量
	SELECT COUNT(*) INTO result
	FROM admin a
	WHERE a.username=username
	AND a.password=PASSWORD;
	
	SELECT IF(result>0,'登陆成功','密码错误！') '登录情况';#使用变量

END$

CALL myp3('john','0000');

#3.创建带out模式的存储过程
DROP PROCEDURE IF EXISTS myp4;
#案例1：根据女神名返回对应的男神信息
DELIMITER $
CREATE PROCEDURE myp4(IN beautyName VARCHAR(20),OUT id INT,OUT boyName VARCHAR(20), OUT userCP INT)
BEGIN
	SELECT b.id,b.boyName,b.userCp INTO id,boyName,userCp
	FROM boys b
	RIGHT JOIN beauty g
	ON b.id=g.boyfriend_id
	WHERE g.name=beautyName;
END $

SET @bsName='';
CALL myp4('柳岩',@id,@bsaName,@Cp);

SELECT @id;

#4. 创建带inout模式的存储过程
#案例1：传入a,b两个值，最终要求a和b都翻倍并返回

DROP PROCEDURE IF EXISTS myp5;
DELIMITER $
CREATE PROCEDURE myp5(INOUT a INT, INOUT b INT)
BEGIN
	SET a=a*2;
	SET b=b*2;
	

END $

SET @num1:=5;
SET @num2:=20;
CALL myp5(@num1,@num2);
SELECT @num1,@num2;

```

### 9.2 删除

- `DROP PROCEDURE 存储过程名`



### 9.3 查看存储过程信息

- `show create procedure 存储过程名`



综合案例:

```mysql
#案例5.传入女神名，返回女神名和男神名
DROP PROCEDURE IF EXISTS myp6;
DELIMITER $
CREATE PROCEDURE myp6(IN NAME VARCHAR(20),OUT girlName VARCHAR(20),OUT boyName VARCHAR(20))
BEGIN
	SELECT g.name,b.boyName INTO girlName, boyName
	FROM boys b
	RIGHT JOIN beauty g
	ON b.id=g.boyfriend_id
	WHERE g.name=NAME;
END $

CALL myp6('苍老师',@girl,@boy);
SELECT @girl,@boy;
```



## 10. 函数-function

- 存储过程：可以无返回也可以有1-多个返回，适用于批量插入，更新
- 函数：有且仅有一个返回，适于做处理数据后返回一个结果



### 10.1 创建&调用

```mysql
#创建
create function 函数名(参数列表) returns 返回类型
begin
	函数体
end

#调用
select 函数名(参数列表);

#查看
SHOW CREATE FUNCTION 函数名;

#删除
DROP FUNCTION 函数名;
```

1. 注意：
   - 参数列表： 包含两部分：参数名 参数类型
   - 函数体：肯定会有return语句

2. 函数体中仅有一句话，可以省略begin end
3. 使用delimiter语句设置结束标记



### 10.2 实例:

```mysql
#控制是否可以信任存储函数创建者
SET GLOBAL log_bin_trust_function_creators=TRUE;

DELIMITER $
CREATE FUNCTION myf1() RETURNS INT
BEGIN
	DECLARE num INT DEFAULT 0;
	SELECT COUNT(*) INTO num
	FROM employees;
	RETURN num;
	
END $

SET @num=myf1();
SELECT @num;
```



## 11.JDBC

Driver: 数据库驱动

Connection: 数据库连接

Statement: 数据库操作指令



```java
//Driver指定连接驱动
Driver driver = new com.mysql.cj.jdbc.Driver();
String url = "jdbc:mysql://localhost:3306/girls";

Properties info = new Properties();
info.setProperty("user", "root");
info.setProperty("password", "root");

//driver.connect指定数据库连接 url & user
Connection connection = driver.connect(url, info);

//statement指定数据库操作指令
String sql = "insert into admin(username, password)" +
    "values(?, ?)";
String name = UUID.randomUUID().toString().substring(0, 5);
String password = "12345";

PreparedStatement statement = connection.prepareStatement(sql);

statement.setObject(1, name);
statement.setObject(2, password);

statement.execute();
System.out.println("插入完成");
statement.close();
```



 PreparedStatement vs Statement

- 代码的可读性和可维护性。
- **PreparedStatement 能最大可能提高性能：**
  - DBServer会对**预编译**语句提供性能优化。因为预编译语句有可能被重复调用，所以<u>语句在被DBServer的编译器编译后的执行代码被缓存下来，那么下次调用时只要是相同的预编译语句就不需要编译，只要将参数直接传入编译过的语句执行代码中就会得到执行。</u>
  - 在statement语句中,即使是相同操作但因为数据内容不一样,所以整个语句本身不能匹配,没有缓存语句的意义.事实是没有数据库会对普通语句编译后的执行代码缓存。这样<u>每执行一次都要对传入的语句编译一次。</u>
  - (语法检查，语义检查，翻译成二进制命令，缓存)
- PreparedStatement 可以防止 SQL 注入 
  - 通过setObject方法指定占位符对应参数；而Statement只能使用字符串拼接。