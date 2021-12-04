

# 设计方法模式探讨：Spring Bean 源码阅读报告 3

## 引言

施工中，请勿通行。

## TODO...

这里有个小问题我们可以顺带关注一下：既然 DefaultSingletonBeanRegistry 是专门管理单例 Bean 的注册表，那么它是如何保证单例的？

我们可以注意到它的一个较为特殊的成员 Set<String> singletonsCurrentlyInCreation，专门来存放正在创建的单例 Bean 的名字。之所以只是名字而不是 Bean 本身，是因为 Bean 还在创建之中。另外，有两个专门设计的方法 beforeSingletonCreation 和 afterSingletonCreation，

``` java
/** Names of beans that are currently in creation. */
	private final Set<String> singletonsCurrentlyInCreation =
			Collections.newSetFromMap(new ConcurrentHashMap<>(16));

/**
	 * Callback before singleton creation.
	 * <p>The default implementation register the singleton as currently in creation.
	 * @param beanName the name of the singleton about to be created
	 * @see #isSingletonCurrentlyInCreation
	 */
	protected void beforeSingletonCreation(String beanName) {
		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}
	}

	/**
	 * Callback after singleton creation.
	 * <p>The default implementation marks the singleton as not in creation anymore.
	 * @param beanName the name of the singleton that has been created
	 * @see #isSingletonCurrentlyInCreation
	 */
	protected void afterSingletonCreation(String beanName) {
		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) {
			throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
		}
	}
```

