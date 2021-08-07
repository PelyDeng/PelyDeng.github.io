title: MyBatis 整合到 Spring 原理
author: Peilin Deng
tags: []
categories:
  - MyBatis
date: 2021-08-07 15:41:00
---
### 1. xml 配置
> 通过在Spring的 applicationContext.xml文件中做以下配置, 指定MyBatis 的mapper.xml 文件扫描路径, MapperScannerConfigurer 是 Mybatis 用于整合 Spring 的核心对象.

<!--more-->

```xml
<!--配置扫描器，将mybatis的接口实现加入到  IOC容器中  -->
<!--
    <mybatis-spring:scan #base-package="com.dpl.crud.dao"/>
-->
    <bean id="mapperScanner" class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="com.gupaoedu.crud.dao"/>
    </bean>
```


### 2. MapperScannerConfigurer 对象
```java
public class MapperScannerConfigurer
    implements BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware, BeanNameAware {
    ......
    ......
}
```
#### 该对象实现了几个接口: BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware, BeanNameAware <br>

> **1). BeanDefinitionRegistryPostProcessor**:  <br>
> ![upload successful](/images/pasted-0.png) <br>

> 如下, 重写了 **++postProcessBeanDefinitionRegistry++**(...)  方法, 在该方法中进行扫描对应的 Mapper 文件

```java
@Override
  public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) {
      processPropertyPlaceHolders();
    }

    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.setMapperFactoryBeanClass(this.mapperFactoryBeanClass);
    if (StringUtils.hasText(lazyInitialization)) {
      scanner.setLazyInitialization(Boolean.valueOf(lazyInitialization));
    }
    scanner.registerFilters();
    // *重点在这里
    scanner.scan(
        StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
  }
```

> 重点在 **scanner.scan**(...)方法

```java
public int scan(String... basePackages) {
        int beanCountAtScanStart = this.registry.getBeanDefinitionCount();
        
        // 继续进入该方法
        this.doScan(basePackages);
        
        if (this.includeAnnotationConfig) {
            AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
        }

        return this.registry.getBeanDefinitionCount() - beanCountAtScanStart;
    }
```

![upload successful](/images/pasted-2.png)

> 该方法有两个实现, 首先进入 **ClassPathMapperScanner** 实现

```java
@Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    // *这里调了父类 ClassPathBeanDefinitionScanner 的doScan 方法, 进入该方法实现
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages)
          + "' package. Please check your configuration.");
    } else {
      processBeanDefinitions(beanDefinitions);
    }
    return beanDefinitions;
  }
```

> 这里调了父类 **ClassPathBeanDefinitionScanner** 的 **doScan**() 方法, 进入该方法实现

```java
protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Assert.notEmpty(basePackages, "At least one base package must be specified");
        
        // *创建 BeanDefinitionHolder 集合, 里面封装的是 BeanDefinition 和 beanName
        Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet();
        
        String[] var3 = basePackages;
        int var4 = basePackages.length;

        for(int var5 = 0; var5 < var4; ++var5) {
            String basePackage = var3[var5];
            Set<BeanDefinition> candidates = this.findCandidateComponents(basePackage);
            Iterator var8 = candidates.iterator();

            while(var8.hasNext()) {
                BeanDefinition candidate = (BeanDefinition)var8.next();
                ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
                candidate.setScope(scopeMetadata.getScopeName());
                String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
                if (candidate instanceof AbstractBeanDefinition) {
                    this.postProcessBeanDefinition((AbstractBeanDefinition)candidate, beanName);
                }

                if (candidate instanceof AnnotatedBeanDefinition) {
                    AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition)candidate);
                }
                
                // 检查每个 BeanDefinition 是否在容器中存在, 不存在则返回true
                if (this.checkCandidate(beanName, candidate)) {
                    // 创建 BeanDefinitionHolder 对象封装 BeanDefinition
                    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                    definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                    beanDefinitions.add(definitionHolder);
                    
                    // *将创建好的对象注册到 BeanDefinitionRegistry 容器中交由Spring管理
                    this.registerBeanDefinition(definitionHolder, this.registry);
                }
            }
        }

        return beanDefinitions;
    }
```

> 上面将mapper对象封装好了注册到 BeanDefinitionRegistry 容器中交由Spring管理, 接着返回 BeanDefinitionHolder 集合继续处理

```java
@Override
  public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages)
          + "' package. Please check your configuration.");
    } else {
    
      // *接下来处理BeanDefinition
      processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
  }
```

```java
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
      definition = (GenericBeanDefinition) holder.getBeanDefinition();
      String beanClassName = definition.getBeanClassName();
      LOGGER.debug(() -> "Creating MapperFactoryBean with name '" + holder.getBeanName() + "' and '" + beanClassName
          + "' mapperInterface");

      definition.getConstructorArgumentValues().addGenericArgumentValue(beanClassName); // issue #59
      
      // mapper接口是Bean的原始类，但是Bean的实际类被替换成了 MapperFactoryBean
      // 这里的 this.mapperFactoryBeanClass 是 MapperFactoryBean.class 对象
      // private Class<? extends MapperFactoryBean> mapperFactoryBeanClass = MapperFactoryBean.class;
      definition.setBeanClass(this.mapperFactoryBeanClass);

      definition.getPropertyValues().add("addToConfig", this.addToConfig);

      boolean explicitFactoryUsed = false;
      if (StringUtils.hasText(this.sqlSessionFactoryBeanName)) {
        definition.getPropertyValues().add("sqlSessionFactory",
            new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionFactory != null) {
        definition.getPropertyValues().add("sqlSessionFactory", this.sqlSessionFactory);
        explicitFactoryUsed = true;
      }

      if (StringUtils.hasText(this.sqlSessionTemplateBeanName)) {
        if (explicitFactoryUsed) {
          LOGGER.warn(
              () -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate",
            new RuntimeBeanReference(this.sqlSessionTemplateBeanName));
        explicitFactoryUsed = true;
      } else if (this.sqlSessionTemplate != null) {
        if (explicitFactoryUsed) {
          LOGGER.warn(
              () -> "Cannot use both: sqlSessionTemplate and sqlSessionFactory together. sqlSessionFactory is ignored.");
        }
        definition.getPropertyValues().add("sqlSessionTemplate", this.sqlSessionTemplate);
        explicitFactoryUsed = true;
      }

      if (!explicitFactoryUsed) {
        LOGGER.debug(() -> "Enabling autowire by type for MapperFactoryBean with name '" + holder.getBeanName() + "'.");
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
      }
      definition.setLazyInit(lazyInitialization);
    }
  }
```

> 这里 mapper接口是Bean的原始类，但是Bean的实际类被替换成了 MapperFactoryBean, MapperFactoryBean 对象又是什么东西?

### 3. MapperFactoryBean 对象
```java
// *要点一: 该类继承了 SqlSessionDaoSupport, SqlSessionDaoSupport 对象里面可以获取 SqlSessionTemplate 对象
// SqlSessionTemplate 对象是 DefaultSqlSession 的替代品, 实现同一事务的情况下保证线程安全的, 每次请求的时候都会创建一个新的 SqlSession
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {

  private Class<T> mapperInterface;

  private boolean addToConfig = true;

  public MapperFactoryBean() {
    // intentionally empty
  }

  public MapperFactoryBean(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }

  @Override
  protected void checkDaoConfig() {
    super.checkDaoConfig();

    notNull(this.mapperInterface, "Property 'mapperInterface' is required");

    Configuration configuration = getSqlSession().getConfiguration();
    if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
      try {
        configuration.addMapper(this.mapperInterface);
      } catch (Exception e) {
        logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
        throw new IllegalArgumentException(e);
      } finally {
        ErrorContext.instance().reset();
      }
    }
  }

  // *要点二: 该类实现了 FactoryBean<T> 接口, 重写 getObject() 方法,
  // 从 SqlSessionTemplate 对象中获取 Mapper 接口的代理对象
  // 后面的逻辑与之前未整合 Spring 的逻辑一致了.....
  @Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }

  @Override
  public Class<T> getObjectType() {
    return this.mapperInterface;
  }

  @Override
  public boolean isSingleton() {
    return true;
  }

  public void setMapperInterface(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }


  public Class<T> getMapperInterface() {
    return mapperInterface;
  }

  public void setAddToConfig(boolean addToConfig) {
    this.addToConfig = addToConfig;
  }

  public boolean isAddToConfig() {
    return addToConfig;
  }
}
```

> 允许注入MyBatis映射器接口的BeanFactory。 可以使用SqlSessionFactory或预配置的SqlSessionTemplate进行设置。<br>
> ***要点一:** 该类继承了 SqlSessionDaoSupport, SqlSessionDaoSupport 对象里面可以获取 SqlSessionTemplate 对象 <br>
> **SqlSessionTemplate** 对象是 DefaultSqlSession 的替代品, **实现同一事务的情况下还保证线程安全的, 每次请求的时候都会创建一个新的 DefualtSqlSession** <br>

> ***要点二:** 该类实现了 FactoryBean<T> 接口, 重写 getObject() 方法, <br>
> 从 SqlSessionTemplate 对象中获取 Mapper 接口的代理对象 <br>
> 后面的逻辑与之前未整合 Spring 的逻辑一致了..... <br>

### 4. SqlSessionTemplate 对象
#### 为什么说 SqlSessionTemplate 在实现同一事务的情况下还保证线程安全的?

```java
public class SqlSessionTemplate implements SqlSession, DisposableBean {

  private final SqlSessionFactory sqlSessionFactory;

  private final ExecutorType executorType;

  private final SqlSession sqlSessionProxy;

  private final PersistenceExceptionTranslator exceptionTranslator;

  ......
  
  public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
    notNull(executorType, "Property 'executorType' is required");

    this.sqlSessionFactory = sqlSessionFactory;
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;
    
    // 1. 在初始化的时候通过这里创建 SqlSessionFactory 代理类, 
    // 该代理类实现SqlSession接口，定义了方法拦截器，如果调用代理类实例中实现SqlSession接口定义的方法，
    // 该调用则被导向 SqlSessionInterceptor 的invoke方法
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class }, 
        new SqlSessionInterceptor()
    );
  }
  
  ......
  
  private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      
      // 2. 为什么说 SqlSessionTemplate 在实现同一事务的情况下还保证线程安全的?
      // 此处则是关键, 进入该方法 getSqlSession(...)
      SqlSession sqlSession = getSqlSession(SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);
          
      try {
        // 调用从Spring的事物上下文获取事物范围内的sqlSession对象
        Object result = method.invoke(sqlSession, args);
        
        //然后判断一下当前的sqlSession是否被Spring托管 如果未被Spring托管则自动commit
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator
              .translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }

}
```

> 1. 在初始化的时候在构造方法中创建 SqlSessionFactory 代理类, <br> 该代理类实现SqlSession接口，定义了方法拦截器，如果调用代理类实例中实现SqlSession接口定义的方法，<br>
该调用则被导向 SqlSessionInterceptor 的invoke方法 <br>
**(代理对象的 InvocationHandler 就是 SqlSessionInterceptor，如果把它命名为SqlSessionInvocationHandler则更好理解！）**

> 2. 为什么说 SqlSessionTemplate 在实现同一事务的情况下还保证线程安全的? <br>
此处则是关键, 进入该方法 <br>
SqlSession sqlSession = getSqlSession(SqlSessionTemplate.this.sqlSessionFactory, <br> &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 
SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);

```java
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
    notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);

    // 进入 TransactionSynchronizationManager.getResource(...) 方法
    // 根据sqlSessionFactory从当前线程对应的资源map中获取SqlSessionHolder，
    // 当sqlSessionFactory创建了sqlSession，
    // 就会在事务管理器中添加一对映射：key为sqlSessionFactory，value为SqlSessionHolder，该类保存sqlSession及执行方式
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    //从SqlSessionHolder中提取SqlSession对象
    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
      return session;
    }

    LOGGER.debug(() -> "Creating a new SqlSession");
    
    // 如果当前事物管理器中获取不到SqlSessionHolder对象就重新创建一个
    session = sessionFactory.openSession(executorType);

    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

    return session;
  }
```

> **进入 TransactionSynchronizationManager.getResource(...) 方法** <br>
> SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

```java
@Nullable
    public static Object getResource(Object key) {
        Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
        
        // 这里 value 为返回结果, 可以进去看这个 value 是什么对象, 进入 doGetResource() 方法
        Object value = doGetResource(actualKey);
        
        if (value != null && logger.isTraceEnabled()) {
            logger.trace("Retrieved value [" + value + "] for key [" + actualKey + "] bound to thread [" + Thread.currentThread().getName() + "]");
        }
        return value;
    }
```

> **这里 value 为返回结果, 可以进去看这个 value 是什么对象, 进入 doGetResource() 方法** <br>
> Object value = doGetResource(actualKey);


```java
@Nullable
    private static Object doGetResource(Object actualKey) {
    
        // 最后从 resources 这个集合中获取的对象, 然而这个 resources 就是一个 ThreadLocal 对象, 保证了线程安全
        Map<Object, Object> map = (Map)resources.get();
        
        if (map == null) {
            return null;
        } else {
            Object value = map.get(actualKey);
            if (value instanceof ResourceHolder && ((ResourceHolder)value).isVoid()) {
                map.remove(actualKey);
                if (map.isEmpty()) {
                    resources.remove();
                }
                value = null;
            }
            return value;
        }
    }
```

> 最后从 resources 这个集合中获取的对象, 然而这个 resources 就是一个 ThreadLocal 对象, 保证了线程安全 <br>
> Map<Object, Object> map = (Map)resources.get();

```java
public abstract class TransactionSynchronizationManager {
    ......
    
    private static final ThreadLocal<Map<Object, Object>> resources = new NamedThreadLocal("Transactional resources");

    ......
}
```

### 5. Mybatis 不是有 SqlSessionManager 了吗？为什么又提供了 SqlSessionTemplate？
> **SqlSessionManager**: SqlSessionManager 是由开发者自身决定如何使用 SqlSession 的, 是适合在不整合 Spring 框架的时候使用。

> **SqlSessionTemplate**: SqlSessionTemplate 是 MyBatis 专门为 Spring 提供的，支持 Spring 框架的一个 SqlSession 获取接口。<br>
> 主要是为了继承 Spring，并同时将是否共用 SqlSession 的权限交给 Spring 去管理。



---