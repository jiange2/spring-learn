## singletonBeanRegistry

singletonBeanRegistry 接口：

此接口是针对Spring中的单例Bean设计的。提供了统一访问单例Bean的功能，BeanFactory可实现此接口以提供访问内部单例Bean的能力.

```java
public interface SingletonBeanRegistry {

	void registerSingleton(String beanName, Object singletonObject);

	@Nullable
	Object getSingleton(String beanName);

	boolean containsSingleton(String beanName);

	String[] getSingletonNames();

	int getSingletonCount();

	Object getSingletonMutex();

}
```

- registerSingleton 注册单例Bean
- getSingleton 获取单例Bean
- getSingletonNames 获取所有单例Bean
- getSingletonCount 获取单例Bean的数量
- getSingletonMute (4.2) //TO DO

## DefaultSingletonBeanRegistry的实现

##### 注册单例Bean

```java
@Override
public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
	Assert.notNull(beanName, "Bean name must not be null");
	Assert.notNull(singletonObject, "Singleton object must not be null");
	synchronized (this.singletonObjects) {
		Object oldObject = this.singletonObjects.get(beanName);
		if (oldObject != null) {
			throw new IllegalStateException("Could not register object [" + singletonObject +
					"] under bean name '" + beanName + "': there is already object [" + oldObject + "] bound");
		}
		addSingleton(beanName, singletonObject);
	}
}

protected void addSingleton(String beanName, Object singletonObject) {
	synchronized (this.singletonObjects) {
		this.singletonObjects.put(beanName, singletonObject);
		this.singletonFactories.remove(beanName);
		this.earlySingletonObjects.remove(beanName);
		this.registeredSingletons.add(beanName);
	}
}
```

如果不管singletonFactories和earlySingletonObjects的移除操作，这段代码还是比较清晰易懂的。直接把要注册的Singleton put到singletonObjects这个map里面并且添加一个注册名到registeredSingletons这个List。

但是关注回earlySingletonObjects,什么是earlySingleton?singletonFactories又有何作用?

##### BeanFactory注册Singleton

earlySingletonObjects的early是指的是还没有完全注入属性的Bean。

而引入earlySingleton的目的是为了解决Bean循环引用的问题。

在介绍单例注入之前我们先看这几个，注册流程中会用到的方法。

```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
	Assert.notNull(singletonFactory, "Singleton factory must not be null");
	synchronized (this.singletonObjects) {
		if (!this.singletonObjects.containsKey(beanName)) {
			this.singletonFactories.put(beanName, singletonFactory);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
}
```

添加一个SingletonFactory，代码比较简单，就直接在singletonFactories这个map注册一个 beanName -> singletonFactory。singletonFactory提供了getObject方法，让我们获得singleton实例。(ObjectFactory会对Bean做一些操作，但是不影响注册逻辑)

```java
@Override
@Nullable
public Object getSingleton(String beanName) {
	return getSingleton(beanName, true);
}

@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
	Object singletonObject = this.singletonObjects.get(beanName);
	if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
		synchronized (this.singletonObjects) {
			singletonObject = this.earlySingletonObjects.get(beanName);
			if (singletonObject == null && allowEarlyReference) {
				ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
				if (singletonFactory != null) {
					singletonObject = singletonFactory.getObject();
					this.earlySingletonObjects.put(beanName, singletonObject);
					this.singletonFactories.remove(beanName);
				}
			}
		}
	}
	return singletonObject;
}
```


```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
	Assert.notNull(beanName, "Bean name must not be null");
	synchronized (this.singletonObjects) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null) {
			if (this.singletonsCurrentlyInDestruction) {
				throw new BeanCreationNotAllowedException(beanName,
						"Singleton bean creation not allowed while singletons of this factory are in destruction " +
						"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
			}
			beforeSingletonCreation(beanName);
			boolean newSingleton = false;
			boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
			if (recordSuppressedExceptions) {
				this.suppressedExceptions = new LinkedHashSet<>();
			}
			try {
				singletonObject = singletonFactory.getObject();
				newSingleton = true;
			}
			catch (IllegalStateException ex) {
				// Has the singleton object implicitly appeared in the meantime ->
				// if yes, proceed with it since the exception indicates that state.
				singletonObject = this.singletonObjects.get(beanName);
				if (singletonObject == null) {
					throw ex;
				}
			}
			catch (BeanCreationException ex) {
				if (recordSuppressedExceptions) {
					for (Exception suppressedException : this.suppressedExceptions) {
						ex.addRelatedCause(suppressedException);
					}
				}
				throw ex;
			}
			finally {
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = null;
				}
				afterSingletonCreation(beanName);
			}
			if (newSingleton) {
				addSingleton(beanName, singletonObject);
			}
		}
		return singletonObject;
	}
}
```
