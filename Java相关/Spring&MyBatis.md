# MyBatis

#: 占位符，可进行预编译与类型匹配操作。

- select * from tablename where id = #{id};  #{id}->'12'->12
- 防止sql注入，替换时加上引号''

$: 占位符， 不进行数据类型匹配，直接替换

- 造成sql注入问题，eg password = ${pw}; 若pw=' ' or 1=1; 则password会匹配成功。
- 应用场景: 传入数据库对象(表名/排序列)



# Spring

## IOC - Inversion of Control

IOC： 将设计好的对象交给容器控制，由容器帮助查找及注入依赖的对象(反转)；而非在对象内部直接控制。

- 反转的是创建和查找对象依赖的控制权，由IOC容器进行注入组合对象**（体系结构灵活?）**  由容器帮对象去找相应的依赖对象并注入而非对象主动寻找 (别找我们，我们找你)

1. 创建主动类
2. 查看主动类是否有依赖对象需要注入
3. 创建需注入的依赖对象，将其注入到主动类
4. **由IOC容器管理依赖对象的生命周期。**



DI： 依赖注入。 **应用程序依赖于IOC容器，由IOC容器提供对象所需的外部资源，并由IOC容器将对象所需的外部依赖资源创建并注入**



反射实现： 无论是xml配置还是注解(@Component  @Repository,@Service,@Controller)形式，最终都是通过反射形式进行创建对象

- @Autowired： 按类型注入，若有多个类型则按名称 (多个类型可用时可通过@Qualifier匹配名称)
- @Resource : 按名称注入，若无名称则按类型注入 

1. Class.forName(类的完整路径)
2. newInstance()



### IOC有关注解

@Configuration注解的类为配置类，类似于配置文件；

#### 1. 给容器注册组件的方法

<u>@Value注解给Bean属性赋值，支持SpEL表达式#{} ${}</u>

- 使用${}注入配置文件值需在配置类使用@PropertySource(value="")设置配置文件路径

<u>@Conditional： 按照一定的条件进行判断，满足条件才注册bean，通过({ConditionImpl.class}) 判断是否传入值</u>

- Condition接口 **ConditionContext可获取ioc使用的bean工厂、类加载器、环境信息、bean定义的注册类等配置信息，根据其中某些值判断返回true./false**

*@Bean注解配置类中的return Bean方法*

- @Bean注解配置类中的方法，返回实例bean。bean ID为方法名或@Bean注解带名。
  - 默认singleton饿加载，**ketongguo @Lazy修改单例bean懒加载** **可通过@Scope指定prototype懒加载**

*@ComponentScan value指定包扫描路径*Controller/Service/Repository/Component注解的类加入bean工厂。

- 注解实例工厂AnnotationConfigApplicationContext传参配置类class对象，通过getBean()方法获取实例

*@Import(.class)： 注解于配置类，将对应类注册进工厂，名字默认为全类名。*

- 可通过实现ImportSelector并注入Impl来批量注册。

*FactoryBean实现类注册*

- getObject();  getObjectType();  isSingleton()
- **在配置类中通过@Bean注解获取FactoryBeanImpl的方法**将FactoryBeanImpl注册入工厂中，调用getBean方法时通过FactoryBeanImpl的getObject()方法获取实例bean
- **想获取工厂实例，getBean("&factory_name")**

#### 2. 依赖注入

*@Autowire：默认按类型注入；*

- 类型冲突时可指定@Qualifier("")名称注入
- required可指定true/false : 容器是否必须有组件

*@Resource：默认按名称注入；*

- 只有单一实例bean时不会冲突，多个实例bean时需指定正确的name

*Aware接口*：创建对象时调用接口规定方法注入相关Spring底层组件  eg.ApplicationContextAware

- xxxAware 有对应的xxAwareProcessor进行处理

原理:AutowiredAnnotationBeanPostProcessor

#### 3. bean生命周期配置  ***

1. 通过@Bean注解指定init-method和destroy-method方法指定初始化和销毁方法
2. 让Bean实现InitializingBean和DisposableBean方法执行初始化/销毁逻辑
   - destroy方法在bean容器关闭时调用；afterPropertiesSet()在bean初始化完成赋值完成后调用。 
3. Javax包下的@PostConstruct / @PreDestroy注解bean中方法来定义初始化和销毁方法
4. BeanPostProcessor接口 bean的后置处理器，在bean初始化前后进行一些处理工作。  **Impl需添加@Component注解** *postProcessBefore/AfterInitialization*
   - before: 在初始化工作(init-method)之前处理
   - after: 在(init-method)之后工作



Spring - bean生命周期概览

![20200804152257590](D:\Huawei Share\Screenshot\20200804152257590.png)



## AOP

三步执行：

1. 将业务逻辑组件与切面类都加入容器中，**@Aspect**指定切面类
2. 切面类中@PointCut; @Before...
   - 通过配置**@PointCut("execution(方法)")**注解指定切入点，增强注解只需指定@PointCut注解的方法即可
3. 配置类中开启注解Aop模式 @EnableAspectJAutoProxy



术语描述

1. 连接点JoinPoint : 可以被动态代理拦截目标类的方法 (可以被增强的方法)
2. 切入点PointCut : 拦截的连接点(真正被增强的方法)
3. 通知Advice : 对切入点增强的内容
   1. 前置通知@Before : 方法执行之前通知
   2. 后置通知@AfterReturning : 方法完成且正常返回时执行通知
   3. 环绕通知@Around : 方法调用前与调用后
   4. 异常通知@AfterThrowing : 方法运行出现异常后通知
   5. 最终通知@After : 方法执行后不考虑其结果通知
4. 切面Aspect：把通知应用到切入点的过程

执行顺序：

1. **@Around**
2. **@Before**
3. **@AfterReturning** / **@AfterThrowing**
4. **@After**
5. **@Around**

使用描述：

- @Component放入IOC容器；@Aspect生成代理对象

- @PointCut定义切入点，注解于代理对象方法上，此后代理对象方法名即可代表切入点，需定义value

  - ```java
    @Pointcut(value = "execution(* Spring5.AOP.AspJ.Jie.Usera.add(..))")
    public void pointdemo(){
    }
    
    //表示在方法执行之后执行(最终通知)
    @Order(0)
    @After(value = "execution(* Spring5.AOP.AspJ.Jie.Usera.add(..))")
    public void after1(){
        System.out.println("add after order-0");
    }
    
    
    
    //前置通知
    //@Before注解表示作为前置通知
    @Order(3)
    @Before(value = "pointdemo()")
    public void before(){
        System.out.println("add before order-3");
    }
    ```

    

- @Order()通知的优先级，越小越优先



## Trancational

给要回滚的事务表注@Trancational注解

**配置类@EnableTransactionManagement**



配置事务管理器管理事务 TransactionManager

- 管理数据源，控制数据源的连接。 **TransactionManager(dataSource())**



## MVC

MVC: 架构思想，将软件按照模型model，视图view与控制器controller来划分

模型mode：模型层，对数据进行处理 - JavaBean

- 实体类bean：存储业务数据 entity类
- 业务类bean： Dao, Service层对象。处理数据访问与业务逻辑。

视图View：与用户进行交互，展示数据。  html/jsp页面及其有关的前端内容

控制器Controller：接受客户端请求并进行响应，controller层



@RequestMapping注解 : 将请求与处理请求的控制器方法关联

- 支持ant风格路径： ?任意单个字符；*任意个字符；**任意层级目录
- 支持restful风格，通过占位符将url的地址转换为对应的请求`"/testRest/{id}/{username}"`
- 衍生的@get/postMapping对应method属性的get/post



获取请求参数的方法

1. request.getParameter("")
2. 控制器方法获取请求参数（**形参与请求参数同名**）
3. @RequestParam(value,required,defaultValue)注解于请求参数, value为指定为形参复制的请求参数名，required为是否必须传入此请求参数
4. 通过POJO获取请求参数，Pojo对应属性名需与请求参数一致



域

request域：生命周期为当前请求到响应结束。

- 获取域方法：request.set/getAttribute()

session域：声明周期为session生命周期，默认为浏览器打开到关闭，可手动设置。

- 获取session域方法：session.set/getAttribute()

application/setvletContext域：生命周期为服务器启动-关闭

- Servletcontext：set/getAttribute()



#### 跨域问题

出现原因: **浏览器的同源策略限制，限制一个域的js脚本与另一个域的内容交互**（同源：两个页面具有相同的协议http/https，主机host与端口号port）

解决策略：dataType : 'jsonp'