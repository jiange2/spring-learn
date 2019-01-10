## Application Context简介

Spring的本质就是一个容器，而ApplicationContext就是Spring的容器。
以ClassPathApplicationContext为例来分析 ApplicationContext.

ClassPathApplicationContex继承关系图:

![ClassPathApplicationContex继承关系图](/image/ApplicationContext/ClassPathXmlApplicationContext.png)

我们通过ClassPathXmlApplicationContext 实现的接口和继承类，来知道ApplicationContext包含了哪些功能，以及分模块的去了解ApplicationContext的实现细节。

### ApplicationContext 的 Interfaces

###### Aware / BeanNameAware
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

比如通过实现BeanNameAware，Bean可以获取自己Bean的Name。

当ApplicationContext加载这个Bean发现他实现了BeanNameAware就会调用setBeanName方法，传入这个Bean的Name。

详细介绍： [Aware](/note/applicationContext/aware.md)

### LifeCycle

对于容器管理的对象来说，一般都是有生命周期的。比如Servlet就可以通过实现init和destroy方法，来监听容器对象的初始化和销毁。在Spring容器的bean也可以通过LifeCycle实现这样的功能。

LifeCycle:
```java
public interface Lifecycle {
	void start();
	void stop();
	boolean isRunning();
}
```
