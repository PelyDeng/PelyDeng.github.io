title: Spring 事务传播原理及数据库事务操作原理
author: Peilin Deng
tags:
  - Spring
  - 事务
categories:
  - 摘抄笔记
date: 2021-08-20 18:43:00
---
### 1. 什么是事务?
事务(Transaction) 是访问并可能更新数据库中各个数据的一个程序执行单元(unit).  
特点: 事务是恢复和并发控制的基本单位, 事务应该具备四个属性
**原子性、一致性、隔离性、持久性**.
这四个属性通常称为 **ACID** 特性.

### 2. Spring 事务传播属性
![Spring 事务传播属性](/images/img-63.png)

### 3. 数据库事务的隔离级别
> http://pdyun.cc:4000/2021/08/27/Mysql-%E6%95%B0%E6%8D%AE%E5%BA%93%E4%BA%8B%E5%8A%A1%E9%9A%94%E7%A6%BB%E7%BA%A7%E5%88%AB/
![数据库的隔离级别](/images/img-64.png)

![](/images/img-66.png)

### 4. Spring 事务的隔离级别
![Spring 事务的隔离级别](/images/img-67.png)