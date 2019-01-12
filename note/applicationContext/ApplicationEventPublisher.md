## ApplicationContext

ApplicationContext提供了事件发布和事件监听的功能，通过这个功能可以实现后台异步操作。

```java
public interface ApplicationEventPublisher {

	default void publishEvent(ApplicationEvent event) {
		publishEvent((Object) event);
	}

	void publishEvent(Object event);

}
```

###### 例子:

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
