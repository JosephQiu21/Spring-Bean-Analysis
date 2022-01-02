

# 设计方法模式探讨：Spring Bean 源码阅读报告 3

[TOC]

## 引言

至此，我们对 Spring Bean IoC 容器的理解还仅仅处于较为初级的阶段。对于很多实现上的细节（如生命周期管理、循环依赖解决等），并没有仔细梳理；而对于很多类、流程的设计意图，我们也完全无从知晓。鉴于有限的时间和精力，最后一次源码阅读报告中，我同样将聚焦最简单的问题：

Spring Bean 使用了哪些设计方法模式？体现了哪些设计原则？

## 什么是设计模式？

首先我们需要了解设计模式的概念。

> In [software engineering](https://en.wikipedia.org/wiki/Software_engineering), a **software design pattern** is a general, [reusable](https://en.wikipedia.org/wiki/Reusability) solution to a commonly occurring problem within a given context in [software design](https://en.wikipedia.org/wiki/Software_design). It is not a finished design that can be transformed directly into [source](https://en.wikipedia.org/wiki/Source_code) or [machine code](https://en.wikipedia.org/wiki/Machine_code). Rather, it is a description or template for how to solve a problem that can be used in many different situations.

设计模式是针对软件设计中各种问题所提出的抽象解决方案，它并不直接用来完成代码的编写。设计模式能使**不稳定**依赖于**相对稳定**、**具体**依赖于**相对抽象**，避免**紧耦合**，以增强软件设计面对并适应变化的能力。

需要指出，并非所有的软件模式都是设计模式，设计模式**特指软件“设计”层次上的问题**。还有其他非设计模式的模式，如架构模式。同时，算法不能算是一种设计模式，因为算法主要是用来解决计算上的问题，而非设计上的问题。

## Spring Bean 使用的设计方法模式

### 上期回顾：Bean 的存取

![getBean-flowchart2](https://josephqiu-1305443579.cos.ap-beijing.myqcloud.com/uPic/getBean-flowchart2.png)

在上次的分析中，我们进行了 debug，基本理清了 `getBean()` 方法访问单例注册表 `DefaultSingletonBeanRegistry` 的路径，如上图所示。在这个过程中遇到的一些类，正体现了很重要的设计模版方法。

### AbstractApplicationContext（一）：模版方法模式

一切开始于 `ApplicationContext.getBean()`——在 `ApplicationContext` 接口中调用 `getBean()` 方法。debug 时我们发现这个方法进入到 `AbstractApplicationContext` 的 `getBean()` 方法。`AbstractApplicationContext` 是个抽象类，代码中有如下注释：

> Abstract implementation of the ApplicationContext interface. Doesn't mandate the type of storage used for configuration; simply implements common context functionality. Uses the Template Method design pattern, requiring concrete subclasses to implement abstract methods.

`AbstractApplicationContext` 是对 `ApplicationContext` 接口的抽象实现（其中所有 getBean() 方法都是对 `BeanFactory` 接口中 `getBean()` 方法的实现），其中 `getBean()` 方法都要求先获得一个 `BeanFactory`，再调用 `BeanFactory` 中的 `getBean()` 方法。这里的 `getBeanFactory()` 即为抽象方法，要求子类完成具体实现。

``` java
@Override
	public <T> T getBean(Class<T> requiredType) throws BeansException {
		assertBeanFactoryActive();
		return getBeanFactory().getBean(requiredType);
	}
```

``` java
@Override
	public abstract ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
```

这就是模版方法设计模式：一个抽象类公开定义了执行它的方法的方式/模板。它的子类可以按需要重写方法实现，但调用将以抽象类中定义的方式进行。`AbstractApplicationContext` 本身并没有组合 `BeanFactory`——如果这样，它就需要考虑不同情形，组合多种 `BeanFactory`。而利用模板方法模式，`BeanFactory` 的获取被延迟到各个子类中进行。在这个 demo 中，我们在子类 `AbstractRefreshableApplicationContext` 中获得组合的 `DefaultListableBeanFactory`。

虽然是个非常简单的应用，但也体现了模板方法模式的优点和思想：

1. 封装不变部分（先获得 `BeanFactory`，再调用 `getBean()` 方法），扩展可变部分（获得哪种 `BeanFactory`？）。
2. 提取公共代码，便于维护。
3. 行为由父类控制，子类实现。

### AbstractApplicationContext（二）：抽象工厂模式

在 Spring 中，我们对获得的 Bean 实例有各种不同的要求，如多个独立实例或单一共享实例（分别对应多例/单例模式）：

> Depending on the bean definition, the factory will return either an independent instance of a contained object (the Prototype design pattern), or a single shared instance (a superior alternative to the Singleton design pattern, in which the instance is a singleton in the scope of the factory).

如果 `ApplicationContext`/BeanFactory 直接实现各种不同需求的 Bean 获取与配置，内部逻辑将变得过于复杂。因此，每种需求将由各种不同的工厂创建。最终 `ApplicationContext` 与 `BeanFactory` 都成为抽象工厂，即“工厂的工厂”，将 Bean 的具体获得过程与具体工厂的调用封装起来，应用直接对抽象工厂发起请求，抽象工厂将请求交给具体工厂处理。

经过上次的分析不难发现，Spring Bean 中使用的工厂有相当多的层次结构：

![DefaultListableBeanFactory](https://josephqiu-1305443579.cos.ap-beijing.myqcloud.com/uPic/DefaultListableBeanFactory.png)

而我们上次对单例 Bean 发起的 `getBean()` 请求最终被 `DefaultListableBeanFactory` 处理，它正是最终的具体工厂。

工厂的抽象层次也可以层层细分，每一层抽象工厂设立的依据则是**子工厂的共性**。也就是说，我们可以将有共同方法、共同功能的子工厂用一层抽象工厂封装。比如，我们可以在 Spring Bean 源码中阅读到 `AbstractBeanFactory` 的设立意图：

> Does not assume a listable bean factory: can therefore also be used as base class for bean factory implementations which obtain bean definitions from some backend resource (where bean definition access is an expensive operation).
>
> This class provides a singleton cache (through its base class DefaultSingletonBeanRegistry, singleton/prototype determination, FactoryBean handling, aliases, bean definition merging for child bean definitions, and bean destruction (org.springframework.beans.factory.DisposableBean interface, custom destroy methods). 
>
> Furthermore, it can manage a bean factory hierarchy (delegating to the parent in case of an unknown bean), through implementing the org.springframework.beans.factory.HierarchicalBeanFactory interface.

`AbstractBeanFactory` 并不是对 listable 工厂的抽象封装（该功能将由更细粒度的工厂实现），因此可以作为其它一些具体工厂的基类；但它继承了 `DefaultSingletonBeanRegistry`，从而提供了单例缓存。将单例缓存支持设置在这一级，因此所有需要单例缓存的工厂都可以将 `AbstractBeanFactory` 作为基类。

每一级工厂提供哪些功能的抽象封装、将哪些细分功能留给子工厂实现、设立多少层次，都是在组织工厂框架时需要仔细考虑的问题。

### DefaultSingletonBeanRegistry：单例模式

上次分析中说到，`DefaultSingletonBeanRegistry` 是存放单例 Bean 的注册表。为什么我们需要单例 Bean？如何实现单例 Bean 的管理？

在我们的系统中，有一些对象其实我们只需要一个，比如说：线程池、缓存、对话框等等。事实上，这一类对象只能有一个实例，而创造多个实例则可能导致程序行为异常、资源使用过量、不一致性等糟糕的结果。

而且，使用单例对象可以省略重复创建对象花费的时间，这是一笔可观的系统开销。

从而，Spring 中 Bean 的默认作用域就是单例（Singleton）的。（当然也提供对多例等其它作用域的支持）

实现单例的方法中，常见的有：

* 懒汉式：在首次调用时进行实例化，后续调用时使用之前生成的实例。
* 饿汉式：项目启动时实例化所有单例对象。

而 Spring 实现单例模式的方法则是**单例注册表**。

我们回过头去查看 `DefaultSingletonBeanRegistry` 的代码，其单例池 `singletonObjects` 为一个 `ConcurrentHashMap`，是一个线程安全的哈希表。

``` java
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
```

`getBean()` 方法最终调用到 doGetBean()，它将试图从这个单例池中获取 Bean（通过 `getSingleton()` 方法）。若 Bean 已经被完全创建好，存储在一级缓存中，则直接获取，否则需要创建并初始化。

在创建过程中又涉及到循环依赖的解决（A 依赖 B，B 又依赖 A）。正是为了解决循环依赖，`DefaultSingletonBeanRegistry` 中设置了三级缓存：

1. 单例池（`singletonObjects`）
2. 提前曝光对象（`earlySingletonObjects`）
3. 提前曝光对象工厂（`singletonFactories`）

具体解决的流程和原理较为复杂，简单描述就是：

- 在创建 Bean 的过程中，获取到实例对象后会提前暴露出去，生成一个 `ObjectFactory` 对象，放入三级缓存中
- 在后续设置属性过程中，如果出现循环依赖，则可以通过三级缓存中对应的 `ObjectFactory#getObject()` 获取这个早期对象，避免再次初始化

之所以使用三级缓存，与 Spring AOP 代理有关，这就涉及我的未知领域了……

## Spring 体现的设计原则

最后我们简单看一下 Spring 体现的设计原则。

### 接口职责单一

我们看一下 Spring IoC 容器提供的接口：

![ApplicationContext](https://josephqiu-1305443579.cos.ap-beijing.myqcloud.com/uPic/ApplicationContext.png)

其中：

* `BeanDefinitionRegistry` 定义了关于 BeanDefinition 的注册、移除、查询方法
* `ResourceLoader` 定义了加载类路径/文件系统资源等资源的方法
* `BeanFactory` 提供了面向客户的容器接口

正是因为接口指责单一，使得不同接口之间可任意组合，分别扩展。如果需要对某一部分做扩展，则只需要提供该接口的新的实现，对其它接口不会产生影响。

`ApplicationContext` 正是继承了多个指责单一的接口： `EnvironmentCapable`, `ListableBeanFactory`, `HierarchicalBeanFactory`, `MessageSource`, `ApplicationEventPublisher`, `ResourcePatternResolver`。

并且 `ApplicationContext` 的实现类中并没有直接实现这些接口中的方法，而是通过代理模式，将接口的实现代理到了成员变量上：

``` java

	/** ResourcePatternResolver used by this context. */
	private ResourcePatternResolver resourcePatternResolver;

	/** LifecycleProcessor for managing the lifecycle of beans within this context. */
	@Nullable
	private LifecycleProcessor lifecycleProcessor;

	/** MessageSource we delegate our implementation of this interface to. */
	@Nullable
	private MessageSource messageSource;

	/** Helper class used in event publishing. */
	@Nullable
	private ApplicationEventMulticaster applicationEventMulticaster;

```

可以看到 `ApplicationContext` 在其抽象实现类 `AbstractApplicationContext` 中对 `ResourcePatternResolver`、`ListableBeanFactory`、`MessageSource` 等多个接口做了组合，如果需要对其中一个接口做扩展，只需要单独实现该接口，并组合进已有的 `ApplicationContext` 中即可。

## 总结与感想

Spring Bean 的源码阅读与分析之旅到此也要告一段落了。时间、精力、难度、经验，种种原因限制下，做到这里大约是我的极限了。那么，我们完成了哪些工作，解答了哪些问题呢？

1. Spring 框架的核心技术
2. Spring 容器的使用方式
3. Bean 的存储位置
4. getBean() 方法的执行路径
5. Spring Bean 下各个类在 Bean 存取过程中的作用
6. Bean 存取过程中体现的设计方法与原则

虽然了解掌握的内容依然相当有限，但足以获得一些学习与成长的快乐。

而更多的，是感触：

1. “盲人摸象”：Spring 框架既是庞然大物，但在结构层次的组织、模块的划分之下，又显得错落有致。而对于 Java 编程经验欠缺，OOP 思想浅尝辄止的我们来说，阅读 Spring 源码有一种“盲人摸象”之感。虽然是盲人，但也要通过各种方法建立起整体认知，把握框架设计的层次结构。
2. “Architecture Astronauts”：这是 John Carmack 在 Facebook Connect 大会的主题演讲中对那些只从抽象层次审视软件开发的 Programmer 的讽刺性称呼。我对这些抽象设计思想并不带有讽刺的态度，但也会觉得 Astronauts 的说法很形象：在这种层次上，很容易有凌空蹈虚之感。虽然设计原则、设计方法的总结对软件设计与开发是极好的指导，但有时写着写着就会觉得自己是在将源码强行解释，这可能并非这些原则、方法提出的本意。我觉得还是应该在具体的软件代码开发、维护与重构过程中理解这些原则要来得好些。
3. “Comfort Zone”：习惯了 C/Verilog 这些接近底层实现的语言，我会形成一种舒适圈，并且觉得 Java 有些难以接近。看着 Java 的语法，便会不由自主地思考它是怎么实现的，内存中的栈又是如何变化的。C 确实是简洁、具象的语言，但也注定不太适合企业级的软件开发。虽然现在接触、实现的项目都规模有限，但未来无可避免地会参与到大规模项目的开发，需要考虑各类协作、维护、扩展、重构的问题——这也正是 OOP 存在、发展的意义。我也必须应该走出这个舒适圈，面对更现实的世界（笑）。

## 参考资料

1. 维基百科「Software design pattern」条目，https://en.wikipedia.org/wiki/Software_design_pattern
1. Spring Framework 官方文档，https://docs.spring.io/spring-framework/docs/current/reference/html/
1. 菜鸟教程设计模式相关解析，https://www.runoob.com/design-pattern/design-pattern-tutorial.html 等
1. 从 Spring 看框架设计原则，https://www.jianshu.com/p/963649e42d5
1. Design Patterns used in Spring Framework，https://www.sourcecodeexamples.net/2018/04/design-patterns-used-in-spring-framework.html
