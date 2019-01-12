## Application Context简介

Spring的本质就是一个容器，而ApplicationContext就是Spring的容器。

以ClassPathApplicationContext为例来分析 ApplicationContext.

ClassPathApplicationContex继承关系图:

![ClassPathApplicationContex继承关系图](/image/ApplicationContext/ClassPathXmlApplicationContext.png)

我们通过ClassPathXmlApplicationContext 实现的接口和继承类，来知道ApplicationContext包含了哪些功能，以及分模块的去了解ApplicationContext的实现细节。

### ApplicationContext 的 Interfaces

#### InitializingBean

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
Spring配置也支持destroy-method

#### Aware / BeanNameAware
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

#### LifeCycle

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

##### ApplicationContext的LifeCycle实现和LifeCycle Bean的关系

当ApplicationContext调用refresh或start方法的时候，会调用所有LifeCycle bean的start方法。而ApplicationContext调用stop方法的时候就会调用LifeCycle bean的stop方法。

而ApplicationContext是通过LifeCycleProcessor完成实现的。

详情: [LifeCycleProcessor](/note/applicationContext/LifeCycleProcessor.md)

##### MessageResource

##### ApplicationEventPublisher

ApplicationContext提供了事件发布和事件监听的功能，通过这个功能可以实现后台异步操作。

```java
public interface ApplicationEventPublisher {

	default void publishEvent(ApplicationEvent event) {
		publishEvent((Object) event);
	}

	void publishEvent(Object event);

}
```

例子:
事件对象:
```java
public class EmailEvent extends ApplicationEvent {
    public EmailEvent(Object source) {
        super(source);
    }
}
```
事件监听器：事件监听器需实现ApplicationListener这个接口。ApplicationEvent的泛型类型就是要监听事件类型。
```java
@Component
public class EmailEventListener implements ApplicationListener<EmailEvent> {
    @Override
    public void onApplicationEvent(EmailEvent event) {
        System.out.println(event);
    }
}
```
测试类:
```java
@Test
public void test(){
    ClassPathXmlApplicationContext appContext =
            new ClassPathXmlApplicationContext(new String[]{"classpath:app-context.xml"});
	  //发布Email事件
    appContext.publishEvent(new EmailEvent(new Object()));
}
```
输出结果:

	appcontext.eventlistener.EmailEvent[source=java.lang.Object@54e28de4]

需要注意的是，如果我们希望事件是异步进行的，我们需要注入线程池到SimpleApplicationEventMulticaster (事件广播器)。

```xml
<bean id="taskExecutor" class="org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor">
    <property name="rejectedExecutionHandler">
        <bean class="java.util.concurrent.ThreadPoolExecutor$CallerRunsPolicy"/>
    </property>
</bean>

<bean id="applicationEventMulticaster" class="org.springframework.context.event.SimpleApplicationEventMulticaster">
    <property name="taskExecutor" ref="taskExecutor"/>
</bean>
```

注意: bean的id必须配置为applicationEventMulticaster。因为Spring有很多这样的HardCode,根据bean的id和class来找需要的bean。

ApplicationEventPublisher更多的一些细节: [ApplicationEventPublisher](/note/applicationContext/ApplicationEventPublisher.md)
