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
@Override
public void start() {
	startBeans(false);
	this.running = true;
}

@Override
public void onRefresh() {
	startBeans(true);
	this.running = true;
}

private void startBeans(boolean autoStartupOnly) {
	Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
	Map<Integer, LifecycleGroup> phases = new HashMap<>();
  // 分组
	lifecycleBeans.forEach((beanName, bean) -> {
		if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
			int phase = getPhase(bean);
			LifecycleGroup group = phases.get(phase);
			if (group == null) {
				group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
				phases.put(phase, group);
			}
			group.add(beanName, bean);
		}
	});
  // 分组排序启动
	if (!phases.isEmpty()) {
		List<Integer> keys = new ArrayList<>(phases.keySet());
		Collections.sort(keys);
		for (Integer key : keys) {
			phases.get(key).start();
		}
	}
}

protected int getPhase(Lifecycle bean) {
	return (bean instanceof Phased ? ((Phased) bean).getPhase() : 0);
}
```

ApplicationContext启动的时候会调用onRefresh方法，但不会调用start方法。OnRefresh调用startBeans，autoStartupOnly为true，这个时候只会调用autoStartupOnly为true的SmartLifeCycle Bean。所以要显式调用applicationContext的start方法才能启动普通的LifeCycle Bean。

###### 分组stop

分组创建:
```java
group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
```

对于SmartLifeCycle Bean，前面提到SmartLifeCycle的stop回调机制来实现异步stop的。applicationContext中有会有阻塞机制等待bean stop完成，但是applicationContext有超时机制，超过了时间就会直接结束。

stop:
```java
public void stop() {
	if (this.members.isEmpty()) {
		return;
	}
	if (logger.isDebugEnabled()) {
		logger.debug("Stopping beans in phase " + this.phase);
	}
	this.members.sort(Collections.reverseOrder());
	CountDownLatch latch = new CountDownLatch(this.smartMemberCount);
	Set<String> countDownBeanNames = Collections.synchronizedSet(new LinkedHashSet<>());
	Set<String> lifecycleBeanNames = new HashSet<>(this.lifecycleBeans.keySet());
	for (LifecycleGroupMember member : this.members) {
		if (lifecycleBeanNames.contains(member.name)) {
			doStop(this.lifecycleBeans, member.name, latch, countDownBeanNames);
		}
		else if (member.bean instanceof SmartLifecycle) {
			latch.countDown();
		}
	}
	try {
		latch.await(this.timeout, TimeUnit.MILLISECONDS);
		if (latch.getCount() > 0 && !countDownBeanNames.isEmpty() && logger.isInfoEnabled()) {
			logger.info("Failed to shut down " + countDownBeanNames.size() + " bean" +
					(countDownBeanNames.size() > 1 ? "s" : "") + " with phase value " +
					this.phase + " within timeout of " + this.timeout + ": " + countDownBeanNames);
		}
	}
	catch (InterruptedException ex) {
		Thread.currentThread().interrupt();
	}
}
```

代码是通过concurrent包的CountDownLatch来完成超时中断功能的。CountDownLatch的初始值为分组bean的数量，每一个LifeCycle调用stop方法的callback，latch的值都会-1。`latch.await(this.timeout, TimeUnit.MILLISECONDS);`这句代码会阻塞，直到超时或者latch的值为0。
