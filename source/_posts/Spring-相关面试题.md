title: Spring 相关面试答疑
author: Peilin Deng
tags:
  - Spring
categories:
  - 面试答疑
  - ''
date: 2022-02-25 15:57:00
---
**1. 使用 Spring 框架能给我们带来哪些好处?**
> 1. 简化开发, 解放双手
> 2. 提供了内置的解决方案 BOP、IoC、AOP
> 3. 声明式事务管理, TransactionManager
> 4. 提供诸多的工具类, 围绕 Spring 生态, 比如 JdbcTemplate, BeanUtils


**2. BeanFactory 和 ApplicationContext 有什么区别?**
> 1. ApplicationContext 是 BeanFactory 的实现类
> 2. BeanFactory 是顶层设计(抽象), 而 ApplicationContext 是 User Interface
> 3. 功能会非常丰富, API是最全的, 一般会认为 ApplicationContext 就是 Ioc容器, 但是Ioc 的功能是在 DefaultListableBeanFactory 类中完成的, 但是有共同的接口


**<mark>3. 请解释 Spring Bean 的生命周期</mark>**
> 所谓生命周期, 从创建, 到调用, 到销毁 ( **作用域决定了生命周期的长短**)
> 1. 单例 Bean: 从容器的启动到 Spring容器的销毁, 如果是延迟加载的 Bean, 会在调用前创建
> 2. 原型 Bean: 在调用前创建, 在调用后销毁.


**4. Spring Bean 各作用域之间的区别?**
> 1. **sigleton** 作用域全局, 在任何地方可以通过 Ioc 拿到
> 2. **prototype** 作用域全局
> 3. **request** 在一次请求发起和结束之间
> 4. **session** 在一个 session 创建和失效之间, 根据配置的 session 失效时长 
> 5. **global-session** 可以理解为容器中的一个应用 ( **Spring 5 不再支持** )


**<mark>5. Spring 中的 Bean 是线程安全吗?</mark>**
> **不一定, 要结合情况. <br>
spring 中的 Bean 是从IOC容器中获得的, IOC容器中的 Bean 是通过配置得到的,<br>
我们在配置 Bean 的时候, 如果该Bean 的 scope 为原型, 则每次都会创建一个新对象, <br>
因此该 Bean 不存在线程之间的竞争, 所以不会存在线程安全的问题; 如果该 Bean 为单例, <br>
所有线程会共享一个单例实例Bean , 所以可能会存在线程安全问题.**


**<mark>6. Spring 中用到了那些设计模式?</mark>**
> 工厂模式、单例模式(容器式单例)、代理模式、享元模式、门面模式、适配器模式、委派模式、<br>
装饰器模式、责任链模式、解释器模式、策略模式、建造者模式、观察者模式、访问者模式 ...


**<mark>7. 讲述Spring 的基本实现思路</mark>**
> (见1. Spring 实现的基本思路 MarkDown ... )


**<mark>8. Spring、SpringBoot、SpringCloud 有什么区别?</mark>**
> **Spring** 是已有的生态, 继承了各种工具, 能完成我们日常开发的所有功能 <br>
> **SpringBoot** 基于 Spring, 更加简化了开发, 官方层面提供了一套脚手架, 一键搭建, 节省时间, 我们只需要遵从约定, 就能体验到更加简便的开发, 实现零配置, 并且全面的去 Servlet 化, 能够自运行, 部署也更加简便.<br>
> **SpringCloud** 基于 SpringBoot, 主要用于搭建分布式微服务, 集成了各种如 注册中心、服务发现、服务监控、配置中心、负载、熔断等... 打造一个微服务生态.


**<mark>9. Spring 事务实现原理</mark>**
> Spring 事务管理分为<mark>编程式</mark>和<mark>声明式</mark>两种,  编程式事务指的是通过编码方式实现事务; <mark>声明式事务基于 AOP</mark> <br>
> 1. **before**() -> 从连接池获取 connection 连接
> 2. 我们业务操作sql 
> 3. **after**() -> 根据条件(捕获异常等...) 来决定 **connection.commit**  或者**connection.rollback**


**10. BeanFactory 和 FactoryBean 的区别?**
> BeanFactory 是 Ioc 容器的顶层设计, 用于从容器获取 Bean <br>
> FactoryBean 用来构建 Bean 的一个包装类, 用于给容器创建 Bean


**11. 项目中如何应用AOP?**
> 声明式事务管理、做日志、权限等...

**<mark>12. @Qualifier是干啥用的?</mark>**
> 默认情况下，@Autowired 按类型装配 Spring Bean。如果容器中有多个相同类型的 bean，则框架将抛出 NoUniqueBeanDefinitionException， 以提示有多个满足条件的 bean 进行自动装配。 

通过将 @Qualifier 注解与我们想要使用的特定 Spring bean 的名称一起进行装配，Spring 框架就能从多个相同类型并满足装配要求的 bean 中找到我们想要的，避免让Spring脑裂。

```java
    @Component
    public class FooService {
        @Autowired
        @Qualifier("fooFormatter")
        private Formatter formatter;
        
        //todo 
    }
```
我们需要做的是@Component或者@Bean注解中声明的value属性以确定名称。其实我们也可以在 Formatter 实现类上使用 @Qualifier 注释，而不是在 @Component 或者 @Bean 中指定名称，也能达到相同的效果：
```java
     @Component
     @Qualifier("fooFormatter")
     public class FooFormatter implements Formatter {
         public String format() {
             return "foo";
         }
     }
 
     @Component
     @Qualifier("barFormatter")
     public class BarFormatter implements Formatter {
         public String format() {
             return "bar";
         }
     }
```

**13. 构造方法注入和设值注入有什么区别?**

> 使用构造函数依赖注入时，Spring保证所有一个对象所有依赖的对象先实例化后，才实例化这个对象。（没有他们就没有我原则）
        
使用set方法依赖注入时，Spring首先实例化对象，然后才实例化所有依赖的对象

**14. FileSystemResource 和 ClassPathResource 有何区别?**

在FileSystemResource 中需要给出spring-config.xml文件在你项目中的相对路径或者绝对路径。

在ClassPathResource中spring会在ClassPath中自动搜寻配置文件，所以要把ClassPathResource 文件放在ClassPath下。

如果将spring-config.xml保存在了src文件夹下的话，只需给出配置文件的名称即可，因为src文件夹是默认。

简而言之，ClassPathResource在环境变量中读取配置文件，FileSystemResource在配置文件中读取配置文件。











