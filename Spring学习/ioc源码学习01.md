# 一：基于XML的applicationcontext子类

 本篇就来分析ClassPathXmlApplicationContext。

## 该类的UML图

![images\ClassPathXmlApplicationContext的体系结构](images\ClassPathXmlApplicationContext的体系结构.jpg)

#### Application的体系结构图

![images\ApplicationContext体系结构](images\ApplicationContext体系结构.png)

从上面的两张UML图中可以看出ClassPathXmlApplicationContext的父类结构图，

其中，最顶端的ResourceLoader代表了**加载资源的一种方式，正是策略模式的实现**，作用是可以将第三方资源加载的ioc容器中。

但是在第二张图中，有两个绿色标识的类**FileSystemXmlApplicationContext** 和 **AnnotationConfigApplicationContext** 这两个类。

**1、FileSystemXmlApplicationContext** 的构造函数需要一个 xml 配置文件在系统中的路径，其他和 ClassPathXmlApplicationContext 基本上一样。

2、**AnnotationConfigApplicationContext**就是spring中基于注解的读取方式，目前已经替代了XML的配置方式，其利用了字节码加载机制。

#### ClassPathXmlApplicationContext构造器源码：

```java
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent) throws BeansException {
        super(parent);
        //解析传入路径对应的的xml文件

        this.setConfigLocations(configLocations);
        if (refresh) {
            this.refresh();
        }

    }
```

下面，就来具体分析这个构造器中的过程。

## 1.构造器

首先一直点击super查看父类构造器，直到最顶层AbstractApplicationContext。

```java
 public AbstractApplicationContext(ApplicationContext parent) {
        this();
        this.setParent(parent);
    }
public AbstractApplicationContext() {
        this.logger = LogFactory.getLog(this.getClass());
        this.id = ObjectUtils.identityToString(this);
        this.displayName = ObjectUtils.identityToString(this);
        this.beanFactoryPostProcessors = new ArrayList();
        this.active = new AtomicBoolean();
        this.closed = new AtomicBoolean();
        this.startupShutdownMonitor = new Object();
        this.applicationListeners = new LinkedHashSet();
        this.resourcePatternResolver = this.getResourcePatternResolver();
    }
 protected ResourcePatternResolver getResourcePatternResolver() {
        return new PathMatchingResourcePatternResolver(this);
        //PathMatchingResourcePatternResolver支持Ant风格的路径解析。

    }
```

## 2.AbstractRefreshableConfigApplicationContext

该类继承自AbstractRefreshableApplicationContext，该类作用是设置配置文件（比如xml配置，proporties，profile）的前置路径，比如解析“classpath:”。

```java
public void setConfigLocation(String location) {
        this.setConfigLocations(StringUtils.tokenizeToStringArray(location, ",; \t\n"));
    }

    public void setConfigLocations(String... locations) {
        if (locations != null) {
            Assert.noNullElements(locations, "Config locations must not be null");
            this.configLocations = new String[locations.length];

            for(int i = 0; i < locations.length; ++i) {
                this.configLocations[i] = this.resolvePath(locations[i]).trim();
            }
        } else {
            this.configLocations = null;
        }

    }
```

既然读取了路径，那么就要解析路径，读取对应的文件，具体方法如下：

```java
protected String resolvePath(String path) {
        return this.getEnvironment().resolveRequiredPlaceholders(path);
    }
```

此方法的目的在于将占位符(placeholder)解析成实际的地址。比如可以这么写:  `new ClassPathXmlApplicationContext("classpath:config.xml");`那么classpath:就是需要被解析的。

getEnvironment方法来自于ConfigurableApplicationContext接口，源码很简单，如果为空就调用createEnvironment创建一个。

```java
//在AbstractApplicationContext类中
protected ConfigurableEnvironment createEnvironment() {
        return new StandardEnvironment();
    }
```

Environment是ioc容器的环境，那么就来分析Environment的结构。

### Environment接口

#### (1)继承体系：

![images\Environment](images\Environment.jpg)

#### （2）

Environment接口代表当前应用所处的环境，从图中可以看出，其主要与profile、Property相关。

##### profile

Spring Profile特性是从3.1开始的，其主要是为了解决这样一种问题: 线上环境和测试环境使用不同的配置或是数据库或是其它。有了Profile便可以在 不同环境之间无缝切换。**Spring容器管理的所有bean都是和一个profile绑定在一起的。**使用了Profile的配置文件示例:

```xml
<beans profile="develop">  
    <context:property-placeholder location="classpath*:jdbc-develop.properties"/>  
</beans>  
<beans profile="production">  
    <context:property-placeholder location="classpath*:jdbc-production.properties"/>  
</beans>  
<beans profile="test">  
    <context:property-placeholder location="classpath*:jdbc-test.properties"/>  
</beans>
```

在启动代码中可以用如下代码设置活跃(当前使用的)Profile:

```java
context.getEnvironment().setActiveProfiles("dev");
```

##### Property

这里的Property指的是程序运行时的一些参数，引用注释:

> > properties files, JVM system properties, system environment variables, JNDI, servlet context parameters, ad-hoc Properties objects,Maps, and so on.

#### （3）Environment构造器

```java
 private final MutablePropertySources propertySources = new MutablePropertySources(this.logger);
public AbstractEnvironment() {
    customizePropertySources(this.propertySources);
}
```

##### propertySources接口

该接口的实现类MutablePropertySources，此接口实际上是PropertySource的容器，默认的MutablePropertySources实现内部含有一个CopyOnWriteArrayList作为存储载体。

```java
 MutablePropertySources(Log logger) {
        //使用线程安全的arrayList，基于写复制保证线程安全

        this.propertySourceList = new CopyOnWriteArrayList();
        this.logger = logger;
    }
```

```java
public class StandardEnvironment extends AbstractEnvironment {
/** System environment property source name: {@value} */
        public static final String 
SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME = "systemEnvironment";

   /** JVM system properties property source name: {@value} */
     public static final String SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME = "systemProperties";
    public StandardEnvironment() {

    }

    protected void customizePropertySources(MutablePropertySources propertySources) {
        propertySources.addLast(new MapPropertySource("systemProperties", this.getSystemProperties()));
        propertySources.addLast(new SystemEnvironmentPropertySource("systemEnvironment", this.getSystemEnvironment()));
    }
}
```

PropertySource接口代表了键值对的Property来源。继承体系：

![images\PropertySource](images\PropertySource.jpg)

而PropertySource中的存储载体CopyOnWriteArrayList，其添加的元素为：systemProperties和systemEnvironment，这两个对应的方法在AbstractEnvironment中，

```java
 public Map<String, Object> getSystemProperties() {
        try {
             //获取系统的属性

            return System.getProperties();
        } catch (AccessControlException var2) {
            return new ReadOnlySystemAttributesMap() {
                protected String getSystemAttribute(String attributeName) {
                    try {
                        //如果安全管理器阻止获取全部属性，就尝试获取单个属性

                        return System.getProperty(attributeName);
                    } catch (AccessControlException var3) {
                        if (AbstractEnvironment.this.logger.isInfoEnabled()) {
                            AbstractEnvironment.this.logger.info("Caught AccessControlException when accessing system property '" + attributeName + "'; its value will be returned [null]. Reason: " + var3.getMessage());
                        }

                        return null;
                    }
                }
            };
        }
    }
```

这里的实现很有意思，如果安全管理器阻止获取全部的系统属性，那么会尝试获取单个属性的可能性，如果还不行就抛异常了。

getSystemEnvironment方法也是一个套路，不过最终调用的是System.getenv，可以获取jvm和OS的一些版本信息。

获取系统的属性与环境的相关操作就分析完了，下面继续分析路径解析的问题。

#### （4）路径Placeholder处理

##### AbstractEnvironment.resolveRequiredPlaceholders:

```java
public String resolveRequiredPlaceholders(String text) throws IllegalArgumentException {
    //text即配置文件路径，比如classpath:config.xml       
        return this.propertyResolver.resolveRequiredPlaceholders(text);

    }
```

##### propertyResolver是一个PropertySourcesPropertyResolver对象:

```java
private final ConfigurablePropertyResolver propertyResolver =
            new PropertySourcesPropertyResolver(this.propertySources);
```

##### propertyResolver接口

PropertyResolver继承体系(排除Environment分支):

![images\PropertyResolver](images\PropertyResolver.jpg)

这个接口正是用来解析PropertyResource。

###### 解析

AbstractPropertyResolver.resolveRequiredPlaceholders:

```java
 public String resolveRequiredPlaceholders(String text) throws IllegalArgumentException {
        if (this.strictHelper == null) {
            this.strictHelper = this.createPlaceholderHelper(false);
        }

        return this.doResolvePlaceholders(text, this.strictHelper);
    }
    
private PropertyPlaceholderHelper createPlaceholderHelper(boolean ignoreUnresolvablePlaceholders) {
 //三个参数分别是${, }, :
 return new PropertyPlaceholderHelper(this.placeholderPrefix, this.placeholderSuffix,
 this.valueSeparator, ignoreUnresolvablePlaceholders);
}
```

doResolvePlaceholders:

```java
 private String doResolvePlaceholders(String text, PropertyPlaceholderHelper helper) {
 //PlaceholderResolver接口依然是策略模式的体现
        return helper.replacePlaceholders(text, new PlaceholderResolver() {
            public String resolvePlaceholder(String placeholderName) {
                return AbstractPropertyResolver.this.getPropertyAsRawString(placeholderName);
            }
        });
    }
```

其实代码执行到这里的时候还没有进行xml配置文件的解析，那么这里的解析placeHolder是什么意思呢，原因在于可以这么写:

```java
System.setProperty("spring", "classpath");
ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("${spring}:config.xml");
SimpleBean bean = context.getBean(SimpleBean.class);
```

这样就可以正确解析。placeholder的替换其实就是字符串操作，也就是${}符的解析，这里只说一下正确的属性是怎么来的。实现的关键在于PropertySourcesPropertyResolver.getProperty:

```java
@Override
protected String getPropertyAsRawString(String key) {
    return getProperty(key, String.class, false);
}
protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
    //解析propertySource中的资源

    if (this.propertySources != null) {
        for (PropertySource<?> propertySource : this.propertySources) {
            Object value = propertySource.getProperty(key);
            return value;
        }
    }
    return null;

}
```

从上面代码就可以看出，很明显，从系统中获取，也就是从System.getProperty和System.getenv获取，但是由于环境变量是无法自定义的，所以其实此处只能通过System.setProperty指定。

注意，classpath:XXX这种写法的classpath前缀到目前为止还没有被处理。

### 小结

    到这，解析配置文件的操作就已经完成了，通过AbstractRefreshableConfigApplicationContext设置配置文件的前置路径（classpath：），Environment接口是IOC容器的环境配置，与profile（应用环境，比如dev，生产环境等）与Property有关，这里的Property是获取系统（比如jvm，os）属性，指的是程序运行时的一些参数。

    然后具体的就是profile与Property相关的操作，首先通过propertySources接口实现类中的copyonwriteArrayList来作为存储集合，这个存储载体添加了系统的环境与系统的属性信息，   获取系统的属性与环境的方法在AbstractEnvironment类中。

     既然已经获取了系统的环境与属性，那么就可以解析路径了，通过解析路径Placeholder处理，Placeholder代表的是 ${}的模式，即以获取系统属性的方式获取对应的classpath: ，使用的是propertyResolver接口来解析属性资源。

> 简单来说，就是获取系统的属性与环境，利用propertySources保存，propertySources也保存了配置的信息，比如可以将classpath:对应路径保存在系统属性中，然后利用propertyResolver接口解析PropertyResource资源，继而解析系统属性保存的对应路径classpath：对应的xml文件的路径 。

上面的过程对应的是ClassPathXmlApplicationContext构造函数中的

**this.setConfigLocations(configLocations);**

```java
public void setConfigLocations(String... locations) {
        if (locations != null) {
            Assert.noNullElements(locations, "Config locations must not be null");
            this.configLocations = new String[locations.length];

            for(int i = 0; i < locations.length; ++i) {
                this.configLocations[i] = this.resolvePath(locations[i]).trim(); //解析xml文件路径

            }
        } else {
            this.configLocations = null;
        }

    }
    
protected String resolvePath(String path) {
//将路径解析为实际的地址，在上面已经分析过了，比如classpath：的解析。获取对应的系统环境与属性，并保存在prototypeSource，然后利用propertyResolver接口来解析对应的path。
        return this.getEnvironment().resolveRequiredPlaceholders(path);
    }
```

## 3:refresh

spring bean的解析就在此方法中，所以是重点。

### AbstractApplicationContext.refresh:

```java
public void refresh() throws BeansException, IllegalStateException {
        Object var1 = this.startupShutdownMonitor;
        //加同步

        synchronized(this.startupShutdownMonitor) {
            //容器准备工作

            this.prepareRefresh();
            //beanfactory的创建

            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            // Prepare the bean factory for use in this context.

            this.prepareBeanFactory(beanFactory);

            try {
             // Allows post-processing of the bean factory in context subclasses.
                this.postProcessBeanFactory(beanFactory);
                // Invoke factory processors registered as beans in the context.
                this.invokeBeanFactoryPostProcessors(beanFactory);
                // Register bean processors that intercept bean creation.
                this.registerBeanPostProcessors(beanFactory);
                // Initialize message source for this context.
                this.initMessageSource();
                 // Initialize event multicaster for this context.
                this.initApplicationEventMulticaster();
                // Initialize other special beans in specific context subclasses.
                this.onRefresh();
                 // Check for listener beans and register them.
                this.registerListeners();
                // Instantiate all remaining (non-lazy-init) singletons.
                this.finishBeanFactoryInitialization(beanFactory);
                    // Last step: publish corresponding event.
                this.finishRefresh();
            } catch (BeansException var9) {
                if (this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                }
        // Destroy already created singletons to avoid dangling resources.
                this.destroyBeans();
                // Reset 'active' flag.
                this.cancelRefresh(var9);
                throw var9;
            } finally {
            // Reset common introspection caches in Spring's core, since we

            // might not ever need metadata for singleton beans anymore...
                this.resetCommonCaches();
            }

        }
    }
```

首先，可以发现，bean的解析是线程安全同步的，同一时间，只能有一个AbstractApplicationContext子类实例读取对应的配置文件进行bean解析。

然后通过源码还需要了解一个知识点，BeanFactory，applicationcontext的父类，ioc容器中生产bean的基本组件。

![images\BeanFacory体系](images\BeanFacory体系.png)

1. ApplicationContext 继承了 ListableBeanFactory，这个 Listable 的意思就是，通过这个接口，我们可以获取多个 Bean，大家看源码会发现，最顶层 BeanFactory 接口的方法都是获取单个 Bean 的。
2. ApplicationContext 继承了 HierarchicalBeanFactory，Hierarchical 单词本身已经能说明问题了，也就是说我们可以在应用中起多个 BeanFactory，然后可以将各个 BeanFactory 设置为父子关系。
3. AutowireCapableBeanFactory 这个名字中的 Autowire 大家都非常熟悉，它就是用来自动装配 Bean 用的，但是仔细看上图，ApplicationContext 并没有继承它，不过不用担心，不使用继承，不代表不可以使用组合，如果你看到 ApplicationContext 接口定义中的最后一个方法 getAutowireCapableBeanFactory() 就知道了。
4. ConfigurableListableBeanFactory 也是一个特殊的接口，看图，特殊之处在于它继承了第二层所有的三个接口，而 ApplicationContext 没有。这点之后会用到。



下面就来具体分析refresh中大概15个方法的具体内容。

#### （1）prepareRefresh

准备工作，记录启动时间，启动标志，初始化PropertySources，获取环境中合法的属性信息（即校验xml配置）。

```java
 protected void prepareRefresh() {
   // 记录启动时间，
   // 将 active 属性设置为 true，closed 属性设置为 false，它们都是 AtomicBoolean 类型
        this.startupDate = System.currentTimeMillis();
        this.closed.set(false);
        this.active.set(true);
        if (this.logger.isInfoEnabled()) {
            this.logger.info("Refreshing " + this);
        }
        //初始化PropertySources,根据不同环境初始化不同的资源，比如web初始化

        this.initPropertySources();
        this.getEnvironment().validateRequiredProperties();
        this.earlyApplicationEvents = new LinkedHashSet();
    }
```

##### 属性校验

AbstractEnvironment.validateRequiredProperties:

```java
public void validateRequiredProperties() throws MissingRequiredPropertiesException {
        this.propertyResolver.validateRequiredProperties();
    }
```

AbstractPropertyResolver.validateRequiredProperties:

```java
 public void validateRequiredProperties() {
        MissingRequiredPropertiesException ex = new MissingRequiredPropertiesException();
        Iterator var2 = this.requiredProperties.iterator();

        while(var2.hasNext()) {
            String key = (String)var2.next();
            if (this.getProperty(key) == null) {
                //判断校验的过程

                ex.addMissingRequiredProperty(key);
            }
        }

        if (!ex.getMissingRequiredProperties().isEmpty()) {
            throw ex;
        }
    }
```

requiredProperties是通过setRequiredProperties方法设置的，保存在一个list里面，默认是空的，也就是不需要校验任何属性。

```java
private final Set<String> requiredProperties = new LinkedHashSet();
```

#### (2)BeanFactory的创建

由obtainFreshBeanFactory调用AbstractRefreshableApplicationContext.refreshBeanFactory:

```
/ 这步比较关键，这步完成后，配置文件就会解析成一个个 Bean 定义，注册到 BeanFactory 中，
      // 当然，这里说的 Bean 还没有初始化，只是配置信息都提取出来了，
      // 注册也只是将这些信息都保存到了注册中心(说到底核心是一个 beanName-> beanDefinition 的 map)
```

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
        this.refreshBeanFactory();
        ConfigurableListableBeanFactory beanFactory = this.getBeanFactory();
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Bean factory for " + this.getDisplayName() + ": " + beanFactory);
        }

        return beanFactory;
    }
    
  protected final void refreshBeanFactory() throws BeansException {
      //如果有BeanFactory，那么先进行销毁。

        if (this.hasBeanFactory()) {
            this.destroyBeans();
            this.closeBeanFactory();
        }

        try {
         //创建了一个DefaultListableBeanFactory对象
            DefaultListableBeanFactory beanFactory = this.createBeanFactory();
            //设置序列化id

            beanFactory.setSerializationId(this.getId());
            //beanFactory的定制            

            this.customizeBeanFactory(beanFactory);
            //注册bean

            this.loadBeanDefinitions(beanFactory);
            Object var2 = this.beanFactoryMonitor;
            synchronized(this.beanFactoryMonitor) {
                this.beanFactory = beanFactory;
            }
        } catch (IOException var5) {
            throw new ApplicationContextException("I/O error parsing bean definition source for " + this.getDisplayName(), var5);
        }
    }
```

##### BeanFactory接口

此接口正如之前所说，就是bean的容器，

![images\BeanFactory](images\BeanFactory.jpg)



##### BeanFactory定制

AbstractRefreshableApplicationContext.customizeBeanFactory方法用于给子类提供一个自由配置的机会，默认实现:

```java
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
    if (this.allowBeanDefinitionOverriding != null) {
        //默认false，不允许覆盖
        beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    if (this.allowCircularReferences != null) {
        //默认false，不允许循环引用
        beanFactory.setAllowCircularReferences(this.allowCircularReferences);
    }
}
```

##### Bean加载（注册）

AbstractXmlApplicationContext.loadBeanDefinitions，这个便是核心的bean加载了:

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
 // Create a new XmlBeanDefinitionReader for the given BeanFactory.
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
        // Configure the bean definition reader with this context's
    // resource loading environment.
        beanDefinitionReader.setEnvironment(this.getEnvironment());
        beanDefinitionReader.setResourceLoader(this);
        beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
        // Allow a subclass to provide custom initialization of the reader,
    // then proceed with actually loading the bean definitions.
    //默认空实现
        this.initBeanDefinitionReader(beanDefinitionReader);
        this.loadBeanDefinitions(beanDefinitionReader);
    }
```

bean注册进入ioc就变为了beanDefinition，

##### EntityResolver

此处只说明用到的部分继承体系:

[![EntityResolver继承体系](https://github.com/seaswalker/spring-analysis/raw/master/note/images/EntityResolver.jpg)

EntityResolver接口在org.xml.sax中定义。DelegatingEntityResolver用于schema和dtd的解析。

##### BeanDefinitionReader

![images\BeanDefinitionReader](images\BeanDefinitionReader.jpg)

##### 路径解析（ant）

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
        Resource[] configResources = this.getConfigResources();
        if (configResources != null) {
            reader.loadBeanDefinitions(configResources);
        }

        String[] configLocations = this.getConfigLocations();
        if (configLocations != null) {
            reader.loadBeanDefinitions(configLocations);
        }

    }
```

AbstractBeanDefinitionReader.loadBeanDefinitions:

```java
 public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
        Assert.notNull(resources, "Resource array must not be null");
        int counter = 0;
        Resource[] var3 = resources;
        int var4 = resources.length;

        for(int var5 = 0; var5 < var4; ++var5) {
            Resource resource = var3[var5];
            counter += this.loadBeanDefinitions((Resource)resource); //加载BeanDefinition

        }
        // 最后返回 counter，表示总共加载了多少的 BeanDefinition
        return counter;
    }
```

之后调用:

```java
//第二个参数为空
public int loadBeanDefinitions(String location, Set<Resource> actualResources) {
    ResourceLoader resourceLoader = getResourceLoader();
    //参见ResourceLoader类图，ClassPathXmlApplicationContext实现了此接口
    if (resourceLoader instanceof ResourcePatternResolver) {
        // Resource pattern matching available.
        try {
            Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
            int loadCount = loadBeanDefinitions(resources);
            if (actualResources != null) {
                for (Resource resource : resources) {
                // 用一个 ThreadLocal 来存放配置文件资源

                    actualResources.add(resource);
                }
            }
            return loadCount;
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException(
                    "Could not resolve bean definition resource pattern [" + location + "]", ex);
        }
    }
    else {
        // Can only load single resources by absolute URL.
        Resource resource = resourceLoader.getResource(location);
        int loadCount = loadBeanDefinitions(resource);
        if (actualResources != null) {
            actualResources.add(resource);
        }
        return loadCount;
    }
}
```

getResource的实现在AbstractApplicationContext：

```java
@Override
public Resource[] getResources(String locationPattern) throws IOException {
    //构造器中初始化，PathMatchingResourcePatternResolver对象
    return this.resourcePatternResolver.getResources(locationPattern);
}
```

PathMatchingResourcePatternResolver是ResourceLoader继承体系的一部分。

```java
@Override
public Resource[] getResources(String locationPattern) throws IOException {
    Assert.notNull(locationPattern, "Location pattern must not be null");
    //classpath:
    if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) {
        // a class path resource (multiple resources for same name possible)
        //matcher是一个AntPathMatcher对象
        if (getPathMatcher().isPattern(locationPattern
            .substring(CLASSPATH_ALL_URL_PREFIX.length()))) {
            // a class path resource pattern
            return findPathMatchingResources(locationPattern);
        } else {
            // all class path resources with the given name
            return findAllClassPathResources(locationPattern
                .substring(CLASSPATH_ALL_URL_PREFIX.length()));
        }
    } else {
        // Only look for a pattern after a prefix here
        // (to not get fooled by a pattern symbol in a strange prefix).
        int prefixEnd = locationPattern.indexOf(":") + 1;
        if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) {
            // a file pattern
            return findPathMatchingResources(locationPattern);
        }
        else {
            // a single resource with the given name
            return new Resource[] {getResourceLoader().getResource(locationPattern)};
        }
    }
}
```

isPattern:

```java
@Override
public boolean isPattern(String path) {
    return (path.indexOf('*') != -1 || path.indexOf('?') != -1);
}
```

可以看出配置文件路径是支持ant风格的，也就是可以这么写:

```java
new ClassPathXmlApplicationContext("con*.xml");
```

##### 配置文件的加载

入口方法在AbstractBeanDefinitionReader中：

```java
//加载
Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
//解析
int loadCount = loadBeanDefinitions(resources);
```

最终逐个调用XmlBeanDefinitionReader的loadBeanDefinitions方法:

```java
@Override
public int loadBeanDefinitions(Resource resource) {
    return loadBeanDefinitions(new EncodedResource(resource));
}
```

Resource是代表一种资源的接口，其类图:

![images\Resource](images\Resource.jpg)

EncodedResource扮演的其实是一个装饰器的模式，为InputStreamSource添加了字符编码(虽然默认为null)。这样为我们自定义xml配置文件的编码方式提供了机会。

之后关键的源码只有两行:

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    InputStream inputStream = encodedResource.getResource().getInputStream();
    InputSource inputSource = new InputSource(inputStream);
    return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
}
```

InputSource是org.xml.sax的类。

doLoadBeanDefinitions：

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource) {
    Document doc = doLoadDocument(inputSource, resource);
    return registerBeanDefinitions(doc, resource);
}
```

doLoadDocument:

```java
protected Document doLoadDocument(InputSource inputSource, Resource resource) {
    return this.documentLoader.loadDocument(inputSource, getEntityResolver(), this.errorHandler,
        getValidationModeForResource(resource), isNamespaceAware());
}
```

documentLoader是一个DefaultDocumentLoader对象，此类是DocumentLoader接口的唯一实现。getEntityResolver方法返回ResourceEntityResolver，上面说过了。errorHandler是一个SimpleSaxErrorHandler对象。

校验模型其实就是确定xml文件使用xsd方式还是dtd方式来校验，忘了的话左转度娘。Spring会通过读取xml文件的方式判断应该采用哪种。

NamespaceAware默认false，因为默认配置了校验为true。

DefaultDocumentLoader.loadDocument:

```java
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver, ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {
//这里就是老套路了，可以看出，Spring还是使用了dom的方式解析，即一次全部load到内存
        DocumentBuilderFactory factory = this.createDocumentBuilderFactory(validationMode, namespaceAware);
        if (logger.isDebugEnabled()) {
            logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
        }

        DocumentBuilder builder = this.createDocumentBuilder(factory, entityResolver, errorHandler);
        return builder.parse(inputSource);
    }
```

createDocumentBuilderFactory比较有意思:

```java
protected DocumentBuilderFactory createDocumentBuilderFactory(int validationMode, boolean namespaceAware{
    DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
    factory.setNamespaceAware(namespaceAware);
    if (validationMode != XmlValidationModeDetector.VALIDATION_NONE) {
        //此方法设为true仅对dtd有效，xsd(schema)无效
        factory.setValidating(true);
        if (validationMode == XmlValidationModeDetector.VALIDATION_XSD) {
            // Enforce namespace aware for XSD...
             //开启xsd(schema)支持
            factory.setNamespaceAware(true);
             //这个也是Java支持Schema的套路，可以问度娘
            factory.setAttribute(SCHEMA_LANGUAGE_ATTRIBUTE, XSD_SCHEMA_LANGUAGE);
        }
    }
    return factory;
}
```

现在，已经完成了找到对应的xml文件并准备进行bean的解析，xml文件的解析使用的是DefaultDocumentLoader。

##### Bean解析：

XmlBeanDefinitionReader.registerBeanDefinitions:注册BeanDefinitions。

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        BeanDefinitionDocumentReader documentReader = this.createBeanDefinitionDocumentReader();
        int countBefore = this.getRegistry().getBeanDefinitionCount();
        documentReader.registerBeanDefinitions(doc, this.createReaderContext(resource));
        return this.getRegistry().getBeanDefinitionCount() - countBefore;
    }
```

（1）createBeanDefinitionDocumentReader:

```java
protected BeanDefinitionDocumentReader createBeanDefinitionDocumentReader() {
        return (BeanDefinitionDocumentReader)BeanDefinitionDocumentReader.class.cast(BeanUtils.instantiateClass(this.documentReaderClass));
        //利用了反射

    }
```

documentReaderClass默认是DefaultBeanDefinitionDocumentReader，这其实也是策略模式，通过setter方法可以更换其实现。

注意cast方法，代替了强转。

（2）createReaderContext：

```java
public XmlReaderContext createReaderContext(Resource resource) {
        return new XmlReaderContext(resource, this.problemReporter, this.eventListener, this.sourceExtractor, this, this.getNamespaceHandlerResolver());
    }
```

problemReporter是一个FailFastProblemReporter对象。

eventListener是EmptyReaderEventListener对象，此类里的方法都是空实现。

sourceExtractor是NullSourceExtractor对象，直接返回空，也是空实现。

getNamespaceHandlerResolver默认返回DefaultNamespaceHandlerResolver对象，用来获取xsd对应的处理器。

XmlReaderContext的作用感觉就是这一堆参数的容器，糅合到一起传给DocumentReader，并美其名为Context。可以看出，Spring中到处都是策略模式，大量操作被抽象成接口。

（3）DefaultBeanDefinitionDocumentReader.registerBeanDefinitions:

```java
@Override
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    Element root = doc.getDocumentElement();
    doRegisterBeanDefinitions(root);
}
```

doRegisterBeanDefinitions: 这是解析xml中具体的标签

```java
protected void doRegisterBeanDefinitions(Element root) {
    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = createDelegate(getReaderContext(), root, parent);
    //默认的命名空间即
    //http://www.springframework.org/schema/beans
    if (this.delegate.isDefaultNamespace(root)) {
        //检查profile属性
        String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
        if (StringUtils.hasText(profileSpec)) {
            //profile属性可以以,分割
            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                    profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                return;
            }
        }
    }
    //preProcessXml方法是个空实现，供子类去覆盖，目的在于给子类一个把我们自定义的标签转为Spring标准标签的机会,.

    preProcessXml(root);
    parseBeanDefinitions(root, this.delegate);
    postProcessXml(root);
    this.delegate = parent;
}
```

delegate的作用在于处理beans标签的嵌套，其实Spring配置文件是可以写成这样的:

```xml
<?xml version="1.0" encoding="UTF-8"?>    
<beans>    
    <bean class="base.SimpleBean"></bean>
    <beans>
        <bean class="java.lang.Object"></bean>
    </beans>
</beans>
```

xml(schema)的命名空间其实类似于java的包名，命名空间采用URL，比如Spring的是这样:

```xml
<?xml version="1.0" encoding="UTF-8"?>    
<beans xmlns="http://www.springframework.org/schema/beans"></beans>
```

xmlns属性就是xml规范定义的用来设置命名空间的。这样设置了之后其实里面的bean元素全名就相当于[http://www.springframework.org/schema/beans:bean，可以有效的防止命名冲突。命名空间可以通过规范定义的org.w3c.dom.Node.getNamespaceURI方法获得。](http://www.springframework.org/schema/beans:bean%EF%BC%8C%E5%8F%AF%E4%BB%A5%E6%9C%89%E6%95%88%E7%9A%84%E9%98%B2%E6%AD%A2%E5%91%BD%E5%90%8D%E5%86%B2%E7%AA%81%E3%80%82%E5%91%BD%E5%90%8D%E7%A9%BA%E9%97%B4%E5%8F%AF%E4%BB%A5%E9%80%9A%E8%BF%87%E8%A7%84%E8%8C%83%E5%AE%9A%E4%B9%89%E7%9A%84org.w3c.dom.Node.getNamespaceURI%E6%96%B9%E6%B3%95%E8%8E%B7%E5%BE%97%E3%80%82)

注意一下profile的检查, AbstractEnvironment.acceptsProfiles:

```java
@Override
public boolean acceptsProfiles(String... profiles) {
    Assert.notEmpty(profiles, "Must specify at least one profile");
    for (String profile : profiles) {
        if (StringUtils.hasLength(profile) && profile.charAt(0) == '!') {
            if (!isProfileActive(profile.substring(1))) {
                return true;
            }
        } else if (isProfileActive(profile)) {
            return true;
        }
    }
    return false;
}
```

原理很简单，注意从源码可以看出，**profile属性支持!取反**。接着继续。

DefaultBeanDefinitionDocumentReader.parseBeanDefinitions：

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        if (delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();

            for(int i = 0; i < nl.getLength(); ++i) {
                Node node = nl.item(i);
                if (node instanceof Element) {
                    Element ele = (Element)node;
                    if (delegate.isDefaultNamespace(ele)) {
                        this.parseDefaultElement(ele, delegate);
                    } else {
                        delegate.parseCustomElement(ele);
                    }
                }
            }
        } else {
            delegate.parseCustomElement(root);
        }

    }
```

可见，对于非默认命名空间的元素交由delegate处理。

##### 默认命名空间的解析

即import, alias, bean, 嵌套的beans四种元素。parseDefaultElement:

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
        if (delegate.nodeNameEquals(ele, "import")) {
            this.importBeanDefinitionResource(ele);
        } else if (delegate.nodeNameEquals(ele, "alias")) {
            this.processAliasRegistration(ele);
        } else if (delegate.nodeNameEquals(ele, "bean")) {
            this.processBeanDefinition(ele, delegate);
        } else if (delegate.nodeNameEquals(ele, "beans")) {
            this.doRegisterBeanDefinitions(ele);
        }

    }
```

1. import标签：导入其他的XML文件。 <import resource="CTIContext.xml" />

2. alias：别名，假如有一个bean名为componentA-dataSource，但是另一个组件想以componentB-dataSource的名字使用，就可以这样定义:

<alias name="componentA-dataSource" alias="componentB-dataSource"/>

>  别名关系的保存使用Map完成，key为别名，value为本来的名字。

3. bean :

bean节点是Spring最最常见的节点了。

DefaultBeanDefinitionDocumentReader.processBeanDefinition:

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele); //解析bean标签的核心方法

    if (bdHolder != null) {
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // Register the final decorated instance.
            BeanDefinitionReaderUtils.registerBeanDefinition
                (bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                    bdHolder.getBeanName() + "'", ele, ex);
        }
        // Send registration event.
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

最终调用BeanDefinitionParserDelegate.parseBeanDefinitionElement(Element ele, BeanDefinition containingBean)，源码较长，分部分说明。

首先获取到id和name属性，**name属性支持配置多个，以逗号分隔，如果没有指定id，那么将以第一个name属性值代替。id必须是唯一的，name属性其实是alias的角色，可以和其它的bean重复，如果name也没有配置，那么其实什么也没做**。

```java
String id = ele.getAttribute(ID_ATTRIBUTE);
String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
List<String> aliases = new ArrayList<String>();
if (StringUtils.hasLength(nameAttr)) {
    //按,分隔
    String[] nameArr = StringUtils.tokenizeToStringArray
        (nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
    aliases.addAll(Arrays.asList(nameArr));
}
String beanName = id;
if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
    //name的第一个值作为id
    beanName = aliases.remove(0);
}
//默认null
if (containingBean == null) {
    //校验id是否已重复，如果重复直接抛异常
    //校验是通过内部一个HashSet完成的，出现过的id都会保存进此Set
    //private final Set<String> usedNames = new HashSet();

    checkNameUniqueness(beanName, aliases, ele);
}
```

beanName的生成：如果name和id属性都没有指定，那么Spring会自己生成一个, BeanDefinitionParserDelegate.parseBeanDefinitionElement:

```java
beanName = this.readerContext.generateBeanName(beanDefinition);
String beanClassName = beanDefinition.getBeanClassName();
aliases.add(beanClassName);
```

可见，Spring同时会把类名作为其别名。

最终调用的是BeanDefinitionReaderUtils.generateBeanName:

```java
public static String generateBeanName(
        BeanDefinition definition, BeanDefinitionRegistry registry, boolean isInnerBean) {
    String generatedBeanName = definition.getBeanClassName();
    if (generatedBeanName == null) {
        if (definition.getParentName() != null) {
            generatedBeanName = definition.getParentName() + "$child";
             //工厂方法产生的bean
        } else if (definition.getFactoryBeanName() != null) {
            generatedBeanName = definition.getFactoryBeanName() + "$created";
        }
    }
    String id = generatedBeanName;
    if (isInnerBean) {
        // Inner bean: generate identity hashcode suffix.
        id = generatedBeanName + GENERATED_BEAN_NAME_SEPARATOR + 
            ObjectUtils.getIdentityHexString(definition);
    } else {
        // Top-level bean: use plain class name.
        // Increase counter until the id is unique.
        int counter = -1;
         //用类名#自增的数字命名
        while (counter == -1 || registry.containsBeanDefinition(id)) {
            counter++;
            id = generatedBeanName + GENERATED_BEAN_NAME_SEPARATOR + counter;
        }
    }
    return id;
}
```

bean解析：分部分说明parseBeanDefinitionElement（Element ele, String beanName, BeanDefinition containingBean）。

首先获取到bean的class属性和parent属性，配置了parent之后，当前bean会继承父bean的属性。之后根据class和parent创建BeanDefinition对象。

```java
String className = null;
if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
    className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
}
String parent = null;
if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
    parent = ele.getAttribute(PARENT_ATTRIBUTE);
}
//根据class和parent创建BeanDefinition对象。
AbstractBeanDefinition bd = createBeanDefinition(className, parent);
```

BeanDefinition的创建在BeanDefinitionReaderUtils.createBeanDefinition:

```java
public static AbstractBeanDefinition createBeanDefinition(
        String parentName, String className, ClassLoader classLoader) {
    GenericBeanDefinition bd = new GenericBeanDefinition();
    bd.setParentName(parentName);
    if (className != null) {
        if (classLoader != null) {
            bd.setBeanClass(ClassUtils.forName(className, classLoader));
        }
        else {
            bd.setBeanClassName(className);
        }
    }
    return bd;
}
```

之后是解析bean的其它属性，其实就是读取其配置，调用相应的setter方法保存在BeanDefinition中:

```java
parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
```

之后解析bean的decription子元素:

```xml
<bean id="b" name="one, two" class="base.SimpleBean">
    <description>SimpleBean</description>
</bean>
```

就仅仅是个描述。

然后是meta子元素的解析，meta元素在xml配置文件里是这样的:

```xml
<bean id="b" name="one, two" class="base.SimpleBean">
    <meta key="name" value="skywalker"/>
</bean>
```

注释上说，这样可以将任意的元数据附到对应的bean definition上。解析过程源码:

```java
public void parseMetaElements(Element ele, BeanMetadataAttributeAccessor attributeAccessor) {
        NodeList nl = ele.getChildNodes();

        for(int i = 0; i < nl.getLength(); ++i) {
            Node node = nl.item(i);
            if (this.isCandidateElement(node) && this.nodeNameEquals(node, "meta")) {
                Element metaElement = (Element)node;
                String key = metaElement.getAttribute("key");
                String value = metaElement.getAttribute("value");
                //就是一个key, value的载体，无他
                BeanMetadataAttribute attribute = new BeanMetadataAttribute(key, value);
                attribute.setSource(this.extractSource(metaElement));//sourceExtractor默认是NullSourceExtractor，返回的是空
                attributeAccessor.addMetadataAttribute(attribute);
            }
        }

    }
```

AbstractBeanDefinition继承自BeanMetadataAttributeAccessor类，底层使用了一个LinkedHashMap保存metadata。

lookup-method解析：

此标签的作用在于当一个bean的某个方法被设置为lookup-method后，**每次调用此方法时，都会返回一个新的指定bean的对象**。用法示例:

```xml
<bean id="apple" class="cn.com.willchen.test.di.Apple" scope="prototype"/>
<!--水果盘-->
<bean id="fruitPlate" class="cn.com.willchen.test.di.FruitPlate">
    <lookup-method name="getFruit" bean="apple"/>
</bean>
```

数据保存在Set中，对应的类是MethodOverrides

replace-mothod解析:

此标签用于替换bean里面的特定的方法实现，替换者必须实现Spring的MethodReplacer接口，有点像aop的意思。

配置文件示例:

```xml
<bean name="replacer" class="springroad.deomo.chap4.MethodReplace" />  
<bean name="testBean" class="springroad.deomo.chap4.LookupMethodBean">
    <replaced-method name="test" replacer="replacer">
        <arg-type match="String" />
    </replaced-method>  
</bean> 
```

构造参数(constructor-arg)解析:

作用一目了然，使用示例:

```xml
<bean class="base.SimpleBean">
    <constructor-arg>
        <value type="java.lang.String">Cat</value>
    </constructor-arg>
</bean>
```

type一般不需要指定，除了泛型集合那种。除此之外，constructor-arg还支持name, index, ref等属性，可以具体的指定参数的位置等。构造参数解析后保存在BeanDefinition内部一个ConstructorArgumentValues对象中。如果设置了index属性，那么以Map<Integer, ValueHolder>的形式保存，反之，以List的形式保存。

property解析:

非常常用的标签，用以为bean的属性赋值，支持value和ref两种形式，示例:

```xml
<bean class="base.SimpleBean">
    <property name="name" value="skywalker" />
</bean>
```

value和ref属性不能同时出现，如果是ref，那么将其值保存在不可变的RuntimeBeanReference对象中，其实现了BeanReference接口，此接口只有一个getBeanName方法。如果是value，那么将其值保存在TypedStringValue对象中。最终将对象保存在BeanDefinition内部一个MutablePropertyValues对象中(内部以ArrayList实现)。

###### Bean注册

BeanDefinitionReaderUtils.registerBeanDefinition:

```java
public static void registerBeanDefinition(
    BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry) {
    // Register bean definition under primary name.
    String beanName = definitionHolder.getBeanName();
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());
    // Register aliases for bean name, if any.
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            registry.registerAlias(beanName, alias);
        }
    }
}
```

registry其实就是DefaultListableBeanFactory对象，registerBeanDefinition方法主要就干了这么两件事:

```java
@Override
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition) {
    this.beanDefinitionMap.put(beanName, beanDefinition);
    this.beanDefinitionNames.add(beanName);
}
```

可以发现，一个是map，一个是List，registerAlias方法的实现在其父类SimpleAliasRegistry，就是把键值对放在了一个ConcurrentHashMap里。

ComponentRegistered事件触发:

默认是个空实现，前面说过了。

##### beBeanDefiniton的数据结构

![images\BeanDefinition](images\BeanDefinition.jpg)



4. beans:beans元素的嵌套直接递归调用DefaultBeanDefinitionDocumentReader.parseBeanDefinitions。

#### 其他命名空间的解析：

入口在DefaultBeanDefinitionDocumentReader.parseBeanDefinitions->BeanDefinitionParserDelegate.parseCustomElement(第二个参数为空):

```java
public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
        String namespaceUri = this.getNamespaceURI(ele);
        NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
        if (handler == null) {
            this.error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
            return null;
        } else {
            return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
        }
    }
```

NamespaceHandlerResolver由XmlBeanDefinitionReader初始化，是一个DefaultNamespaceHandlerResolver对象，也是NamespaceHandlerResolver接口的唯一实现。

其resolve方法:

```java
@Override
public NamespaceHandler resolve(String namespaceUri) {
    Map<String, Object> handlerMappings = getHandlerMappings();
    Object handlerOrClassName = handlerMappings.get(namespaceUri);
    if (handlerOrClassName == null) {
        return null;
    } else if (handlerOrClassName instanceof NamespaceHandler) {
        return (NamespaceHandler) handlerOrClassName;
    } else {
        String className = (String) handlerOrClassName;
        Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
        NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
        namespaceHandler.init();
        handlerMappings.put(namespaceUri, namespaceHandler);
        return namespaceHandler;
    }
}
```

容易看出，Spring其实使用了一个Map了保存其映射关系，key就是命名空间的uri，value是**NamespaceHandler对象或是Class完整名，如果发现是类名，那么用反射的方法进行初始化，如果是NamespaceHandler对象，那么直接返回**。

NamespaceHandler映射关系来自于各个Spring jar包下的META-INF/spring.handlers文件，以spring-context包为例:

```html
http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler
http\://www.springframework.org/schema/jee=org.springframework.ejb.config.JeeNamespaceHandler
http\://www.springframework.org/schema/lang=org.springframework.scripting.config.LangNamespaceHandler
http\://www.springframework.org/schema/task=org.springframework.scheduling.config.TaskNamespaceHandler
http\://www.springframework.org/schema/cache=org.springframework.cache.config.CacheNamespaceHandler
```

##### NamespaceHandler继承体系

[![NamespaceHandler继承体系](https://github.com/seaswalker/spring-analysis/raw/master/note/images/NamespaceHandler.jpg)](https://github.com/seaswalker/spring-analysis/blob/master/note/images/NamespaceHandler.jpg)

##### init

resolve中调用了其init方法，此方法用以向NamespaceHandler对象注册BeanDefinitionParser对象。**此接口用以解析顶层(beans下)的非默认命名空间元素，比如`<context:annotation-config />`**。

所以这样逻辑就很容易理解了:  **每种子标签的解析仍是策略模式的体现，init负责向父类NamespaceHandlerSupport注册不同的策略，由父类的NamespaceHandlerSupport.parse方法根据具体的子标签调用相应的策略完成解析的过程**。



此部分比较重要。下面是BeanFactory的结构。

![D:\gitww\personalMarkDown\Spring学习\images\Beanfactory_structure](D:\gitww\personalMarkDown\Spring学习\images\Beanfactory_structure.jpg)

### 小结（2）BeanFactory的创建

由obtainFreshBeanFactory调用AbstractRefreshableApplicationContext.refreshBeanFactory。

如果有BeanFactory，那么先销毁，然后1.创建一个创建了一个DefaultListableBeanFactory对象，即ioc容器，其内部保存了三个数据结构，如上图所示，beandefinationMap，aliasMap，beanDefinitionnames数组集合。

2.定制化BeanFactory，默认是不允许bean覆盖与循环依赖；3.然后是在容器中注册bean。



    bean的注册进入ioc容器中就变为了beanDefinition，这也是ioc容器中的元数据。

    beanDefinition包含了作用域，父bean的名字，bean class对象或名称等等相关的信息，可以将bean看做为对应的beanDefinition的一个字段。所以，接下来就研究如何向容器中注册beanDefinition对象。 （当然，解析符合spring的xml文件格式，使用了EntityResolver，EntityResolver接口在org.xml.sax中定义。DelegatingEntityResolver用于schema和dtd的解析。）



     注册beanDefinition对象这个过程利用了BeanDefinitionReader类，首先是路径解析（支持ant风格），首先，获取ClassPathXmlApplicationContext构造函数中的

**this.setConfigLocations(configLocations);**中已经解析好对应配置文件前置路径（比如classpath:）的resource属性， 找到对应的配置文件路径信息，并保存（使用ThreadLocal保存）。

    然后进行配置文件的加载，入口方法在AbstractBeanDefinitionReader中，先加载。然后进行解析loadBeanDefinitions(resources);，利用了EncodedResource进行加载，EncodedResource扮演的其实是一个装饰器的模式，为InputStreamSource添加了字符编码(虽然默认为null)。这样为我们自定义xml配置文件的编码方式提供了机会。具体过程是通过input流读取对应的文件，然后传入inputSource中，然后进行解析，xml文件的解析使用的是DefaultDocumentLoader，利用文档模型解析xml文件结构，包括约束校验与数据校验。

    接着就是bean的解析，XmlBeanDefinitionReader.registerBeanDefinitions:注册BeanDefinitions。利用了反射，创建对应的BeanDefinitions对象并解析具体的标签

为该对象设置属性，在这个过程中，使用了XmlReaderContext将参数糅合到一起传入给容器，使用doRegisterBeanDefinitions: 解析xml中具体的标签。 使用了delegate处理嵌套标签，可见，对于非默认命名空间的元素交由delegate处理。

    最后是空间默认标签的解析（也就是doRegisterBeanDefinitions要干的事）。

有import, alias, bean, 嵌套的beans四种元素。 其中解析bean标签是重点（重点理解）。

     到这里，bean对应的BeanDefinitions就创建并设置完成了，然后注册到BeanFactory中，注册的时候干了两件事情，name名加入到list中，alias别名加入到map中，到这BeanFactory的创建就完成了。

> 简单总结：创建BeanFactory实例，加载并解析对应的配置文件，根据标签生成BeanDefinitions实例并设置属性，然后将BeanDefinitions注册进BeanFactory中。


