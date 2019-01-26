## BeanFactory

---

### 1、前言

ApplicationContext将很多不同的组件(LifeCycleProcessor,ResourceLoader,MessageSource,Environment)聚合在一起,而这些组件其实都是为ApplicationContext最核心部分功能IOC容器提供服务。而BeanFactory就是IOC容器的抽象。

| 组件 | 功能 |
| ---- | ---- |
|LifeCycleProcessor|Bean生命周期|
|ResourceLoader| 读取配置和资源|
|MessageSource| I18N配置读取|
|Environment|系统变量管理，Profile|
|ApplicationEvent|事件推送，异步处理能力|

而BeanFactory包含的则是最核心Bean管理功能，包括Bean加载和访问。

---

### 2、BeanFactory接口

在阅读BeanFactory源码之前，我们先看一下BeanFactory提供了些什么功能。

```java
public interface BeanFactory {

	// BeanName 为&打头的Bean就是FactoryBan
	String FACTORY_BEAN_PREFIX = "&";

	Object getBean(String name) throws BeansException;

	<T> T getBean(String name, Class<T> requiredType) throws BeansException;

	Object getBean(String name, Object... args) throws BeansException;

	<T> T getBean(Class<T> requiredType) throws BeansException;

	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

	<T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

	<T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

	boolean containsBean(String name);

	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

	boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

	Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	String[] getAliases(String name);

}
```

可以看到大致是一些获取Bean和判断Bean的各种属性。先简单介绍这里面的一些概念：

**Name和Aliase :**

Spring一般可以通过BeanName和Bean的type来获取bean。那Bean的Name是根据我们配置决定的，主要是跟id和name这两个属性有关系。name的逻辑：
 - 假如配置了id，BeanName就是id。
 - 同样，配置了name，那BeaName就是配置的name。
 - 如果同时配置了id，那BeanName就是id，而配置的name就是Aliase。
 - 如果配置了多个name，会取首个name作为BeanName,其他是Aliase
 - 如果都没配的话就是`appcontext.(全限定类名)#0。`

无论是BeanName或是Aliase都可以通过getBean(String name)的方式来获取bean。

**BeanProvider :**

这是4.3之后的新特性，主要是提供了获取bean的其他方式，以及解决byType查找时，非唯一的情况。

### 3、BeanFactory源码
