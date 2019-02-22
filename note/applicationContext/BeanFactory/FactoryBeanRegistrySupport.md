## FactoryBean 之 FactoryBeanRegistrySupport

---

FactoryBeanRegistrySupport提供了一些FactoryBean的相关方法供BeanFactory内部使用。

PS: 由于FactoryBean是Bean（用&符号获取）,通过FactoryBean生成的Bean也是Bean。下面的文中的FactoryBean指的是用&符号获取的那个Bean，FactoryBean生成的Bean指的是直接用BeanName获取的Bean。

#### 代码分析

##### 管理FactoryBean生产的单例

FactoryBeanRegistrySupport的成员变量：

```java
private final Map<String, Object> factoryBeanObjectCache = new ConcurrentHashMap<>(16);
```

factoryBeanObjectCache是用于存放FactoryBean单例的Map。

普通Bean的单例是由配置决定的，而FactoryBean生成的Bean是由FactoryBean的isSingleton这个方法决定的。

##### FacotryBean的创建

getObjectFromFactoryBean源码:

```java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
  // FactoryBean生成的Bean是单例 且 FactoryBean是单例
	if (factory.isSingleton() && containsSingleton(beanName)) {
		synchronized (getSingletonMutex()) {
      // 尝试从缓冲中拿
			Object object = this.factoryBeanObjectCache.get(beanName);
      // 缓存没有
      if (object == null) {
        // 尝试创建
				object = doGetObjectFromFactoryBean(factory, beanName);
				Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
				if (alreadyThere != null) {
					object = alreadyThere;
				}
				else {
					if (shouldPostProcess) {
						if (isSingletonCurrentlyInCreation(beanName)) {
							return object;
						}
						beforeSingletonCreation(beanName);
						try {
							object = postProcessObjectFromFactoryBean(object, beanName);
						}
						catch (Throwable ex) {
							throw new BeanCreationException(beanName,
									"Post-processing of FactoryBean's singleton object failed", ex);
						}
						finally {
							afterSingletonCreation(beanName);
						}
					}
					if (containsSingleton(beanName)) {
						this.factoryBeanObjectCache.put(beanName, object);
					}
				}
			}
			return object;
		}
	}
	else {
		Object object = doGetObjectFromFactoryBean(factory, beanName);
		if (shouldPostProcess) {
			try {
				object = postProcessObjectFromFactoryBean(object, beanName);
			}
			catch (Throwable ex) {
				throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
			}
		}
		return object;
	}
}
```

- 代码首先是单例多例会有不同处理，只有在FactoryBean和FacotryBean都是单例的情况下才会去尝试从缓存中取Bean。


#### FactoryBean和FactoryBean生成的Bean的辨析

||FactoryBean|FactoryBean生成的Bean|
|--|--|--|
|种类|FactoryBean是一个普通Bean，它和普通的Bean总体上没有区别|FactoryBean生成的Bean是一种特殊的Bean|
|创建途径|通过CreateBean方法创建|通过FactoryBean的getObject方法|
|是否是单例|配置决定|由FactoryBean的isSingleton方法决定的|
|单例存放点|DefaultSingletonBeanRegistry的SingletonObjects|FactoryBeanRegistrySupport的factoryBeanObjectCache|
|创建方式|可以像普通Bean进行依赖注入|因为是通过getObject方法生成，无法直接注入依赖|
|获取方法|需要在BeanName前加 '&' getBean("&" + BeanName)|直接用BeanName|
