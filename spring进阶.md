# Spring事务

事务：逻辑上的操作，要么全部成功，要么全部失败

Spring中很多业务方法都包括了多个原子性的数据库操作，这些原子性的数据库操作是有依赖的，要么都执行，要么都不执行（**但是要对应的数据库引擎支持事务**）

## Spring支持两种方式的事务管理

### 1.编程式事务管理

通过 `TransactionTemplate`或者`TransactionManager`手动管理事务，实际应用中很少使用

### 2.声明式事务管理

推荐使用（代码侵入性最小），实际是通过 AOP 实现（基于`@Transactional` 的全注解方式使用最多）

# Spring框架用到的设计模式

## 1.控制反转和依赖注入

DI（依赖注入）是实现控制反转的一种设计模式，依赖注入就是将实例变量传入到一个对象中去（前面已经讲过了控制反转了）

## 2.工厂设计模式

Spring使用工厂设计模式可以通过BeanFactory和ApplicationContext来创建bean对象

- BeanFactory：延迟注入（使用到bean的时候才会注入），相比于applicationContext来说会占用更少的内存，程序启动更快
- ApplicationContext：容器启动的时候，不管有没有用到，一次性创建所有的bean（扩展了BeanFactory）

ApplicationContext有三个实现类

1. ClassPathXMLApplication：吧上下文文件当作类路径资源
2. FileSystemXMLApplication：吧文件系统中的xml文件载入上下文定义信息
3. XMLWebApplicationContext：从Web系统中的xml文件载入上下文定义信息

## 3.单例设计模式

在系统中一些对象一般只需要一个，如：线程池、缓存、对话框、注册表、日志对象等，这一类对象只能有一个实例，如果制造多个实例就可能导致一些问题的产生，如：程序的行为异常、资源使用过量、或者不一致性的结果

### 好处：

- 对于频繁使用的对象，可以省略创建对象所需要的时间，对于重量级对象就更加友好（减少了系统开销）
- 由于new的次数减少，因而对系统内存的使用频率也会降低，减轻了gc压力，缩短了gc停顿时间

Spring中bean的默认作用域就是singleton（单例）的，除了singleton作用域，Spring中bean还有以下几种作用域：

- prototype
- request
- session
- application/global-session
- websocket

### 实现

Spring通过ConcurrentHashMap实现单例注册表的特殊方式来实现单例模式

```java
// 通过 ConcurrentHashMap（线程安全） 实现单例注册表
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<String, Object>(64);

public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "'beanName' must not be null");
        synchronized (this.singletonObjects) {
            // 检查缓存中是否存在实例
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                //...省略了很多代码
                try {
                    singletonObject = singletonFactory.getObject();
                }
                //...省略了很多代码
                // 如果实例对象在不存在，我们注册到单例注册表中。
                addSingleton(beanName, singletonObject);
            }
            return (singletonObject != NULL_OBJECT ? singletonObject : null);
        }
    }
    //将对象添加到单例注册表
    protected void addSingleton(String beanName, Object singletonObject) {
            synchronized (this.singletonObjects) {
                this.singletonObjects.put(beanName, (singletonObject != null ? singletonObject : NULL_OBJECT));

            }
        }
}
```

## 4.代理设计模式

### 应用方面

**Spring AOP 就是基于动态代理的**，如果要代理的对象，实现了某个接口，那么 Spring AOP 会使用 **JDK Proxy** 去创建代理对象，而对于没有实现接口的对象，就无法使用 JDK Proxy 去进行代理了，这时候 Spring AOP 会使用 **Cglib** 生成一个被代理对象的子类来作为代理

当然也能使用AspectJ来进行相关操作

> 动态代理

## 5.模板方法

模板方法是一种行为设计模式，定义了一个操作中算法的骨架，而将一些步骤延迟到子类中

```java
public abstract class Template {
    //这是我们的模板方法
    public final void TemplateMethod(){
        PrimitiveOperation1();
        PrimitiveOperation2();
        PrimitiveOperation3();
    }

    protected void  PrimitiveOperation1(){
        //当前类实现
    }

    //被子类实现的方法
    protected abstract void PrimitiveOperation2();
    protected abstract void PrimitiveOperation3();

}
public class TemplateImpl extends Template {

    @Override
    public void PrimitiveOperation2() {
        //当前类实现
    }

    @Override
    public void PrimitiveOperation3() {
        //当前类实现
    }
}
```

Spring 中 `JdbcTemplate`、`HibernateTemplate` 等以 Template 结尾的对数据库操作的类，它们就使用到了模板模式。一般情况下，我们都是使用继承的方式来实现模板模式，但是 Spring 并没有使用这种方式，而是使用 Callback 模式与模板方法模式配合，既达到了代码复用的效果，同时增加了灵活性

## 6.观察者模式

观察者模式是一种对象行为型模式，他表示的是一种对象与对象之间具有依赖关系，当一个对象发生改变的时候，依赖这个对象的所有对象也会做出反应（如Spring事件驱动模型）

### Spring事件驱动模型的三种角色

#### （1）事件角色

#### （2）事件监听角色

#### （3）事件发布角色

#### Spring的事件流程

1. 定义一个事件
2. 定义一个事件监听
3. 使用事件发布者发布消息

```java
// 定义一个事件,继承自ApplicationEvent并且写相应的构造函数
public class DemoEvent extends ApplicationEvent{
    private static final long serialVersionUID = 1L;

    private String message;

    public DemoEvent(Object source,String message){
        super(source);
        this.message = message;
    }

    public String getMessage() {
         return message;
          }


// 定义一个事件监听者,实现ApplicationListener接口，重写 onApplicationEvent() 方法；
@Component
public class DemoListener implements ApplicationListener<DemoEvent>{

    //使用onApplicationEvent接收消息
    @Override
    public void onApplicationEvent(DemoEvent event) {
        String msg = event.getMessage();
        System.out.println("接收到的信息是："+msg);
    }

}
// 发布事件，可以通过ApplicationEventPublisher  的 publishEvent() 方法发布消息。
@Component
public class DemoPublisher {

    @Autowired
    ApplicationContext applicationContext;

    public void publish(String message){
        //发布事件
        applicationContext.publishEvent(new DemoEvent(this, message));
    }
}
```

## 7.适配器模式

### （1）Spring AOP中的适配器模式

### （2）Spring MVC中的适配器模式

## 8.装饰者模式

# Springboot的自动装配

为了优化Spring的xml配置，引入了Springboot，只需要添加相关依赖，无需配置就能使用，还能通过修改application.properties或application.yml来修改项目相关配置信息

Springboot这主要得力于其自动装配

## 概念

Springboot定义了一套接口规范：Springboot启动时会扫描外部引用jar包中的META-INF/Spring.factories文件，将文件中配置的类信息加载到Spring容器，并执行类中定义的各种操作（对于外部的jar只需要按照Springboot定义的标准就能吧自己的功能装置进Springboot）

简单来说：通过注解或者一些简单的配置就能在Spring Boot的帮助下实现某块功能

## 实现

首先看启动类那里的注解SpringBootApplication（Springboot的核心注解）

主要包含三部分

- @EnableAutoConfiguration：启用SpringBoot的自动配置机制
- @Configuration：允许在上下文中注册额外的bean或导入其他配置类
- @ComponentScan：扫描被@Component（@Service，@Controller）注解的bean，注解默认会扫描启动类所在的包下所有的类，可以自定义不扫描某些bean

```java
@ComponentScan(excludeFilters = {@Filter(type = FilterType.CUSTOM,classes = xxx.class),
@Filter(type = FilterType.CUSTOM,classes = xxx.class)
})
```

#### （1）@EnableAutoConfiguration：实现自动装配的核心注解