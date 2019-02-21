## FactoryBeanRegistrySupport

---

FactoryBeanRegistrySupport提供了一些FactoryBean的相关方法供BeanFactory内部使用。

#### 管理FactoryBean生产的单例

FactoryBeanRegistrySupport的成员变量：

```java
private final Map<String, Object> factoryBeanObjectCache = new ConcurrentHashMap<>(16);
```

factoryBeanObjectCache是用于存放FactoryBean单例的Map。




#### FactoryBean和FactoryBean生成的Bean的辨析

||FactoryBean|FactoryBean生成的Bean|
|--|--|--|
|种类|FactoryBean是一个普通Bean，它和普通的Bean总体上没有区别|FactoryBean生成的Bean是一种特殊的Bean|
|创建途径|通过CreateBean方法创建|通过FactoryBean的getObject方法|
|是否是单例|配置决定|由FactoryBean的isSingleton方法决定的|
|单例存放点|DefaultSingletonBeanRegistry的SingletonObjects|FactoryBeanRegistrySupport的factoryBeanObjectCache|
|创建方式|可以像普通Bean进行依赖注入|因为是通过getObject方法生成，无法直接注入依赖|
|获取方法|需要在BeanName前加 '&' getBean("&" + BeanName)|直接用BeanName|
