## LifeCycleProcessor

ApplicationContext通过调用LifeCycleProcessor实现调用LifeCycle Bean的生命周期回调。

```Java
public interface LifecycleProcessor extends Lifecycle {
	void onRefresh();
	void onClose();
}
```
当ApplicationContext调用refresh会调用LifecycleProcessor的onRefresh方法,调用close会调用
LifecycleProcessor的onClose方法。

---

#### LifeCycleProcessor的初始化

AbstractApplicationContext源码:
```Java
public static final String LIFECYCLE_PROCESSOR_BEAN_NAME = "lifecycleProcessor";

protected void initLifecycleProcessor() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
		this.lifecycleProcessor =
				beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
	}
	else {
		DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
		defaultProcessor.setBeanFactory(beanFactory);
		this.lifecycleProcessor = defaultProcessor;
		beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
	}
}
```
ApplicationContext会看你是否配置了bean name为`lifecycleProcessor`的bean。如果没有就创建默认的LifeCycleProcessor。

所以我们可以通过配置`lifecycleProcessor`来替换掉默认的LifeCycleProcessor。

LifeCycleProcessor必须设置beahFactory，因为LifeCycleProcessor实际上管理的是容器里面的bean。

---

#### Start和OnRefresh

```java
```
