# 一：简介

## 框架结构

![XVe1noRCMt](images\spring-overview.png.pagespeed.ce.XVe1noRCMt.png)

（1）核心的容器由spring-core， spring-beans，spring-context，spring-context-support，和spring-expression （Spring表达式语言）。就是图中的core Container中的结构。

- core与beans提供了框架的基本功能，包括IOC与依赖注入功能。 其核心是利用了BeanFactory类，利用工厂模式实现了bean的单例管理。

- context（上下文）模块是建立在core与beans的基础上的，它是提供了一个框架式的对象访问方式（比如可以利用ClassPathXmlApplicationContext对象去读取配置的xml文件中定义的相应bean对象）。   它的上下文模块从Beans模块中继承功能，并添加支持国际化（使用，例如，资源集合），事件传播，资源负载，并且透明创建上下文，例如，Servlet容器。Context模块还支持Java EE的功能，如EJB，JMX和基本的远程处理。ApplicationContext接口是Context模块的焦点。 spring-context-support支持整合普通第三方库到Spring应用程序上下文，特别是用于高速缓存（ehcache，JCache）和调度（CommonJ，Quartz）的支持。

- spring-expression模块即提供了EL表达式，用法是可以代替jsp直接在html中操作元素对象，但是在如今的前后端分离中，该模块可以被忽略。

（2）然后是上面的Aop、Instrumentation、messaging

- aop：面向切面编程，可以基于动态代理+反射实现，也可以通过cglib以创建字节码的方式实现。  Aspectj是aop的一种实现方式（原本这种技术是静态的，spring重写了，使其能在运行时动态织入），目前已成主流。

- spring-instrument模块提供了类植入(instrumentation)支持和类加载器的实现,可以应用在特定的应用服务器中。 作用是进行服务器整合，该spring-instrument-tomcat 模块包含了支持Tomcat的植入代理。

- Spring框架4包括spring-messaging(消息传递模块)，其中包含来自Spring Integration的项目，例如，Message，MessageChannel，MessageHandler，和其他用来传输消息的基础应用。该模块还包括一组用于将消息映射到方法的注释(annotations)，类似于基于Spring MVC注解的编程模型，简单来说就是利用注解向bean对象set数据，就是利用了messaging模块。

（3）：最上层的Data访问\集成层

数据访问/集成层由JDBC，ORM，OXM，JMS和事务模块组成。

面向连接数据库的模块，提供了比如jdbc连接池，整合mybatis等。

- spring-jdbc模块提供了一个[JDBC](http://docs.spring.io/spring/docs/5.0.0.M5/spring-framework-reference/html/jdbc.html#jdbc-introduction)  –抽象层，消除了需要的繁琐的JDBC编码和数据库厂商特有的错误代码解析。

- spring-tx模块支持用于实现特殊接口和所有POJO（普通Java对象）的类的[编程和声明式事务](http://docs.spring.io/spring/docs/5.0.0.M5/spring-framework-reference/html/transaction.html)  管理。

- spring-orm模块为流行的对象关系映射([object-relational mapping](http://docs.spring.io/spring/docs/5.0.0.M5/spring-framework-reference/html/orm.html#orm-introduction)  )API提供集成层，包括JPA和Hibernate。使用spring-orm模块，您可以将这些O / R映射框架与Spring提供的所有其他功能结合使用，例如前面提到的简单声明性事务管理功能。

- spring-oxm模块提供了一个支持[对象/ XML映射](http://docs.spring.io/spring/docs/5.0.0.M5/spring-framework-reference/html/oxm.html)实现的抽象层，如JAXB，Castor，JiBX和XStream。

- spring-jms模块([Java Messaging Service](http://docs.spring.io/spring/docs/5.0.0.M5/spring-framework-reference/html/jms.html)) 包含用于生产和消费消息的功能。自Spring Framework 4.1以来，它提供了与 spring-messaging模块的集成。

（4）：web层

Web层由spring-web，spring-webmvc和spring-websocket 模块组成。

- spring-web提供了基本的面向web的集成功能，比如多文件上传，以及初始化一个使用了Servlet侦听器和面向Web的应用程序上下文的IoC容器。它还包含一个HTTP客户端和Spring的远程支持的Web相关部分。

- webmvc，即springMVC，包含用于Web应用程序的Spring的模型-视图-控制器([*MVC*](http://docs.spring.io/spring/docs/5.0.0.M5/spring-framework-reference/html/mvc.html#mvc-introduction))和REST Web Services实现。

（5）最底层的test层

测试模块，支持junit进行单元测试和集成测试，它提供了Spring ApplicationContexts的一致[加载](http://docs.spring.io/spring/docs/5.0.0.M5/spring-framework-reference/html/integration-testing.html#testcontext-ctx-management)和这些上下文的[缓存](http://docs.spring.io/spring/docs/5.0.0.M5/spring-framework-reference/html/integration-testing.html#testcontext-ctx-management-caching)。它还提供可用于独立测试代码的[模仿(mock)对象](http://docs.spring.io/spring/docs/5.0.0.M5/spring-framework-reference/html/unit-testing.html#mock-objects)。

# 二：IOC容器

## 1.经典术语

依赖注入与控制反转ioc

IOC又叫依赖注入（DI），它描述了对象的定义和依赖的一个过程，也就是说，依赖的对象通过构造参数、工厂方法参数或者属性注入，当对象实例化后依赖的对象才被创建，当创建bean后容器注入这些依赖对象。这个过程基本上是反向的，因此命名为控制反转（IoC），它通过直接使用构造类来控制实例化，或者定义它们之间的依赖关系，或者类似于服务定位模式的一种机制。  （这个过程简单类比于使用单例先创建对象，然后将其用map集合保存，通过对应的xml文件保存map中的key，通过读取xml文件中的key，就可以获得map中value保存的对应的对象）。

> `org.springframework.beans`和`org.springframework.context`是Spring框架中IoC容器的基础，[`BeanFactory`](http://docs.spring.io/spring-framework/docs/5.0.0.M5/javadoc-api/org/springframework/beans/factory/BeanFactory.html)接口提供一种高级的配置机制能够管理任何类型的对象。[`ApplicationContext`](http://ifeve.com/spring-ioc-1-2/href=%22http://docs.spring.io/spring-framework/docs/5.0.0.M5/javadoc-api/org/springframework/context/ApplicationContext.html)是`BeanFactory`的子接口。它能更容易集成Spring的AOP功能、消息资源处理（比如在国际化中使用）、事件发布和特定的上下文应用层比如在网站应用中的`WebApplicationContext。`
> 
> `BeanFactory`提供了配置框架和基本方法，`ApplicationContext`添加更多的企业特定的功能。`ApplicationContext`是`BeanFactory`的一个子接口.

在Spring中，IOC容器管理的对象称为beans，bean就是由Spring IoC容器实例化、组装和以其他方式管理的对象。此外bean只是你应用中许多对象中的一个。Beans以及他们之间的依赖关系是通过容器配置元数据反映出来。

## 2.容器概述：

### org.springframework.context.ApplicationContext

ApplicationContext接口代表了Spring的IOC容器，它负责实例化、配置、组装之前的beans。容器通过读取配置元数据获取对象的实例化、配置和组装的描述信息。它配置的0元数据用xml、Java注解或Java代码表示。它允许你表示组成你应用的对象以及这些对象之间丰富的内部依赖关系。

spring中提供了几个开箱即用的ApplicationContext接口的实现类，在独立应用程序中通常创建一个[`ClassPathXmlApplicationContext`](http://docs.spring.io/spring-framework/docs/5.0.0.M5/javadoc-api/org/springframework/context/support/ClassPathXmlApplicationContext.html)或[`FileSystemXmlApplicationContext`](http://docs.spring.io/spring-framework/docs/5.0.0.M5/javadoc-api/org/springframework/context/support/FileSystemXmlApplicationContext.html)实例对象。虽然XML是用于定义配置元数据的传统格式，你也可以指示容器使用Java注解或代码作为元数据格式，但要通过提供少量XML配置来声明启用对这些附加元数据格式的支持。springboot中统一支持注解的方式。

在大多数场景中，用户代码并不需要显示创建一个或多个springIOC实例（即new ApplicationContext对象），这在web模块中已经封装好了。

在开发中，你应用中所有的类都由元数据组装到一起 ，所以当`ApplicationContext`创建和实例化后，你就有了一个完全可配置和可执行的系统或应用。

![images\ioccontainer-magic](images\ioccontainer-magic.png)

上图表示的是，你定义的类，比如pojo类作为元数据，开发人员再交给IOC容器并告诉Spring容器怎样去实例化、配置和装备你应用中的对象，最后交给用户代码使用。

#### Configuration元数据

就是将用户定义的类配置为beans的过程，最早使用xml的方式配置，目前已被基于注解的java配置代理。

- [基于注解配置：](http://docs.spring.io/spring/docs/5.0.0.M5/spring-framework-reference/html/beans.html#beans-annotation-config)  在Spring2.5中有过介绍支持基于注解的配置元数据
- [基于Java配置：](http://docs.spring.io/spring/docs/5.0.0.M5/spring-framework-reference/html/beans.html#beans-java)  从Spring3.0开始，由Spring JavaConfig提供的许多功能已经成为Spring框架中的核心部分。这样你可以使用Java程序而不是XML文件定义外部应用程序中的bean类。使用这些新功能，可以查看`@Configuration`,`@Bean`,`@Import`和`@DependsOn`这些注解

Spring配置由必须容器管理的一个或通常多个定义好的bean组成。基于XML配置的元数据中，这些bean通过标签定义在顶级标签内部。在Java配置中通常在使用`@Configuration`注解的类中使用`@Bean`注解方法。

至于元数据怎么配置的，网上随便找个demo都有例子。

下面的容器加载元数据的内容都是基于Xml配置的，使用注解的java配置则封装了这个过程。

## 3.实例化容器

即实例化ApplicationContext接口的对应子类，然后传入需要加载的Configuration的元数据的文件路径。  

```java
ApplicationContext context = new ClassPathXmlApplicationContext("services.xml", "daos.xml");
```

上面就是典型的加载配置类中元数据的操作。

在原始的spring项目中，通常一个项目有多个configuration的配置文件，那么就需要一个整合，如下所示：

```java
<beans>
    <import resource="services.xml"/>
    <import resource="resources/messageSource.xml"/>
    <import resource="/resources/themeSource.xml"/>

    <bean id="bean1" class="..."/>
    <bean id="bean2" class="..."/>
</beans>
```

这样，只需要在创建ClassPathXmlApplicationContext对象传入主xml文件即可。

##### Groovy Bean 定义 DSL

```java
beans {
    dataSource(BasicDataSource) {
        driverClassName = "org.hsqldb.jdbcDriver"
        url = "jdbc:hsqldb:mem:grailsDB"
        username = "sa"
        password = ""
        settings = [mynew:"setting"]
    }
    sessionFactory(SessionFactory) {
        dataSource = dataSource
    }
    myService(MyService) {
        nestedBean = { AnotherBean bean ->
            dataSource = dataSource
        }
    }
}
```

这种定义bean的方式看起来更清晰，在很大程度上等同于 XML bean 定义，甚至支持 Spring 的 XML configuration 命名空间。它还允许通过`importBeans`指令 importing XML bean definition files。

## 4.使用容器

通过容器的getBean方法。

`ApplicationContext`是高级工厂的接口，能够维护不同 beans 及其依赖项的注册表。通过使用方法`T getBean(String name, Class<T> requiredType)`，您可以检索 beans 的实例。

# 三：beans概述

Spring IoC 容器管理一个或多个 beans。这些 beans 是使用您提供给容器的 configuration 元数据创建的(对于 example，以 XML `<bean/>`定义的形式)。

在容器的内部中，beans定义表示为Beandefiniton Object，（因为在spring中，一个bean对象不可能只有用户自定义的类中的信息，还包括其他的信息，比如是否单例，初始化与destroy对应方法，这些在xml标签中都可以找到） 。所以Beandefiniton Object包括以下元数据：

- package-qualified class name：通常是bean 被定义的实际实现类。

- Bean 行为 configuration 元素，它们 state bean 应该如何在容器中运行(范围，生命周期回调等)。

- 引用 bean 执行其工作所需的其他 beans。这些 references 也称为协作者或依赖项。

- 要在新创建的 object 中设置的其他 configuration 设置 - 用于 example，池的大小限制或在管理连接池的 bean 中使用的连接数。

此元数据转换为构成每个 bean 定义的一组 properties。以下 table 描述了这些 properties：

| 属性                       | 解释在......                                                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| Class                    | [实例化 Beans](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-class)                  |
| Name                     | [命名 Beans](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-beanname)                        |
| Scope                    | [Bean的作用域](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-scopes)                  |
| Constructor arguments    | [依赖注入](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-collaborators=)              |
| Properties               | [依赖注入](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-collaborators)               |
| Autowiring mode          | [自动化协作者](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-autowire)                  |
| Lazy initialization mode | [Lazy-initialized Beans](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-lazy-init) |
| Initialization method    | [初始化回调](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-lifecycle-initializingbean) |
| Destruction method       | [毁灭回调](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-lifecycle-disposablebean)    |

除了包含有关如何创建特定 bean 的信息的 bean 定义之外，`ApplicationContext` implementations 还允许注册在容器外部(由用户)创建的现有 objects（比如工厂方法创建的对象注入容器）。这是通过`getBeanFactory()`方法访问 ApplicationContext 的 BeanFactory 来完成的，该方法返回 BeanFactory `DefaultListableBeanFactory` implementation。 `DefaultListableBeanFactory`通过`registerSingleton(..)`和`registerBeanDefinition(..)`方法支持此注册。但是，典型的 applications 只能使用通过常规 bean 定义元数据定义的 beans。

> Bean 元数据和手动提供的 singleton 实例需要尽早注册，以便容器在自动装配和其他内省步骤中正确推理它们。虽然在某种程度上支持覆盖现有元数据和现有 singleton 实例，但是在运行时注册新的 beans(与对工厂的实时访问同时)并未得到官方支持，并且可能导致并发访问 exceptions，bean 容器中的 state 不一致。

### 1：Bean的命名

每个 bean 都有一个或多个标识符。这些标识符在承载 bean 的容器中必须是唯一的。 bean 通常只有一个标识符。但是，如果它需要多个，则额外的可以被视为别名。 默认的bean名称是类名（首字母小写）。 

对于bean别名，使用alias。

### 2：实例化Beans

bean 定义本质上是 creating 一个或多个 objects 的配方。容器在询问时查看命名 bean 的配方，并使用由 bean 定义封装的 configuration 元数据来创建(或获取)实际的 object。

（1）：使用构造函数实例化； spring容器中采用了默认的空构造函数函数进行初始化（如果在类中定义了构造函数但是没有默认的空构造函数，就会报错）。     如果需要向对应的构造函数提供参数，那么比如在xml中，就需要在bean标签中使用相应的构造器标签。

（2）：使用静态工厂方法实例化对象。

```java
<bean id="clientService"
    class="examples.ClientService"
    factory-method="createInstance"/> //factory-method中必须是静态方法并且返回id对应的对象。
```

这种 bean 定义的一个用途是在 legacy code 中调用`static`工厂。

（3）：使用实例工厂方法实例化对象

与通过[静态工厂方法](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-class-static-factory-method)进行实例化类似，使用实例工厂方法进行实例化会从容器中调用现有 bean 的 non-static 方法来创建新的 bean。

```java
<!-- the factory bean, which contains a method called createInstance() -->
<bean id="serviceLocator" class="examples.DefaultServiceLocator">
    <!-- inject any dependencies required by this locator bean -->
</bean>

<!-- the bean to be created via the factory bean -->
<bean id="clientService"
    factory-bean="serviceLocator"
    factory-method="createClientServiceInstance"/>
    
    //工厂类
    public class DefaultServiceLocator {

    private static ClientService clientService = new ClientServiceImpl();

    public ClientService createClientServiceInstance() {
        return clientService;
    }
}
```

实例工厂方法的好处是，一个工厂class可以包含多个工厂方法。那么就表名了，工厂bean本身就可以通过依赖注入（DI）进行管理和配置。

> 在 Spring 文档中，“factory bean”指的是 Spring 容器中配置的 bean，它通过[例](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-class-instance-factory-method)或[静态](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-class-static-factory-method)工厂方法创建 objects。相比之下，`FactoryBean`(注意大写)指的是 Spring-specific [FactoryBean](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-extension-factorybean)。
> 
> 静态工厂方法与工厂方法实例化bean的区别在于，静态与对象无关，但是比较死板，静态类无法被容器复用，而工厂bean比较灵活，通过创建多个工厂方法，可以用于不同bean实例的创建。

# 四：依赖

依赖注入(DI)是一个 process，其中 objects 仅通过构造函数 arguments，工厂方法的 arguments 或 object 实例在构造之后设置的 properties 定义它们的依赖项(即，它们工作的其他 objects)或者从工厂方法返回。然后容器在创建 bean 时注入这些依赖项。这个 process 基本上是 bean 本身的逆(因此 name，控制反转)，它通过使用 classes 或 Service Locator pattern 的直接构造来控制其依赖项的实例化或位置。

使用 DI 原理，Code 更干净，当 objects 具有依赖关系时，解耦更有效。 object 不查找其依赖项，也不知道依赖项的位置或 class。因此，您的 classes 变得更容易测试，特别是当依赖关系在接口或 abstract base classes 上时，它们允许在单元测试中使用 stub 或 mock implementations。

注入的方式主要有：

（1）使用构造方法注入，这个需要在configuration xml中对应的bean标签中使用相应的构造标签传入数据。

```java
 <bean id="thingOne" class="x.y.ThingOne">
        <constructor-arg ref="thingTwo"/>  //传入引用

        <constructor-arg ref="thingThree"/>
    </bean>
    
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg type="int" value="7500000"/> //传入数据

    <constructor-arg type="java.lang.String" value="42"/>
</bean>
//您还可以使用构造函数参数 name 进行 value 消歧，如下面的 example 所示：
<bean id="exampleBean" class="examples.ExampleBean">
    <constructor-arg name="years" value="7500000"/>
    <constructor-arg name="ultimateAnswer" value="42"/>
</bean>
```

请记住，要使这项工作开箱即用，必须在启用 debug flag 的情况下编译 code，以便 Spring 可以从构造函数中查找参数 name。如果您不能或不想使用 debug flag 编译 code，则可以使用[@ConstructorProperties](https://download.oracle.com/javase/6/docs/api/java/beans/ConstructorProperties.html) JDK annotation 显式 name 构造函数 arguments。然后 sample class 必须如下所示：

```java
package examples;
public class ExampleBean {
    // Fields omitted,使用构造方法注解注入

    @ConstructorProperties({"years", "ultimateAnswer"})
    public ExampleBean(int years, String ultimateAnswer) {
        this.years = years;
        this.ultimateAnswer = ultimateAnswer;
    }
}
```

（2）使用set方法依赖注入

Setter-based DI 是在调用 no-argument 构造函数或 no-argument `static`工厂方法来实例化 bean 之后，在 beans 上调用 setter 方法的容器来完成的。

`ApplicationContext`支持它管理的 beans 的 constructor-based 和 setter-based DI。在通过构造函数方法注入了一些依赖项之后，它还支持 setter-based DI。您可以以`BeanDefinition`的形式配置依赖项，并将其与`PropertyEditor`实例结合使用，以将 properties 从一种格式转换为另一种格式。但是，大多数 Spring 用户不直接使用这些 classes(即以编程方式)，而是使用 XML `bean`定义，带注释的组件(即用，`@Controller`等注释的 classes)，或 Java-based `@Configuration` classes 中的`@Bean`方法。然后，这些源在内部转换为`BeanDefinition`的实例，并用于加载整个 Spring IoC 容器实例。

Constructor-based 或 setter-based DI？

由于您可以混合使用 constructor-based 和 setter-based DI，因此将构造函数用于强制依赖项和 setter 方法或使用 configuration 方法作为可选依赖项是一个很好的经验法则。请注意，在 setter 方法上使用[@Required](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-required-annotation)  annotation 可用于使 property 成为必需的依赖项。

#### 2 循环依赖：

如果您主要使用构造函数注入，则可以创建无法解析的循环依赖关系场景。

对于 example：Class A 需要通过构造函数注入实现 class B，而 class B 需要通过构造函数注入实现 class A.如果为 classes A 和 B 配置 beans 以相互注入，则 Spring IoC 容器会在运行时检测到此循环 reference，并抛出`BeanCurrentlyInCreationException`。

一种可能的解决方案是编辑某些 classes 的 source code，以便由 setter 而不是构造函数配置。或者，避免构造函数注入并仅使用 setter 注入。换句话说，尽管不推荐使用，但您可以使用 setter 注入配置循环依赖项。

如果不存在循环依赖关系，则当一个或多个协作 beans 被注入依赖 bean 时，每个协作 bean 在被注入依赖 bean 之前完全配置。这意味着，如果 bean A 依赖于 bean B，Spring IoC 容器在 bean A 上调用 setter 方法之前完全配置 bean B.换句话说，bean 被实例化(如果它不是 pre-instantiated singleton)，设置其依赖项，并调用相关的生命周期方法(例如[配置的 init 方法](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-lifecycle-initializingbean)或[InitializingBean 回调方法](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-lifecycle-initializingbean))。

#### 3 lazy-initialized bean

懒加载bean。

#### 4自动装配Autowired

Spring 容器可以自动配合协作 beans 之间的关系。您可以通过检查`ApplicationContext`的内容让 Spring 自动为您的 bean 解析协作者(其他 beans)。自动装配具有以下优点：

- 自动装配可以显着减少指定 properties 或构造函数 arguments 的需要。 (其他机制，如 bean 模板[在本章的其他地方讨论过](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-scopes-singleton)在此 regard.)也很有价值

- 随着 objects 的发展，自动装配可以更新 configuration。例如，如果需要向 class 添加依赖项，则可以自动满足该依赖项，而无需修改 configuration。因此，自动装配在开发期间尤其有用，而不会在 code base 变得更稳定时否定切换到显式布线的选项。

使用 XML-based configuration 元数据(请参阅[依赖注入](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-child-bean-definitions))时，可以使用`<bean/>`元素的`autowire`属性为 bean 定义指定 autowire 模式。自动装配功能有四种模式。您为每个 bean 指定自动装配，因此可以选择要自动装配的那些。以下 table 描述了四种自动装配模式：

| 模式            | 说明                                                                                                                                                                                               |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `no`          | (默认)无自动装配。 Bean references 必须由`ref`元素定义。不建议对较大的部署更改默认设置，因为明确指定协作者可以提供更好的控制和清晰度。在某种程度上，它记录了系统的结构。                                                                                                 |
| `byName`      | property name 自动装配。 Spring 查找 bean，其 name 与需要自动装配的 property 相同。例如，如果 bean 定义由 name 设置为 autowire 并且它包含`master`  property(即，它具有`setMaster(..)`方法)，则 Spring 将查找名为`master`的 bean 定义并使用它来设置 property。 |
| `byType`      | 如果容器中只存在 property 类型的一个 bean，则允许 property 自动装配。如果存在多个，则抛出致命的 exception，这表示您不能对该 bean 使用`byType`自动装配。如果没有匹配的 beans，则不会发生任何事情(property 未设置)。                                                       |
| `constructor` | 类似于`byType`但适用于构造函数 arguments。如果容器中没有构造函数参数类型的一个 bean，则会引发致命错误。                                                                                                                                  |

考虑自动装配的局限和缺点：

- `property`和`constructor-arg`设置中的显式依赖项始终覆盖自动装配。您无法自动装配简单的 properties，例如 primitives，`Strings`和`Classes`(以及此类简单 properties 的数组)。这个限制是 by-design。

- 自动装配不如显式布线精确。虽然如前面的 table 所述，Spring 谨慎避免在可能产生意外结果的模糊性的情况下进行猜测。您的 Spring-managed object 之间的关系不再明确记录。

- 可能无法为可能从 Spring 容器生成文档的工具提供接线信息。

- 容器中的多个 bean 定义可以 match 由 setter 方法或构造函数参数指定的类型以进行自动装配。对于数组，集合或`Map`实例，这不一定是个问题。但是，对于期望单个 value 的依赖关系，这种歧义不是任意解决的。如果没有唯一的 bean 定义，则抛出 exception。

在后一种情况下，您有几种选择：

- 放弃自动装配，支持显式布线。

- 通过将属性设置为`false`，避免为 bean 定义自动装配，如[下一节](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-collaborators)中所述。

- 通过将元素的`primary`属性设置为`true`，将单个 bean 定义指定为主要候选者。

- 使用 annotation-based configuration 实现更多 fine-grained 控件，如[Annotation-based Container Configuration](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-autowire-candidate)中所述。

### 5方法注入：（实现ApplicationAware接口）

在大多数 application 场景中，容器中的大多数 beans 都是[单身](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-autowired-annotation)。当 singleton bean 需要与另一个 singleton bean 协作或 non-singleton bean 需要与另一个 non-singleton bean 协作时，通常通过将一个 bean 定义为另一个的 property 来处理依赖关系。当 bean 生命周期不同时会出现问题。假设 singleton bean A 需要使用 non-singleton(原型)bean B，可能在 A 上的每个方法调用上。容器只创建 singleton bean A 一次，因此只有一次机会来设置 properties。容器不能为 bean A 提供 bean B 的新实例，每个 time B 都需要一个。

    解决的方式是放弃一些控制反转，可以`ApplicationContextAware`接口，获取容器实例，并通过[对容器进行 getBean(“B”)调用](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-aware)请求(一个通常是新的)bean B 实例每 time bean A 需要它。  这就是ApplicationAware接口的作用。

### 6：Bean的范围

| 范围                                                                                                                               | 描述                                                                                                                                         |
| -------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| [singleton](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-scopes-custom)     | (默认)为每个 Spring IoC 容器的单个 object 实例定义单个 bean 定义。                                                                                            |
| [原型](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-scopes-singleton)         | 为任意数量的 object 实例定义单个 bean 定义。                                                                                                              |
| [请求](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-scopes-prototype)         | 将单个 bean 定义范围限定为单个 HTTP 请求的生命周期。也就是说，每个 HTTP 请求都有自己的 bean 实例，该实例是在单个 bean 定义的后面创建的。仅在 web-aware Spring  `ApplicationContext`的 context 中有效。 |
| [session](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-scopes-request)      | 将单个 bean 定义范围限定为 HTTP  `Session`的生命周期。仅在 web-aware Spring  `ApplicationContext`的 context 中有效。                                              |
| [应用](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-scopes-session)           | 将单个 bean 定义范围限定为`ServletContext`的生命周期。仅在 web-aware Spring  `ApplicationContext`的 context 中有效。                                              |
| [WebSocket](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/web.html#beans-factory-scopes-application) | 将单个 bean 定义范围限定为`WebSocket`的生命周期。仅在 web-aware Spring  `ApplicationContext`的 context 中有效。                                                   |

#### （1）单例范围：

只管理 singleton bean 的一个共享实例，并且 beans 的所有请求都带有一个或多个 match bean 定义的 ID 导致 Spring 容器返回的一个特定 bean 实例。

此单个实例存储在此类 singleton beans 的缓存中，并且所有后续请求和 references 都指向 bean return 缓存的 object。 单例的生命周期与容器相同。

#### （2）原型（多例）范围

bean 部署的 non-singleton 原型范围导致每次都会创建一个新的 bean 实例，并对该特定 bean 发出请求。也就是说，bean 被注入到另一个 bean 中，或者通过容器上的`getBean()`方法调用来请求它。通常，您应该为所有有状态 beans 使用原型范围，为 stateless beans 使用 singleton 范围。

spring只会创建多例对象，之后就不再管理，与其他范围相比，Spring 不管理原型 bean 的完整生命周期。容器实例化，配置和组装原型 object 并将其交给 client，没有该原型实例的进一步 record。

### 7：Aware接口

| 名称                               | 注入依赖                                                                 | 解释在......                                                                                                                                          |
| -------------------------------- | -------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| `ApplicationContextAware`        | 声明`ApplicationContext`。                                              | [ApplicationContextAware 和 BeanNameAware](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-aware) |
| `ApplicationEventPublisherAware` | 封闭`ApplicationContext`的 Event 发布者。                                   | [ApplicationContext 的附加功能](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-aware)                |
| `BeanClassLoaderAware`           | Class loader 用于加载 bean classes。                                      | [实例化 Beans](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#context-introduction)                              |
| `BeanFactoryAware`               | 声明`BeanFactory`。                                                     | [ApplicationContextAware 和 BeanNameAware](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-class) |
| `BeanNameAware`                  | 声明 bean 的名称。                                                         | [ApplicationContextAware 和 BeanNameAware](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-factory-aware) |
| `BootstrapContextAware`          | 资源适配器`BootstrapContext`容器运行。通常仅在 JCA 感知`ApplicationContext`实例中可用。    | [JCA CCI](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/integration.html#beans-factory-aware)                          |
| `LoadTimeWeaverAware`            | 定义的 weaver 用于在 load time 处理 class 定义。                                | [Load-time 在 Spring Framework 中使用 AspectJ 进行编织](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#cci)           |
| `MessageSourceAware`             | 用于解析消息的已配置策略(支持参数化和国际化)。                                             | [ApplicationContext 的附加功能](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#aop-aj-ltw)                         |
| `NotificationPublisherAware`     | Spring JMX 通知发布者。                                                    | [通知](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/integration.html#context-introduction)                              |
| `ResourceLoaderAware`            | 配置加载程序以 low-level 访问资源。                                              | [资源](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#jmx-notifications)                                        |
| `ServletConfigAware`             | 当前`ServletConfig`容器运行。仅在 web-aware Spring  `ApplicationContext`中有效。  | [Spring MVC](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/web.html#resources)                                         |
| `ServletContextAware`            | 当前`ServletContext`容器运行。仅在 web-aware Spring  `ApplicationContext`中有效。 | [Spring MVC](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/web.html#mvc)                                               |

### 8：Bean定义继承

ean 定义可以包含许多 configuration 信息，包括构造函数 arguments，property 值和 container-specific 信息，例如初始化方法，静态工厂方法 name 等。 child bean 定义从 parent 定义继承 configuration 数据。 child 定义可以覆盖某些值或根据需要添加其他值。使用 parent 和 child bean 定义可以节省大量的 typing。实际上，这是一种模板形式。

### 9：扩展spring容器

#### （1）：使用BeanPostProcessor

`BeanPostProcessor`接口定义了您可以实现的回调方法，以提供您自己的(或覆盖容器的默认)实例化逻辑，dependency-resolution 逻辑等。如果要在 Spring 容器完成实例化，配置和初始化 bean 之后实现某些自定义逻辑，则可以插入一个或多个`BeanPostProcessor` implementations。

#### （2）使用BeanFactoryPostProcessor自定义Configuration元数据

此接口的语义类似于`BeanPostProcessor`的语义，但有一个主要区别：`BeanFactoryPostProcessor`对 bean configuration 元数据进行操作。也就是说，Spring IoC 容器允许`BeanFactoryPostProcessor`读取 configuration 元数据，并可能在容器实例化除`BeanFactoryPostProcessor`实例之外的任何 beans 之前更改它。

#### （3）使用FactoryBean自定义实例化逻辑

`FactoryBean`接口是可插入 Spring IoC 容器的实例化逻辑的一个点。如果你有一个复杂的初始化 code，用 Java 表示，而不是(可能)冗长的 XML，你可以创建自己的`FactoryBean`，在 class 中编写复杂的初始化，然后将自定义`FactoryBean`插入容器。`FactoryBean`接口提供了三种方法：

- `Object getObject()`：返回此工厂创建的 object 的实例。可以共享实例，具体取决于此工厂是返回单例还是原型。

- `boolean isSingleton()`：如果`FactoryBean`返回单例，则返回`true`，否则返回`false`。

- `Class getObjectType()`：返回`getObject()`方法返回的 object 类型，如果事先不知道类型，则返回`null`。

当你需要向一个容器询问一个实际的`FactoryBean`实例本身而不是它生成的 bean 时，在调用`ApplicationContext`的`getBean()`方法时，_BE 的`id`前面带有＆符号(`&`)。因此，对于具有`id`  `myBean`的给定`FactoryBean`，在容器上调用`getBean("myBean")`将返回`FactoryBean`的乘积，而调用`getBean("&myBean")`则返回`FactoryBean`实例本身。

### Bean的生命周期图

![images\bean的生命周期](images\bean的生命周期.jpg)



# 总结：

今天首先学习了spring框架的结构，需要认识spring框架图中每一层，每个模块都是干什么的。

然后学习了IOC容器，spring的核心莫过于IOC容器， 了解了控制反转与依赖注入，

在spring中，IOC对应的实例对象可以通过ApplicationContext接口对应的子类创建，准确来说，ApplicationContext继承自BeanFactory（这才是容器的核心），但是ApplicationContext实现了国际化，集成了许多功能，所以还是以ApplicationContext为主。

在spring中，通过configuration配置元数据（即beans），spring容器就是beans元素的集合，目前主流来说，是通过java注解来配置bean的。 通过容器对象（ApplicationContext）可以读取配置信息，然后将配置中的元数据按各自的规则在spring容器中进行注册。

之后学习了Bean，在容器的内部中，beans定义表示为Beandefiniton Object，（因为在spring中，一个bean对象不可能只有用户自定义的类中的信息，还包括其他的信息，比如是否单例，初始化与destroy对应方法，这些在具体的xmlbean标签中都可以找到） 。 

Bean的实例化方式有三种，spring内部默认是使用默认的构造方法创建，  第二个是通过静态工厂方法创建对象并获取， 第三个是使用工厂方法的模式创建。第二种与第三种的区别是，工厂方法的模式创建可以复用对应的工厂类（对应的工厂类可以创建多个工厂方法，可以用于不同bean实例的创建，使用工厂方法模式也可以在对应的bean中实现依赖注入）。

Bean的依赖注入，简单理解就是给对象设置属性值（对应的值如果是对象，在容器中已经创建好了，直接注入即可，这就是依赖注入，可以解耦）。Bean的依赖注入的方式有：（1）：通过构造方法注入（需要定义相应的构造函数，并在bean相应构造函数标签中注入值，这个时候spring就不再使用默认的构造函数了）。（2）通过set方法注入，这也是pojo的注入方式，在 setter 方法上使用[@Required](https://www.docs4dev.com/docs/zh/spring-framework/5.1.3.RELEASE/reference/core.html#beans-required-annotation) annotation 可用于使 property 成为必需的依赖项。

循环依赖？ 比如利用构造方法的注入模式，A构造中有B，B构造中有A，这样在容器创建bean时就会产生循环依赖现象而抛出错误。 解决方法，使用set方法注入，如果 bean A 依赖于 bean B，Spring IoC 容器在 bean A 上调用 setter 方法之前完全配置 bean B.

自动装配：Autowired，它有四种模式，默认会匹配到对应的类的实例注入，但是，当一个接口有两个子类实现时，在自动装配这个接口时，就无法识别使用哪个，抛出异常，在这种情况下就不能使用@autowired，可以使用其他的派生注解。

bean的范围，有6种，单例，多例（原型），request，session，application，websocket，认识每种范围的特点与生命周期，主要是单例与多例的区别，其四种都是单个bean存在的生命周期与其相同。

Aware接口：要是想在在一个bean的定义中获取spring容器内部的对象，那么就需要对应的Aware接口，比如某个类实现ApplicationContextAware，就可以获取ApplicationContext对象，这样就可以操作容器内部的bean。

bean也可以继承，用于扩展，比如在xml中就可以在bean中通过parent相应标签得以实现。

aware接口只是针对某些实现它的bean定制初始化过程，spring可以针对所有的bean或者部分bean做扩展。于是还学习了三种扩展spring容器的方式：

（1）使用BeanPostProcessor，这个接口定义了在bean的生命周期中可以进行回调的方法，该接口中包含两个方法，postProcessBeforeInitialization和postProcessAfterInitialization。 postProcessBeforeInitialization方法会在容器中的Bean初始化之前执行， postProcessAfterInitialization方法在容器中的Bean初始化之后执行。这样就可以在bean的初始化前后插入自定义的逻辑。

（2）BeanFactoryPostProcessor，作用与（1）类似，但是这个是作用在configuration上的，可以对bean configuration 元数据进行操作。简单来说就是可以增加或者删除或修改configuration 中的元数据。

（3）使用FactoryBean扩展，作用是可以直接向容器注册进对应实现的类，而不用在xml文件中声明bean，也可以理解为，直接工厂生成创建对应类的实例bean（少去了注册的过程）。   但是这样创建对应的bean后，获取factorybean的实例需要getBean("&myBean")才能返回其实例本身。

（通过学习FactoryBean，面试题BeanFactory与FactoryBean的区别就分清了，BeanFactory是容器的核心类，FactoryBean的作用如上，直接向容器创建bean）。

最后认识Bean的生命周期图。




