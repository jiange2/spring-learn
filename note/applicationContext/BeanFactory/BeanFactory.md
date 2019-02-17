## BeanFactory

---

#### 目录

1. ![BeanFactory简介](/note/applicationContext/BeanFactory/BeanFactroy简介.md)
2. ![BeanDefinition]((/note/applicationContext/BeanFactory/BeanDefinition.md)
3. Aliase
   - ![Aliase](/note/??)
4. SingleTon
5. temp

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

### 2、BeanFactory基础概念介绍

** Name和Aliase :**



** BeanDefinition :**

![BeanDefinition详情](/note/applicationContext/BeanFactory/BeanDefinition.md)

** BeanProvider :**

这是4.3之后的新特性，主要是提供了获取bean的其他方式，以及解决byType查找时，非唯一的情况。

---

### 3、BeanFactory

![BeanFactory继承关系图](/image/ApplicationContext/BeanFactory/DefaultListableBeanFactory.png)

** AliasRegistry ：** 这个接口包含一些Alias注册和获取的方法。

** singletonBeanRegistry : **

** BeanFactory : **

---

### 4、Alias注册和获取



---

### 5、BeanDefinitionRegistry

```Java
public interface BeanDefinitionRegistry extends AliasRegistry {

	void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException;

	void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

	BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

	boolean containsBeanDefinition(String beanName);

	String[] getBeanDefinitionNames();

	int getBeanDefinitionCount();

	boolean isBeanNameInUse(String beanName);

}

```

- BeanDefinitionRegistry提供了BeanDefinination的管理和访问方法。

- BeanDefinitionRegistry继承了AliasRegistry,因为Alias其实也是BeanDefinition的一个特性。

#### 6、DefaultListableBeanFactory
