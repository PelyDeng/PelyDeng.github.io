title: MyBatis 插件的执行顺序
author: Peilin Deng
tags:
  - MyBatis
  - 自定义插件
categories:
  - 摘抄笔记
date: 2021-08-20 18:58:00
---
多个插件执行顺序 
---
###### 本文转载自 https://zhuanlan.zhihu.com/p/266735787
###### 在 mybatis 中允许针对 SQL 在执行前后进行扩展操作，而这些扩展操作也叫做插件。<br>允许用插件来拦截的方法包括：
> MyBatis 默认支持对**4大对象**（**Executor，StatementHandler，ParameterHandler，ResultSetHandler**）上的方法执行拦截，具体支持的方法为：
> - **Executor** (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)，主要用于sql重写。
> - **ParameterHandler** (getParameterObject, setParameters)，用于参数处理。
> - **ResultSetHandler** (handleResultSets, handleOutputParameters)，用于结果集二次处理。
> - **StatementHandler** (prepare, parameterize, batch, update, query)，用于jdbc层的控制。
>
> 通过插件可以实现 SQL 打印，分页插件等功能。<br>
这时候就会延伸出一个问题：如果存在多个插件，这些插件的执行顺序是怎样的？

### 一、插件的执行顺序
###### 插件执行顺序一共有两种

#### 1、不同拦截对象的执行顺序
> Mybatis 针对以上这四种对象的拦截的执行顺序是固定的，因为 Mybatis代码的执行流程是固定的。<br>
以 **SimpleExecutor**#query 来说，这四种对象的执行代码如下：<br>
> ![](/images/img-71.png)

> 通过源码知道，执行顺序为：<br>
**Executor` -> `StatementHandler` -> `ParameterHandler` -> `StatementHandler` -> `ResultSetHandler**
> 
> 虽然中间 StatementHandler 执行了多次，但是总的来说，执行顺序优先级从高到低为：<br>
**Executor` -> `StatementHandler` -> `ParameterHandler` -> `ResultSetHandler**


#### 2、同种拦截对象的执行顺序
###### 针对同种对象如果存在多种拦截器对象，其拦截顺序如何？

> 首先在前面，知道了插件功能的实现是通过代理的方式对原有的如：Executor、ParameterHandler 等进行了代理增强，而经过代理后的原有的 Executor 、ParameterHandler 等对象会以如下方式存在：
> ![](/images/img-72.png)
>
> 如图，存在多个拦截器都是先进后出，针对代理模式来说来说，可以对被代理的原始对象的处理前后进行代码增强操作。
> 
> 而那个拦截器优先执行，取决于在生成代理对象时的顺序，也就是包裹在最外层的插件（拦截器）优先执行。
> 
> 来回顾一下代理对象生成时的逻辑然后结合mybatis-config.xml 配置文件的关于 <plugins> 的配置信息，即可知道相关的执行顺序。

###### 来看代理生成代码 InterceptorChain#pluginAll，如下：
> ![](/images/img-73.png)
> 如上述代码所示，通过遍历 interceptors List 列表对 target 对象进行包装（target 可以是 Executor or ParameterHandler等）。
>
> 也就是在 List 列表的最开始的 interceptor 插件最先被包裹在 target 对象外层，也就是如下图所示：
> ![upload successful](/images/img-74.png)
> 
> 如上图所示，在配置文件中越靠前的插件配置，在 interceptors List 列表中的位置自然越靠前，其执行顺序自然越靠后。
> 
#### 总结：同个对象的多个拦截器执行顺序根据配置文件 mybatis-config.xml 插件配置顺序有关，配置越靠前，执行顺序越靠后