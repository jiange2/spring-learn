## Application Context简介

Spring的最核心的部分就是容器，而ApplicationContext就是Spring的容器。

以ClassPathApplicationContext为例来分析 ApplicationContext.

ClassPathApplicationContex继承关系图:

![ClassPathApplicationContex继承关系图](/image/ApplicationContext/ClassPathXmlApplicationContext.png)

我们可以发现一个ApplicationContext实现和继承了很多接口和抽象类。通过这些接口和继承类，来知道ApplicationContext包含了哪些功能，以及分模块的去了解ApplicationContext的实现细节。

### ApplicationContext 的 Interfaces
---
#### InitializingBean

```java
public interface InitializingBean {
	void afterPropertiesSet() throws Exception;
}
```

实现了这个接口的Bean，在Spring容器给Bean设置了属性值后会调用这个方法。相对应的Spring中还有DisposableBean这个接口，实现了这个接口的bean，applicationContext会在销毁这个bean时调用这个接口的destroy方法。

Spring并不推荐实现这个两个接口，因为这样会将代码会和Spring耦合。
> We recommend that you do not use the InitializingBean interface, because it unnecessarily couples the code to Spring.

如果需要实现init和destroy这样的功能。可以通过xml配置或者使用`@PostConstruct`和`@PreDestroy`注解 (这两个注解是java标准库的注解，而不是spring的)。

xml配置:
```xml
<beans default-init-method="init">
	<bean class="com.test.InitBean" init-method="init" destroy-method="destroy"/>
</beans>

在beans可以设置default-init-method，也就是设置beans下面的bean的默认初始化方法。
```
注解方式：
```java
@Component
public class InitBean {

    @PostConstruct
    public void init(){
        System.out.println("@PostConstruct init");
    }

    @PreDestroy
    public void destroy(){
        System.out.println("@PreDestroy destroy");
    }
}
```

---

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

但是Spring建议尽量避免使用这些接口，因为这样会使代码和Spring耦合。如果在bean中需要使用ApplicationContext这种对象可以使用`@Autowired`这个注解。

```java
@Autowired
private ApplicationContext applicationContext;
```

其他Aware接口： [Aware](/note/applicationContext/Aware.md)

---

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

当ApplicationContext调用refresh或start方法的时候，会调用所有LifeCycle bean的start方法。而ApplicationContext调用stop方法的时候就会调用LifeCycle bean的stop方法。

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

SmartLifeCycle提供了异步的stop回调。通过调用参数的callback来通知ApplicationContext stop方法执行结束。

回调使用示例：
```java
@Override
public void stop(Runnable callback) {
    new Thread(() -> {
        //clear的代码
        callback.run();
    }).start();
}
```
##### ApplicationContext的LifeCycle实现和LifeCycle Bean的关系

ApplicationContext通过调用LifeCycleProcessor实现调用LifeCycle Bean的生命周期回调。

LifeCycleProcessor: [LifeCycleProcessor](/note/applicationContext/LifeCycleProcessor.md)

---

##### MessageResource

---

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

ApplicationEventPublisher更多的一些细节: [ApplicationEventPublisher](/note/applicationContext/ApplicationEventPublisher.md)

---

#### EnvironmentCapable

```java
public interface EnvironmentCapable {

	Environment getEnvironment();

}
```
Environment主要是两个方面的封装Profiles和Properties的抽象。
而ApplicationContext实现了EnvironmentCapable这个接口，在ApplicationContext初始化的会创建一个StandardEnviroment。

##### Profile
Profile可以理解为容器的分组。ApplicationContext会根据Environment当前处于active状态的Profile来决定加载哪些bean。

```xml
<!-- 当Environment active的Prifile为Test时,读取test目录的property -->
<beans profile="test">
    <context:property-placeholder
            location="classpath*:test/*.properties" />
</beans>

<!-- 当Environment active的Prifile为Production,读取production目录的property -->
<beans profile="production">
    <context:property-placeholder
            location="classpath*:production/*.properties" />
</beans>
```

详情: [Profile](/note/applicationContext/Env-Profile.md)

##### Properties
Enviroment管理了各种属性配置,包括自定义Properties,JVM system properties,system environment variables, JNDI, servlet context parameters, ad-hoc Properties objects, Map objects, 等等.

通过Enviroment，我们可以通过统一接口可以访问各种Properties。

详情: [Properties](/note/applicationContext/Env-Properties.md)

Spring官方文档: [Spring Environment Abstraction](https://docs.spring.io/spring/docs/5.1.4.RELEASE/spring-framework-reference/core.html#beans-environment)

---

### Resource和ResourceLoader

##### Rsource
Resource是Spring对各种资源文件的封装，通过Resource进行对资源统一的访问。

为什么Spring要对资源进行封装？

>Spring官方文档：
>
>Java’s standard java.net.URL class and standard handlers for various URL prefixes, unfortunately, are not quite adequate enough for all access to low-level resources. For example, there is no standardized URL implementation that may be used to access a resource that needs to be obtained from the classpath or relative to a ServletContext. While it is possible to register new handlers for specialized URL prefixes (similar to existing handlers for prefixes such as http:), this is generally quite complicated, and the URL interface still lacks some desirable functionality, such as a method to check for the existence of the resource being pointed to.

总的来说就是java标准库的java.net.URL类,虽然对不同协议头的资源做了统一封装。但是在易用性和完整性，spring并不是很满意，所以就做了这样一个封装。

例外这段文字中提到了一个点值得我们思考:

**URL没有classpath和servletContext资源这两种协议**。所以其实classpath这个我们十分常用的路径头，可以理解为在Spring里面的一种特殊文件传输协议。在Spring中classpath和我们常用的http,ftp,file等协议头没什么区别，都是文件获取的协议。

##### ResourceLoader

ApplicationContext 实现了 ResourceLoader 这个接口。通过这个接口的方法，我们可以传入不同文件协议的路径来获取资源文件。

详情: [Resource和ResourceLoader](/note/appplicationContext/Resource和ResourceLoader.md)

#### BeanFactory

ApplicationContext最核心的功能就是管理bean，说白了ApplicationContext就是一个BeanFactroy，其他接口实现都是为BeanFactory服务的。

BeanFactory提供了ApplicationContext最核心的功能。

BeanFactory详情: [BeanFactory](/applicationContext/BeanFactory.md)
