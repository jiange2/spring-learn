## SingleTon 单例注册

---

#### 简介

我们知道Spring容器的 Bean 可以是单例 （把 scope 配置为 singleton 或者不设置 scope，即默认是单例），这些单例的 Bean 被初始化之后就被Spring注册到BeanFactory管理了起来，方便下次直接使用。除了配置的单例Bean，我们还可以调用BeanFactory的Bean直接注册一个单例Bean。

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

singletonObjects就是管理所有单例bean的一个map。

#### 自动注册的单例 (配置scope为singleton的bean)

自动注册,也就是配置的Singleton Bean的注册要比手动注册的复杂得多。原因是，自动注册需要解决循环依赖的问题。

##### 循环引用

spring在把Bean创建之后需要给Bean注入参数，假如 A 依赖了 B，那创建 A 的时候就是在 spring 容器里面查找 B，假如这个时候 B 也还没创建那就会创建 B。如果 B 又依赖了 A ，那就会有循环依赖的问题。

A -> B -> A

##### earlySingletonObjects和singletonFactories

为了解决循环依赖的问题, `DefaultSingletonBeanRegistry` 中用了 `earlySingletonObjects` 和 `singletonFactories`这两个map来解决这个问题。

earlySingletonObjects中的 early 指的是new 出来但是还没注入参数的Bean，而`earlySingletonObjects`管理的正是这种Bean。在上面的源码中，`addSingleton`的方法。在添加单例之后就移除了`earlySingletonObjects`中相同beanName的bean，因为已经有了注入好的Bean，就不需要'early'的了。

singletonFactories spring不会直接创建bean，而是把创建bean的方法（createBean）封装成factory保存在`singletonFactories`里。

##### getSingleton源码

spring通过getSingleton的这几个方法实现注册配置的bean并解决循环依赖问题。这几个方法有一个比较复杂的调用流程，我们先看这几个方法大概做了些什么，什么我们再通过调用流程深入了解这些方法的功用。

```java
@Override
@Nullable
public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}

@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    //  判断singletonObjects是否存在要找bean，这个方法大部分情况都会在这里结束，后面的代码只有在存在循环引用才会调用。
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```

##### isSingletonCurrentlyInCreation

第二行代码：`isSingletonCurrentlyInCreation(beanName)`。

```java
public boolean isSingletonCurrentlyInCreation(String beanName) {
	return this.singletonsCurrentlyInCreation.contains(beanName);
}
```
isSingletonCurrentlyInCreation的作用判断这个beanName是不是正在创建，也就是判断是不是循环依赖了。

举个例子假如 A依赖了B，B又依赖了A （A -> B -> A）。创建 A 时，会先把 A 的beanName添加到 singletonsCurrentlyInCreation这个set里面，然后再去查找B，发现B还没创建就去创建B，同样创建B也会添加到singletonsCurrentlyInCreation，再去查找依赖A，这个时候isSingletonCurrentlyInCreation就会返回true，也就是循环依赖了。

这个方法依次会从singletonObjects，earlySingletonObjects，singletonFactories查找bean，关于具体什么时候，bean会在那个地方，后面调用流程会有介绍。

##### getSingleton(String beanName, ObjectFactory<?> singletonFactory)

```Java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    synchronized (this.singletonObjects) {
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null) {
            beforeSingletonCreation(beanName);
            boolean newSingleton = false;
            boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
            if (recordSuppressedExceptions) {
                this.suppressedExceptions = new LinkedHashSet<>();
            }
            try {
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            } catch (IllegalStateException ex) {
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    throw ex;
                }
            } catch (BeanCreationException ex) {
                if (recordSuppressedExceptions) {
                    for (Exception suppressedException : this.suppressedExceptions) {
                        ex.addRelatedCause(suppressedException);
                    }
                }
                throw ex;
            } finally {
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = null;
                }
                afterSingletonCreation(beanName);
            }
            if (newSingleton) {
                addSingleton(beanName, singletonObject);
            }
        }
        return singletonObject;
    }
}

protected void beforeSingletonCreation(String beanName) {
	if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
		throw new BeanCurrentlyInCreationException(beanName);
	}
}

protected void afterSingletonCreation(String beanName) {
	if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) {
		throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
	}
}
```

`getSingleton(String beanName, ObjectFactory<?> singletonFactory)`，这个方法是getSingleton另一个重载方法，在创建之前把beanName放到singletonsCurrentlyInCreation，再通过调用传入的singletonFactory getObject创建bean，在创建之后再把singletonsCurrentlyInCreation移除，最后再把创建的bean注册到singletonObjects。

这个方法虽然叫getSingleton但实际上主要功能是创建bean再把bean注册。

#### 调用流程

假设A依赖了B，B依赖了A，我们分析一下spring是如何调用这几个方法解决依赖循环的问题的，其中加粗的方法是我们前面介绍过的方法，其他方法涉及太多其他的细节，就不在这里分析了。

|方法|描述|singletonObject池|
|--|--|--|
|1.doGetBean("A")|获取bean A|singleton：<br>earlySingleton:<br>singletonFactory:|
|2.getSingleton("A")|直接取singleton A,所有池没有A所以找不到|singleton：<br>earlySingleton:<br>singletonFactory:|
|第2步getSingleton("A")结束||singleton：<br>earlySingleton:<br>singletonFactory:|
|3. getSingleton("A", ObjectFactory)|前面已经了解带ObjectFactory参数的getSingleton方法其实包含了创建和注册bean。根据下面源码，调用ObjectFacotry的getObject方法其实就是调用BeanFacotry的子类实现的createBean|singleton：<br>earlySingleton:<br>singletonFactory:|
|4.createBean("A",..)|getSingleton("A", ObjectFactory)创建bean的操作|singleton：<br>earlySingleton:<br>singletonFactory:|
|5.doCreateBean("A",..)|-|singleton：<br>earlySingleton:<br>singletonFactory:|
|6.addSingletonFactory("A", ObjectFactory)|把用于创建A的ObjectFacotry注册到singletonFactories|singleton：<br>earlySingleton:<br>singletonFactory:A|
|第6步的方法结束|-|singleton：<br>earlySingleton:<br>singletonFactory:A|
|7.populateBean("A",..)|给A注入依赖|singleton：<br>earlySingleton:<br>singletonFactory:A|
|8.doGetBean("B")|因为A依赖B，所以需要查找B|singleton：<br>earlySingleton:<br>singletonFactory:A|
|9.getSingleton("B")|获取B的单例|singleton：<br>earlySingleton:<br>singletonFactory:A|
|第9步执行的方法结束|-|singleton：<br>earlySingleton:<br>singletonFactory:A|
|10. getSingleton("B", ObjectFactory)|因为B还没创建所以创建B|singleton：<br>earlySingleton:<br>singletonFactory:A|
|11.createBean("B",..)|创建B|singleton：<br>earlySingleton:<br>singletonFactory:A|
|12.doCreateBean("B",..)|-|singleton：<br>earlySingleton:<br>singletonFactory:A|
|13.addBeanFactory("B",facotry)|把用于创建B的ObjectFacotry注册到singletonFactories|singleton：<br>earlySingleton:<br>singletonFactory:A,B|
|14.populateBean("B",..)|注入B的依赖|singleton：<br>earlySingleton:<br>singletonFactory:A,B|
|15.doGetBean("A")|因为B依赖了A，所以查找A|singleton：<br>earlySingleton:<br>singletonFactory:A,B|
|16.getSingleton("A")|这个时候已经有A的ObjectFacotry了，根据前面的代码，getSingleton会用objectFactory创建出Bean A，然后把A的objectFactory移除，再把Bean A放到earlySingletonObjects|singleton：<br>earlySingleton:A<br>singletonFactory:B|
|第15,16步结束|成功创建Bean A|singleton：<br>earlySingleton:A<br>singletonFactory:B|
|第14步结束|成功注入Bean A|singleton：<br>earlySingleton:A<br>singletonFactory:B|
|第11，12，13步结束|成功创建Bean B|singleton：<br>earlySingleton:A<br>singletonFactory:B|
|17.addSingleton("B", Bean B);|把创建成功的Bean B注册到singletonObjects|singleton：B<br>earlySingleton:A<br>singletonFactory:|
|第10步结束||singleton：B<br>earlySingleton:A<br>singletonFactory:|
|第7,8,9结束|成功把beanB 注入到Bean A|singleton：B<br>earlySingleton:A<br>singletonFactory:|
|18.addSingleton("A", Bean B);|添加A到singletonObjects|singleton：A,B<br>earlySingleton:<br>singletonFactory:|
|第1,3结束|成功创建A|singleton：A,B<br>earlySingleton:<br>singletonFactory:|


###### 部分源码：

3.getSingleton("A", ObjectFactory)

```java
sharedInstance = getSingleton(beanName, () -> {
	try {
		return createBean(beanName, mbd, args);
	}
	catch (BeansException ex) {
		destroySingleton(beanName);
		throw ex;
	}
});
```

6.addSingletonFactory(beanName, ObjectFactory)

```Java
// bean这个参数就是new出来未注入的bean
addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
```


##### inCreationCheckExclusions

`getSingleton(String beanName, ObjectFactory<?> singletonFactory)`这个方法创建之前bean之前会把beanName添加到singletonsCurrentlyInCreation，但在添加之前还要检查是否在inCreationCheckExclusions这个set里面，在这个set里面的beanName是不检查创建的名单，也就是如果要创建的bean是在inCreationCheckExclusions里面就不会添加到singletonsCurrentlyInCreation。



```java
public void setCurrentlyInCreation(String beanName, boolean inCreation) {
	Assert.notNull(beanName, "Bean name must not be null");
	if (!inCreation) {
		this.inCreationCheckExclusions.add(beanName);
	}
	else {
		this.inCreationCheckExclusions.remove(beanName);
	}
}
```

通过这个方法可以设置某个bean不注册进earlySingletonObjects，假如互相依赖的两个bean都不注册进earlySingletonObjects，那就会抛出异常。

### DefaultSingletonBeanRegistry的成员变量

我们知道DefaultSingletonBeanRegistry主要维护管理了BeanFactory的单例Bean，除了singletonObjects这个管理了所有单例的map外，还有很多其他成员变量。我们来看一下DefaultSingletonBeanRegistry的所有成员变量。

```java
/** Cache of singleton objects: bean name to bean instance. */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/** Cache of singleton factories: bean name to ObjectFactory. */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

/** Cache of early singleton objects: bean name to bean instance. */
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
```
singletonObjects，singletonFactories，earlySingletonObjects对于这三个变量，上面已经有了很大的篇幅介绍，单例的注册也主要是维护这几个map。

```java
/** Names of beans that are currently in creation. */
private final Set<String> singletonsCurrentlyInCreation =
		Collections.newSetFromMap(new ConcurrentHashMap<>(16));

/** Names of beans currently excluded from in creation checks. */
private final Set<String> inCreationCheckExclusions =
		Collections.newSetFromMap(new ConcurrentHashMap<>(16));
```

这两个set前面也有介绍到，主要是用来判断某个是否正在被创建，是bean是否循环的主要依据。

```java
/** List of suppressed Exceptions, available for associating related causes. */
@Nullable
private Set<Exception> suppressedExceptions;
```

用来纪录所有创建bean时相关的异常。

```java
/** Disposable bean instances: bean name to disposable instance. */
private final Map<String, Object> disposableBeans = new LinkedHashMap<>();

/** Map between containing bean names: bean name to Set of bean names that the bean contains. */
private final Map<String, Set<String>> containedBeanMap = new ConcurrentHashMap<>(16);

/** Map between dependent bean names: bean name to Set of dependent bean names. */
private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<>(64);

/** Map between depending bean names: bean name to Set of bean names for the bean's dependencies. */
private final Map<String, Set<String>> dependenciesForBeanMap = new ConcurrentHashMap<>(64);
```

待续...
