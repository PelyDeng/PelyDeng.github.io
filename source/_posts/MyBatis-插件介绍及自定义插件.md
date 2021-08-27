title: MyBatis 插件介绍及自定义插件
author: Peilin Deng
tags:
  - MyBatis
  - 自定义插件
categories:
  - 摘抄笔记
date: 2021-08-20 18:57:00
---
mybatis自定义插件（拦截器）开发详解
---

###### mybatis插件（准确的说应该是around拦截器，因为接口名是interceptor，而且invocation.proceed要自己调用，配置中叫插件）功能非常强大，**可以让我们无侵入式的对SQL的执行进行干涉，从SQL语句重写、参数注入、结果集返回等每个主要环节，典型的包括权限控制检查与注入、只读库映射、K/V翻译、动态改写SQL。**

> MyBatis 默认支持对**4大对象**（**Executor，StatementHandler，ParameterHandler，ResultSetHandler**）上的方法执行拦截，具体支持的方法为：
>
> - **Executor** (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)，主要用于sql重写。
> - **ParameterHandler** (getParameterObject, setParameters)，用于参数处理。
> - **ResultSetHandler** (handleResultSets, handleOutputParameters)，用于结果集二次处理。
> - **StatementHandler** (prepare, parameterize, batch, update, query)，用于jdbc层的控制。

大多数情况下仅在Executor做插件比如SQL重写、结果集脱敏，ResultSetHandler和StatementHandler仅在高级场景中使用，而且某些场景中非常有价值。

四大对象的在sql执行过程中的调用链如下：

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

编写Interceptor的实现类
---

#### 一. Executor拦截器

###### 案例1:

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


###### 案例2: 带反射

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

#### 二. ResultSetHandler拦截器

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

#### 最后将插件配置到mybatis-config.xml中，如下：

```xml
<!-- mybatis-config.xml  注册插件-->
<plugins>
    <plugin interceptor="com.dpl.mybatis.plugin.SelectPruningColumnPlugin">
        <property name="someProperty" value="100"/>
    </plugin>
</plugins>
```