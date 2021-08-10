title: MyBatis 运行时序图
author: Peilin Deng
tags:
  - MyBatis
categories:
  - 摘抄笔记
  - ''
date: 2021-08-10 19:22:00
---
### 架构分层

![架构分层](/images/pasted-4.png)

<!--more-->

#### 1. 创建会话工厂类

![创建会话过程](/images/pasted-5.png)

#### 2. 创建会话

![创建会话过程](/images/pasted-6.png)

#### 3. 获取代理对象
![获取代理对象过程](/images/pasted-7.png)

#### 4. 调用代理对象方法, 执行SQL

![调用代理对象方法, 执行SQL过程](/images/pasted-8.png)