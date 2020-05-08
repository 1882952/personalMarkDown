继续上篇中的refresh方法的分析。

## （3）prepareBeanFactory

此方法对beanFactory进行一些特征的设置工作，"特征"包含以下几个方面：

首先设置bean的类加载器。

### BeanExpressionResolver

此接口只有一个实现: StandardBeanExpressionResolver。接口只含有一个方法:

```java
Object evaluate(String value, BeanExpressionContext evalContext)
```

prepareBeanFactory将一个此对象放入BeanFactory:

```java
 beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
```

StandardBeanExpressionResolver对象内部有一个关键的成员: SpelExpressionParser,其整个类图:

![images\ExpressionParser](images\ExpressionParser.jpg)

这便是Spring3.0开始出现的Spel表达式的解释器。 也就是说设置BeanExpressionResolver是为了装载spel表达式的解释器。

### PropertyEditorRegistrar

此接口用于向Spring注册java.beans.PropertyEditor，只有一个方法:

```java
registerCustomEditors(PropertyEditorRegistry registry)
```

实现也只有一个: ResourceEditorRegistrar。

在编写xml配置时，我们设置的值都是字符串形式，所以在使用时肯定需要转为我们需要的类型，PropertyEditor接口正是定义了这么个东西。

```java
  beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, this.getEnvironment()));
```

BeanFactory也暴露了registerCustomEditors方法用以添加自定义的转换器，所以这个地方是组合模式的体现。

我们有两种方式可以添加自定义PropertyEditor:

- 通过`context.getBeanFactory().registerCustomEditor`

- 通过Spring配置文件:
  
  ```xml
  <bean class="org.springframework.beans.factory.config.CustomEditorConfigurer">
    <property name="customEditors">
            <map>
                <entry key="base.Cat" value="base.CatEditor" /> 
        </map>
    </property>
  </bean>
  ```

参考: [深入理解JavaBean(2)：属性编辑器PropertyEditor](http://blog.csdn.net/zhoudaxia/article/details/36247883)

也就是为BeanFactory添加配置文件内容的转换器（比如将xml的字符串形式转换为需要的类型）。

### 环境注入

在Spring中我们自己的bean可以通过实现EnvironmentAware等一系列Aware接口获取到Spring内部的一些对象。 

```java
 beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
```

ApplicationContextAwareProcessor核心的invokeAwareInterfaces方法:

```java
 private void invokeAwareInterfaces(Object bean) {
        if (bean instanceof Aware) {
            if (bean instanceof EnvironmentAware) {
                ((EnvironmentAware)bean).setEnvironment(this.applicationContext.getEnvironment());
            }
            if (bean instanceof EmbeddedValueResolverAware) {                ((EmbeddedValueResolverAware)bean).setEmbeddedValueResolver(this.embeddedValueResolver);

            }
    //*********************
    }
```

这一步也就是让实现aware接口对应的类中获取到spring内部的一些对象。

### 依赖解析忽略

此部分设置哪些接口在进行依赖注入的时候应该被忽略:

```java
beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
```

### bean的伪装

有些对象并不在BeanFactory中，但是依然想让其可以被装配，这个时候就需要伪装一下。

```java
beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
beanFactory.registerResolvableDependency(ResourceLoader.class, this);
beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
beanFactory.registerResolvableDependency(ApplicationContext.class, this);
```

伪装关系保存在一个Map<Class<?>, Object>里。



```java
//还有一步，这一步是添加应用的监听设置
  beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
```

### LoadTimeWeaver

如果配置了此bean，那么：

```java
if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
    beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
    // Set a temporary ClassLoader for type matching.
    beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
}
```

这个东西具体是干什么的在后面context:load-time-weaver中说明。（猜测是动态织入）。

### 注册环境：

注册具体的环境（prototypesource中利用并发ArrayList保存的环境）

源码:

```java
if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
    beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
}
if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
    beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
}
if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
    beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().
        getSystemEnvironment());
}
```

containsLocalBean特殊之处在于不会去父BeanFactory寻找。

到这，对prepareBeanFactory（）：对BeanFactory进行一些特征设置就分析完了，

小结一下过程。

### 小结：

1. 设置BeanExpressionResolver，目的是为了装载spel表达式。

2. 设置PropertyEditorRegistrar，目的是为了装载配置文件中内容的转换器（比如将xml的字符串形式转换为需要的类型）。

3. 环境注入，在Spring中我们自己的bean可以通过实现EnvironmentAware等一系列Aware接口获取到Spring内部的一些对象。(即与aware接口对应的对象相关)

4. 依赖解析的忽略，就是将aware接口获取到的对应的spring内部对象的依赖关系忽略掉。

5. bean伪装，让不在BeanFactory中的对象也可以被装配，伪装关系保存在一个Map<Class<?>, Object>里。

6. LoadTimeWeaver：猜测是与动态织入相关。

7. 注册应用具体环境，即获取到prototypesource中利用并发ArrayList保存的环境信息，之前已经分析过了。

## （4）postProcessBeanFactory

this.postProcessBeanFactory(beanFactory);

此方法允许子类在所有的bean尚未初始化之前注册BeanPostProcessor。空实现且没有子类覆盖。

但是看其子类重写的该方法，比如web相关类，添加了servlet相关的上下文信息。

## （5）invokeBeanFactoryPostProcessors

BeanFactoryPostProcessor接口允许我们在bean正是初始化之前改变其值。这也是扩展spring容器的三种方式之一，此接口只有一个方法:

```java
void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory);
```

有两种方式可以向Spring添加此对象:

- 通过代码的方式:
  
  ```java
  context.addBeanFactoryPostProcessor
  ```

- 通过xml配置的方式:
  
  ```xml
  <bean class="base.SimpleBeanFactoryPostProcessor" />
  ```

注意此时尚未进行bean的初始化工作，初始化是在后面的finishBeanFactoryInitialization进行的，所以在BeanFactoryPostProcessor对象中获取bean会导致提前初始化。

此方法的关键源码:

```java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory,
        getBeanFactoryPostProcessors());
}
```

getBeanFactoryPostProcessors获取的就是AbstractApplicationContext的成员beanFactoryPostProcessors(ArrayList)，但是很有意思，**只有通过context.addBeanFactoryPostProcessor这种方式添加的才会出现在这个List里，所以对于xml配置方式，此List其实没有任何元素。玄机就在PostProcessorRegistrationDelegate里**。

核心思想就是使用BeanFactory的getBeanNamesForType方法获取相应的BeanDefinition的name数组，之后逐一调用getBean方法获取到bean(初始化)，getBean方法后面再说。

注意此处有一个优先级的概念，如果你的BeanFactoryPostProcessor同时实现了Ordered或者是PriorityOrdered接口，那么会被首先执行。

### 小结：

使用扩展spring容器的三种方式之一的BeanFactoryPostProcessor接口在bean初始化之前改变其值，这也会导致该bean的初始化。 对于BeanFactoryPostProcessor的获取就是AbstractApplicationContext的成员beanFactoryPostProcessors(ArrayList)，

```java
 private final List<BeanFactoryPostProcessor> beanFactoryPostProcessors;
```

但是很有意思，**只有通过context.addBeanFactoryPostProcessor这种方式添加的才会出现在这个List里，所以对于xml配置方式，此List其实没有任何元素。玄机就在PostProcessorRegistrationDelegate里**。

核心思想就是使用BeanFactory的getBeanNamesForType方法获取相应的BeanDefinition的name数组，之后逐一调用getBean方法获取到bean(初始化)。

## (6)registerBeanPostProcessors

此部分实质上是在BeanDefinitions中寻找BeanPostProcessor，之后调用BeanFactory.addBeanPostProcessor方法保存在一个List中，注意添加时仍然有优先级的概念，优先级高的在前面。

## (7)initMessageSource

此接口用以支持Spring国际化。继承体系如下:

![images\MessageSource](images\MessageSource.jpg)

AbstractApplicationContext的initMessageSource()方法就是在BeanFactory中查找MessageSource的bean，如果配置了此bean，那么调用getBean方法完成其初始化并将其保存在AbstractApplicationContext内部messageSource成员变量中，用以处理ApplicationContext的getMessage调用，因为从继承体系上来看，ApplicationContext是MessageSource的子类，此处是委托模式的体现。如果没有配置此bean，那么初始化一个DelegatingMessageSource对象，此类是一个空实现，同样用以处理getMessage调用请求。

## （8）事件驱动

此接口代表了Spring的事件驱动(监听器)模式。一个事件驱动包含三部分:

#### 事件

java的所有事件对象一般都是java.util.EventObject的子类，Spring的整个继承体系如下:

[![EventObject继承体系](https://github.com/seaswalker/spring-analysis/raw/master/note/images/EventObject.jpg)](https://github.com/seaswalker/spring-analysis/blob/master/note/images/EventObject.jpg)

#### 发布者

##### ApplicationEventPublisher

[![ApplicationEventPublisher继承体系](https://github.com/seaswalker/spring-analysis/raw/master/note/images/ApplicationEventPublisher.jpg)](https://github.com/seaswalker/spring-analysis/blob/master/note/images/ApplicationEventPublisher.jpg)

一目了然。

##### ApplicationEventMulticaster

ApplicationEventPublisher实际上正是将请求委托给ApplicationEventMulticaster来实现的。其继承体系:

[![ApplicationEventMulticaster继承体系](https://github.com/seaswalker/spring-analysis/raw/master/note/images/ApplicationEventMulticaster.jpg)](https://github.com/seaswalker/spring-analysis/blob/master/note/images/ApplicationEventMulticaster.jpg)

#### 监听器

所有的监听器是jdk EventListener的子类，这是一个mark接口。继承体系:

[![EventListener继承体系](https://github.com/seaswalker/spring-analysis/raw/master/note/images/EventListener.jpg)](https://github.com/seaswalker/spring-analysis/blob/master/note/images/EventListener.jpg)

可以看出SmartApplicationListener和GenericApplicationListener是高度相似的，都提供了事件类型检测和顺序机制，而后者是从Spring4.2加入的，Spring官方文档推荐使用后者代替前者。

#### 初始化

前面说过ApplicationEventPublisher是通过委托给ApplicationEventMulticaster实现的，所以refresh方法中完成的是对ApplicationEventMulticaster的初始化:

```java
// Initialize event multicaster for this context.
initApplicationEventMulticaster();
```

initApplicationEventMulticaster则首先在BeanFactory中寻找ApplicationEventMulticaster的bean，如果找到，那么调用getBean方法将其初始化，如果找不到那么使用SimpleApplicationEventMulticaster。

#### 事件发布

AbstractApplicationContext.publishEvent核心代码:

```java
protected void publishEvent(Object event, ResolvableType eventType) {
    getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
}
```

SimpleApplicationEventMulticaster.multicastEvent:

```java
//执行事件，利用了Excutor框架执行事件。
@Override
public void multicastEvent(final ApplicationEvent event, ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        Executor executor = getTaskExecutor();
        if (executor != null) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    invokeListener(listener, event);
                }
            });
        } else {
            invokeListener(listener, event);
        }
    }
}
```

##### 监听器获取

获取当然还是通过beanFactory的getBean来完成的，值得注意的是Spring在此处使用了缓存(ConcurrentHashMap)来加速查找的过程。

##### 同步/异步

可以看出，如果executor不为空，那么监听器的执行实际上是异步的。那么如何配置同步/异步呢?

全局

```xml
<task:executor id="multicasterExecutor" pool-size="3"/>
<bean class="org.springframework.context.event.SimpleApplicationEventMulticaster">
    <property name="taskExecutor" ref="multicasterExecutor"></property>
</bean>
```

task schema是Spring从3.0开始加入的，使我们可以不再依赖于Quartz实现定时任务，源码在org.springframework.core.task包下，使用需要引入schema：

```xml
xmlns:task="http://www.springframework.org/schema/task"
xsi:schemaLocation="http://www.springframework.org/schema/task http://www.springframework.org/schema/task/spring-task-4.0.xsd"
```

可以参考:  [Spring定时任务的几种实现](http://gong1208.iteye.com/blog/1773177)

###### 注解

开启注解支持:

```xml
<!-- 开启@AspectJ AOP代理 -->  
<aop:aspectj-autoproxy proxy-target-class="true"/>  
<!-- 任务调度器 -->  
<task:scheduler id="scheduler" pool-size="10"/>  
<!-- 任务执行器 -->  
<task:executor id="executor" pool-size="10"/>  
<!--开启注解调度支持 @Async @Scheduled-->  
<task:annotation-driven executor="executor" scheduler="scheduler" proxy-target-class="true"/>  
```

在代码中使用示例:

```java
@Component  
public class EmailRegisterListener implements ApplicationListener<RegisterEvent> {  
    @Async  
    @Override  
    public void onApplicationEvent(final RegisterEvent event) {  
        System.out.println("注册成功，发送确认邮件给：" + ((User)event.getSource()).getUsername());  
    }  
}  
```

参考:  [详解Spring事件驱动模型](http://jinnianshilongnian.iteye.com/blog/1902886)

### 小结：

此方法initApplicationEventMulticaster（）是初始化了spring事件驱动（事件监听模式）。 在spring中，事件驱动分为三个模块；

1. 事件：EventObject的子类对象。

2. 发布者：ApplicationEventPublisher，此类将事件包装并且发布，实际上是将事件委托给ApplicationEventMulticaster执行。

3. 监听器：所有的监听器是jdk EventListener的子类，这是一个mark接口。spring也是利用事件监听机制执行事件。

初始化是利用了initApplicationEventMulticaster方法，首先在BeanFactory中寻找ApplicationEventMulticaster的bean，如果找到，那么调用getBean方法将其初始化，如果找不到那么使用SimpleApplicationEventMulticaster。

事件发布是将事件委托给了ApplicationEventMulticaster，而该类的是将事件包装为任务，然后交由Excutor框架执行，具体执行过程是调用对应的事件监听器，传入事件。（单线程事件的机制：回调）。

对于监听器的获取还是利用了beanFactory的getBean来完成的。

如果executor不为空，那么监听器的执行实际上是异步的。可以通过注解开启异步任务调度的执行。

## （9）this.onRefresh();

这又是一个模版方法，允许子类在进行bean初始化之前进行一些定制操作。默认空实现。

## （10） ApplicationListener的注册

即应用相关监听器的注册，registerListeners（）方法，还是通过BeanFactory调用getBeanNamesForType方法，获取到对应的事件监听对象，然后与相应的事件执行对象ApplicationEventMulticaster联系起来。

## （11）singleton初始化

this.finishBeanFactoryInitialization(beanFactory);到这，才是对应的单例bean的初始化。 这一部分比较重要。

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
        if (beanFactory.containsBean("conversionService") && beanFactory.isTypeMatch("conversionService", ConversionService.class)) {
            beanFactory.setConversionService((ConversionService)beanFactory.getBean("conversionService", ConversionService.class));
        }

        if (!beanFactory.hasEmbeddedValueResolver()) {
            beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
                public String resolveStringValue(String strVal) {
                    return AbstractApplicationContext.this.getEnvironment().resolvePlaceholders(strVal);
                }
            });
        }

        String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
        String[] var3 = weaverAwareNames;
        int var4 = weaverAwareNames.length;

        for(int var5 = 0; var5 < var4; ++var5) {
            String weaverAwareName = var3[var5];
            this.getBean(weaverAwareName);
        }

        beanFactory.setTempClassLoader((ClassLoader)null);
        beanFactory.freezeConfiguration();
        beanFactory.preInstantiateSingletons();
    }
```

分部分说明：

### ConversionService

此接口用于类型之间的转换，在Spring里其实就是把配置文件中的String转为其它类型，从3.0开始出现，目的和jdk的PropertyEditor接口是一样的，参考ConfigurableBeanFactory.setConversionService注释:

> > Specify a Spring 3.0 ConversionService to use for converting property values, as an alternative to JavaBeans PropertyEditors. @since 3.0

### StringValueResolver

用于解析注解的值。接口只定义了一个方法:

```java
String resolveStringValue(String strVal);
```

### LoadTimeWeaverAware

实现了此接口的bean可以得到LoadTimeWeaver，此处仅仅初始化。

### **单例的初始化**

DefaultListableBeanFactory.preInstantiateSingletons:

```java
@Override
public void preInstantiateSingletons() throws BeansException {
    List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX 
                    + beanName);
                boolean isEagerInit;
                if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                    isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
                        @Override
                        public Boolean run() {
                            return ((SmartFactoryBean<?>) factory).isEagerInit();
                        }
                    }, getAccessControlContext());
                }
                else {
                    isEagerInit = (factory instanceof SmartFactoryBean &&
                            ((SmartFactoryBean<?>) factory).isEagerInit());
                }
                if (isEagerInit) {
                    getBean(beanName);
                }
            }
            else {
                getBean(beanName);
            }
        }
    }

    // Trigger post-initialization callback for all applicable beans...
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            final SmartInitializingSingleton smartSingleton = 
                (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged(new PrivilegedAction<Object>() {
                    @Override
                    public Object run() {
                        smartSingleton.afterSingletonsInstantiated();
                        return null;
                    }
                }, getAccessControlContext());
            }
            else {
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}
```

这个方法时初始化单例对象的核心，所以逐步分析。

首先进行Singleton的初始化，其中如果bean是FactoryBean类型(注意，只定义了factory-method属性的普通bean并不是FactoryBean)，并且还是SmartFactoryBean类型，那么需要判断是否需要eagerInit(isEagerInit是此接口定义的方法)。

#### <1>getBean

这里便是bean初始化的核心逻辑。源码比较复杂，分开说。以getBean(String name)为例。AbstractBeanFactory.getBean:

```java
public Object getBean(String name) throws BeansException {
        return this.doGetBean(name, (Class)null, (Object[])null, false);
    }
```

第二个参数表示bean的Class类型，第三个表示创建bean需要的参数，最后一个表示不需要进行类型检查。

##### <2>beanName的转化

```java
final String beanName = transformedBeanName(name);
```

这里是将FactoryBean的前缀去掉以及将别名转为真实的名字。

##### <3> 手动注册单例Bean的检查

前面注册环境一节说过，Spring其实手动注册了一些单例bean。这一步就是检测是不是这些bean。如果是，那么再检测是不是工厂bean，如果是返回其工厂方法返回的实例，如果不是返回bean本身。

```java
Object sharedInstance = getSingleton(beanName);
if (sharedInstance != null && args == null) {
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
}
```

##### <4>检查父容器

如果父容器存在并且存在此bean定义，那么交由其父容器初始化:

```java
BeanFactory parentBeanFactory = getParentBeanFactory();
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
    // Not found -> check parent.
    //此方法其实是做了前面beanName转化的逆操作，因为父容器同样会进行转化操作
    String nameToLookup = originalBeanName(name);
    if (args != null) {
        // Delegation to parent with explicit args.
        return (T) parentBeanFactory.getBean(nameToLookup, args);
    } else {
        // No args -> delegate to standard getBean method.
        return parentBeanFactory.getBean(nameToLookup, requiredType);
    }
}
```

##### <5>依赖的初始化

bean可以由depends-on属性配置依赖的bean。Spring会首先初始化依赖的bean。

```java
String[] dependsOn = mbd.getDependsOn();
if (dependsOn != null) {
    for (String dependsOnBean : dependsOn) {
         //检测是否存在循环依赖
        if (isDependent(beanName, dependsOnBean)) {
            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
            "Circular depends-on relationship between '" + beanName + "' and '" + dependsOnBean + "'");
        }
        registerDependentBean(dependsOnBean, beanName);
        getBean(dependsOnBean);
    }
}
```

registerDependentBean进行了依赖关系的注册，这么做的原因是Spring在即进行bean销毁的时候会首先销毁被依赖的bean。依赖关系的保存是通过一个ConcurrentHashMap<String, Set>完成的，key是bean的真实名字。

##### <6>singleton初始化

虽然这里大纲是Singleton初始化，但是getBean方法本身是包括所有scope的初始化，在这里一次说明了。

```java
if (mbd.isSingleton()) {
    sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
        @Override
        public Object getObject() throws BeansException {
            return createBean(beanName, mbd, args);
        }
    });
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```

###### [1]getSingleton

首先会检查是否存在，如果存在，直接返回:

```java
synchronized (this.singletonObjects) {
    Object singletonObject = this.singletonObjects.get(beanName);
}
```

所有的单例bean都保存在这样的数据结构中:  `ConcurrentHashMap<String, Object>`。

#### bean创建

源码位于AbstractAutowireCapableBeanFactory.createBean，主要分为几个部分:

##### lookup-method检测

此部分用于检测lookup-method标签配置的方法是否存在:

```java
RootBeanDefinition mbdToUse = mbd;
mbdToUse.prepareMethodOverrides();
```

prepareMethodOverrides:

```java
public void prepareMethodOverrides() throws BeanDefinitionValidationException {
    // Check that lookup methods exists.
    MethodOverrides methodOverrides =
                prepareMethodOverride(mo);
            }
        }
    }
}
```

prepareMethodOverride:

```java
protected void prepareMethodOverride(MethodOverride mo)  {
    int count = ClassUtils.getMethodCountForName(getBeanClass(), mo.getMethodName());
    if (count == 0) {
        throw new BeanDefinitionValidationException(
                "Invalid method override: no method with name '" + mo.
                  );
    } else if (count == 1) {
        // Mark override as not overloaded, to avoid the overhead of arg type checking.
        mo.setOverloaded(false);
    }
}
```

 InstantiationAwareBeanPostProcessor触发

在这里触发的是其postProcessBeforeInitialization和postProcessAfterInstantiation方法。

```java
Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
if (bean != null) {
    return bean;
}
Object beanInstance = doCreateBean(beanName, mbdToUse, args);
return beanInstance;
```

继续:

```java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
    Object bean = null;
    if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
        // Make sure bean class is actually resolved at this point.
        if (!mbd determineTargetType(beanName, mbd);
            if 
                bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
                if 
                    bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
                }
            }
        }
        mbd.beforeInstantiationResolved = (bean != null);
    }
    return bean;
}
```

从这里可以看出，**如果InstantiationAwareBeanPostProcessor返回的不是空，那么将不会继续执行剩下的Spring初始化流程，此接口用于初始化自定义的bean，主要是在Spring内部使用**。

###### doCreateBean

真正的创建bean对象的方法。

同样分为几部分

###### 创建(createBeanInstance)

关键代码:

```java
BeanWrapper instanceWrapper = null;
if (instanceWrapper == null) {
    instanceWrapper = createBeanInstance(beanName, mbd, args);
}
```

createBeanInstance的创建过程又分为以下几种情况:

- 工厂bean

- 调用instantiateUsingFactoryMethod方法:
  
  ```java
  protected BeanWrapper instantiateUsingFactoryMethod(
    String beanName, RootBeanDefinition mbd, Object[] explicitArgs) {
    return new ConstructorResolver(this).instantiateUsingFactoryMethod(beanName, mbd, explicitArgs);
  }
  ```
  
  注意，此处的工厂bean指的是配置了factory-bean/factory-method属性的bean，不是实现了FacrotyBean接口的bean。如果没有配置factory-bean属性，那么factory-method指向的方法必须是静态的。此方法主要做了这么几件事:
  
  - 初始化一个BeanWrapperImpl对象。
  
  - 根据设置的参数列表使用反射的方法寻找相应的方法对象。
  
  - InstantiationStrategy:
  
  bean的初始化在此处又抽成了策略模式，类图:
  
  [![InstantiationStrategy类图](https://github.com/seaswalker/spring-analysis/raw/master/note/images/InstantiationStrategy.jpg)](https://github.com/seaswalker/spring-analysis/blob/master/note/images/InstantiationStrategy.jpg)
  
  instantiateUsingFactoryMethod部分源码:
  
  ```java
  beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(
      mbd, beanName, this.beanFactory, factoryBean, factoryMethodToUse, argsToUse);
  ```
  
  getInstantiationStrategy返回的是CglibSubclassingInstantiationStrategy对象。此处instantiate实现也很简单，就是调用工厂方法的Method对象反射调用其invoke即可得到对象，SimpleInstantiationStrategy.
  
  instantiate核心源码:
  
  ```java
  @Override
  public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner,
      Object factoryBean, final Method factoryMethod, Object... args) {
      return factoryMethod
  }
  ```

构造器自动装配

createBeanInstance部分源码:

```java
// Need to determine the constructor...
Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
if (ctors != null ||
  mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
    //配置了<constructor-arg>子元素
  mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
  return
}
```

determineConstructorsFromBeanPostProcessors源码:

```java
protected Constructor<?>[] determineConstructorsFromBeanPostProcessors(Class<?> beanClass, String beanName) {
  if (beanClass != null && hasInstantiationAwareBeanPostProcessors()) {
      for (BeanPostProcessor bp : getBeanPostProcessors()) {
          if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
              SmartInstantiationAwareBeanPostProcessor ibp = 
                  (SmartInstantiationAwareBeanPostProcessorConstructor<?>[] ctors = ibp.determineCandidateConstructors(beanClass, beanName);
              if (ctors != null) {
                  return ctors;
              }
          }
      }
  }
  return null;
}
```

可见是由SmartInstantiationAwareBeanPostProcessor决定的，默认是没有配置这种东西的。

之后就是判断bean的自动装配模式，可以通过如下方式配置:

```xml
<bean id="student" class="base.Student" primary="true" autowire="default" />
```

- autowire共有以下几种选项:
  
  - no: 默认的，不进行自动装配。在这种情况下，只能通过ref方式引用其它bean。
  - byName: 根据bean里面属性的名字在BeanFactory中进行查找并装配。
  - byType: 按类型。
  - constructor: 以byType的方式查找bean的构造参数列表。
  - default: 由父bean决定。
  
  参考:  [Spring - bean的autowire属性(自动装配)](http://www.cnblogs.com/ViviChan/p/4981539.html)
  
  autowireConstructor调用的是ConstructorResolver.autowireConstructor，此方法主要做了两件事:
  
  - 得到合适的构造器对象。
  
  - 根据构造器参数的类型去BeanFactory查找相应的bean:
    
    入口方法在ConstructorResolver.resolveAutowiredArgument:
    
    ```java
    protected Object resolveAutowiredArgument(
            MethodParameter param, String beanName, Set<String> autowiredBeanNames, 
            TypeConverter typeConverter) {
        return this.beanFactory.resolveDependency(
                new DependencyDescriptor(param, true), beanName, 
                autowiredBeanNames, typeConverter);
    }
    ```

    最终调用的还是CglibSubclassingInstantiationStrategy.instantiate方法，关键源码:
    
    ```java
    @Override
    public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner,
          final Constructor<?> ctor, Object... args) {
      if (bd.getMethodOverrides().isEmpty()) {
               //反射调用
          return BeanUtils.instantiateClass(ctor, args);
      } else {
          return instantiateWithMethodInjection(bd, beanName, owner, ctor, args);
      }
    }
    ```
    
    可以看出，如果配置了lookup-method标签，**得到的实际上是用Cglib生成的目标类的代理子类**。
    
    CglibSubclassingInstantiationStrategy.instantiateWithMethodInjection:
    
    ```java
    @Override
    protected Object instantiateWithMethodInjection(RootBeanDefinition bd, String beanName, BeanFactory     owner,Constructor<?> ctor, Object... args) {
      // Must generate CGLIB subclass...
      return new CglibSubclassCreator(bd, owner).instantiate(ctor, args);
    }
    ```

- 默认构造器
  
  一行代码，很简单:
  
  ```java
  // No special handling: simply use no-arg constructor.
  return instantiateBean(beanName, mbd);
  ```

###### MergedBeanDefinitionPostProcessor

触发源码:

```java
synchronized (mbd.postProcessingLock) {
    if (!mbd.postProcessed) {
        applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
        mbd.postProcessed = true;
    }
}
```

此接口也是Spring内部使用的，不管它了。

###### 属性解析

入口方法: AbstractAutowireCapableBeanFactory.populateBean，它的作用是: 根据autowire类型进行autowire by name，by type 或者是直接进行设置，简略后的源码:

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    //所有<property>的值
    PropertyValues pvs = mbd.getPropertyValues();

    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
            mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

        // Add property values based on autowire by name if applicable.
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }

        // Add property values based on autowire by type if applicable.
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE
            autowireByType(beanName, mbd, bw, newPvs);
        }

        pvs = newPvs;
    }
    //设值
    applyPropertyValues(beanName, mbd, bw, pvs);
}
```

autowireByName源码:

```java
protected void autowireByName(
        String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {
    //返回所有引用(ref="XXX")的bean名称
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
    for (String propertyName : propertyNames) {
        if (containsBean(propertyName)) {
             //从BeanFactory获取
            Object bean = getBean(propertyName);
            pvs.add(propertyName, bean);
            registerDependentBean(propertyName, beanName);
        }
    }
}
```

autowireByType也是同样的套路，所以可以得出结论:  **autowireByName和autowireByType方法只是先获取到引用的bean，真正的设值是在applyPropertyValues中进行的。**

###### 属性设置

Spring判断一个属性可不可以被设置(存不存在)是通过java bean的内省操作来完成的，也就是说，属性可以被设置的条件是**此属性拥有public的setter方法，并且注入时的属性名应该是setter的名字**。

###### 初始化

此处的初始化指的是bean已经构造完成，执行诸如调用其init方法的操作。相关源码:

```java
// Initialize the bean instance.
Object exposedObject = bean;
try {
    populateBean(beanName, mbd, instanceWrapper);
    if (exposedObject != null) {
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
}
```

initializeBean:

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                invokeAwareMethods(beanName, bean);
                return null;
            }
        }, getAccessControlContext());
    }
    else {
        invokeAwareMethods(beanName, bean);
    }

    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    invokeInitMethods(beanName, wrappedBean, mbd);

    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```

主要的操作步骤一目了然。

- Aware方法触发:
  
  我们的bean有可能实现了一些XXXAware接口，此处就是负责调用它们:
  
  ```java
  private void invokeAwareMethods(final String beanName, final Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ((BeanClassLoaderAware) bean).setBeanClassLoader(getBeanClassLoader());
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
  }
  ```

- BeanPostProcessor触发，没什么好说的

- 调用init方法:
  
  在XML配置中，bean可以有一个init-method属性来指定初始化时调用的方法。从原理来说，其实就是一个反射调用。不过注意这里有一个InitializingBean的概念。
  
  此接口只有一个方法：
  
  ```java
  void afterPropertiesSet() throws Exception;
  ```
  
  如果我们的bean实现了此接口，那么此方法会首先被调用。此接口的意义在于: 当此bean的所有属性都被设置(注入)后，给bean一个利用现有属性重新组织或是检查属性的机会。感觉和init方法有些冲突，不过此接口在Spring被广泛使用。

##### getObjectForBeanInstance

位于AbstractBeanFactory，此方法的目的在于如果bean是FactoryBean，那么返回其工厂方法创建的bean，而不是自身。

##### 《《《 多例的初始化

AbstractBeanFactory.doGetBean相关源码:

```java
else if (mbd.isPrototype()) {
    // It's a prototype -> create a new instance.
    Object prototypeInstance = null;
    try {
        beforePrototypeCreation(beanName);
        prototypeInstance = createBean(beanName, mbd, args);
    }
    finally {
        afterPrototypeCreation(beanName);
    }
    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
}
```

### beforePrototypeCreation

此方法用于确保在同一时刻只能有一个此bean在初始化。

### createBean

和单例的是一样的，不在赘述。

### afterPrototypeCreation

和beforePrototypeCreation对应的，你懂的（多例创建后要执行的方法）。

### 总结

可以看出，初始化其实和单例是一样的，只不过单例多了一个是否已经存在的检查。

## 其它Scope初始化

其它就指的是request、session。此部分源码:

```java
else {
    String scopeName = mbd.getScope();
    final Scope scope = this.scopes.get(scopeName);
    if (scope == null) {
        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
    }
    Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
        @Override
        public Object getObject() throws BeansException {
            beforePrototypeCreation(beanName);
            try {
                return createBean(beanName, mbd, args);
            }
            finally {
                afterPrototypeCreation(beanName);
            }
        }
    });
    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
}
```

scopes是一个LinkedHashMap<String, Scope>，可以调用 ConfigurableBeanFactory定义的registerScope方法注册其值。

Scope接口继承体系:

![images\Scope](images\Scope.jpg)

根据socpe.get的注释，此方法如果找到了叫做beanName的bean，那么返回，如果没有，将调用ObjectFactory创建之。Scope的实现参考类图。



## 总结：

对于（11），主要是单例对象的初始化过程，这个过程确实比较复杂，首先是进行类型转换，把配置文件中bean对应class的字符串转化为具体的类型，然后解析注解对应的值。

然后是单例的初始化，getBean方法时核心逻辑，首先进行的是手动注册的单例bean的检查（在spring设置环境时就设置过一些单例bean），再检查父容器是否存在该bean的定义，如果有，则让父容器创建bean，在创建单例之前，还需要让依赖先初始化（让一个并发map保存其关系），在这之后，才是真正的单例初始化，需要注意的是，getBean包含的是所有作用域bean的初始化，单例只是其中一种。

  在初始化时，先检查容器中是否存在该单例，如果有，就直接返回，如果没有，则创建对应的bean对象，可以对照着bean的生命周期图来分析，首先是lookup-method标签检测，然后是Beanpostprocesser对应方法（pre与after两钩子）的调用，  到这个，才是真正的创建对象doCreateBean。

在spring中，创建bean对象的方式总体上有两种，利用工厂方法创建与利用构造函数创建，通过工厂方法的创建简单来说，就是通过反射获取工厂方法的method对象，然后调用获取对象即可。   通过构造方法的获取，可以配合构造器自动装配，对应的

autowireConstructor调用的是ConstructorResolver.autowireConstructor，此方法主要做了两件事:

- 得到合适的构造器对象。

- 根据构造器参数的类型去BeanFactory查找相应的bean:

通过自动装配也可获取属性中的set方法注入的对象并引用给属性。

然后初始化完成，在这个过程中，如果有实现aware接口，就在对应的aware方法中注入对应的aware对象。



其余作用域的，比如多例，与单例基本相似，但是没有检查时候含有该bean的操作，并且在创建后，容器不会管理多例的生命周期。



## （12）this.finishRefresh();

完成ioc容器创建与bean单例的创建过程后，设置ioc生命周期，并发布事件。



到这里，ioc原码的主体分析就完成了，剩余的就是容器的销毁与应用关闭，主要是销毁单例。






