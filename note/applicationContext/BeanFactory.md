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

### 2、BeanFactory基础概念介绍

**Name和Aliase :**

Spring一般可以通过BeanName和Bean的type来获取bean。那Bean的Name是根据我们配置决定的，主要是跟id和name这两个属性有关系。name的逻辑：
 - 假如配置了id，BeanName就是id。
 - 同样，配置了name，那BeaName就是配置的name。
 - 如果同时配置了id，那BeanName就是id，而配置的name就是Aliase。
 - 如果配置了多个name，会取首个name作为BeanName,其他是Aliase
 - 如果都没配的话就是`appcontext.(全限定类名)#0。`

无论是BeanName或是Aliase都可以通过getBean(String name)的方式来获取bean。

**BeanDefinition :**



**BeanProvider :**

这是4.3之后的新特性，主要是提供了获取bean的其他方式，以及解决byType查找时，非唯一的情况。

---

### 3、BeanFactory

![BeanFactory继承关系图](/image/ApplicationContext/BeanFactory/DefaultListableBeanFactory.png)

** AliasRegistry ：** 这个接口包含一些Alias注册和获取的方法。

** singletonBeanRegistry : **

** BeanFactory : **

---

### 4、Bean注册

**AliasRegistry: **

```java

public interface AliasRegistry {

	void registerAlias(String name, String alias);

	void removeAlias(String alias);

	boolean isAlias(String name);

	String[] getAliases(String name);

}

```

AliasRegistry提供了管理访问别名的方法。

**register alias: **

```Java
public void registerAlias(String name, String alias) {
	Assert.hasText(name, "'name' must not be empty");
	Assert.hasText(alias, "'alias' must not be empty");
	synchronized (this.aliasMap) {
    // name == alias,相当于要注册的alias其实就是name
		if (alias.equals(name)) {
			this.aliasMap.remove(alias);
			if (logger.isDebugEnabled()) {
				logger.debug("Alias definition '" + alias + "' ignored since it points to same name");
			}
		}
		else {
			String registeredName = this.aliasMap.get(alias);
			if (registeredName != null) {
        //注册过一摸一样的了
				if (registeredName.equals(name)) {
					return;
				}
        //要改alias的name，如果allowAliasOverriding是false（默认true）就抛异常
				if (!allowAliasOverriding()) {
					throw new IllegalStateException("Cannot define alias '" + alias + "' for name '" +
							name + "': It is already registered for name '" + registeredName + "'.");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Overriding alias '" + alias + "' definition for registered name '" +
							registeredName + "' with new target name '" + name + "'");
				}
			}
      //检查循环
			checkForAliasCircle(name, alias);
			this.aliasMap.put(alias, name);
			if (logger.isTraceEnabled()) {
				logger.trace("Alias definition '" + alias + "' registered for name '" + name + "'");
			}
		}
	}
}
```

aliasMap存放了alias -> name的映射。可以看出来要用alias找name是比较方便的，但是要从name找alias就需遍历整个map。

** alias循环：**

从上面源码看，在把alias注册的最后一步需要检查alias循环，那什么是alias循环呢？

```Java
protected void checkForAliasCircle(String name, String alias) {
	if (hasAlias(alias, name)) {
		throw new IllegalStateException("Cannot register alias '" + alias +
				"' for name '" + name + "': Circular reference - '" +
				name + "' is a direct or indirect alias for '" + alias + "' already");
	}
}

public boolean hasAlias(String name, String alias) {
	for (Map.Entry<String, String> entry : this.aliasMap.entrySet()) {
		String registeredName = entry.getValue();
		if (registeredName.equals(name)) {
			String registeredAlias = entry.getKey();
			if (registeredAlias.equals(alias) || hasAlias(registeredAlias, alias)) {
				return true;
			}
		}
	}
	return false;
}
```


**BeanDefinitionRegistry: **

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

BeanDefinitionRegistry提供了BeanDefinination的管理和访问方法。

BeanDefinitionRegistry继承了AliasRegistry,因为Alias其实也是BeanDefinition的一个特性。

---

### 3、BeanFactory接口

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



### 4、BeanFactory源码

##### getBean by name：

```java
@Override
public Object getBean(String name) throws BeansException {
	return doGetBean(name, null, null, false);
}

protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
		@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

	final String beanName = transformedBeanName(name);
	Object bean;

	// Eagerly check singleton cache for manually registered singletons.
	Object sharedInstance = getSingleton(beanName);
	if (sharedInstance != null && args == null) {
		if (logger.isTraceEnabled()) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
						"' that is not fully initialized yet - a consequence of a circular reference");
			}
			else {
				logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
			}
		}
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}

	else {
		// Fail if we're already creating this bean instance:
		// We're assumably within a circular reference.
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}

		// Check if bean definition exists in this factory.
		BeanFactory parentBeanFactory = getParentBeanFactory();
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// Not found -> check parent.
			String nameToLookup = originalBeanName(name);
			if (parentBeanFactory instanceof AbstractBeanFactory) {
				return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
						nameToLookup, requiredType, args, typeCheckOnly);
			}
			else if (args != null) {
				// Delegation to parent with explicit args.
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else if (requiredType != null) {
				// No args -> delegate to standard getBean method.
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
			else {
				return (T) parentBeanFactory.getBean(nameToLookup);
			}
		}

		if (!typeCheckOnly) {
			markBeanAsCreated(beanName);
		}

		try {
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			checkMergedBeanDefinition(mbd, beanName, args);

			// Guarantee initialization of beans that the current bean depends on.
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dep : dependsOn) {
					if (isDependent(beanName, dep)) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
					}
					registerDependentBean(dep, beanName);
					try {
						getBean(dep);
					}
					catch (NoSuchBeanDefinitionException ex) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
					}
				}
			}

			// Create bean instance.
			if (mbd.isSingleton()) {
				sharedInstance = getSingleton(beanName, () -> {
					try {
						return createBean(beanName, mbd, args);
					}
					catch (BeansException ex) {
						// Explicitly remove instance from singleton cache: It might have been put there
						// eagerly by the creation process, to allow for circular reference resolution.
						// Also remove any beans that received a temporary reference to the bean.
						destroySingleton(beanName);
						throw ex;
					}
				});
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}

			else if (mbd.isPrototype()) {
				// It's a prototype -> create a new instance.
				Object prototypeInstance = null;
				try {
					beforePrototypeCreation(beanName);
					prototypeInstance = createBean(beanName, mbd, args);
				}
				finally {
					afterPrototypeCreation(beanName);
				}
				bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
			}

			else {
				String scopeName = mbd.getScope();
				final Scope scope = this.scopes.get(scopeName);
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
				}
				try {
					Object scopedInstance = scope.get(beanName, () -> {
						beforePrototypeCreation(beanName);
						try {
							return createBean(beanName, mbd, args);
						}
						finally {
							afterPrototypeCreation(beanName);
						}
					});
					bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException(beanName,
							"Scope '" + scopeName + "' is not active for the current thread; consider " +
							"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
							ex);
				}
			}
		}
		catch (BeansException ex) {
			cleanupAfterBeanCreationFailure(beanName);
			throw ex;
		}
	}

	// Check if required type matches the type of the actual bean instance.
	if (requiredType != null && !requiredType.isInstance(bean)) {
		try {
			T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
			if (convertedBean == null) {
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
			return convertedBean;
		}
		catch (TypeMismatchException ex) {
			if (logger.isTraceEnabled()) {
				logger.trace("Failed to convert bean '" + name + "' to required type '" +
						ClassUtils.getQualifiedName(requiredType) + "'", ex);
			}
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
	}
	return (T) bean;
}
```
