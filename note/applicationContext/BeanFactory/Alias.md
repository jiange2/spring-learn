## Alias

---

Spring一般可以通过BeanName和Bean的type来获取bean。那Bean的Name是根据我们配置决定的，主要是跟id和name这两个属性有关系。name的逻辑：
 - 假如配置了id，BeanName就是id。
 - 同样，配置了name，那BeaName就是配置的name。
 - 如果同时配置了id，那BeanName就是id，而配置的name就是Aliase。
 - 如果配置了多个name，会取首个name作为BeanName,其他是Aliase
 - 如果都没配的话就是`appcontext.(全限定类名)#0。`

无论是BeanName或是Aliase都可以通过getBean(String name)的方式来获取bean。

##### AliasRegistry:

```java

public interface AliasRegistry {

	void registerAlias(String name, String alias);

	void removeAlias(String alias);

	boolean isAlias(String name);

	String[] getAliases(String name);

}

```

AliasRegistry提供了管理访问别名的方法。

##### register alias:

```Java
public void registerAlias(String name, String alias) {
	Assert.hasText(name, "'name' must not be empty");
	Assert.hasText(alias, "'alias' must not be empty");
	synchronized (this.aliasMap) {
    // name == alias,相当于要注册的alias其实就是name
		if (alias.equals(name)) {
			this.aliasMap.remove(alias);
			if (logger.isDebugEnabled()) {
				logger.debug("Alias definition '" + alias + "' ignored since it points to same name");
			}
		}
		else {
			String registeredName = this.aliasMap.get(alias);
			if (registeredName != null) {
        //注册过一摸一样的了
				if (registeredName.equals(name)) {
					return;
				}
        //要改alias的name，如果allowAliasOverriding是false（默认true）就抛异常
				if (!allowAliasOverriding()) {
					throw new IllegalStateException("Cannot define alias '" + alias + "' for name '" +
							name + "': It is already registered for name '" + registeredName + "'.");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Overriding alias '" + alias + "' definition for registered name '" +
							registeredName + "' with new target name '" + name + "'");
				}
			}
      //检查循环
			checkForAliasCircle(name, alias);
			this.aliasMap.put(alias, name);
			if (logger.isTraceEnabled()) {
				logger.trace("Alias definition '" + alias + "' registered for name '" + name + "'");
			}
		}
	}
}
```

aliasMap存放了alias -> name的映射。可以看出来要用alias找name是比较方便的，但是要从name找alias就需遍历整个map。

##### alias循环映射：

从上面源码看，在把alias注册的最后一步需要检查alias循环，那什么是alias循环呢？

```Java
protected void checkForAliasCircle(String name, String alias) {
	if (hasAlias(alias, name)) {
		throw new IllegalStateException("Cannot register alias '" + alias +
				"' for name '" + name + "': Circular reference - '" +
				name + "' is a direct or indirect alias for '" + alias + "' already");
	}
}
```

我们可以发现hasAlias的参数是name、alias，而checkForAliasCircle调用hasAlias的时候传的是alias、name，把两个参数调换着传。我们在看hasAlias的源码之前，可以简单推断，如果hasAlias的结果是true的话,其实就是说已经存在了name -> alias的情况，如果我们再注册一个 alias -> name ，那就出现了循环的情况，这个时候会抛出异常，也就是alias循环是不被允许的。

##### alias传递映射:

```java
public boolean hasAlias(String name, String alias) {
	for (Map.Entry<String, String> entry : this.aliasMap.entrySet()) {
		String registeredName = entry.getValue();
		if (registeredName.equals(name)) {
			String registeredAlias = entry.getKey();
			if (registeredAlias.equals(alias) || hasAlias(registeredAlias, alias)) {
				return true;
			}
		}
	}
	return false;
}
```

`registeredAlias.equals(alias) || hasAlias(registeredAlias, alias)` 通过这句,我们发现 hasAlias除了 alias 直接指向 name 的情况(`registeredAlias.equals(alias)`)。还有间接指向的情况(`hasAlias(registeredAlias, alias)`)

举个例子：

假如hasAlias的参数是 name:C, alias:A

那假如注册过 A -> C, 那返回结果肯定是true。

而对于另外一种传递引用的也会是true。 比如注册过： A -> B,B -> C。代码首先找到了C的直接alias B,但是发现B并不等于A，那就会递归去找B的Alias有没有A。这个时候就可以找到A，那结果就是true。

所以对于前面判断循环就不只有直接的互相指向才算是循环(A->B,B->A)，间接互相指向也算是循环(A->B,B->C,C->A)。

##### 根据alias获取name：

```Java
public String canonicalName(String name) {
	String canonicalName = name;
	String resolvedName;
	do {
		resolvedName = this.aliasMap.get(canonicalName);
		if (resolvedName != null) {
			canonicalName = resolvedName;
		}
	}
	while (resolvedName != null);
	return canonicalName;
}
```

方法比较简单，就是要处理一下传递映射的问题。

假如传入的参数name是A：

1. 对于直接只有 A->B 这样的映射，那结果就是B。

2. 对于有传递映射比如： A->B, B->C的情况。结果就是C。

从这个逻辑也可以看出来为什么alias不允许循环映射。

##### 获取name的所有alias：

```Java
@Override
	public String[] getAliases(String name) {
		List<String> result = new ArrayList<>();
		synchronized (this.aliasMap) {
			retrieveAliases(name, result);
		}
		return StringUtils.toStringArray(result);
	}

	private void retrieveAliases(String name, List<String> result) {
		this.aliasMap.forEach((alias, registeredName) -> {
			if (registeredName.equals(name)) {
				result.add(alias);
				retrieveAliases(alias, result);
			}
		});
	}
```

因为传递映射的关系，所以alias的关系有可能会形成树状结构，所以这里的代码就跟递归遍历树差不多。

##### resolveAliases ：

```java
public void resolveAliases(StringValueResolver valueResolver) {
	Assert.notNull(valueResolver, "StringValueResolver must not be null");
	synchronized (this.aliasMap) {
		Map<String, String> aliasCopy = new HashMap<>(this.aliasMap);
		aliasCopy.forEach((alias, registeredName) -> {
			String resolvedAlias = valueResolver.resolveStringValue(alias);
			String resolvedName = valueResolver.resolveStringValue(registeredName);
			if (resolvedAlias == null || resolvedName == null || resolvedAlias.equals(resolvedName)) {
				this.aliasMap.remove(alias);
			}
			else if (!resolvedAlias.equals(alias)) {
				String existingName = this.aliasMap.get(resolvedAlias);
				if (existingName != null) {
					if (existingName.equals(resolvedName)) {
						// Pointing to existing alias - just remove placeholder
						this.aliasMap.remove(alias);
						return;
					}
					throw new IllegalStateException(
							"Cannot register resolved alias '" + resolvedAlias + "' (original: '" + alias +
							"') for name '" + resolvedName + "': It is already registered for name '" +
							registeredName + "'.");
				}
				checkForAliasCircle(resolvedName, resolvedAlias);
				this.aliasMap.remove(alias);
				this.aliasMap.put(resolvedAlias, resolvedName);
			}
			else if (!registeredName.equals(resolvedName)) {
				this.aliasMap.put(alias, resolvedName);
			}
		});
	}
}
```

待续。。。
