## Application Context简介

Spring的本质就是一个容器，而ApplicationContext就是Spring的容器。
以ClassPathApplicationContext为例来分析 ApplicationContext.

ClassPathApplicationContex继承关系图:

![ClassPathApplicationContex继承关系图](/image/ApplicationContext/ClassPathXmlApplicationContext.png)

我们通过ClassPathXmlApplicationContext 实现的接口和继承类，来知道ApplicationContext包含了哪些功能，以及分模块的去了解ApplicationContext的实现细节。

### ApplicationContext 的 Interfaces

##### InitializingBean

```java
public interface InitializingBean {
	void afterPropertiesSet() throws Exception;
}
```

实现了这个接口的Bean，在Spring容器给Bean设置了属性值后会调用这个方法。

在Spring中，还可以通过配置init-method的方式添加初始化方法。和实现InitializingBean这个方式相比，init-method耦合度更低。

```xml
<bean class="com.test.InitBean" init-method="init" destroy-method="destroy"/>
```
Spring配置同时也支持destroy-method

##### Aware / BeanNameAware
```java
public interface BeanNameAware extends Aware {
	void setBeanName(String name);
}
```
ApplicationContext实现了这个BeanNameAware接口，这个对ApplicationContext作用不是很大, 调用setBeanName就是设置ApplicationContext的ID和DisplayName。

AbstractRefreshableConfigApplicationContext的实现：

```java
@Override
public void setBeanName(String name) {
	if (!this.setIdCalled) {
		super.setId(name);
		setDisplayName("ApplicationContext '" + name + "'");
	}
}
```
Aware本身的作用是可以让每个Spring管理的Bean可以获取ApplicationContext或Bean本身的一些信息。

比如通过实现了BeanNameAware的，当ApplicationContext加载这个Bean发现他实现了BeanNameAware就会调用setBeanName方法，传入这个Bean的Name。这样,这个Bean可以获取自己的Bean Name。


详细介绍： [Aware](/note/applicationContext/aware.md)

##### LifeCycle

对于容器管理的对象来说，一般都是有生命周期的。比如Servlet就可以通过实现init和destroy方法，来监听容器对象的初始化和销毁。在Spring容器的bean也可以通过LifeCycle实现这样的功能。

LifeCycle:
```java
public interface Lifecycle {
	void start();
	void stop();
	boolean isRunning();
}
```

在Spring中还可以通过实现SmartLifeCyle来让ApplicationContext里面的Bean按优先级调用start和stop方法。

SmartLifeCycle:

```java
public interface SmartLifecycle extends Lifecycle, Phased {

	int DEFAULT_PHASE = Integer.MAX_VALUE;

	default boolean isAutoStartup() {
		return true;
	}

	default void stop(Runnable callback) {
		stop();
		callback.run();
	}

	@Override
	default int getPhase() {
		return DEFAULT_PHASE;
	}
}

public interface Phased {
	int getPhase();
}
```

phase是这个bean的优先级，默认是Integer.MAX_VALUE。phase值越小越早启动，越晚关闭。所以默认phase的bean优先级最低。

###### ApplicationContext的LifeCycle和BeanLifeCycle的关系

当ApplicationContext调用refresh或start方法的时候，会调用所有LifeCycle bean的start方法。而ApplicationContext调用stop方法的时候就会调用LifeCycle bean的stop方法。

而ApplicationContext是通过LifeCycleProcessor完成实现细节的。

详情: [LifeCycleProcessor](/note/applicationContext/LifeCycleProcessor.md)
