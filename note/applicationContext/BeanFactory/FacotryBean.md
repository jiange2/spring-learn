## FactoryBean

---

#### 简介

 Spring中有两种类型的Bean，一种是普通Bean，另一种是工厂Bean，即FactoryBean。工厂Bean跟普通Bean不同，其返回的对象不是指定类的一个实例，其返回的是该工厂Bean的getObject方法所返回的对象。

 #### BeanFactory和FactoryBean

 很多人可能会因为这两者的名字而混淆，但实际上这两者的名字是十分清晰的。

 FactoryBean是Spring容器里面一种特殊的Bean, 它是一个工厂Bean，Spring调用getBean方法获取到FactoryBean的时候会调用它的getObject把返回Object作为返回结果，而不是直接返回FactoryBean。

 Factory是一个管理Bean的Factory，是spring IOC容器的核心，里面管理了所有的Bean，当然包括配置的FactoryBean。

 #### 例子

 Product.class和ProductFactoryBean

 ``` java
 public class Product {}

 public class ProductFactoryBean implements FactoryBean {
     @Override
     public Object getObject() throws Exception {
         return new Product();
     }

     @Override
     public Class<?> getObjectType() {
         return Product.class;
     }
 }
```

ProductFactoryBean要实现FactoryBean这个接口。

配置:

```xml
<bean id="product" class="appcontext.factorybean.ProductFactoryBean"/>
```

测试代码:

```java
ClassPathXmlApplicationContext appContext =
        new ClassPathXmlApplicationContext("classpath:app-context.xml");
System.out.println(appContext.getBean("product"));
```

结果: `appcontext.factorybean.Product@134593bf`

可以看到打印的结果是Product而不是ProductFactoryBean。

#### 获取FactoryBean本身

FactoryBean本身也是一个Bean，如果我们不想获取FactoryBean生产的实例，而是获取它本身，我们可以通过 '&' + beanName 来获取FactoryBean本身。

测试代码:
```java
ClassPathXmlApplicationContext appContext =
       new ClassPathXmlApplicationContext("classpath:app-context.xml");
System.out.println(appContext.getBean("&product"));
```

结果: `appcontext.factorybean.ProductFactoryBean@134593bf`

#### 为什么要有FactoryBean?

使用FactoryBean更加灵活，因为创建Bean的流程完全由我们自己控制。
