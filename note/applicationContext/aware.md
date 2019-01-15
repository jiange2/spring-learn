## Aware

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
|||
