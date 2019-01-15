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

#### BeanPostProcessor
