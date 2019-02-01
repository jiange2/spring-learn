## BeanDefinition

##### 1.什么是BeanDefinition

BeanDefinition是对我们配置的Bean的封装。我们使用xml配置的`<bean></bean>`，或者注解`@Component`,`@Bean`都会被解析成BeanDefinition。

##### 2.BeanDefinition结构
![ClassPathApplicationContex继承关系图](/image/ApplicationContext/BeanFactory/RootBeanDefinition.png)

AbstractBeanDefinition继承了三个主要接口，AttributeAccessor、BeanMetadataElement、BeanDefinition。

##### 3.BeanMetadataElement

```Java
public interface BeanMetadataElement {

	@Nullable
	Object getSource();

}
```

待续...

##### 4. AttributeAccessor

```Java
public interface AttributeAccessor {

	void setAttribute(String name, @Nullable Object value);

	@Nullable
	Object getAttribute(String name);

	@Nullable
	Object removeAttribute(String name);

	boolean hasAttribute(String name);

	String[] attributeNames();

}
```
待续...

#### 5. BeanDefinition

```Java
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

	String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;

	String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;

	int ROLE_APPLICATION = 0;

	int ROLE_SUPPORT = 1;

	int ROLE_INFRASTRUCTURE = 2;

	void setParentName(@Nullable String parentName);

	@Nullable
	String getParentName();

	void setBeanClassName(@Nullable String beanClassName);

	@Nullable
	String getBeanClassName();

	void setScope(@Nullable String scope);

	@Nullable
	String getScope();

	void setLazyInit(boolean lazyInit);

	boolean isLazyInit();

	void setDependsOn(@Nullable String... dependsOn);

	@Nullable
	String[] getDependsOn();

	void setAutowireCandidate(boolean autowireCandidate);

	boolean isAutowireCandidate();

	void setPrimary(boolean primary);

	boolean isPrimary();

	void setFactoryBeanName(@Nullable String factoryBeanName);

	@Nullable
	String getFactoryBeanName();

	void setFactoryMethodName(@Nullable String factoryMethodName);

	@Nullable
	String getFactoryMethodName();

	ConstructorArgumentValues getConstructorArgumentValues();

	default boolean hasConstructorArgumentValues() {
		return !getConstructorArgumentValues().isEmpty();
	}

	MutablePropertyValues getPropertyValues();

	default boolean hasPropertyValues() {
		return !getPropertyValues().isEmpty();
	}

	void setInitMethodName(@Nullable String initMethodName);

	@Nullable
	String getInitMethodName();

	void setDestroyMethodName(@Nullable String destroyMethodName);

	@Nullable
	String getDestroyMethodName();

	void setRole(int role);

	int getRole();

	void setDescription(@Nullable String description);

	@Nullable
	String getDescription();

	boolean isSingleton();

	boolean isPrototype();

	boolean isAbstract();

	@Nullable
	String getResourceDescription();

	@Nullable
	BeanDefinition getOriginatingBeanDefinition();
}
```

beanDefinition提供了很多get、set方法都是和可配置的参数一一对应的。

|属性|xml对应的配置|作用|
|---|---|---|
|beanClassName|class|配置的bean的全限定类名|
|Scope|scope|配置bean单例还是多例|
|lazyInit|lazy-init|默认false，也就是beanFactory启动时就初始化bean|
|dependsOn|depends-on|配置依赖|
|autowireCandidate|autowire-candidate|设置是否参与自动注入(这个属性设置是针对by type注入，如果名字符合的话还是会注入的)|
|primary|primary|设置为自动注入的首选项，当by type注入时，找到多个符合的类型，设primary为true优先注入（断路器作用，当by type查找，找到第一个primary为true的时候就会取这个bean为结果，结束查找）|
|factoryBeanName|factory-bean|bean工厂|
|factoryMethodName|factory-method|bean工厂方法|
|initMethodName|init-method|初始化的钩子函数|
|destroyMethodName|destory-method|摧毁的钩子函数|
|abstract|abstract|默认false, true意味着这个bean只能被继承，不能实例化|
|autowireMode(AbstractBeanDefinition)|default-autowire|设置自动注入模式，默认为NO|
|qualifiers(AbstractBeanDefinition)|qualifier|标记, by type查到可以根据qualifier筛选一些bean|

参数解释：

** role : **

Role这个属性是用来标记这个Bean是属于哪种，bean。

- Application: 一般来说是用户自定义的Bean
- Support: 支持类Bean
- Infrastructure: Spring内部使用的Bean

** propertyValue: **

保存了改Bean配置注入的property和value的数据结构。

##### AbstractBeanDefinition

参数解释：

** autowireMode: **

设置了 autowireMode 的 bean 根据模式扫描 bean的set方法或 构造函数来自动注入。

- no 不自动注入
- byName 根据set方法的名字去查找相应beanName的bean
- byType 根据参数类型查找bean
- constructor 根据构造函数注入

** methodOverride: **

对spring管理的bean的方法进行重写。 (配置replace-method或lookup-method会解析成methodOverride)

//TODO
涉及AOP，待续。。。

方法：

** validate: **

```Java
public void validate() throws BeanDefinitionValidationException {
  // 如果有方法重写（需要代理）又是factory bean(无法代理)那就抛出异常
	if (hasMethodOverrides() && getFactoryMethodName() != null) {
		throw new BeanDefinitionValidationException(
				"Cannot combine static factory method with method overrides: " +
				"the static factory method must create the instance");
	}

	if (hasBeanClass()) {
		prepareMethodOverrides();
	}
}

public void prepareMethodOverrides() throws BeanDefinitionValidationException {
	// Check that lookup methods exists.
	if (hasMethodOverrides()) {
		Set<MethodOverride> overrides = getMethodOverrides().getOverrides();
		synchronized (overrides) {
			for (MethodOverride mo : overrides) {
				prepareMethodOverride(mo);
			}
		}
	}
}

protected void prepareMethodOverride(MethodOverride mo) throws BeanDefinitionValidationException {
	int count = ClassUtils.getMethodCountForName(getBeanClass(), mo.getMethodName());
	if (count == 0) {
		throw new BeanDefinitionValidationException(
				"Invalid method override: no method with name '" + mo.getMethodName() +
				"' on class [" + getBeanClassName() + "]");
	}
	else if (count == 1) {
		mo.setOverloaded(false);
	}
}
```

这里校验的后续处理工作其实和前面的methodOverride机制有关。methodOverride需要去检查参数的问题，这就涉及到了方法重载。（待续。。。）

##### 6、父子BeanDefinition

Spring容器的Bean之间是可以有继承关系的。

继承配置：

```xml
<bean id="parent" class="appcontext.parent.Parent">
    <property name="name" value="parent"/>
</bean>

<bean id="son" class="appcontext.parent.Son" parent="parent"/>
```
```java
public class Parent {
    private String name;
    public void setName(String name) {
        this.name = name;
    }
}
public class Son {
    private String name;
    public void setName(String name) {
        this.name = name;
    }
}
```

通过指定bean的parent这个参数，Son继承了所有Parent的所有配置。当然Son也可以覆盖parent的配置。通过这种方式可以把共用配置抽出来做成模板。

其中Parent的class不可以不指定，然后把abstract属性设置为true，这样parent就相当于一个纯配置模板，不能被实例化。

```xml
<bean id="parent" abstract="true">
    <property name="name" value="parent"/>
</bean>
```
** RootBeanDefinition,ChildBefinition和GenericBeanDefinition **

RootBeanDefinition,ChildBefinition和GenericBeanDefinition是AbstractBeanDefinition的3种实现。ChildDefinition必须指定其Parent，所以一般表示子BeanDefinition。但是在2.5之后一般都用通用的GenericBeanDefinition，因为GenericBeanDefinition更加灵活，可以随时指定不同的Parent。

##### 7、BeanDefinition注册
