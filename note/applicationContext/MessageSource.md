## MessageSource

MessageSource类图:

![MessageSource类图](/image/ApplicationContext/MessageSource/StaticMessageSource.png)

#### ResourceBundleMessageSource配置

ResourceBundleMessageSource是MessageSource其中一种实现,我们以ResourceBundleMessageSource为例演示MessageSource的基本用法

xml 配置:
```xml
<!-- bean的id一定是‘messageSource’, spring经常会有类似这样的hardcode -->
<bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
    <property name="basename" value="message-source"/>
</bean>
```
```Properties
// message-source.properties文件
key=value
```

`<property name="basename" value="message-source"/>`这里的basename指的是properties文件的根路径。

测试代码:
```Java
@Test
public void test(){
    ClassPathXmlApplicationContext appContext =
            new ClassPathXmlApplicationContext("classpath:app-context.xml");
    System.out.println(appContext.getMessage("key",null, Locale.getDefault()));
}
```
控制台:

    value

#### 父MessageSource

Spring有两种MessageSource的实现,ResourceBundleMessageSource和StaticMessageSource。这两种实现都继承了HierarchicalMessageSource。

```Java
public interface HierarchicalMessageSource extends MessageSource {

	void setParentMessageSource(@Nullable MessageSource parent);

	@Nullable
	MessageSource getParentMessageSource();

}
```

可以看到HierarchicalMessageSource可以设置ParentMessageSource。当当前MessageSource找不到这个Message的时候会尝试到父MessageSource去找。

xml 配置:
```xml
<bean id="parentMessageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
    <property name="basename" value="parent-message-source"/>
</bean>

<bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
    <property name="parentMessageSource" ref="parentMessageSource"/>
    <property name="basename" value="message-source"/>
</bean>
```
```Properties
// message-source.properties文件
key=value

// parent-message-source文件
parent-key=parent-value
```
测试代码:
```Java
@Test
public void test(){
    ClassPathXmlApplicationContext appContext =
            new ClassPathXmlApplicationContext("classpath:app-context.xml");
    System.out.println(appContext.getMessage("key",null, Locale.getDefault()));
    System.out.println(appContext.getMessage("parent-key",null,Locale.getDefault()));
}
```
控制台:
```
value
parent-value
```
#### Common Message

#### MessageFormat

#### MessageSourceResolvable

#### 源码分析
