title: MyBatis 插件的使用及原理
author: Peilin Deng
tags:
  - MyBatis
  - 插件
  - ''
categories:
  - 摘抄笔记
date: 2021-08-20 18:59:00
---
# MyBatis插件关键对象
---
> **Interceptor** 接口：自定义拦截器（实现类）

> **InterceptorChain**：存放插件的容器

> **Plugin**：h对象；提供创建代理类的方法

> **Invocation**：对被代理对象的封装

# 1. 自定义插件

mybatis插件（准确的说应该是around拦截器，因为接口名是interceptor，而且invocation.proceed要自己调用，配置中叫插件）功能非常强大，**可以让我们无侵入式的对SQL的执行进行干涉，从SQL语句重写、参数注入、结果集返回等每个主要环节，典型的包括权限控制检查与注入、只读库映射、K/V翻译、动态改写SQL。**

MyBatis 默认支持对**4大对象**（**Executor，StatementHandler，ParameterHandler，ResultSetHandler**）上的方法执行拦截，具体支持的方法为：
- **Executor** (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)，主要用于sql重写。

- **ParameterHandler** (getParameterObject, setParameters)，用于参数处理。

- **ResultSetHandler** (handleResultSets, handleOutputParameters)，用于结果集二次处理。

- **StatementHandler** (prepare, parameterize, batch, update, query)，用于jdbc层的控制。

大多数情况下仅在Executor做插件比如SQL重写、结果集脱敏，ResultSetHandler和StatementHandler仅在高级场景中使用，而且某些场景中非常有价值。

**四大对象的在sql执行过程中的调用链如下：**

![](/images/img-70.png)

具体的方法定义可以参见每个类方法的签名，这里就不详细展开了。这四个类被创建后不是直接返回，而是创执行了interceptorChain.pluginAll(parameterHandler)才返回。如下所示：

```java
//Configuration 中
public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement, parameterObject, boundSql);
    parameterHandler = (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
    return parameterHandler;
}

public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
                                            ResultHandler resultHandler, BoundSql boundSql) {
    ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
    resultSetHandler = (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
    return resultSetHandler;
}

public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
}

public Executor newExecutor(Transaction transaction) {
    return newExecutor(transaction, defaultExecutorType);
}

public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType ** null ? defaultExecutorType : executorType;
    executorType = executorType ** null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH ** executorType) {
        executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE ** executorType) {
        executor = new ReuseExecutor(this, transaction);
    } else {
        executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
        executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}

```
## 1-1 编写Interceptor的实现类
---

### Executor拦截器

#### 案例1:
```java
@Intercepts({
    // @Signature(type = Executor.class, method = /* org.apache.ibatis.executor.Executor中定义的方法,参数也要对应 */"update", args = { MappedStatement.class, Object.class}),
    @Signature(
        type = Executor.class, method = "query", 
        args = { MappedStatement.class, Object.class,RowBounds.class, ResultHandler.class }
    ) 
})
public class SelectPruningColumnPlugin implements Interceptor {
    public static final ThreadLocal<ColumnPruning> enablePruning = new ThreadLocal<ColumnPruning>(){
        @Override
        protected ColumnPruning initialValue()
        {
            return null;
        }
    };
    
    Logger logger = LoggerFactory.getLogger(SelectPruningColumnPlugin.class);
    
    static int MAPPED_STATEMENT_INDEX = 0;// 这是对应上面的args的序号
    static int PARAMETER_INDEX = 1;
    static int ROWBOUNDS_INDEX = 2;
    static int RESULT_HANDLER_INDEX = 3;
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        if (enablePruning.get() != null && enablePruning.get().isEnablePruning()) {
            Object[] queryArgs = invocation.getArgs();
            MappedStatement mappedStatement = (MappedStatement) queryArgs[MAPPED_STATEMENT_INDEX];
            Object parameter = queryArgs[PARAMETER_INDEX];
            BoundSql boundSql = mappedStatement.getBoundSql(parameter);
            String sql = boundSql.getSql();// 获取到SQL ，进行调整
            String name = mappedStatement.getId();
            logger.debug("拦截的方法名是:" + name + ",sql是" + sql + ",参数是" + JsonUtils.toJson(parameter));
            String execSql = pruningColumn(enablePruning.get().getReserveColumns(), sql);
            logger.debug("修改后的sql是:" + execSql);

            // 重新new一个查询语句对像
            BoundSql newBoundSql = new BoundSql(mappedStatement.getConfiguration(), execSql, boundSql.getParameterMappings(), boundSql.getParameterObject());
            // 把新的查询放到statement里
            MappedStatement newMs = copyFromMappedStatement(mappedStatement, new BoundSqlSqlSource(newBoundSql));
            for (ParameterMapping mapping : boundSql.getParameterMappings()) {
                String prop = mapping.getProperty();
                if (boundSql.hasAdditionalParameter(prop)) {
                    newBoundSql.setAdditionalParameter(prop, boundSql.getAdditionalParameter(prop));
                }
            }
            queryArgs[MAPPED_STATEMENT_INDEX] = newMs;
            // 因为涉及分页查询PageHelper插件，所以不能设置为null，需要业务上下文执行完成后设置为null
//            enablePruning.set(null);
        }
        Object result = invocation.proceed();
        return result;
    }
    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }
    @Override
    public void setProperties(Properties properties) {
    }
```


#### 案例2: 带反射

```java
@Intercepts({
        @Signature(
            type = Executor.class,method = "query",
            args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
        )
})
public class MyPageInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("将逻辑分页改为物理分页");
        Object[] args = invocation.getArgs();
        MappedStatement ms = (MappedStatement) args[0]; // MappedStatement
        BoundSql boundSql = ms.getBoundSql(args[1]); // Object parameter
        RowBounds rb = (RowBounds) args[2]; // RowBounds
        // RowBounds为空，无需分页
        if (rb ** RowBounds.DEFAULT) {
            return invocation.proceed();
        }

        // 将原 RowBounds 参数设为 RowBounds.DEFAULT，关闭 MyBatis 内置的分页机制
        //args[2] = RowBounds.DEFAULT;

        // 在SQL后加上limit语句
        String sql = boundSql.getSql();
        String limit = String.format("LIMIT %d,%d", rb.getOffset(), rb.getLimit());
        sql = sql + " " + limit;

        // 自定义sqlSource
        SqlSource sqlSource = new StaticSqlSource(ms.getConfiguration(), sql, boundSql.getParameterMappings());

        // 修改原来的sqlSource
        Field field = MappedStatement.class.getDeclaredField("sqlSource");
        field.setAccessible(true);
        field.set(ms, sqlSource);

        // 执行被拦截方法
        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
    }
}
```
### ResultSetHandler拦截器

```java
@Intercepts({
    @Signature(
        type = ResultSetHandler.class, 
        method = "handleResultSets", 
        args = { Statement.class}
    ) 
})
public class OptMapPlugin implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object target = invocation.getTarget();
        Statement stmt = (Statement) invocation.getArgs()[0];
        if (target instanceof DefaultResultSetHandler) {
            DefaultResultSetHandler resultSetHandler = (DefaultResultSetHandler) target;
            Class clz = resultSetHandler.getMappedStatement().getResultMaps().get(0).getType();
            if (clz ** OptMap.class) {
                List<Object> resultList = new ArrayList<Object>();
                OptMap optMap = new OptMap();
                resultList.add(optMap);
                resultSet2OptMap(resultSetHandler.getConfiguration(),resultSetHandler,optMap,stmt.getResultSet());
                return resultList;
            }
            return invocation.proceed();
        }
        //如果没有进行拦截处理，则执行默认逻辑
        return invocation.proceed();
    }
```

### 注册插件
最后将插件配置到mybatis-config.xml

```xml
<!-- mybatis-config.xml  注册插件-->
<plugins>
    <plugin interceptor="com.dpl.mybatis.plugin.SelectPruningColumnPlugin">
        <property name="someProperty" value="100"/>
    </plugin>
</plugins>
```


# 2. 插件实现原理

## 2-1. 初始化操作
首先我们来看下在全局配置文件加载解析的时候做了什么操作。

![](/images/img-121.png)

进入方法内部可以看到具体的解析操作
```java
	private void pluginElement(XNode parent) throws Exception {
        if (parent != null) {
            for (XNode child : parent.getChildren()) {
                // 获取<plugin> 节点的 interceptor 属性的值
                String interceptor = child.getStringAttribute("interceptor");
                // 获取<plugin> 下的所有的properties子节点
                Properties properties = child.getChildrenAsProperties();
                // 获取 Interceptor 对象
                Interceptor interceptorInstance = (Interceptor)
                        resolveClass(interceptor).getDeclaredConstructor().newInstance();
                // 设置 interceptor的 属性
                interceptorInstance.setProperties(properties);
                // Configuration中记录 Interceptor
                configuration.addInterceptor(interceptorInstance);
            }
        }
    }
```
该方法用来解析全局配置文件中的plugins标签，然后对应的创建Interceptor对象，并且封装对应的属性信息。最后调用了Configuration对象中的方法。

configuration.addInterceptor(interceptorInstance)

```java
	public void addInterceptor(Interceptor interceptor) {
        interceptorChain.addInterceptor(interceptor);
    }
```
通过这个代码我们发现我们自定义的拦截器最终是保存在了 InterceptorChain 这个对象中。而 InterceptorChain 的定义为
```java
public class InterceptorChain {
  // 保存所有的 Interceptor  也就我所有的插件是保存在 Interceptors 这个List集合中的
  private final List<Interceptor> interceptors = new ArrayList<>();
  //
  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      // 获取拦截器链中的所有拦截器 target = interceptor.plugin(target); // 创建对应的拦截器的代理对象
    }
    return target;
  }
  public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
  }
  public List<Interceptor> getInterceptors() {
    return Collections.unmodifiableList(interceptors);
  }
}
```

## 2.2 如何创建代理对象

在解析的时候创建了对应的Interceptor对象，并保存在了InterceptorChain中，那么这个拦截器是如何和对应的目标对象进行关联的呢？

首先拦截器可以拦截的对象是<mark>Executor</mark>,<mark>ParameterHandler</mark>,<mark>ResultSetHandler</mark>,<mark>StatementHandler</mark>. 那么我们来看下这四个对象在创建的时候又什么要注意的

### 2.2.1 Executor

![upload successful](/images/img-122.png)

我们可以看到Executor在装饰完二级缓存后会通过 **pluginAll** 来创建 Executor 的代理对象。

```java
public Object pluginAll(Object target) {
  for (Interceptor interceptor : interceptors) {
    // 获取拦截器链中的所有拦截器 target = interceptor.plugin(target); // 创建对应的拦截器的代理对象
  }
  return target;
}
```
进入plugin方法中，我们会进入到
```java
// 决定是否触发 intercept()方法
default Object plugin(Object target) {
	return Plugin.wrap(target, this);
}
```
然后进入到MyBatis给我们提供的Plugin工具类的实现 wrap方法中。
```java
/**
* 创建目标对象的代理对象
*    目标对象 Executor  ParameterHandler  ResultSetHandler StatementHandler
* @param target 目标对象
* @param interceptor 拦截器
* @return */
public static Object wrap(Object target, Interceptor interceptor) {
  // 获取用户自定义 Interceptor中@Signature注解的信息
  // getSignatureMap 负责处理@Signature 注解
  Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
  // 获取目标类型
  Class<?> type = target.getClass();
  // 获取目标类型 实现的所有的接口
  Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
  // 如果目标类型有实现的接口 就创建代理对象
  if (interfaces.length > 0) {
    return Proxy.newProxyInstance(
    type.getClassLoader(),
      interfaces,
    new Plugin(target, interceptor, signatureMap));
  }
  // 否则原封不动的返回目标对象
  return target;
}
```
Plugin中的各个方法的作用
```java
public class Plugin implements InvocationHandler {
  private final Object target;
  // 目标对象
  private final Interceptor interceptor;
  // 拦截器
  private final Map<Class<?>, Set<Method>> signatureMap;
  // 记录 @Signature 注解的 信息
  private Plugin(Object target, Interceptor interceptor, Map<Class<?>,
  Set<Method>> signatureMap) {
    this.target = target;
    this.interceptor = interceptor;
    this.signatureMap = signatureMap;
  }
  /**
* 创建目标对象的代理对象
*    目标对象 Executor  ParameterHandler  ResultSetHandler StatementHandler
* @param target 目标对象
* @param interceptor 拦截器
* @return */
  public static Object wrap(Object target, Interceptor interceptor) {
    // 获取用户自定义 Interceptor中@Signature注解的信息
    // getSignatureMap 负责处理@Signature 注解
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    // 获取目标类型
    Class<?> type = target.getClass();
    // 获取目标类型 实现的所有的接口
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    // 如果目标类型有实现的接口 就创建代理对象
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
      type.getClassLoader(),
      interfaces,
      new Plugin(target, interceptor, signatureMap));
    }
    // 否则原封不动的返回目标对象
    return target;
  }
  /**
* 代理对象方法被调用时执行的代码
* @param proxy
* @param method
* @param args
* @return
* @throws Throwable */
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      // 获取当前方法所在类或接口中，可被当前Interceptor拦截的方法
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {
        // 当前调用的方法需要被拦截 执行拦截操作
        return interceptor.intercept(new Invocation(target, method, args));
      }
      // 不需要拦截 则调用 目标对象中的方法
      return method.invoke(target, args);
    }
    catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }
  private static Map<Class<?>, Set<Method>> getSignatureMap(Interceptor interceptor) {
    Intercepts interceptsAnnotation =
    interceptor.getClass().getAnnotation(Intercepts.class);
    // issue #251
    if (interceptsAnnotation == null) {
      throw new PluginException("No @Intercepts annotation was found in interceptor " + interceptor.getClass().getName());
    }
    Signature[] sigs = interceptsAnnotation.value();
    Map<Class<?>, Set<Method>> signatureMap = new HashMap<>();
    for (Signature sig : sigs) {
      Set<Method> methods = signatureMap.computeIfAbsent(sig.type(), k -> new HashSet<>());
      try {
        Method method = sig.type().getMethod(sig.method(), sig.args());
        methods.add(method);
      }
      catch (NoSuchMethodException e) {
        throw new PluginException("Could not find method on " + sig.type() + "
named " + sig.method() + ". Cause: " + e, e);
      }
    }
    return signatureMap;
  }
  private static Class<?>[] getAllInterfaces(Class<?> type, Map<Class<?>, Set<Method>> signatureMap) {
    Set<Class<?>> interfaces = new HashSet<>();
    while (type != null) {
      for (Class<?> c : type.getInterfaces()) {
        if (signatureMap.containsKey(c)) {
          interfaces.add(c);
        }
      }
      type = type.getSuperclass();
    }
    return interfaces.toArray(new Class<?>[interfaces.size()]);
  }
}
```

### 2.2.2 StatementHandler
在获取StatementHandler的方法中
```java
@Override
public <E> List<E> doQuery(MappedStatement ms, rowBounds, ResultHandler resultHandler, BoundSql
Statement stmt = null;
Object parameter, RowBounds     boundSql) throws SQLException {
  try {
    Configuration configuration = ms.getConfiguration();
    // 注意，已经来到SQL处理的关键对象 StatementHandler >>
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
    // 获取一个 Statement对象
    stmt = prepareStatement(handler, ms.getStatementLog());
    // 执行查询
    return handler.query(stmt, resultHandler);
  }
  finally {
    // 用完就关闭
    closeStatement(stmt);
  }
}
```
在进入newStatementHandler方法
```java
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler      resultHandler, BoundSql boundSql) {
  StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
  // 植入插件逻辑（返回代理对象）
  statementHandler = (StatementHandler)
  interceptorChain.pluginAll(statementHandler);
  return statementHandler;
}
```
可以看到statementHandler的代理对象

2.2.3 ParameterHandler
在上面步骤的RoutingStatementHandler方法中，我们来看看
```java
public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    // StatementType 是怎么来的？ 增删改查标签中的 statementType="PREPARED"，默认值 PREPARED
    switch (ms.getStatementType()) {
        case STATEMENT:
              delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
        case PREPARED:
        // 创建 StatementHandler 的时候做了什么？  >>
        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
        case CALLABLE:
        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
        default:
        throw new ExecutorException("Unknown statement type: " +
        ms.getStatementType());
    }
}
```
然后我们随便选择一个分支进入，比如PreparedStatementHandler

![upload successful](/images/img-123.png)

在newParameterHandler的步骤我们可以发现代理对象的创建
```java
public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject, BoundSql boundSql) {
    ParameterHandler parameterHandler =
    mappedStatement.getLang().createParameterHandler(mappedStatement,
    parameterObject, boundSql);
    // 植入插件逻辑（返回代理对象）
    parameterHandler = (ParameterHandler)
    interceptorChain.pluginAll(parameterHandler);
    return parameterHandler;
}
```

### 2.2.4 ResultSetHandler
在上面的newResultSetHandler()方法中，也可以看到ResultSetHander的代理对象
```java
public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler,
ResultHandler resultHandler, BoundSql boundSql) {
    ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
    // 植入插件逻辑（返回代理对象）
    resultSetHandler = (ResultSetHandler)
    interceptorChain.pluginAll(resultSetHandler);
    return resultSetHandler;
}
```

## 2.3 执行流程
以Executor的query方法为例，当查询请求到来的时候，Executor的代理对象是如何处理拦截请求的呢？我们来看下。当请求到了executor.query方法的时候

![upload successful](/images/img-124.png)

然后会执行Plugin的invoke方法
```java
/**
* 代理对象方法被调用时执行的代码
* @param proxy
* @param method
* @param args
* @return
* @throws Throwable */
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        // 获取当前方法所在类或接口中，可被当前Interceptor拦截的方法
        Set<Method> methods = signatureMap.get(method.getDeclaringClass());
        if (methods != null && methods.contains(method)) {
            // 当前调用的方法需要被拦截 执行拦截操作
            return interceptor.intercept(new Invocation(target, method, args));
        }
        // 不需要拦截 则调用 目标对象中的方法
        return method.invoke(target, args);
    }
    catch (Exception e) {
        throw ExceptionUtil.unwrapThrowable(e);
    }
}
```
然后进入interceptor.intercept 会进入我们自定义的 FirstInterceptor对象中
```java
/**
* 执行拦截逻辑的方法
* @param invocation
* @return
* @throws Throwable */
@Override
public Object intercept(Invocation invocation) throws Throwable {
    System.out.println("FirtInterceptor  拦截之前 ....");
    Object obj = invocation.proceed();
    // 执行目标方法
    System.out.println("FirtInterceptor  拦截之后 ....");
    return obj;
}
```
这个就是自定义的拦截器执行的完整流程

## 2.4 多拦截器

### 插件执行顺序

见 [MyBatis 插件的执行顺序](https://pdyun.cc/2021/08/20/MyBatis-%E6%8F%92%E4%BB%B6%E7%9A%84%E6%89%A7%E8%A1%8C%E9%A1%BA%E5%BA%8F/)

### 插件执行流程

如果我们有多个自定义的拦截器，那么他的执行流程是怎么样的呢？比如我们创建了两个 Interceptor 都是用来拦截 Executor 的query方法，一个是用来执行逻辑A 一个是用来执行逻辑B的。

单个拦截器的执行流程

![](/images/img-125.png)

如果说对象被代理了多次，这里会继续调用下一个插件的逻辑，再走一次Plugin的invoke()方法。这里我们需要关注一下有多个插件的时候的运行顺序。
 
配置的顺序和执行的顺序是相反的。InterceptorChain的List是按照插件从上往下的顺序解析、添加的。

而创建代理的时候也是按照list的顺序代理。执行的时候当然是从最后代理的对象开始。

![](/images/img-126.png)

这个我们可以通过实际的案例来得到验证，最后来总结下Interceptor的相关对象的作用

对象 | 作用
---|---
Interceptor | 自定义插件需要实现接口，实现4个方法
InterceptChain | 配置的插件解析后会保存在Configuration的InterceptChain中
Plugin | 触发管理类，还可以用来创建代理对象
Invocation | 对被代理类进行包装，可以调用proceed()调用到被拦截的方法















