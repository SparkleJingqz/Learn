

# 1.传参设置

- sql语句通常需要通过传参来设置具体执行条件
- 传入单个参数时，mybatis不会做特殊处理，此时传入的值任意都可
- 传入多个参数时，myBatis会做特殊处理 - 多个参数会被封装为一个map，#{}从map中取出指定key的值
  1. parm1 ~ parmN 传入参数
  2. 使用参数的索引传入参数 1~n
  3. 可通过@Param注解命名参数指定map的key



## 1.1 传入pojo

- 若多个参数恰好为业务逻辑的数据模型(根据emp的id、name、email等属性)，可直接传入对应pojo，#{属性名}
- 多个pojo时可使用pojo.属性名来具体指定哪个pojo

## 1.2 传入map

- 若多个参数不是业务模型数据，可指定对应map来获取，根据#{key}来获取对应属性值
- **推荐使用TO(Transfer Object)传输对象** - 建立模型设置变量

## 1.3 参数值的获取

1. #{} 以预编译的形式将参数设置到sql语句中,防止sql注入(类似于PreparedStatement)
   - 使用场景:where,select中指定参数
   - 可通过设置#{xxx, jdbcType=...}来改变默认情况下对数据的处理
2. ${}取出的值直接拼装到sql语句中,存在sql注入问题
   - 使用场景:分组,排序等原生sql不支持占位符的地方



# 2. 关联对象或者集合的查询

- employee对应一个department,一个department对应多个employee. 查询时希望获取对应的对象或者集合
- association 获取对应对象
- collection 获取对应集合

## 2.1 association

1. 级联赋值关联对象查询
2. 分布查询 - 两次查询,基于第一次查询的结果赋予第二次查询的条件(查询出员工deptId,根据其查询对应部门信息)
3. 基于分布查询,set属性设置lazy,aggerssiveLazy开启懒加载 - 得到使用时才加载的sql语句



级联查询

```xml
<resultMap id="EmpDepAso" type="com.example.bean.Employee">
    <!--注:列名使用别名最好-->
    <id column="eid" property="id"></id>
    <result column="last_name" property="lastName"></result>
    <result column="email" property="email"></result>
    <result column="gender" property="gender"></result>

    <!--association指定联合的javaBean对象
        javaType:指定这个属性对象的类型
    指定哪个属性为联合的对象
    -->
    <association property="department" javaType="com.example.bean.Department">
        <id column="did" property="id"></id>
        <result column="dept_name" property="deptName"></result>
    </association>
</resultMap>
```


分步查询:association指定select为对应Mapper中的查询方法

- property指定关联查询属性
- column指定本次查询提供给外部查询条件的参数

```xml
<association 
property="department"             select="com.example.dao.DepartmentMapper.getDeptById"
column="did">
</association>
```



## 2.2 collection

- 基于association的查询集合
- ofType指定集合中元素的属性
- 查询部门时同时获取查询员工的所有信息

级联查询

```xml
<!--connection嵌套结果集的方式,定义关联的集合类型元素封装规则-->
    <!--可能主要应用于外连接查询多个结果集-->
    <resultMap id="MyDept" type="com.example.bean.Department">
        <id column="did" property="id"></id>
        <result column="dept_name" property="deptName"></result>
        <!--定义集合类型的属性封装规则 - connection
            ofType: 指定集合里面的元素类型-->
        <collection property="emps" ofType="com.example.bean.Employee">
            <!--定义集合中元素的封装规则-->
            <id column="eid" property="id"/>
            <result column="last_name" property="lastName"/>
        </collection>
    </resultMap>
```



分布查询

- fetchType指定加载方式
- column指定关联本列查询对象
- select指定外部mapper.方法的查询方法

```xml
<collection
property="emps"                   select="com.example.dao.EmployeeMapperPlus.getEmpsByDeptId"
column="{randomid=id}"
fetchType="lazy">
</collection>
```
# 3. 动态sql

- 程序运行时动态执行的sql，而非先前已静态编译确定的sql



四个表达式：



## 1. if 判断条件

- 先过滤信息, 符合信息进行where判断,若不符合直接跳过
- test表达式遵循OGNL规则， 从参数进行取值判断, 遇见特殊符号应写转义字符  ('')



**用法:**

```mysql
<if test="lastName!=null and lastName!=''">
    last_name like #{lastName}
</if>
```



## 2. where 封装查询条件

if中会有问题,and置于前列时 (and last_name like ***) 若and之前的语句判断失效,则第一个遍历的会出现 where and 这一错误结构

`select * from employee where and`

基于1.问题 处理方法有如下2中:

1. where 1=1 后面的条件都会有 and
2. mybatis的where标签将所有查询条件保存在内,会将where内的if首个and或者or去除
   - where只会去除第一个多出来的and, or 若and,or置于后面会有问题



## 3. set 封装修改条件

类似于where,在修改中一定会用到set,此时会前撤最后一个,仍存在的问题

`update table set ***,`

解决方法: 用set标签,会将最后一个,自动去除

## 4. trim标签,添加/去除整体拼串的前/后字符串

where仍可能存在问题,例如and置于每次查询的后面而非前面则where无法去除

`select * from employee where .... and`

基于该问题提供trim标签

- trim标签四个属性:
  1. prefix: 前缀，trim标签体中是整个字符串拼串后的结果， prefix给拼串后整个字符串加个前缀
  2. prefixoverrides: 前缀覆盖，去掉整个字符串前面多余的字符
  3. suffix：给拼串后整个字符串加一个后缀
  4. suffixOverrides：覆盖整个字符串后面多余的字符



**用法: - 添加前缀where表示筛选, 取消后缀and避免错误**

```mysql
<trim prefix="where"
      prefixOverrides=""
      suffix=""
      suffixOverrides="and">
    <if test="id!=null">
        id=#{id} and
    </if>
    <if test="lastName!=null and lastName!=''">
        last_name like #{lastName} and
    </if>
    <if test="email!=null and email.trim()!=''">
        email like #{email} and
    </if>
    <if test="gender=='W' or gender=='M'">
        gender = #{gender} and
    </if>
</trim>
```

## 5. choose(when, otherwise) 分支选择

*场景导入:带了id就用id查,带了lastName就用lastName查, 只会进入其中一个查询条件*

1. when: 相当于if表达式
2. otherwise: when均不满足时进行的查询

**用法**

```mysql
<choose>
    <when test="id!=null">
        id=#{id}
    </when>
    <when test="lastName!=null">
        last_name like #{lastName}
    </when>
    <otherwise>
        gender = 'W'
    </otherwise>
</choose>
```



## 6. foreach 集合遍历

基`select * from table where id in()`,等遍历集合情况,使用foreach进行批量操作

1. collection:指定要遍历的集合
   - list类型的参数会特殊处理封装在map中,key即为list

2. item:将当前遍历出的元素赋值给指定的变量
3. #{变量名}就能取出变量的值即当前遍历出的元素
4. separator: 给每个元素之间的分隔符
5. open:所有结果拼接一个开始的字符
6. close:所有字符拼接一个结束的字符
7. index:索引
   1. 遍历list时index就是索引,item就是当前值
   2. 遍历map时index表示的就是map的key,itmp就是指map的value



**使用实例1 : 批量查询** -  查询id在指定集合中的员工 （类似于id in语句）

```xml
<select id="getEmpsByConditionForeach" resultType="com.example.bean.Employee">
    select * from tbl_employee where id in
        <foreach collection="d_list" item="item_id" separator=","
        open="(" close=")">
            #{item_id}
        </foreach>
</select>
```



**使用实例2：批量保存** - 根据list将Employee存储于数据库中（根据item指定对应属性 emp.xxx）

- 也可将insert语句置于foreach循环中，分隔符用‘；’ 实现多次执行sql插入 （需开启alloMultiQueries=true属性）

```xml
<!--批量保存，-->
<insert id="addEmps">
    insert into tbl_employee(last_name, gender, email, dept_id) values
    <foreach collection="empList" item="emp" separator=",">
        (#{emp.lastName},#{emp.gender},#{emp.email},#{emp.department.id})
    </foreach>
</insert>
```



## 7. bind标签

- 将OGBL表达式的值绑定到一个变量中,方便后来引用



场景:不想通过%e%进行对员工名的模糊查询，而是不使用% - 通过bind标签绑定来实现模糊查询

- bind标签将属性中lastName的值前后绑定%来实现模糊查询

```xml
<bind name="_lastname" value="'%'+lastName+'%'"/>
select * from tbl_employee where last_name like #{_lastname}
```

## 8. sql标签 - 抽取可重用

- sql抽取，将经常要查询的列名或者插入用的列名抽取出来方便引用。
- **结合include标签一起使用**，include还可以自定义一些property，sql标签内部就能使用自定义的属性 **$()**



```xml
<sql id="columnVal">
    last_name,gender,email,dept_id
</sql>

```

- include标签结合使用：*<include refid="columnVal"></include>*

```xml
<insert id="addEmps">
    insert into tbl_employee(<include refid="columnVal"></include>) values
    <foreach collection="empList" item="emp" separator=",">
        (#{emp.lastName},#{emp.gender},#{emp.email},#{emp.department.id})
    </foreach>
</insert>
```



## 9. myBatis两个内置参数

- 不仅可以将方法传递过来的参数用来判断、取值，也可使用myBatis提供的两个内置参数
  1. _parameter:代表整个参数
     - **单个参数:_parameter就是这个参数**
     - **多个参数:参数会封装为一个map,_parameter代表这个map**
  2. databaseId:若全局配置了DataBasedIdProvider标签,databaseId就代表当前数据库的别名

`last_name=#{_parameter.lastName},`