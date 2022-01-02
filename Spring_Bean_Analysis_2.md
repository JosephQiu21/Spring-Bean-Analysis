

# Bean 是如何被存取的：Spring Bean 源码阅读报告 2

[TOC]

## 序言

上一次报告中，我们最重要的收获是认识了 Spring 框架所提供的 IoC 容器：当我们需要根据当前需求实例化某个组件时，可以通过 XML 等配置信息将所需要的依赖项告知 IoC 容器，容器就会根据这些依赖项创建我们所需的组件，即 Bean。借用官方文档中的 Spring 工作流程图：

<img src="https://josephqiu-1305443579.cos.ap-beijing.myqcloud.com/uPic/xSVfov.png" alt="Spring 容器的工作方式" style="zoom: 33%;" />

一句“Magic Happens Here”不能不引起我们的好奇心：Spring 的 IoC 容器这个黑匣子里，到底有什么 Magic？它是如何完成配置信息读取、实例化 Bean、生命周期管理等工作的？这将是我们本次源码阅读报告的主题。

限于时间、篇幅，我们将关注点集中在这个最简单也最关键的问题上：

**我们所需的 Bean，到底存放在哪？它是如何被存取的？**

## 从 Bean 的获取开始：getBean() 方法

上次报告中提到，Spring Bean 提供了两种常见的 IoC 容器：`BeanFactory` 和 `ApplicationContext`。我们以 `BeanFactory` 为例：

``` java
public interface BeanFactory {

	String FACTORY_BEAN_PREFIX = "&";

	Object getBean(String name) throws BeansException;

	<T> T getBean(String name, Class<T> requiredType) throws BeansException;

	Object getBean(String name, Object... args) throws BeansException;

	<T> T getBean(Class<T> requiredType) throws BeansException;

	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

	<T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

	<T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

	boolean containsBean(String name);

	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

	boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

	@Nullable
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	@Nullable
	Class<?> getType(String name, boolean allowFactoryBeanInit) throws NoSuchBeanDefinitionException;

	String[] getAliases(String name);

}
```

可以看到，`BeanFactory` 是一个接口，仅提供了多种 `getBean()`方法，并未定义字段去存储 Bean，也没有出现 `addBean()` 之类的方法。事实上，它只是提供了一个方法规范，要求所有实现 `BeanFactory` 接口的子类都必须实现 `getBean()` 方法。换句话说，`BeanFactory` 接口只定义了如何**访问**容器内 Bean 的方法， 而具体 Bean 的注册和管理工作，则由各个 `BeanFactory` 的具体实现类负责。

我们通过 Intellij IDEA 分析一下 `BeanFactory` 的子接口和实现类：

<img src="https://josephqiu-1305443579.cos.ap-beijing.myqcloud.com/uPic/D8Lr5K.jpg" alt="BeanFactory 的子接口和实现类" style="zoom: 50%;" />

可见`BeanFactory`有3个子接口:`ListableBeanFactory`、`HierarchicalBeanFactory`、`AutowireCapableBeanFactory`。最终的默认实现类是`DefaultListableBeanFactory`，其实现了上面所有的接口。

阅读 `DefaultListableBeanFactory` 的源码，可以发现它已经实现了 Bean 的注册，正如此处的 `registerSingleton()` 方法：

```java
@Override
	public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
		super.registerSingleton(beanName, singletonObject);
		updateManualSingletonNames(set -> set.add(beanName), set -> !this.beanDefinitionMap.containsKey(beanName));
		clearByTypeCache();
	}
```

于是我们可以顺藤摸瓜，在 `SingletonBeanRegistry` 和 `DefaultSingletonBeanRegistry` 中找到该方法，而最终的实现则是在后者中：

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
```

而事实证明， `DefaultSingletonBeanRegistry` **正是我们寻找的，实现 Bean 存取的地方。**

其最重要的三个成员变量如下：

``` java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {

	/** Cache of singleton objects: bean name to bean instance. */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	/** Cache of singleton factories: bean name to ObjectFactory. */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

	/** Cache of early singleton objects: bean name to bean instance. */
	private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

```

通过注释和官方文档我们可以得知，其中的 `singletonObjects` 即为存放单例 Bean 的 map，由名称映射到 Bean 的实例。而其余两级缓存则用于解决循环依赖。同样，`DefaultSingletonBeanRegistry` 也提供了 `registerSingleton()`、`addSingleton()` 等一系列提供 Bean 存取服务的方法。但问题是，`DefaultSingletonBeanRegistry` 没有 `getBean()` 方法——它没有实现 `BeanFactory`！它只是一个注册表，实现了 `SingletonBeanRegistry`：

``` java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
```

那么，我们在实际应用时用到的 `getBean()` 方法，到底是如何抵达这个 `DefaultSingletonBeanRegistry` 这个注册表，或者说 `singletonObjects` 这个单例池的？

我们不妨来 debug 看看。

## Debug：getBean() 方法如何抵达目的地？

我们依然沿用上一次源码阅读报告中来自廖雪峰老师的 demo，这里使用的容器为 `ApplicationContext`：

``` java
public class Main {
    public static void main(String[] args) {
      	// ApplicationContext: Spring bean
      	// Construct an installation & Load configuration file
        ApplicationContext context = new ClassPathXmlApplicationContext("application.xml");
      	// Use getBean() method
        UserService userService = context.getBean(UserService.class);
      	// Use it normally
        User user = userService.login("bob@example.com", "password");
        System.out.println(user.getName());
    }
}
```

在 `getBean()` 方法处单步进入，我们来到抽象类 `AbstractApplicationContext` 的 `getBean()` 方法（这里以类型为参数，所需为 `UserService` 类型）；接着进入，是 `DefaultListableBeanFactory` 的 `getBean()` 方法；接着是 `resolveBean()` 和 `resolveNamedBean()` 方法，后者再次调用 `getBean()` 方法（这里以名称、类型为参数，所需 Bean 的名称为 `userService`），然后进入 `AbstractBeanFactory` 的 `doGetBean()` 方法，然后——

终于来到了 `DefaultSingletonBeanRegistry` 的 `getSingleton()` 方法！

![getBean 方法调用栈](https://josephqiu-1305443579.cos.ap-beijing.myqcloud.com/uPic/IBIswy.png)

我们来仔细看一下这个过程。最让我们不解的一点是，我们本以为 `AbstractApplicationContext` 必然是通过层层继承，最终继承到 `DefaultSingletonBeanRegistry`，然后 `getBean()` 方法在某一层调用了 `DefaultSingletonBeanRegistry` 提供的 `getSingleton()` 方法。但检查类继承图，发现这两个类互相并不认识！毫无关系的两个类是如何实现调用的？

![相关类继承图](https://josephqiu-1305443579.cos.ap-beijing.myqcloud.com/uPic/VSTXmR.jpg)

除了继承后使用父类方法访问父类变量，还有别的途径吗？有，那就是组合/聚合。

在 `AbstractApplicationContext` 中，先使用 `getBeanFactory()` 方法获得一个 `BeanFactory`，然后再调用 `getBean()` 方法：

``` java
public <T> T getBean(Class<T> requiredType) throws BeansException {
        this.assertBeanFactoryActive();
        return this.getBeanFactory().getBean(requiredType);
}
```

这里的 `getBeanFactory()` 是一个抽象方法，交给子类实现：

``` java
public abstract ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
```

在子类 `AbstractRefreshableApplicationContext` 中，果然**组合了一个字段 `beanFactory`，其为 `DefaultListableBeanFactory`**。

``` java
public abstract class AbstractRefreshableApplicationContext extends AbstractApplicationContext {
    @Nullable
    private Boolean allowBeanDefinitionOverriding;
    @Nullable
    private Boolean allowCircularReferences;
    @Nullable
    private DefaultListableBeanFactory beanFactory;

```

通过类继承图，我们可以发现正是 `DefaultListableBeanFactory` **继承了 `DefaultSingletonBeanRegistory`，从而可以访问单例池**，犹如定义在自己内部；**同时实现了 `BeanFactory`，从而可以使用 `getBean()` 方法。**可以看出，`DefaultListableBeanFactory` 在这里扮演了一个类似中介的角色！

![DefaultListableBeanFactory](https://josephqiu-1305443579.cos.ap-beijing.myqcloud.com/uPic/DefaultListableBeanFactory.png)

到此为止，我们对 `getBean()` 方法抵达 `DefaultSingletonBeanRegistry` 中的单例池 `singletonObjects` 的过程已经较为清晰了，其中关键点如下：

1. 在 `AbstractApplicationContext` 中通过模版方法设计模式，在子类 `AbstractRefreshableApplicationContext` 中实现 `getBeanFactory()` 方法
2. 子类 `AbstractRefreshableApplicationContext` 中组合了一个 `BeanFactory`，那就是 `DefaultListableBeanFactory`。组合关系在类继承图中无法体现，这是我们疑惑的来源
3. `DefaultListableBeanFactory` 在整个过程中扮演了一个中介的角色。它继承了 `DefaultSingletonBeanRegistory`，从而可以访问单例池，犹如定义在自己内部；同时实现了 `BeanFactory`，从而可以使用 `getBean()` 方法

全过程的流程图如下：

![getBean-flowchart2](https://josephqiu-1305443579.cos.ap-beijing.myqcloud.com/uPic/getBean-flowchart2.png)

## 总结&下期预告

本次分析纠结了很久，也没有完全确定写哪一部分的具体工作流程。助教老师推荐的主题，即“Bean 的生命周期管理”，我阅读了不少教程和网络博客，仍有云山雾罩的感觉。最后还是觉得，与其毫无理解地做网络博客和教程的搬运工，不如从最简单的层次出发，基于 debug，研究一下 getBean() 的执行流程，弄清楚 Bean 到底是存放在哪里的。我现在如同执灯入室，厅堂中绝大多数仍是难以捉摸的黑暗，但至少身边的几处是被照亮的、可以看清的。

总体来看，此次分析可能大致符合课程要求，即分析关键工作流程、分析类之间的联系。但我担心内容可能不够充实。另外对于下次分析，要求从设计模式的层次出发，进行设计意图的探讨，我目前大致方向有两点，一是探讨 Spring Bean 中的工厂模式，二是 Spring Bean 在获取单例 Bean 时如何确保单例（可能有个解决循环依赖的问题，这点在网上分析很多？）。不知道老师有什么建议吗？

## 参考资料

1. Spring源码解析(1)：Bean容器，https://zhuanlan.zhihu.com/p/74832770
2. Spring Framework 官方文档，https://docs.spring.io/spring-framework/docs/current/reference/html/













