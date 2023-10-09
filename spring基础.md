- - ### 优势（为什么学）

    - 简化开发，降低企业级开发的复杂性
      - IOC
      - AOP
        - 事务处理
    - 框架整合，高效整合其他技术，提高企业级应用开发与运行效率
      - Mybatis
      - Mybatis-plus
      - Struts
      - Struts2
      - Hibernate
      - .....

    ### 怎么学

    - 学习spring框架设计思想
    - 学习基础操作，思考操作与思想的联系
    - 学习案例

    # spring Framework

    spring Framework是spring生态圈中最基本的项目，是其他项目的根基

    ### Core Container：核心容器

    - Beans
    - Core
    - Context
    - SqEl

    ### AOP：面向切面编程（在不改变原结构的目录下）

    ### Aspects：AOP思想实现

    ### Data Access：数据访问/Data Integration：数据集成

    - JDBC
    - ORM
    - OXM
    - JMS
    - Transactions

    ### Web：Web开发

    - WebSocket
    - Servlet
    - Web
    - Portlet

    ### Test：单元测试与集成测试

    # Core Container核心容器

    解决耦合度偏高问题：在使用对象时，在程序中不要主动使用new产生对象，转换为外部提供对象

    ## Ioc（Inversion of Control）控制反转

    `org.springframework.beans`和 `org.springframework.context` 这两个包是 IoC 实现的基础

    使用对象时，由主动new产生对象转换为**外部**提供对象，此过程中对象创建控制权由程序转移到**外部**，此思想为控制反转

    **spring对Ioc思想进行了实现**

    - spring提供了一个容器，称为**Ioc容器**，用来充当Ioc思想中的**外部**
    - Ioc容器负责对象的创建、初始化等一系列工作，被创建或被管理的对象在Ioc容器中统称为Bean

    **DI（Dependency Injection）依赖注入**

    在容器中建立bean与bean之间的依赖关系的整个过程被称为依赖注入

    > 目标：充分解耦
    >
    > - 使用Ioc容器管理bean（Ioc）
    > - 在Ioc容器内将有依赖关系的bean进行关系绑定（DI）

    实现效果：使用对象时不仅可以直接从Ioc容器中获取，并且获取到的bean已经绑定了所有的依赖

    #### 使用Ioc

    - 管理什么（Service和Dao）
    - 如何将被管理的对象告知Ioc容器（配置）
    - 被管理的对象交给Ioc容器，如何获取到Ioc容器（接口）
    - Ioc容器得到后，如何从容器中获取bean（接口方法）
    - 使用spring导入哪些坐标（pom.xml）

    1. pom.xml导入spring的依赖（spring-framework，spring-context）
    2. 配置spring.xml
    3. 创建bean（bean标签创建）
    4. 初始化Ioc容器：ApplicationContext xxx = new ClassPathXMLApplicationContext("配置文件名.xml");
    5. 获取bean，根据容器xxx.getBean（）获取

    #### 使用依赖注入

    - Ioc的具体实现方式

    ## Spring Bean

    Bean就是指那些被IOC容器管理的对象

    在xml里面通过配置元数据来配置

    ```xml
    <!-- Constructor-arg with 'value' attribute -->
    <bean id="..." class="...">
       <constructor-arg value="..."/>
    </bean>
    ```

    ### 实现方式（声明类）

    - `@Component`：可标注任意类为spring组件，如果一个Bean不知道属于哪个层，可以使用`@Component`注解标注
    - `@Repository`：对应持久层即Dao层，主要用于数据库相关操作
    - `@Service`：对应服务层，主要涉及一些复杂的逻辑，需要用到Dao层
    - `@controller`：对应SpringMVC控制层，主要用于接收用户请求并调用Service层返回数据给前端

    ### 实现方式（声明方法）

    #### @Autowired

    `@Autowired`属于spring内置的主机，默认的注入方式是`byType`（根据类型进行匹配），也就是会优先根据接口类型去匹配并注入Bean（接口的实现类），这种情况就导致当一个接口存在多个实现类的时候，`byType`这个方式就无法正确注入对象了，因为这个时候spring会同时找到多个满足条件的选择，默认情况下不知道选择哪一个

    此时就会变成`byName`（根据名称进行匹配）

    ##### @Qualifier：来显示指定名称

    ```java
    //假设现在SEService这个接口有两个实现类:SEServiceImpl1和SEServiceImpl2
    
    //报错,byName和byType都无法匹配到对象
    @Autowried
    private SEService seService;
    //成功
    @Autowried
    private SEService seServiceImpl1;
    //成功
    @Autowired
    @Qualifier(value = "seServiceImpl1")
    private SEService seService;
    ```

    #### @Resource

    基于JDK的注解，默认注入方式为`byName`，如果无法匹配到对应的Bean的话，注入方式就会变为`byType`

    如果仅指定name就是通过`byName`方式，如果仅指定type就是`byType`方式，如果都指定那么就是`byType`+`byName`

    ```java
    // 报错，byName 和 byType 都无法匹配到 bean
    @Resource
    private SmsService smsService;
    // 正确注入 SmsServiceImpl1 对象对应的 bean
    @Resource
    private SmsService smsServiceImpl1;
    // 正确注入 SmsServiceImpl1 对象对应的 bean（比较推荐这种方式）
    @Resource(name = "smsServiceImpl1")
    private SmsService smsService;
    ```

    #### @Inject

    ### @Autowired和@Resource的区别

    1. `@Autowired`是Spring提供的注解，`@Resource`是JDK提供的注解
    2. `@Autowired`的默认注入方式是byType，`@Resource`默认注入方式是byName
    3. 当一个接口存在多个类的情况下，`@Autowired`（通过`@Qualifier`注解显示指定）和`@Resource`（通过`name`属性显示指定）都需要通过名称才能正确匹配到对应的Bean
    4. `@Autowired`支持在构造函数、方法、字段和参数上使用，`@Resource`主要用于字段和方法上的注入，不支持在构造函数或参数上使用

    ### @Component和@Bean的区别

    1. `@Component`注解用于类，`@Bean`注解作用于方法
    2. `@Component`通常是通过类路径扫描来自动侦测以及自动装配到Spring容器中（可以使用`@ComponentScan`注解来自定义需要自动装配到Spring的bean容器中）,`@Bean`注解通过是我们在标有该注解的地方定义产生这个bean，`@Bean`告诉Spring这是个类的实例，当我需要用到他的时候在使用
    3. `@Bean`注解比`@Component`注解的自定义性更强，而且很多地方只能通过`@Bean`注解来注册bean，比如引入第三方库中的类需要装配到spring容器中时，只能通过`@Bean`来实现

    ### Bean的作用域

    - singleton：IoC容器中只有唯一的bean实例，Spring中bean默认都是单例的，是对单例设计模式的应用
    - prototype：每次获取都会创建一个新的bean实例，也就是说，连续getbean()会得到不同的bean实例
    - request（仅web可用）：每一次http请求都会产生一个新的bean（请求bean），该bean仅在当前http生效
    - session（仅web可用）：每一次来自新session1的http请求都会产生一个新的bean，仅在http session内有效
    - appilcation/global-session（仅web可用）：每个web应用在启动时创建一个Bean（应用bean），该bean仅在当前应用启动时间内有效
    - websocket（仅web）：每一次websocket会话产生一个新的bean
  
    #### 配置
  
    ```java
    //xml方式:
    <bean id="..." class="..." scope="singleton"></bean>
    //注解方式:
    @Bean
    @Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    public Test TsetPrototype(){
         return new Test();
    }
    ```
  
    ### Bean是线程安全的吗
  
    spring中的bean是否是线程安全取决于作用域和状态
  
    这里主要说明singleton和prototype
  
    #### singleton
  
    singleton作用域下，Ioc容器只有唯一的bean实例，可能存在资源竞争问题
  
    - 当前bean有状态（包含可变的成员变量的对象）存在线程安全问题
    - 当前bean无状态（没有定义可变的成员变量，如Dao，service）是线程安全的
  
    解决
  
    - 在bean中尽量避免定义可变的成员变量
    - 在类中定义`ThreadLocal`成员变量，将需要的可变成员变量保存在`ThreadLocal`中
  
    Bean的生命周期
  
    
  
    # AOP切面编程
  
    是对OOP（面向对象编程）的一种延续
  
    ## AOP和OOP基本介绍
  
    - AOP：面向切面编程，将横切关注点从核心业务逻辑中提取出来，形成一个切面，通过字节码操作或动态代理等技术，实现代码的复用和解耦，提高代码的可维护性和可扩展性
      - 横切关注点：多个类或对象的公共行为
        - 日志记录：自定义日志记录注解
        - 性能统计：利用AOP在目标方法的执行前后统计访问方法的执行时间，方便优化和分析
        - 事务管理：@Transactional注解可以让Spring为我们进行事务管理比如回滚异常操作
      - 切面（Aspect）：对横切关注点进行封装的类，一个切面是一个类，切面可以定义多个通知用来实现具体的功能
        - 切入点+通知
      - 连接点（JoinPoint）：目标对象所属类中定义的所有方法均为连接点
        - 切入点：方法调用或者方法执行的某个特定的时刻（如方法调用和异常抛出等）
      - 通知（Advice）：通知就是在切面在某个连接点时的具体操作，分为五种：前置通知，后置通知，返回通知，异常通知，环绕通知，前四种是在目标方法的前后执行，环绕通知可以控制目标方法的执行过程
        - Before（前置通知）：目标对象的方法调用前触发
        - After（后置通知）：目标对象的方法调用后触发
        - AfterReturning（返回通知）：目标对象的方法调用完成，在返回结果值之后触发
        - AfterThrowing（异常通知）：目标对象的方法运行中抛出/触发异常后触发
        - Around（环绕通知）：环绕通知是所有通知类型中可操作范围最大的一种，因为它可以直接拿到目标对象，以及要执行的方法，所以环绕通知可以任意的在目标对象的方法调用前后搞事，甚至不调用目标对象的方法
      - 切点（Pointcut）：一个切点是一个表达式，用来定义哪些连接点需要被切面所增强
      - 织入（Weaving）：织入是将切面和目标对象连接起来的过程，也就是将通知应用到切点匹配的连接点上，分为编译器织入和运行期织入
    - OOP：将业务逻辑按照对象的属性和行为进行封装，通过类，对象，继承，多态等方式实现代码的模块化和层次化提高代码的可读性和可维护性
  
    ## Spring AOP和AspectJ AOP有什么区别
  
    - Spring AOP属于运行时增强，而AspectJ是编译时增强。Spring AOP基于代理，而AspectJ基于字节码操作
    - Spring AOP已经集成了AspectJ，AspectJ是比较完整的AOP框架，相对于Spring AOP来说功能更加强大且更加简单
    - 切面太多时使用AspectJ，较少时无多大差别
  
    ## 多个切面如何控制顺序
  
    ### （1）使用@Oreder注解
  
    ```java
    //值越小优先级越高
    @Oreder(1)
    @Component
    @Aspect
    public class TestAspect implements Order{}
    ```
  
    ### （2）实现Ordered接口重写getOrder方法
  
    ```java
    @Component
    @Aspect
    public class Testspect implements Order {
        @Override
        public int getOrder() {
            // 返回值越小优先级越高
            return 1;
        }
    }
    ```
  
    # Spring MVC
  
    MVC的核心思想是通过将业务逻辑、数据、显示分离来组织代码
  
    - 模型（Model）
    - 视图（View）
    - 控制器（Controller）
  
    ## 组件
  
    - DispatcherServlet：核心的中央处理器，负责接收请求、分发，并给予客户端回应
    - HandlerMapping：处理器映射器，根据url去匹配能查找能处理的Handler，并会将请求设计到的拦截器和Handler一起封装
    - HandlerAdapter：处理器适配器，根据处理器映射器找到的Handler去执行适配的Handler
    - Handler：请求处理器，处理实际请求的处理器
    - ViewResolver：视图解析器，根据handler返回的逻辑视图/视图，解析并渲染真正的视图，并传递给中央处理器响应客户端
  
    ## 工作原理
  
    当客户端发起一个http请求
  
    1. DispatcherServlet拦截请求
    2. DispatcherServlet根据请求信息调用HandlerMapping
    3. HandlerMapping根据url去匹配查找能处理的Handler（也就是controller控制器），并将请求涉及到的拦截器和Handler一起封装
    4. DispatcherServlet调用HandlerAdapter执行Handler
    5. Hander处理完请求之后，会返回一个ModelAndView对象给DispatcherServlet
    6. ViewResolver会根据View查找实际的View
    7. DispatcherServlet吧返回的Model传给View
    8. 吧View返回给请求者
  
    ## 统一异常处理
  
    通常使用`@ControllerAdvice`+`@ExceptionHandler`注解来实现
  
    这种异常处理将所有或者指定的Controller织入异常处理的逻辑（AOP），当`Controller`中的方法抛出异常时，由被`@ExceptionHandler`注解修饰的方法进行处理
  
    `ExceptionHandlerMethodResolver` 中 `getMappedMethod` 方法决定了异常具体被哪个被 `@ExceptionHandler` 注解修饰的方法处理异常
  
    ```java
    @ControllerAdvice
    @ResponseBody
    public class GlobalExceptionHandler {
    
        @ExceptionHandler(BaseException.class)
        public ResponseEntity<?> handleAppException(BaseException ex, HttpServletRequest request) {
          //......
        }
    
        @ExceptionHandler(value = ResourceNotFoundException.class)
        public ResponseEntity<ErrorReponse> handleResourceNotFoundException(ResourceNotFoundException ex, HttpServletRequest request) {
          //......
        }
    }
    ```