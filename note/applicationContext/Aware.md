## Aware

#### Aware Interface

Aware Interfaces:

| Name | Injected Dependency |
| ---- | ------------------- |
|[ApplicationContext](/note/applicationContext/applicationContext.md)Aware  |Declaring ApplicationContext.  |
|ApplicationEventPublisherAware  |	Event publisher of the enclosing  ApplicationContext.  |
|BeanClassLoaderAware  |Class loader used to load the bean classes.  |
|BeanNameAware|Name of the declaring bean.|
|BootstrapContextAware|Resource adapter BootstrapContext the container runs in. Typically available only in JCA aware ApplicationContext instances.
|LoadTimeWeaverAware | Defined weaver for processing class definition at load time.|
|MessageSourceAware|Configured strategy for resolving messages (with support for parametrization and internationalization).|
|NotificationPublisherAware|Spring JMX notification publisher.|
|ResourceLoaderAware|Configured loader for low-level access to resources.|
|ServletConfigAware|Current ServletConfig the container runs in. Valid only in a web-aware Spring ApplicationContext.|
|ServletContextAware| Current ServletContext the container runs in. Valid only in a web-aware Spring ApplicationContext. |

但是Spring建议尽量避免使用这些接口，因为这样会使代码和Spring耦合。如果在bean中需要使用ApplicationContext这种对象可以使用`@Autowired`这个注解。

```java
@Autowired
private ApplicationContext applicationContext;
```
