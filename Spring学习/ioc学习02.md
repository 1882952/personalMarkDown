# 一：注解annotation

annotation-based configuration 提供了 XML 设置的替代方法，它依赖于字节码元数据来连接组件而不是 angle-bracket 声明。  

> 注释注入在 XML 注入之前执行。因此，XML configuration 会覆盖通过这两种方法连接的 properties 的 annotations。

## 1.注解类别

### （1）@requird

`@Required` annotation 适用于 bean property setter 方法，表示该属性必须注入填充，否则会抛出异常。

### （2）@Autowired

自动注入，最常用的注解，这个注解可以用于构造函数，setter方法，应用于字段，甚至可以将其与构造函数混合使用。

```java
public class MovieRecommender {

    private final CustomerPreferenceDao customerPreferenceDao;

    @Autowired
    private MovieCatalog movieCatalog;

    @Autowired
    public MovieRecommender(CustomerPreferenceDao customerPreferenceDao) {
        this.customerPreferenceDao = customerPreferenceDao;
    }

    // ...
}
```

该注解使用的前提是容器需要知道所用之处的具体的类型，但是自动注入只依赖对应的类型，如果一个接口有多个实现类，那么自动注入该接口就会发生异常，这个时候就需要使用其他派生的注解或者在autowired中写上相应的id。

对于通过 classpath 扫描找到的 XML-defined beans 或 component classes，容器通常预先知道具体类型。但是，对于`@Bean`工厂方法，您需要确保声明的 return 类型具有足够的表现力。对于实现多个接口的组件或可能由其 implementation 类型引用的组件，请考虑在工厂方法中声明最具体的 return 类型(至少与引用 bean 的注入点所需的具体类型相同)。

从 Spring Framework 5.0 开始，您还可以使用`@Nullable`  annotation(任何包中的任何类型 - 对于 example，`javax.annotation.Nullable`来自 JSR-305)：

```java
public class SimpleMovieLister {
    @Autowired
    public void setMovieFinder(@Nullable MovieFinder movieFinder) {
        ...
    }
}
```

还可以将`@Autowired`用于 well-known 可解析依赖项的接口：`BeanFactory`，`ApplicationContext`，`Environment`，`ResourceLoader`，`ApplicationEventPublisher`和`MessageSource`。这些接口及其扩展接口(如`ConfigurableApplicationContext`或`ResourcePatternResolver`)将自动解析，无需特殊设置。

> 说明除了使用aware接口，还可以使用autowired实现获取ioc容器内的bean对象。

### （3）使用 @Primary 自动装配

由于按类型自动装配可能会导致多个候选项，因此通常需要对选择 process 进行更多控制。实现此目的的一种方法是使用 Spring 的`@Primary` annotation。 `@Primary`表示当多个 beans 是自动连接到 single-valued 依赖项的候选者时，应该优先选择特定的 bean。如果候选者中只存在一个主 bean，则它将成为自动连接的 value。

```java
@Configuration
public class MovieConfiguration {

    @Bean
    @Primary //默认为主的自动装配

    public MovieCatalog firstMovieCatalog() { ... }
    @Bean

    public MovieCatalog secondMovieCatalog() { ... }

    // ...
}
```

### （4）使用 @Qualifier

当可以确定一个主要候选者时，`@Primary`是一种有效的方式，可以通过多个实例使用类型自动装配。当您需要更多控制选择 process 时，可以使用 Spring 的`@Qualifier` annotation。您可以将限定符值与特定的 arguments 相关联，缩小类型匹配集，以便为每个参数选择特定的 bean。 

```java
    @Autowired
    @Qualifier("main")
    private MovieCatalog movieCatalog; //按照类中注入的基础上，再按照对应的名称（value值）注入
```

@Qualifier适合于多控制的选择，可以使用相同的限定符 value“xxx”定义多个`MovieCatalog` beans，所有这些都被注入`Set<MovieCatalog>`注释`@Qualifier("xxx")`。

> 注意：Qualifier注解在给类的成员注入时不能单独使用，必须配合@autowired，如上所示， 但是在方法参数注入时可以单独使用。

注解具有层级性，所以可以自定义注解。

（2）`@Resource`，它可以通过其唯一的 name 获取代理回到当前 bean。`@Autowired`适用于字段，构造函数和 multi-argument 方法，允许通过参数 level 中的限定符注释缩小范围。相比之下，`@Resource`仅支持字段，bean property setter 方法只支持一个参数。因此，如果注射目标是构造函数或 multi-argument 方法，则应该使用限定符。

### （5）使用泛型的autowired

除了`@Qualifier` annotation 之外，您还可以使用 Java 泛型类型作为隐式的限定形式。

通用限定符也适用于自动装配 lists，`Map`实例和数组。以下 example 自动装配通用`List`：

```java
// Inject all Store beans as long as they have an <Integer> generic
// Store<String> beans will not appear in this list
@Autowired
private List<Store<Integer>> s;
```

### （6）使用 CustomAutowireConfigurer

[CustomAutowireConfigurer 上](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/factory/annotation/CustomAutowireConfigurer.html)是一个`BeanFactoryPostProcessor`，它允许您注册自己的自定义限定符注释类型，即使它们没有使用 Spring 的`@Qualifier` annotation 注释。

### （7）注射@Resource

Spring 还支持使用字段上的 JSR-250  `@Resource`  annotation 或 bean property setter 方法进行注入。这是 Java EE 5 和 6 中的 common pattern(例如，在 JSF 1.2 managed beans 或 JAX-WS 2.0 endpoints 中)。 Spring 也为 Spring-managed objects 支持此 pattern。

`@Resource`采用 name 属性。默认情况下，Spring 将 value 解释为要注入的 bean name。换句话说，它遵循 by-name 语义。

在`@Resource`用法的唯一情况下，没有指定明确的 name，并且类似于`@Autowired`，`@Resource`找到主要类型 match 而不是特定的名为 bean，并解析众所周知的可解析依赖项：`BeanFactory`，`ApplicationContext`，`ResourceLoader`，`ApplicationEventPublisher`和`MessageSource`接口。

### （8） 使用 @PostConstruct 和 @PreDestroy

就是xml中bean标签中init与destory的注解形式。

# 二：classpath扫描与托管组件

可以使用 annotations(用于 example，`@Component`)，AspectJ 类型表达式或您自己的自定义过滤条件来选择哪些 classes 具有向容器注册的 bean 定义。

> 从 Spring 3.0 开始，Spring JavaConfig 项目提供的许多 features 都是核心 Spring Framework 的一部分。这允许您使用 Java 定义 beans 而不是使用传统的 XML files。有关如何使用这些新 features 的示例，请查看`@Configuration`，`@Bean`，`@Import`和`@DependsOn` 注释。

### （1）@component

Spring 提供了进一步的构造型注释：`@Component`，`@Service`和`@Controller`。 `@Component`是任何 Spring-managed component 的通用构造型。 `@Repository`，`@Service`和`@Controller`是`@Component`的特化，用于更具体的用例(分别在持久性，服务和表示层中)。

### （2）使用 Meta-annotations 和 Composed Annotations

即注解的派生性与层次性，可以组合 meta-annotations 来创建“撰写注释”。例如，Spring MVC 的`@RestController` annotation 由`@Controller`和`@ResponseBody`组成。

也可以利用派生性与层次性自定义注解。

### （3）自动检测class并注册bean定义

Spring 可以自动检测原型 classes 并使用`ApplicationContext`注册相应的`BeanDefinition`实例。

```java
@Service
public class SimpleMovieLister {
    private MovieFinder movieFinder;
    @Autowired
    public SimpleMovieLister(MovieFinder movieFinder) {
        this.movieFinder = movieFinder;
    }
}
```

```java
@Repository
public class JpaMovieFinder implements MovieFinder {
    // implementation elided for clarity
}
```

要自动检测这些 classes 并注册相应的 beans，您需要将`@ComponentScan`添加到`@Configuration` class，其中`basePackages`属性是两个 classes 的 common parent 包。

要自动检测这些 classes 并注册相应的 beans，您需要将`@ComponentScan`添加到`@Configuration` class，其中`basePackages`属性是两个 classes 的 common parent 包。 (或者，您可以指定包含每个 class.)的 parent 包的逗号或分号或 space-separated 列表

```java
@Configuration
@ComponentScan(basePackages = "org.example")
public class AppConfig  {
    ...
}
```

> 这是利用注解的自动检测，替代了XML的方式
> 
> ```xml
>   <context:component-scan base-package="org.example"/>
> ```

此外，使用 component-scan 元素时，隐式包含`AutowiredAnnotationBeanPostProcessor`和`CommonAnnotationBeanPostProcessor`。这意味着这两个组件是自动检测并连接在一起的 - 所有这些都没有在 XML 中提供任何 bean configuration 元数据。

### （4）使用过滤器自定义扫描

默认情况下，使用`@Component`，`@Repository`，`@Service`，`@Controller`注释的 classes 或使用`@Component`注释的自定义注释是唯一检测到的候选组件。但是，您可以通过应用自定义筛选器来修改和扩展此行为。

    将它们添加为`@ComponentScan` annotation 的`includeFilters`或`excludeFilters`参数(或`component-scan`元素的`include-filter`或`exclude-filter` child 元素)。

| 过滤器类型          | Example 表达式                  | 描述                                                                |
| -------------- | ---------------------------- | ----------------------------------------------------------------- |
| annotation(默认) | `org.example.SomeAnnotation` | 目标组件中 level 类型的注释。                                                |
| 分配             | `org.example.SomeClass`      | 目标组件可分配给(扩展或实现)的 class(或接口)。                                      |
| AspectJ        | `org.example..*Service+`     | 要由目标组件匹配的 AspectJ 类型表达式。                                          |
| 正则表达式          | `org\.example\.Default.*`    | 要由目标组件 class 名称匹配的正则表达式。                                          |
| 习惯             | `org.example.MyTypeFilter`   | `org.springframework.core.type .TypeFilter`接口的自定义 implementation。 |

```java
//Demo
@Configuration
@ComponentScan(basePackages = "org.example",
        includeFilters = @Filter(type = FilterType.REGEX, pattern = ".*Stub.*Repository"),
        excludeFilters = @Filter(Repository.class))
public class AppConfig {
    ...
}
```

### （5）在组件中定义Bean元数据

Spring 组件还可以将 bean 定义元数据提供给容器。您可以使用用于在`@Configuration` annotated classes 中定义 bean 元数据的相同`@Bean` annotation 来执行此操作。

```java
@Component
public class FactoryMethodComponent {

    @Bean
    @Qualifier("public")
    public TestBean publicInstance() {
        return new TestBean("publicInstance");
    }

    public void doWork() {
        // Component method implementation omitted
    }
}
```

### （6）命名自动检测的组件

当 component 作为 scan process 的一部分自动检测时，其 bean name 由该扫描程序已知的`BeanNameGenerator`策略生成。默认情况下，任何包含 name `value`的 Spring 构造型 annotation(`@Component`，`@Repository`，`@Service`和`@Controller`)都会将 name 提供给相应的 bean 定义。

### （7）为自动检测组件提供范围

与 Spring-managed 组件一样，自动检测组件的默认范围和最常见范围是`singleton`。但是，有时您需要一个可由`@Scope`  annotation 指定的不同范围。您可以在 annotation 中提供范围的 name，如下面的 example 所示：

```java
@Scope("prototype")
@Repository
public class MovieFinderImpl implements MovieFinder {
    // ...
}
```

# 三：java代码中使用注解配置spring

### （1）基本概念@bean与@configuration

Spring 新的 Java-configuration 支持中的中心 artifacts 是`@Configuration`  -annotated classes 和`@Bean`  -annotated 方法。

`@Bean`  annotation 用于指示方法实例化，配置和初始化由 Spring IoC 容器管理的新 object。对于那些熟悉 Spring 的`<beans/>`  XML configuration 的人来说，`@Bean`  annotation 与`<bean/>`元素扮演的角色相同。您可以将`@Bean`  -annotated 方法与任何 Spring  `@Component`一起使用。但是，它们最常用于`@Configuration`  beans。

使用`@Configuration`注释 class 表示其主要目的是作为 bean 定义的源。此外，`@Configuration`  classes 允许通过调用同一 class 中的其他`@Bean`方法来定义 inter-bean 依赖项。最简单的`@Configuration`  class 如下：

```java
@Configuration
public class AppConfig {

    @Bean
    public MyService myService() {
        return new MyServiceImpl();
    }
}
```

### （2） 使用 AnnotationConfigApplicationContext 实例化 Spring 容器

当`@Configuration`  classes 作为输入提供时，`@Configuration`  class 本身被注册为 bean 定义，class 中所有声明的`@Bean`方法也被注册为 bean 定义。

当提供`@Component`和 JSR-330 classes 时，它们被注册为 bean 定义，并且假设在必要时在那些 classes 中使用诸如`@Autowired`或`@Inject`的 DI 元数据。

#### 简单构造

与实例化`ClassPathXmlApplicationContext`时 Spring XML files 用作输入的方式大致相同，在实例化`AnnotationConfigApplicationContext`时可以使用`@Configuration`  classes 作为输入。这允许完全 XML-free 使用 Spring 容器，如下面的示例所示：

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
```

如前所述，`AnnotationConfigApplicationContext`不仅限于使用`@Configuration`  classes。任何`@Component`或 JSR-330 带注释的 class 都可以作为输入提供给构造函数，如下面的 example 所示：

```java
public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(MyServiceImpl.class, Dependency1.class, Dependency2.class);
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
```

##### 使用 register(Class <?>以编程方式构建容器...)

您可以使用 no-arg 构造函数实例化`AnnotationConfigApplicationContext`，然后使用`register()`方法对其进行配置。当以编程方式 building  `AnnotationConfigApplicationContext`时，此方法特别有用。以下 example 显示了如何执行此操作：

```java
public static void main(String[] args) {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext();
    ctx.register(AppConfig.class, OtherConfig.class);
    ctx.register(AdditionalConfig.class);
    ctx.refresh();
    MyService myService = ctx.getBean(MyService.class);
    myService.doStuff();
}
```

##### 使用 scan(String 启用 Component 扫描...)

要启用 component 扫描，您可以按如下方式注释`@Configuration`  class：

```java
@Configuration
@ComponentScan(basePackages = "com.acme") (1)
public class AppConfig  {
    ...
}
```

| **1** | 此 annotation 启用 component 扫描。 |

### （3）使用@Bean

@Bean是方法级别的注解，是 XML `<bean/>`元素的直接模拟。

annotation 支持`<bean/>`提供的一些属性，例如：*  [init-method](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-lifecycle-initializingbean)  *  [destroy-method](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-lifecycle-disposablebean)  *  [自动装配](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-autowire)  *  `name`。

您可以在`@Configuration`  -annotated 或`@Component`  -annotated class 中使用`@Bean`  annotation。

```java
@Configuration
public class AppConfig {

    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl();
    }
}
```

但是这会将高级类型预测的可见性限制为指定的接口类型(`TransferService`)。然后，只有容器已知的完整类型(`TransferServiceImpl`)一次，受影响的 singleton bean 已被实例化。 Non-lazy singleton beans 根据其声明 order 进行实例化，因此您可能会看到不同的类型匹配结果，具体取决于另一个 component 尝试按 non-declared 类型匹配的情况(例如`@Autowired TransferServiceImpl`，只有在`transferService` bean 实例化后才会解析)。

#### bean注解接收生命回调

```java
public class BeanOne {

    public void init() {
        // initialization logic
    }
}
public class BeanTwo {

    public void cleanup() {
        // destruction logic
    }
}

@Configuration
public class AppConfig {

    @Bean(initMethod = "init")
    public BeanOne beanOne() {
        return new BeanOne();
    }

    @Bean(destroyMethod = "cleanup")
    public BeanTwo beanTwo() {
        return new BeanTwo();
    }
}
```

当然，在方法体中调用相关init方法等也能完成init操作，也就可以不用注解的属性值。

### （4）使用@configuration

`@Configuration`是 class-level annotation，表示 object 是 bean 定义的来源。 `@Configuration` classes 通过 public `@Bean` annotated 方法声明 beans。 Calls 到`@Configuration` classes 上的`@Bean`方法也可用于定义 inter-bean 依赖项。

```java
@Configuration
public class AppConfig {

    @Bean
    public ClientService clientService1() {
        ClientServiceImpl clientService = new ClientServiceImpl();
        clientService.setClientDao(clientDao());
        return clientService;
    }

    @Bean
    public ClientService clientService2() {
        ClientServiceImpl clientService = new ClientServiceImpl();
        clientService.setClientDao(clientDao());
        return clientService;
    }

    @Bean
    public ClientDao clientDao() {
        return new ClientDaoImpl();
    }
}
```

`clientDao()`在`clientService1()`中被调用一次，在`clientService2()`中被调用一次。由于此方法创建`ClientDaoImpl`的新实例并将其返回，因此通常需要两个实例(每个服务一个)。这肯定会有问题：在 Spring 中，实例化的 beans 默认具有`singleton`范围。这就是魔术的用武之地：所有`@Configuration` class 都在 startup-time 和`CGLIB`进行了子类化。在子类中，child 方法首先检查容器是否有任何缓存(作用域)beans，然后 calls parent 方法并创建一个新实例。

### （5）撰写java-based的配置

Spring 的 Java-based configuration feature 允许您撰写 annotations，这可以降低 configuration 的复杂性。

#### 使用@Import

就像在 Spring XML files 中使用`<import/>`元素来帮助模块化配置一样，`@Import` annotation 允许从另一个 configuration class 中加载`@Bean`定义。

```java
@Configuration
public class ConfigA {
    @Bean
    public A a() {
        return new A();
    }
}
@Configuration
@Import(ConfigA.class)
public class ConfigB {

    @Bean
    public B b() {
        return new B();
    }
}
```

```java
@Configuration
public class ServiceConfig {

    @Bean
    public TransferService transferService(AccountRepository accountRepository) {
        return new TransferServiceImpl(accountRepository);
    }
}

@Configuration
public class RepositoryConfig {

    @Bean
    public AccountRepository accountRepository(DataSource dataSource) {
        return new JdbcAccountRepository(dataSource);
    }
}

@Configuration
@Import({ServiceConfig.class, RepositoryConfig.class})
public class SystemTestConfig {

    @Bean
    public DataSource dataSource() {
        // return new DataSource
    }
}

public static void main(String[] args) {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(SystemTestConfig.class);
    // everything wires up across configuration classes...
    TransferService transferService = ctx.getBean(TransferService.class);
    transferService.transfer(100.00, "A123", "C456");
}
//上述的代码除了用Import注解外，还可以使用@autowired，但是不能用于静态bean的自动注入。
```

> 要特别注意通过`@Bean`的`BeanPostProcessor`和`BeanFactoryPostProcessor`定义。那些应该通常被声明为`static @Bean`方法，而不是触发它们包含 configuration class 的实例化。否则，`@Autowired`和`@Value`在 configuration class 本身上不起作用，因为它太早创建为 bean 实例。

#### @Configuration Class-centric 将 XML 与 @ImportResource 一起使用

在 application 中`@Configuration` classes 是配置容器的主要机制，仍然可能需要使用至少一些 XML。在这些场景中，您可以使用`@ImportResource`并根据需要定义尽可能多的 XML。这样做可以实现配置容器的“Java-centric”方法，并将 XML 保持在最低限度。

```java
@Configuration
@ImportResource("classpath:/com/acme/properties-config.xml")
public class AppConfig {

    @Value("${jdbc.url}")
    private String url;

    @Value("${jdbc.username}")
    private String username;

    @Value("${jdbc.password}")
    private String password;

    @Bean
    public DataSource dataSource() {
        return new DriverManagerDataSource(url, username, password);
    }
}
```

```xml
properties-config.xml
<beans>
    <context:property-placeholder location="classpath:/com/acme/jdbc.properties"/>
</beans>
```

```java
jdbc.properties
jdbc.url=jdbc:hsqldb:hsql://localhost/xdb
jdbc.username=sa
jdbc.password=
```

# 四：环境抽象

[环境](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/core/env/Environment.html)接口是集成在容器中的抽象，它为 application 环境的两个 key 方面建模：[profiles](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-definition-profiles)和[properties](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-property-source-abstraction)。

profile 是 bean 定义的命名逻辑 group，仅当给定的 profile 为 active 时才向容器注册。 Beans 可以分配给 profile，无论是用 XML 定义还是用 annotations 定义。与 profiles 相关的`Environment`  object 的作用是确定哪些 profiles(如果有)当前是 active，以及哪些 profiles(如果有)默认情况下应该 active。

Properties 在几乎所有 applications 中都发挥着重要作用，可能来自各种来源：properties files，JVM 系统 properties，系统环境变量，JNDI，servlet context 参数，ad-hoc  `Properties`  objects，`Map`objects 等等。与 properties 相关的`Environment`  object 的作用是为用户提供方便的服务接口，用于配置 property 源和从中解析 properties。

> 简单来说，profile是定义区分生产与开发的环境的相关配置。
> 
> Properties就是保存一些静态的数据，比如保存jdbc中数据库连接的相关信息。

### 使用@PropertySource

[@PropertySource](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/context/annotation/PropertySource.html) annotation 提供了一种方便的声明式机制，用于将`PropertySource`添加到 Spring 的`Environment`。如下例子，导入数据库连接信息

```java
@Configuration
@PropertySource("classpath:/com/myco/app.properties")
public class AppConfig {

    @Autowired
    Environment env;

    @Bean
    public TestBean testBean() {
        TestBean testBean = new TestBean();
        testBean.setName(env.getProperty("testbean.name"));
        return testBean;
    }
}
```

# 五：ApplicationContext的附加功能

`org.springframework.beans.factory`包提供了管理和操作 beans 的基本功能，包括以编程方式。除了扩展其他接口以提供更多 application framework-oriented 样式的附加功能外，`org.springframework.context`包还添加了扩展`BeanFactory`接口的[ApplicationContext](https://docs.spring.io/spring-framework/docs/5.1.3.RELEASE/javadoc-api/org/springframework/context/ApplicationContext.html)接口。许多人以完全声明的方式使用`ApplicationContext`，甚至不以编程方式创建它，而是依靠支持 class(如`ContextLoader`)来自动实例化`ApplicationContext`作为 Java EE web application 的正常启动 process 的一部分。

要以更多 framework-oriented 样式增强`BeanFactory`功能，context 包还提供以下功能：

- 通过`MessageSource`接口访问 i18n-style 中的消息。

- 通过`ResourceLoader`接口访问 URL 和 files 等资源。

- Event 发布，即通过使用`ApplicationEventPublisher`接口实现`ApplicationListener`接口的 beans。

- Loading 多个(分层)上下文，让每个上下文通过`HierarchicalBeanFactory`接口聚焦在一个特定层上，例如 application 的 web 层。

### 国际化

`ApplicationContext`接口扩展了一个名为`MessageSource`的接口，因此提供了国际化(“i18n”)功能。 Spring 还提供`HierarchicalMessageSource`接口，可以分层次地解析消息。这些接口共同提供了 Spring 效果消息解析的基础。这些接口上定义的方法包括：

- `String getMessage(String code, Object[] args, String default, Locale loc)`：用于从`MessageSource`检索消息的基本方法。如果未找到指定 locale 的消息，则使用默认消息。传入的任何 arguments 都使用标准 library 提供的`MessageFormat`功能成为替换值。

- `String getMessage(String code, Object[] args, Locale loc)`：与前一个方法基本相同，但有一点不同：无法指定默认消息。如果找不到该消息，则抛出`NoSuchMessageException`。

- `String getMessage(MessageSourceResolvable resolvable, Locale locale)`：前面方法中使用的所有 properties 也包装在一个名为`MessageSourceResolvable`的 class 中，您可以使用此方法。

利用MessageSource`的接口就可以读取外部的资源包，实现国际化。

### 标准和自定义事件

通过`ApplicationEvent` class 和`ApplicationListener`接口提供`ApplicationContext`中的 Event 处理。如果实现`ApplicationListener`接口的 bean 被部署到 context 中，那么每 time被发布到`ApplicationContext`，bean 被通知。从本质上讲，这是标准的 Observer 设计 pattern。

| 事件                      | 说明                                                                                                                                                                                                                                                                                                                                                         |
| ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ContextRefreshedEvent` | 初始化或刷新`ApplicationContext`时发布(对于 example，通过`ConfigurableApplicationContext`接口使用`refresh()`方法)。这里，“初始化”意味着所有 beans 都被加载，post-processor beans 被检测并激活，单例是 pre-instantiated，`ApplicationContext`object 可以使用了。如果 context 尚未关闭_long，则可以多次触发刷新，前提是所选的`ApplicationContext`实际上支持这种“热”刷新。对于 example，`XmlWebApplicationContext`支持热刷新，但`GenericApplicationContext`不支持。 |
| `ContextStartedEvent`   | 通过在`ConfigurableApplicationContext`接口上使用`start()`方法启动`ApplicationContext`时发布。这里，“已启动”意味着所有`Lifecycle`  beans 都会收到明确的启动信号。通常，此信号用于在显式停止后重新启动 beans，但它也可用于启动尚未为自动启动配置的组件(对于 example，尚未在初始化时启动的组件)。                                                                                                                                                           |
| `ContextStoppedEvent`   | 通过在`ConfigurableApplicationContext`接口上使用`stop()`方法停止`ApplicationContext`时发布。这里，“停止”意味着所有`Lifecycle`  beans 都会收到明确的停止信号。可以通过`start()`调用重新启动已停止的 context。                                                                                                                                                                                                    |
| `ContextClosedEvent`    | 通过在`ConfigurableApplicationContext`接口上使用`close()`方法关闭`ApplicationContext`时发布。在这里，“关闭”意味着所有 singleton beans 都被销毁。封闭的 context 到达其生命的终点。它无法刷新或重新启动。                                                                                                                                                                                                           |
| `RequestHandledEvent`   | web-specific event 告诉所有 beans 已经为 HTTP 请求提供服务。请求完成后发布此 event。此 event 仅适用于使用 Spring 的`DispatcherServlet`的 web applications。                                                                                                                                                                                                                                 |

### （3）方便地访问 Low-level 资源

为了最佳地使用和理解 application 上下文，你应该熟悉 Spring 的`Resource`抽象，如[资源](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#resources)中所述。

application context 是`ResourceLoader`，可用于加载`Resource`  objects。  `Resource`本质上是 JDK  `java.net.URL`  class 的 feature 富 version。事实上，的_implement 包装了一个`java.net.URL`的实例，如果合适的话。  `Resource`可以透明的方式从几乎任何位置获取 low-level 资源，包括 classpath，文件系统位置，任何可用标准 URL 描述的位置，以及其他一些变体。如果资源位置 string 是一个没有任何特殊前缀的简单路径，那么这些资源来自特定且适合于实际的 application context 类型。

### （4）方便的web applications

# 六：BeanFactory

`BeanFactory`  API 为 Spring 的 IoC 功能提供了基础。它的特定 contracts 主要用于与 Spring 和相关 third-party 框架的其他部分进行整合，其`DefaultListableBeanFactory`  implementation 是 higher-level  `GenericApplicationContext`容器中的 key 委托。

`BeanFactory`和相关接口(例如`BeanFactoryAware`，`InitializingBean`，`DisposableBean`)是其他 framework 组件的重要集成点。通过不需要任何注释或甚至反射，它们允许容器与其组件之间的非常有效的交互。 Application-level beans 可以使用相同的回调接口，但通常更喜欢声明性依赖注入，通过 annotations 或通过编程 configuration。

请注意，核心`BeanFactory`  API level 及其`DefaultListableBeanFactory`  implementation 不会对 configuration 格式或要使用的任何 component annotations 进行假设。所有这些风格都通过 extensions(例如`XmlBeanDefinitionReader`和`AutowiredAnnotationBeanPostProcessor`)进入，并作为核心元数据表示在共享`BeanDefinition`  objects 上运行。这就是 Spring 容器如此灵活和可扩展的本质。


