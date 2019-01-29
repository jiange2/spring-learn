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
