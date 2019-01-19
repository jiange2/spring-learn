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

AbstractMessage定义了commonMessage用来存放区域无关的消息。

```java
<bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
    <property name="commonMessages">
        <map>
            <entry key="common-key" value="common-value"/>
        </map>
    </property>
    <property name="basename" value="message-source"/>
</bean>
```

#### MessageFormat

MessageSource支持带占位符的Property。例如: `{0} World`，调用的时候加上参数
`System.out.println(appContext.getMessage("key2",new Object[]{"Hello"}, Locale.CHINA));`
。就可以把`{0}`替换成 `Hello`。

这个是通过java标准库的MessageFormat实现的。

#### MessageSourceResolvable

MessageSourceResolvable主要是对Key值, args, 和DefaultMessage的封装。

```java
public interface MessageSourceResolvable {

	@Nullable
	String[] getCodes();

	@Nullable
	default Object[] getArguments() {
		return null;
	}

	@Nullable
	default String getDefaultMessage() {
		return null;
	}
}
```

其中codes是property的key，这里可以设置多个key，返回第一个能找到的key。比如设置`key1`和`key2`。但是配置文件只有`key2`，那就返回key2的value。

MessageSourceResolvable除了作为`String getMessage(MessageSourceResolvable resolvable, Locale locale)`这个方法的参数。还可以作为占位符的arguments。

例子：

properties文件
```java
key=[value {0}]
```
测试代码:
```Java
@Test
public void test(){
    ClassPathXmlApplicationContext appContext =
            new ClassPathXmlApplicationContext(new String[]{"classpath:app-context.xml"});
    // 设置两个key, 但是'ke22'没有。这时候会使用'key'的值
    MessageSourceResolvable msr = new DefaultMessageSourceResolvable(new String[]{"ke22","key"},new Object[]{"args"});
    System.out.println(appContext.getMessage(msr, Locale.getDefault()));
}
```
控制台
```
[value args]
```

MessageSourceResolvable作为arguments:

```java
@Test
public void test(){
    ClassPathXmlApplicationContext appContext =
            new ClassPathXmlApplicationContext(new String[]{"classpath:app-context.xml"});
    MessageSourceResolvable msr = new DefaultMessageSourceResolvable(new String[]{"ke22","key"},new Object[]{"args"});
    String result = appContext.getMessage("key",new Object[]{msr},Locale.getDefault());
    System.out.println(result);
}
```
控制台
```java
//[value {0}]
[value [value args]]
```
`args`是MessageSourceResolvable的占位符参数。而MessageSourceResolvable的结果（`[value args]`）又是getMessage的占位符参数，形成嵌套的效果。

---

#### 参数配置 (待续)

###### MessageSourceSupport

**alwaysUseMessageFormat (false)**:

如果配置为true，即使没有传入参数也会使用MessageFormat。

###### AbstractMessageSource

**useCodeAsDefaultMessage (false)**：

在没有传入default值，又找不到传入的key：

useCodeAsDefaultMessage = true --> 把key值当成value
useCodeAsDefaultMessage = false --> 抛出异常

---

#### 源码分析

###### AbstractMessageSource getMessage:

```Java
@Override
public final String getMessage(String code, @Nullable Object[] args, Locale locale) throws NoSuchMessageException {
	String msg = getMessageInternal(code, args, locale);
	if (msg != null) {
		return msg;
	}
	String fallback = getDefaultMessage(code);
	if (fallback != null) {
		return fallback;
	}
	throw new NoSuchMessageException(code, locale);
}

@Override
public final String getMessage(String code, @Nullable Object[] args, @Nullable String defaultMessage, Locale locale) {
	String msg = getMessageInternal(code, args, locale);
	if (msg != null) {
		return msg;
	}
	if (defaultMessage == null) {
		return getDefaultMessage(code);
	}
	return renderDefaultMessage(defaultMessage, args, locale);
}

protected String getDefaultMessage(String code) {
  // useCodeAsDefaultMessage是可配置的参数
	if (isUseCodeAsDefaultMessage()) {
		return code;
	}
	return null;
}
```

**getMessage(code,args,locale):**

getMessageInternal > getDefaultMessage > NoSuchMessageException

先调用getMessageInternal,没有再getDefaultMessage,再没有就抛出异常。

**getMessage(code,args,defaultMessage,locale):**

getMessageInternal > renderDefaultMessage > getDefaultMessage

先调用getMessageInternal,再调用renderDefaultMessage,最后在尝试defaultMessage。

**getMessageInternal:**
```Java
protected String getMessageInternal(@Nullable String code, @Nullable Object[] args, @Nullable Locale locale) {
	if (code == null) {
		return null;
	}
	if (locale == null) {
		locale = Locale.getDefault();
	}
	Object[] argsToUse = args;

	if (!isAlwaysUseMessageFormat() && ObjectUtils.isEmpty(args)) {
		String message = resolveCodeWithoutArguments(code, locale);
		if (message != null) {
			return message;
		}
	}else {
		argsToUse = resolveArguments(args, locale);

		MessageFormat messageFormat = resolveCode(code, locale);
		if (messageFormat != null) {
			synchronized (messageFormat) {
				return messageFormat.format(argsToUse);
			}
		}
	}

	Properties commonMessages = getCommonMessages();
	if (commonMessages != null) {
		String commonMessage = commonMessages.getProperty(code);
		if (commonMessage != null) {
			return formatMessage(commonMessage, args, locale);
		}
	}

	return getMessageFromParent(code, argsToUse, locale);
}
```

没有传入参数：
在`alwaysUseMessageFormat`（可配置）为false以及没有传入参数的情况下，尝试调用`resolveCodeWithoutArguments`（子类需要实现的方法）获取结果。

传入了参数：
先通过调用`resolveArguments(args, locale);`把[MessageSourceResolvable](#MessageSourceResolvable)解析成字符串,然后尝试调用`resolveCode`（子类需要实现的方法）获取结果。

如果经过上面的步骤还拿不到参数的话就可以尝试通过配置common message获取结果。还没有就尝试父MessageSource。

总结优先级:
`resolveCode|resolveCodeWithoutArguments` > `commonMessages` > `getMessageFromParent` > `defaultMessage` > `getDefaultMessag(key)` > `exception`

---

## SpringMessageSource的实现 （待续）

#### ResourceBundleMessageSource
