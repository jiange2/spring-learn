## SingleTon 单例注册

---

#### 简介

我们知道假如 Bean 可以是单例 （把 scope 配置为 singleton 或者不设置 scope），这些单例的 Bean 被初始化之后就被Spring注册到BeanFactory管理了起来，方便下次直接使用。除了配置的单例Bean，我们还可以调用BeanFactory的Bean直接注册一个单例Bean。

#### SingletonBeanRegistry

SingletonBeanRegistry是BeanFactory默认实现DefaultListableBeanFactory实现的接口之一。这个接口主要提供了一些单例注册、获取等单例相关的方法。

```java
public interface SingletonBeanRegistry {

    void registerSingleton(String beanName, Object singletonObject);

    Object getSingleton(String beanName);

    boolean containsSingleton(String beanName);

    String[] getSingletonNames();

    int getSingletonCount();

    Object getSingletonMutex();

}
```

#### 手动注册单例

前面提到Spring管理着手动注册以及scope配置为singleton的单例Bean注册。这两种Bean的注册方式是不同的。我们先来看手工注册。

通过调用registerSingleton(String beanName, Object singletonObject)这个方法注册的Bean就是手动注册的Bean。

例子:

```java
@Test
public void test(){
    ClassPathXmlApplicationContext appContext =
            new ClassPathXmlApplicationContext("classpath:app-context.xml");
    Object object = new Object();
    appContext.getBeanFactory().registerSingleton("object",object);
    System.out.println(object == appContext.getBean("object"));
}
```

打印的结果为`true`，证明可以拿到注册进去的Bean。

DefaultSingletonBeanRegistry（SingletonBeanRegistry默认实现）源码:

```java
@Override
public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
  Assert.notNull(beanName, "Bean name must not be null");
  Assert.notNull(singletonObject, "Singleton object must not be null");
  synchronized (this.singletonObjects) {
    Object oldObject = this.singletonObjects.get(beanName);
    if (oldObject != null) {
      throw new IllegalStateException("Could not register object [" + singletonObject +
          "] under bean name '" + beanName + "': there is already object [" + oldObject + "] bound");
    }
    addSingleton(beanName, singletonObject);
  }
}

protected void addSingleton(String beanName, Object singletonObject) {
  synchronized (this.singletonObjects) {
    this.singletonObjects.put(beanName, singletonObject);
    this.singletonFactories.remove(beanName);
    this.earlySingletonObjects.remove(beanName);
    this.registeredSingletons.add(beanName);
  }
}
```

代码比较简单,忽略一些校验以及`this.singletonFactories.remove(beanName);`,`this.earlySingletonObjects.remove(beanName);`（这两个变量主要是为配置的bean服务的）,这两句remove代码,可以看到就是往singletonObjects添加了一个 beanName -> singletonObject， 以及往 registeredSingletons 添加了注册进来的beanName。

singletonObject就是管理所有单例bean的一个map。

#### 自动注册的单例 (配置scope为singleton的bean)
