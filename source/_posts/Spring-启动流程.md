title: Spring 启动流程简述
author: Peilin Deng
tags:
  - Spring
categories:
  - 摘抄笔记
date: 2021-08-20 13:47:00
---
> beanDefinitionMap  -> 用来存储 BeanDefinition(Bean 的配置信息)  
factoryBeanObjectCache  ->  用来存储原生 Bean 对象的Map, 指反射创建出的实际对象  
factoryBeanInstanceCache  ->  用来存储 BeanWrapper 的Map, 指原生 Bean 的包装类  

### Spring 启动流程简述
#### 一. 配置阶段
* web.xml
```
DispatcherServlet 路径
设定 init-param ( contextConfigLocation = classPath:application.xml )
设定 url-pattern ( /* )
配置 Annotation 等
```

<!-- more -->

* appication.xml
```
配置 包扫描路径、Bean定义、视图解析配置等......
```
* ......

#### 二. 初始化阶段
* Servlet.init()
```
Spring 是 servlet 编程模型, 容器启动时会调用 servlet 的 init() 方法, 
在该方法中会读取配置进行 IoC 容器及 MVC 组件的初始化.
```

* IoC 部分 (定位、加载、注册) 
```
初始化 IoC 容器
 1. 通过 web.xml 的配置定位 application .xml配置文件. 
 2. 使用 BeanDefinitionReader 读取配置文件, 扫描类并封装成 BeanDefinition
 3. 创建 BeanFatory, 将 *beanDefinition 注册到 DefalutListableBeanFactory 的 beanDefinitionMap 中
```

* DI 、 AOP 部分
```
 4. 初始化非延迟加载的 bean
	0). 标记 bean 为创建中      
	1). 通过反射 new 出 bean 对象, 封装成 BeanWrapper 对象      
	2). 如果 bean 为单例且支持循环依赖则生成三级缓存 singletonFactories, 可提前暴露 bean      
	3). 填充bean属性，解决属性依赖      
	4). 初始化bean的各个Aware接口(各个Aware接口能让bean获取到部分属性: ApplicationContextAware-能获取到ApplicationContex; BeanFactoryAware 能获取到 BeanFactory) 并执行各类 bean 的后处理器, 执行初始化方法, 如果有 AOP 配置需要生成 AOP 代理对象 
	5). 如果存在循环依赖，解决之 – 这里有点问题，这一步是如果之前解决了aop循环依赖，则缓存中放置了提前生成的代理对象，然后使用原始bean继续执行初始化，所以需要再返回最终bean前，把原始bean置换为代理对象返回。      
	6). 此时 bean 已经可以使用, 将 bean 放入一级缓存 singletonObjects , 移除创建中标记以及二三级缓存
```

* MVC 部分
```
 5. 初始化 MVC 九大组件
	// 1). 初始化文件上传解析器
	initMultipartResolver(context);
	// 2). 初始化本地语言环境
	initLocaleResolver(context);
	// 3). 初始化模板处理器
	initThemeResolver(context);
	// 4). 初始化 HandlerMapping 组件
	initHandlerMappings(context);
	// 5). 初始化参数适配器
	initHandlerAdapters(context);
	// 6). 初始化异常拦截器
	initHandlerExceptionResolvers(context);
	// 7). 初始化视图预处理器
	initRequestToViewNameTranslator(context);
	// 8). 初始化视图解析器
	initViewResolvers(context);
	// 9). 初始化 FlashMap 管理器
	( 为了解决请求转发和重定向过程中参数的丢失问题: redirect->重定向, request 参数会丢失 ; forward->转发, 自动将 request 参数系诶带到下一个请求 )
	initFlashMapManager(context);
```

#### 三. 运行阶段
> 1. 从页面点击按钮或者 url 访问资源请求会先到 DispatcherServlet 的 doDispatch() 方法, 该方法会从 HandlerMapping 中通过 url 去匹配对应的控制器及方法  
> 2. 通过参数解析器解析参数并反射执行方法, 返回一个 ModelAndView  
> 3. 通过视图解析器解析 ModelAndView, 决定返回页面或者输出数据  
> 4. 前端根据对应结果来展示