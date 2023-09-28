### 优势（为什么学）

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

### Ioc（Inversion of Control）控制反转

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

# AOP切面编程

是对OOP（面向对象编程）的一种延续

- AOP：面向切面编程，将横切关注点从核心业务逻辑中提取出来，形成一个切面，通过字节码操作或动态代理等技术，实现代码的复用和解耦，提高代码的可维护性和可扩展性
  - 横切关注点：多个类或对象的公共行为
    - 日志记录：自定义日志记录注解
    - 性能统计：利用AOP在目标方法的执行前后统计访问方法的执行时间，方便优化和分析
    - 事务管理：@Transactional注解可以让Spring为我们进行事务管理比如回滚异常操作
  - 切面（Aspect）：对横切关注点进行封装的类，一个切面是一个类，切面可以定义多个通知用来实现具体的功能
  - 连接点（JoinPoint）：连接点是方法调用或者方法执行的某个特定的时刻（如方法调用和异常抛出等）
  - 通知（Advice）：通知就是在切面在某个连接点时的具体操作，分为五种：前置通知，后置通知，返回通知，异常通知，环绕通知，前四种是在目标方法的前后执行，环绕通知可以控制目标方法的执行过程
  - 切点（Pointcut）：一个切点是一个表达式，用来定义哪些连接点需要被切面所增强
  - 织入（Weaving）：织入是将切面和目标对象连接起来的过程，也就是将通知应用到切点匹配的连接点上，分为编译器织入和运行期织入
- OOP：将业务逻辑按照对象的属性和行为进行封装，通过类，对象，继承，多态等方式实现代码的模块化和层次化提高代码的可读性和可维护性