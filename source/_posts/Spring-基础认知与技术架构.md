title: Spring 基础认知与技术架构
author: Peilin Deng
tags:
  - Spring
categories:
  - 摘抄笔记
date: 2021-08-19 23:10:00
---
## Spring
> Spring 是一个轻量级Java开发框架，最早有Rod Johnson创建，目的是为了解决企业级应用开发的业务逻辑层和其他各层的耦合问题。它是一个分层的JavaSE/JavaEE full-stack（一站式）轻量级开源框架，为开发Java应用程序提供全面的基础架构支持。Spring负责基础架构，因此Java开发者可以专注于应用程序的开发。 
  
> Spring最根本的使命是解决企业级应用开发的复杂性，即简化Java开发。


### 1. Spring 简化开发的四个基本策略
> 1. 基于POJO 的轻量级和最小侵入性编程.
> 2. 通过依赖注入和面向接口松耦合.
> 3. 基于切面和惯性进行声明式编程.
> 4. 通过切面和模板减少样版式代码.

<!-- more -->

### 2. Spring 中的编程思想
Spring思想 | 应用场景 (特点) | 一句话归纳
--- | --- | ---
**OOP** | Object Oriented Programming (面向对象编程) 用程序归纳总结生活中一切事物 | 封装、继承、多态.
**BOP** | Bean Oriented Programming (面向Bean编程) 面向Bean (普通Java类) 设计程序, 解放程序员. | 一切从Bean开始.
**AOP** | Aspect Oriented Programming (面向切面编程) 找出多个类中有一定规律的代码, 开发时拆开, 应运行时再合并. 面向切面编程, 及面向规则编程. | 解耦, 专人做专事.
**IoC** | Inversion of Control (控制反转) 将new对象的动作交给Spring管理, 并由Spring保存已创建的对象 (IOC容器). | 转交控制权(即控制权反转).
**DI/D**L | Dependency Injection (依赖注入) 或者Dependency Lookup (依赖查找) , Spring不仅保存自己创建的对象, 而且保存对象与对象之间的关系. 注入即赋值, 主要三种方式 --- 构造方法、set方法、直接赋值. | 自动赋值.


### 3. Spring 注解编程演化
V1.X | V2.0 | V2.5 | V3.X | V4.X | V5.X 
--- | --- | --- | --- | --- | --- | --- 
**注解驱动启蒙时代** | **注解驱动过渡时代** | **引入新的骨架式Annotation** | **注解驱动黄金时代** | **注解驱动完善时代** | **注解驱动成熟时代**


### 4. Spring 模块结构
 > Spring 总共大约有 20 个模块， 由 1300 多个不同的文件构成。 而这些组件被分别整合在核心容器（Core Container） 、 AOP（Aspect Oriented Programming）和设备支持（Instrmentation） 、数据访问与集成（Data Access/Integeration） 、 Web、 消息（Messaging） 、 Test等 6 个模块中。 
 
![Spring 模块结构](/images/img-13.png)


### 5. Spring 系统架构模块功能介绍
#### Spring 核心模块

模块名称 | 主要功能
---|---
**spring-core** | IoC控制反转与DI依赖注入的最基本实现
**spring-beans** | Bean工厂与Bean的装配
**spring-context** | 定义基础的Spring的Context上下文即IoC容器
**spring-context-support** | 对Spring IoC的扩展支持, 以及IoC子容器
**spring-context-indexer** | Spring的类管理组件和Classpath扫描
**spring-expression** | Spring表达式语言

---------------------------------------

##### Spring 面向切面编程模块

模块名称 | 主要功能
---|---
**spring-aop** | 面向切面编程的引用模块, 整合Asm, CGLib、JDKProxy
**spring-aspects** | 继承AspectJ, AOP应用框架
**spring-instrument** | 动态Class Loading模块

---------------------------------------

##### Spring 数据访问与继承模块

模块名称 | 主要功能
---|---
**spring-jdbc** | Spring 提供的JDBC抽象框架的组要实现模块, 用于简化 Spring JDBC 操作
**spring-tx** | Spring JDBC 事务控制实现模块
**spring-orm** | 主要继承 Hibernate, Java Persitence API (JPA) 和 Java Data Object (JDO)
**spring-oxm** | 将Java对象映射成XML数据, 或者将XML数据映射成Java对象
**spring-jms** | Java Messaging Service 能够发送和接收信息

---------------------------------------

##### Spring Web 模块

模块名称 | 主要功能
---|---
**spring-web** | 提供了最基础Web支持, 主要建立于核心容器之上, 通过 Servlet 或者 Listeners 来初始化 IoC 容器
**spring-webmvc** | 实现了 Spring-MVC (model-view-controller) 的 Web 应用
**spring-websokect** | 主要是与 Web 前端的全双工通讯的协议
**spring-webflux** | 一个新的非阻塞函数式 Reactive Web 框架, 可以用来建立异步的. 非阻塞, 事件驱动的服务

---------------------------------------

##### Spring 通信报文 模块

模块名称 | 主要功能
---|---
**spring-messaging** | 从 Spring4 开始新加入的一个模块, 主要职责是为 Spring 框架继承一些基础的报文传输应用

---------------------------------------

##### Spring 集成测试 模块

模块名称 | 主要功能
---|---
**spring-test** | 主要为测试提供支持的

---------------------------------------

##### Spring 集成兼容 模块

模块名称 | 主要功能
---|---
**spring-framwork-bom** | Bill of Materials. 解除 Spring 的不同模块依赖版本不同问题


### 6. Spring 模块之依赖关系图
![模块依赖关系](/images/img-14.png)


### 7. 版本命名规则

#### Spring 版本命名规则
> ![Spring 版本命名规则](/images/img-15.png)

#### 其他常见软件版本命名规则
> ![upload successful](/images/img-16.png)

#### 语义化版本命名通用规则
> ![upload successful](/images/img-17.png)

#### 商业软件中常见的修饰词
> ![upload successful](/images/img-18.png)