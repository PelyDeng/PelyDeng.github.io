title: MyBatis插件关键对象
author: Peilin Deng
tags:
  - MyBatis
  - 自定义插件
categories:
  - 摘抄笔记
date: 2021-08-20 18:59:00
---
MyBatis插件关键对象
---
> **Interceptor** 接口：自定义拦截器（实现类）

> **InterceptorChain**：存放插件的容器

> **Plugin**：h对象；提供创建代理类的方法

> **Invocation**：对被代理对象的封装