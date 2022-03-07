title: Spring 循环依赖-AOP详解
author: Peilin Deng
tags:
  - Spring
categories:
  - 摘抄笔记
date: 2022-02-24 19:10:00
---
<!DOCTYPE html>
<html>
<head>
<meta  charset=utf-8"/>
    <style>
        mark {
            background-color:#f0e0ed; 
            color:#c7254e;
     	 }
    </style>
                     

该文章引用自: [《当Spring AOP遇上循环依赖》](https://blog.csdn.net/chaitoudaren/article/details/105060882)

# 前言
问：Spring如何解决<mark>循环依赖</mark>？

答：Spring通过<mark>提前曝光</mark>机制，利用<mark>三级缓存</mark>解决<mark>循环依赖</mark>（这原理还是挺简单的，参考：三级缓存、图解循环依赖原理）

--------------------------------------------
问：只用一级缓存可以能解决问题吗?

答：<mark>不能</mark>. 在整个处理中, 缓存中放的<mark>是成品和半成品对象</mark>, 如果只使用一级缓存, 成品和半成品对象都存在一个缓存, <mark>有可能获取Bean的时候会获取到半成品对象</mark>, 此时的对象是无法使用的, 因此至少都需要一级缓存和二级缓存把成品对象和半成品对象分开.

--------------------------------------------
再问：<mark>只使用二级缓存</mark>能解决循环依赖吗? 为什么需要<mark>三级缓存</mark>？

答：Spring通过<mark>提前曝光</mark>，直接曝光到<mark>二级缓存</mark>已经可以解决循环依赖问题了; 使用<mark>三级缓存的本质在于AOP代理问题</mark>, 因为三级缓存中存放的是<mark>ObjectFactory(也可以说是放的一个lambda表达式)</mark>,存在AOP的时候,就会调用它来完成代理对象的覆盖过程.
```java
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
```

--------------------------------------------
再细问：如果循环依赖的时候，所有类又都需要<mark>Spring AOP</mark>自动代理，那Spring如何提前曝光？曝光的是<mark>原始bean</mark>还是<mark>代理后的bean</mark>？

这些问题算是Spring源码的压轴题了，如果这些问题都弄明白，恭喜你顺利结业Spring源码了。就单单对Spring这一块的理解，不夸张的说可以达到阿里水准了

# 源码分析
进入正题，在Spring<mark>创建Bean</mark>的核心代码<mark>doGetBean</mark>中，在<mark>实例化bean</mark>之前，会先尝试从<mark>三级缓存</mark>获取bean，这也是Spring解决循环依赖的开始

## (一) 缓存中获取bean
```java
// AbstractBeanFactory.java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		// 2. 尝试从缓存中获取bean
		Object sharedInstance = getSingleton(beanName);
		...
}

```
```java
protected Object getSingleton(String beanName, Boolean allowEarlyReference) {
  // 从一级缓存获取，key=beanName value=bean
  Object singletonObject = this.singletonObjects.get(beanName);
  if (singletonObject <mark> null && isSingletonCurrentlyInCreation(beanName)) {
    synchronized (this.singletonObjects) {
      // 从二级缓存获取，key=beanName value=bean
      singletonObject = this.earlySingletonObjects.get(beanName);
      // 是否允许循环引用
      if (singletonObject <mark> null && allowEarlyReference) {
        /**
           * 三级缓存获取，key=beanName value=objectFactory，objectFactory中存储getObject()方法用于获取提前曝光的实例
           *
           * 而为什么不直接将实例缓存到二级缓存，而要多此一举将实例先封装到objectFactory中？
           * 主要关键点在getObject()方法并非直接返回实例，而是对实例又使用
           * SmartInstantiationAwareBeanPostProcessor的getEarlyBeanReference方法对bean进行处理
           *
           * 也就是说，当spring中存在该后置处理器，所有的单例bean在实例化后都会被进行提前曝光到三级缓存中，
           * 但是并不是所有的bean都存在循环依赖，也就是三级缓存到二级缓存的步骤不一定都会被执行，有可能曝光后直接创建完成，没被提前引用过，
           * 就直接被加入到一级缓存中。因此可以确保只有提前曝光且被引用的bean才会进行该后置处理
            */
        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
        if (singletonFactory != null) {
          /**
             * 通过getObject()方法获取bean，通过此方法获取到的实例不单单是提前曝光出来的实例，
             * 它还经过了SmartInstantiationAwareBeanPostProcessor的getEarlyBeanReference方法处理过。
             * 这也正是三级缓存存在的意义，可以通过重写该后置处理器对提前曝光的实例，在被提前引用时进行一些操作
              */
          singletonObject = singletonFactory.getObject();
          // 将三级缓存生产的bean放入二级缓存中
          this.earlySingletonObjects.put(beanName, singletonObject);
          // 删除三级缓存
          this.singletonFactories.remove(beanName);
        }
      }
    }
  }
  return singletonObject;
}
```

**三级缓存分别是：**
- <mark>singletonObject</mark>：一级缓存，该缓存<mark>key = beanName, value = bean</mark>;这里的bean是已经创建完成的，该bean经历过实例化->属性填充->初始化以及各类的后置处理。因此，一旦需要获取bean时，我们第一时间就会寻找一级缓存
- <mark>earlySingletonObjects</mark>：二级缓存，该缓存<mark>key = beanName, value = bean</mark>;这里跟一级缓存的区别在于，该缓存所获取到的bean是提前曝光出来的，是还没创建完成的。也就是说获取到的bean只能确保已经进行了实例化，但是属性填充跟初始化肯定还没有做完，因此该bean还没创建完成，仅仅能作为指针提前曝光，被其他bean所引用
- 
- <mark>singletonFactories</mark>：三级缓存，该缓存<mark>key = beanName, value = beanFactory</mark>;在bean实例化完之后，属性填充以及初始化之前，如果允许提前曝光，spring会将实例化后的bean提前曝光，也就是把该bean转换成<mark>beanFactory</mark>并加入到三级缓存。在需要引用提前曝光对象时再通过<mark>singletonFactory.getObject()</mark>获取。

这里抛出问题，如果我们直接将提前曝光的对象放到二级缓存<mark>earlySingletonObjects</mark>，Spring循环依赖时直接取就可以解决循环依赖了，为什么还要三级缓存<mark>singletonFactory</mark>然后再通过<mark>getObject()</mark>来获取呢？这不是多此一举？

## (二) 三级缓存的添加
我们回到添加三级缓存，添加<mark>SingletonFactory</mark>的地方，看看<mark>getObject()</mark>到底做了什么操作
```java
	this.addSingletonFactory(beanName, () -> {
		return this.getEarlyBeanReference(beanName, mbd, bean);
	});
```

可以看到在返回<mark>getObject()</mark>时，多做了一步<mark>getEarlyBeanReference</mark>操作，这步操作是<mark>BeanPostProcess</mark>的一种，也就是给子类重写的一个后处理器，目的是用于<mark>被提前引用</mark>时进行拓展。即：曝光的时候并不调用该后置处理器，只有曝光，且被提前引用的时候才调用，确保了<mark>被提前引用</mark>这个时机触发。

## (三) 提前曝光代理earlyProxyReferences
因此所有的重点都落到了<mark>getEarlyBeanReference</mark>上，<mark>getEarlyBeanReference</mark>方法是<mark>SmartInstantiationAwareBeanPostProcessor</mark>所规定的接口。再通过UML的类图查看实现类，仅有<mark>AbstractAutoProxyCreator</mark>进行了实现。也就是说，除了用户在子类重写，否则仅有<mark>AbstractAutoProxyCreator</mark>一种情况
```java
	// AbstractAutoProxyCreator.java
	public Object getEarlyBeanReference(Object bean, String beanName) {
		// 缓存当前bean，表示该bean被提前代理了
		Object cacheKey = getCacheKey(bean.getClass(), beanName);
		this.earlyProxyReferences.put(cacheKey, bean);
		// 对bean进行提前Spring AOP代理
		return wrapIfNecessary(bean, beanName, cacheKey);
	}
```

<mark>wrapIfNecessary</mark>是用于Spring AOP自动代理的。Spring将当前bean缓存到<mark>earlyProxyReferences</mark>中标识<mark>提前曝光的bean</mark>在被提前引用之前，然后进行了Spring AOP代理。

但是经过Spring AOP代理后的bean就已经不再是原来的bean了，经过代理后的bean是一个全新的bean，也就是说代理前后的2个bean连内存地址都不一样了。这时将再引出新的问题：**B提前引用A将引用到A的代理，这是符合常理的，但是最原始的bean A在B完成创建后将继续创建，那么Spring Ioc最后返回的Bean是Bean A呢还是经过代理后的Bean呢？**

这个问题我们得回到Spring AOP代理，Spring AOP代理时机有2个：

1. 当自定义了<mark>TargetSource</mark>，则在bean实例化前完成Spring AOP代理并且直接发生短路操作，返回bean
2. 正常情况下，都是在bean初始化后进行Spring AOP代理
3. 如果要加上今天说的提前曝光代理，<mark>getEarlyBeanReference</mark>可以说3种

第一种情况就没什么好探究的了，直接短路了，根本没有后续操作。而我们关心的是第二种情况，在Spring初始化后置处理器中发生的Spring AOP代理
```java
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			// 调用bean初始化后置处理器处理
			Object current = processor.postProcessAfterInitialization(result, beanName);
			if (current <mark> null) {
				return result;
			}
			result = current;
		}
		return result;
	}
```
```java
	// AbstractAutoProxyCreator.java
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
		if (bean != null) {
			// 获取缓存key
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			// 查看该bean是否被Spring AOP提前代理！而缓存的是原始的bean，因此如果bean被提前代理过，这此处会跳过
			// 如果bean没有被提前代理过，则进入AOP代理
			if (this.earlyProxyReferences.remove(cacheKey) != bean) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```

<mark>earlyProxyReferences</mark>是不是有点熟悉，是的，这就是我们刚刚提前曝光并且进行Spring AOP提前代理时缓存的原始bean，如果缓存的原始bean跟当前的bean是一至的，那么就不进行Spring AOP代理了！返回原始的bean
```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {
		try {
			//
			/**
			 * 4. 填充属性
			 * 如果@Autowired注解属性，则在上方完成解析后，在这里完成注入
			 *
			 * @Autowired
			 * private Inner inner;
			 */
			populateBean(beanName, mbd, instanceWrapper);
			// 5. 初始化
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		// 6. 存在提前曝光情况下
		if (earlySingletonExposure) {
			// earlySingletonReference：二级缓存，缓存的是经过提前曝光提前Spring AOP代理的bean
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				// exposedObject跟bean一样，说明初始化操作没用应用Initialization后置处理器(指AOP操作)改变exposedObject
				// 主要是因为exposedObject如果提前代理过，就会跳过Spring AOP代理，所以exposedObject没被改变，也就等于bean了
				if (exposedObject <mark> bean) {
					// 将二级缓存中的提前AOP代理的bean赋值给exposedObject，并返回
					exposedObject = earlySingletonReference;
				}
				// 引用都不相等了，也就是现在的bean已经不是当时提前曝光的bean了
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					// dependentBeans也就是B, C, D
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					// 被依赖检测异常
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}
```

这个时候我们需要理清一下3个变量

1. <mark>earlySingletonReference</mark>：二级缓存，缓存的是经过提前曝光提前AOP代理的bean
2. <mark>bean</mark>：这个就是经过了实例化、填充、初始化的bean
3. <mark>exposedObject</mark>：这个是经过<mark>了AbstractAutoProxyCreator</mark>的<mark>postProcessAfterInitialization</mark>处理过后的bean，但是在其中因为发现当前bean已经被<mark>earlyProxyReferences</mark>缓存，所以并没有进行AOP处理，而是直接跳过，因此还是跟第2点一样的bean

理清这3个变量以后，就会发现，<mark>exposedObject = earlySingletonReference</mark>;
AOP代理过的Bean赋值给<mark>了exposedObject</mark>并返回，这时候用户拿到的bean就是AOP代理过后的bean了，一切皆大欢喜了。

但是中间还有一个问题！提前曝光的bean在提前引用时被Spring AOP代理了，但是此时的bean只是经过了实例化的bean，还没有进行@Autowire的注入啊！<mark>也就是说此时代理的bean里面自动注入的属性是空的！<mark>

## (四) 提前AOP代理对象的 属性填充、初始化
是的，确实在<mark>Spring AOP提前代理</mark>后没有经过属性<mark>填充</mark>和<mark>初始化</mark>。那么这个代理又是如何保证依赖属性的注入的呢？答案回到Spring AOP最早最早讲的[JDK动态代理](https://blog.csdn.net/chaitoudaren/article/details/104833279)上找，JDK动态代理时，会将<mark>目标对象target</mark>保存在最后生成的代理$proxy中，当调用$proxy方法时会<mark>回调h.invoke</mark>，而<mark>h.invoke</mark>又会回调<mark>目标对象target</mark>的原始方法。因此，其实在Spring AOP动态代理时，<mark>原始bean</mark>已经被保存在提前曝光代理中了。而后<mark>原始Bean</mark>继续完成<mark>属性填充</mark>和<mark>初始化</mark>操作。因为AOP代理<mark>$proxy</mark>中保存着<mark>traget</mark>也就是是原始bean的引用，因此后续<mark>原始bean</mark>的完善，也就相当于Spring AOP中的<mark>target</mark>的完善，这样就保证了Spring AOP的<mark>属性填充</mark>与<mark>初始化</mark>了！

## (五) 循环依赖遇上Spring AOP 图解
为了帮助大家理解，这里灵魂画手画张流程图帮助大家理解

首先又bean A，bean B，他们循环依赖注入，同时bean A还需要被Spring AOP代理，例如事务管理或者日志之类的操作。

原始bean A，bean B图中用a，b表示，而代理后的bean A我们用aop.a表示

![](/images/img-89.png)