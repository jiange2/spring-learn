## Application Context简介

Spring的本质就是一个容器，而ApplicationContext就是Spring的容器。
以ClassPathApplicationContext为例来分析 ApplicationContext.

ClassPathApplicationContex继承关系图:

![ClassPathApplicationContex继承关系图](https://raw.githubusercontent.com/jiange2/spring-learn/master/image/ApplicationContext/ClassPathXmlApplicationContext.png)

我们通过ClassPathXmlApplicationContext 实现的接口和继承类，来知道ApplicationContext包含了哪些功能，以及分模块的去了解ApplicationContext的实现细节。

### ApplicationContext 的 Interfaces

###### Aware / BeanNameAware

ApplicationContext实现了这个BeanNameAware接口，这个对ApplicationContext作用不是很大。
Aware的作用是可以让每个Spring管理的Bean可以获取ApplicationContext或Bean本身的一些信息。

比如通过实现BeanNameAware，Bean可以获取自己Bean的Name。

详细： [Aware](/aware.md)
