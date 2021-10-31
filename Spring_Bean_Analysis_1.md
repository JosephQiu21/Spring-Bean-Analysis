# 基本概念和使用：Spring Bean 源码阅读报告1

## 序言

作为面向对象程序设计和 Java 语言的初学者，本次源码阅读报告是一个从 0 到 1 的过程。我希望通过对 Spring Bean 的分析，了解它的功能、理解它蕴含的面向对象思想、甚至窥探设计者的意图和思路。此类顶级开源项目无疑能在思维、实践上给予我们多层次的指导。

第一次的源码阅读报告将从 Spring 框架的官方文档出发，**理清一些概念上的基本问题**，**了解 Spring Bean 的具体功能**，并作初步分析。

我们从最简单的问题开始：**什么是 Spring？什么是 Spring Bean**？

## 什么是 Spring 框架

### Spring 框架的模块

> The Spring Framework is divided into modules. Applications can choose which modules they need. At the heart are the modules of the core container, including a configuration model and a dependency injection mechanism. Beyond that, the Spring Framework provides foundational support for different application architectures, including messaging, transactional data and persistence, and web. It also includes the Servlet-based Spring MVC web framework and, in parallel, the Spring WebFlux reactive web framework.

Spring 框架提供了数个模块，可助力企业环境 Java 程序开发（此处的“企业环境”，大概指的是项目规模大、协作程度高）。其中的核心是与核心容器相关的几个模块，包含**配置模型**、**依赖项注入机制**等。此外，Spring 框架也提供了对消息传递、事务性数据、持久性等应用程序体系结构的支持。

即使并不清楚它们的作用，我们也将最主要的模块列举如下：

- 支持 IoC 和 AOP 的容器；
- 支持 JDBC 和 ORM 的数据访问模块；
- 支持声明式事务的模块；
- 支持基于 Servlet 的 MVC 开发；
- 支持基于 Reactive 的 Web 开发；
- 以及集成 JMS、JavaMail、JMX、缓存等其他模块。

### Spring 框架的核心技术：IoC

> Foremost amongst these is the Spring Framework’s Inversion of Control (IoC) container. A thorough treatment of the Spring Framework’s IoC container is closely followed by comprehensive coverage of Spring’s Aspect-Oriented Programming (AOP) technologies.

Spring 框架的核心技术有两个：IoC 容器（Inversion of Control，控制反转）、AOP 技术（Aspect-Oriented Programming，面向切面编程）。

由于本次分析的主题为 Spring Bean，其并不涉及后者，因此我们只介绍前者。

传统的应用程序中，很多不同的组件都需要通过构造器创建并持有相同的实例。实例化一个组件其实很难，比如你可能需要先读取配置。所以，让这些共同需要的实例共享才是合理的选择。但谁负责创建、分配这些实例？谁负责销毁实例，并保证所有的使用者也都全部被销毁？我们很难处理。**当一个系统的组件数量过多，其生命周期、依赖关系如果靠组件自身维护，会大大增加系统复杂度和耦合度。**

下面的例子来自廖雪峰老师的 Java 教程，直观地表现了 IoC 的原理：

为了从数据库中查询书籍，我们需要实例化一个 `HikariDataSource`；而为了实例化它，我们需要先实例化配置 `HikariConfig`。

``` java
public class BookService {
    // Read configuration first
    private HikariConfig config = new HikariConfig();
    // Use constructor to construct datasource
    private DataSource dataSource = new HikariDataSource(config);

    public Book getBook(long bookId) {
        try (Connection conn = dataSource.getConnection()) {
            ...
            return book;
        }
    }
}
```

IoC 则将控制权从应用程序转移到 IoC 容器。所有组件不再由应用程序自己创建和配置，而是由 IoC 容器负责。

```java
public class BookService {
    private DataSource dataSource;
		
  	// Injection
    public void setDataSource(DataSource dataSource) {
        this.dataSource = dataSource;
    }
}
```

在上面的例子中，`BookService` 自己并不会创建  ` DataSource` ，而是等待 IoC 容器通过 `setDataSource()` 方法来注入一个 `DataSource`。

通过将自己构建改为注入，组件不必关心如何创建所需的依赖项，因此不必读取配置等代码；共享组件变得简单，只需要将组件注入到不同对象中。

怎么告诉容器组件的创建方法和各组件的依赖关系呢？最简单的配置是通过 XML 文件实现的。（Spring 的容器配置方式其实还有**注解**和 **@configuration 配置类**两种。）

```xml
<beans>
    <bean id="dataSource" class="HikariDataSource" />
    <bean id="bookService" class="BookService">
        <property name="dataSource" ref="dataSource" />
    </bean>
    <bean id="userService" class="UserService">
        <property name="dataSource" ref="dataSource" />
    </bean>
</beans>
```

该 XML 文件指示 IoC 容器创建 3 个 Bean，将 id 为 `dataSource` 的组件通过属性 `dataSource` （即调用 `setDataSource()` 方法）注入到另外两个组件中。

在 Spring 的 IoC 容器中，我们将这样的组件成为 Bean。配置一个组件就是配置一个 Bean。

最后我们给出 IoC 和 IoC 容器较为书面、精确的定义：

IoC，即控制反转，指的是使对象仅通过**构造器参数/工厂方法参数/对象实例上设置的属性**定义它们的依赖。然后，容器在创建 bean 时注入那些依赖项。

IoC 容器，则是为组件运行提供的一个环境，以及一些底层服务。这些服务包括组件的生命周期管理、配置组装、AOP 支持等。

## 什么是 Spring Bean

我此次分析的模块为 Spring Bean，其组件位于 Spring 的 `org.springframework.beans` 包下。这个包下的所有类主要解决了三件事：Bean 的定义、Bean 的创建以及对 Bean 的解析。

对于 Spring Bean 内类的关系、每个类的作用，应当在这里简单分析一下，因为时间关系先跳过。这周内会作补充。

## 如何使用 Spring Bean

下面的实例同样来自廖雪峰老师的 Java 教程。我们实现用于用户注册/登录的服务 `Userservice` 和用于发送邮件通知的服务 `MailService`。由于这部分代码确实显示了 Spring Bean 的使用方法，我在这里摘录部分，并作简单解释。

```java
// User Service
public class UserService {
    private MailService mailService;
		
  	// Dependency injection
    public void setMailService(MailService mailService) {
        this.mailService = mailService;
    }

    private List<User> users = new ArrayList<>(List.of( // users:
            new User(1, "bob@example.com", "password", "Bob"), // bob
            new User(2, "alice@example.com", "password", "Alice"), // alice
            new User(3, "tom@example.com", "password", "Tom"))); // tom

    public User login(String email, String password) {
        for (User user : users) {
            if (user.getEmail().equalsIgnoreCase(email) && user.getPassword().equals(password)) {
                mailService.sendLoginMail(user);
                return user;
            }
        }
        throw new RuntimeException("login failed.");
    }

    public User getUser(long id) {
        return this.users.stream().filter(user -> user.getId() == id).findFirst().orElseThrow();
    }

    public User register(String email, String password, String name) {
        users.forEach((user) -> {
            if (user.getEmail().equalsIgnoreCase(email)) {
                throw new RuntimeException("email exist.");
            }
        });
        User user = new User(users.stream().mapToLong(u -> u.getId()).max().getAsLong() + 1, email, password, name);
        users.add(user);
        mailService.sendRegistrationMail(user);
        return user;
    }
}

// Mail Service
public class MailService {
    private ZoneId zoneId = ZoneId.systemDefault();

    public void setZoneId(ZoneId zoneId) {
        this.zoneId = zoneId;
    }

    public String getTime() {
        return ZonedDateTime.now(this.zoneId).format(DateTimeFormatter.ISO_ZONED_DATE_TIME);
    }

    public void sendLoginMail(User user) {
        System.err.println(String.format("Hi, %s! You are logged in at %s", user.getName(), getTime()));
    }

    public void sendRegistrationMail(User user) {
        System.err.println(String.format("Welcome, %s!", user.getName()));

    }
}
```

然后我们需要编写 XML 配置文件，告诉 Spring 的 IoC 容器如何创建、组装 Bean：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

  	// 1st Bean
    <bean id="userService" class="com.itranswarp.learnjava.service.UserService">
      	// Bean of mailService is injected here
        <property name="mailService" ref="mailService" />
    </bean>

  	// 2nd Bean
    <bean id="mailService" class="com.itranswarp.learnjava.service.MailService" />
</beans>
```

最后在 `main()` 方法中，创建一个 IoC 容器实例、加载配置文件，让 Spring 容器为我们创建装配好配置文件中指定的所有 Bean，就可以从容器中取出 Bean 并使用它：

```java
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

运行该样例，得到输出：

```bash
Hi, Bob! You are logged in at 2021-10-31T11:12:15.029302+08:00[Asia/Shanghai]
Bob
```

上面的 `ApplicationContext` 也就是 Spring IoC 容器的一种。它有很多实现类，而我们选择的是 `ClassPathXmlApplicationContext`，即它会自动从 classpath 中查找指定的 XML 配置文件。通常，我们根据 Bean 的类型使用 `getBean()` 方法获取 Bean 的引用。

Spring 也提供了另一种容器 `BeanFactory`，它的实现是按需创建，即第一次获取 Bean 时才创建这个 Bean，而 `ApplicationContext` 会一次性创建所有 Bean。事实上，`BeanFactory` 提供了管理 Bean 的配置框架和基本功能， `ApplicationContext` 则是 `BeanFactory` 的超集和子接口，添加了 AOP 功能的集成、用于国际化的消息资源处理等功能。

![ApplicationContext 继承图](https://josephqiu-1305443579.cos.ap-beijing.myqcloud.com/uPic/lwar6V.jpg)

在这一节中，我们通过实例接触了 Bean 和 IoC 容器的使用方法。知道了怎么用，接下来就需要弄懂 Spring Bean 是怎么实现这些功能——创建、实例化、依赖注入、生命周期管理的。我们会通过源码阅读来完成。也许这学期的三次作业不足以将这些过程的机理都分析清楚，那么后续可能会选择其中的一部分作尽可能详尽清楚的剖析。

## 参考资料

1. Spring Framework 官方文档：https://docs.spring.io/spring-framework/docs/current/reference/html/

2. 廖雪峰的官方网站 - Java 教程 - Spring 开发 - IoC 原理：https://www.liaoxuefeng.com/wiki/1252599548343744/1282381977747489

