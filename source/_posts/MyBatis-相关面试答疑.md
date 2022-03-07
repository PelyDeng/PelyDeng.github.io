title: MyBatis 相关面试答疑
author: Peilin Deng
tags:
  - MyBatis
categories:
  - 面试答疑
date: 2021-08-20 19:35:00
---
# 一、配置解析过程

- 解析 MyBatis 配置文件
  首先通过创建 SqlSessionFactoryBuilder 对象的 build() 方法来创建 SqlSessionFactory对象, 其中用 XMLConfigBuilder 读取并解析 MyBatis 配置文件读取各个标签的属性封装成 Configuration 对象. 

  在解析 mapper.xml 的时候, 主要做了两个事情: 
  - (1) 将 mapper.xml 文件中的 '增删改查' 语句封装成 MappedStatment 对象; 
  - (2) 将 mapper 接口文件 与 MapperProxyFactory[用于后面创建mapper代理对象] 对象相关联; 
  
  最后返回一个默认的 **DefaultSqlSessionFactory** 对象.


# 二、SqlSession 的创建过程
- SqlSession 的实现类是什么?
**默认是 DefaultSqlSession.** 

- 其他还创建了什么对象?
创建 SqlSession 时还通过事务工厂创建了 **事务对象**、**Executor对象**( **SIMPLE/REUSE/BATCH** 默认为**SIMPLE**, 当二级缓存开关打开时, 会封装成 **CachingExecutor** )
  - **SimpleExecutor**:
    默认的执行器,  根据对应的sql直接执行即可, 不会做一些额外的操作. 
  - **BatchExecutor**: 
    通过批量操作来优化性能的执行器.
  - **ReuseExecutor**:
    可重用的执行器, 重用对象是Statement, 该执行器会缓存同一个sql的Statement, 省去Statement的重新创建, 优化性能.

# 三、Mapper 对象
1. Mapper 对象是什么对象?
mapper 对象是一个JDK动态代理对象. 

2. 为什么要从 SqlSession 里面去获取?

3. 为什么传进去一个接口, 然后还要用接口类型来接收?
底层使用了JDK的动态代理, 给接口创建了一个代理对象.

# 四、执行 SQL
1. 为什么 Mybatis 的动态代理不需要实现类?
2. 接口没有实现类, 调用的是什么方法?
3. 接口方法怎么找到要执行的 SQL ?
4. 方法参数是怎么转换成 SQL 参数的?
5. 结果集怎么转换成对象?

# 五、Mybatis 核心对象
```txt
Configuration 
SqlSession 
Executor 
MapperProxy 
StatementHandler 
ParameterHandler 
ResultSetHandler 
MappedStatement 
MapperMethod 
SqlSource 
BoundSql
```




