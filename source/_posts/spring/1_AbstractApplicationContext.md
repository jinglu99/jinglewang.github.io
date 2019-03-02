---
title: （一）AbstractApplicationContext分析
date: 2018/11/27 17:48:25  # 文章发表时间
toc: true
tags:
- Java
- Spring 
categories:
- Java
- Spring源码
thumbnail: http://pic.jingl.wang/2019-03-02-150042.png
---



AbstractApplicationContext(下称AAC) 是Spring应用上下文的基类，通过调用此类的refresh()方法来初始化spring容器，它的依赖关系如图所示，大部分的接口方法在AAC中进行实现。
<!--more-->

![](http://pic.jingl.wang/2018-11-27-085135.png)

## BeanFactory

BeanFactory是bean工厂的顶级接口，提供了一些访问bean容器的方法，他是bean容器最基本的视图，主要的子类有 `HierarchicalBeanFactory` 和 `ListableBeanFactory` 

它提供的方法有：

```java
	//根据bean name, 或者bean type获得bean
	Object getBean(String name) throws BeansException;	
	<T> T getBean(String name, Class<T> requiredType) throws BeansException;
	Object getBean(String name, Object... args) throws BeansException;
	<T> T getBean(Class<T> requiredType) throws BeansException;
	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

	//获得ObjectProvider 
	//ObjectFactory的变体 用于注入操作，以后深入
	<T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);
	<T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

	//是否存在bean
	boolean containsBean(String name);

	//是否为单例的bean
	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	
	//是否为原型域的bean
	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

	//name对应的bean是否与typeToMatch类型一致
	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;
	boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

	//获得bean的类型
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	//获得bean的别名
	String[] getAliases(String name);
```

接下来 在看一下对他进行补充的两个主要的子类`HierarchicalBeanFactory` 和 `ListableBeanFactory` 

### HierarchicalBeanFactory

`HierarchicalBeanFactory` 这个接口的意义是使BeanFactory可以获得父子关系，他提供了2个方法：

```java
//获得当前BeanFactory的父工厂类
BeanFactory getParentBeanFactory();

//在当前工厂类中是否存在某个bean
boolean containsLocalBean(String name);
```

通过这两个方法可以在具有父子关系的bean工厂中查找一个bean。

### ListableBeanFactory

`ListableBeanFactory` 使BeanFactory 获得了枚举所有bean实例的能力。如果当前工厂类是具有父子关系的，n那么这个接口中的方法只会考虑当前工厂类中的bean。它提供了这些方法：

```java
//确认当前bean工厂是否存在某个bean的定义
boolean containsBeanDefinition(String beanName);

//统计当前工厂中bean定义的数量，会忽略用其他方法的注册的bean。
int getBeanDefinitionCount();

//获得所有bean定义的名称
String[] getBeanDefinitionNames();

//通过类型寻找Bean定义
String[] getBeanNamesForType(ResolvableType type);
String[] getBeanNamesForType(@Nullable Class<?> type);
String[] getBeanNamesForType(@Nullable Class<?> type, boolean includeNonSingletons, boolean allowEagerInit);

//通过类型获得Bean
<T> Map<String, T> getBeansOfType(@Nullable Class<T> type) throws BeansException;
<T> Map<String, T> getBeansOfType(@Nullable Class<T> type, boolean includeNonSingletons, boolean allowEagerInit)
      throws BeansException;

//获得所有存在某个注解的bean name
String[] getBeanNamesForAnnotation(Class<? extends Annotation> annotationType);

//通过注解获得bean
Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> annotationType) throws BeansException;

//获得一个bean的注解
@Nullable
<A extends Annotation> A findAnnotationOnBean(String beanName, Class<A> annotationType)
      throws NoSuchBeanDefinitionException;
```

除了getBeanDefinitionCount和containsBeanDefinition，这个接口中的方法没有被当作频烦调用的方法设计，实现可以会慢。

## ApplicationContext

`ApplicationContext` 是所有应用上下文锁都需要实现的接口，是spring容器中很重要的接口，他整合了EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,  MessageSource, ApplicationEventPublisher, ResourcePatternResolver接口，以此提供了：

* 访问应用组件的工厂方法（ListableBeanFactory）
* 加载资源的能力（ResourceLoader）
* 发布注册事件消息的能力（ApplicationEventPublisher）
* 处理消息，支持国际化的能力（MessageSource）
* 获得父ApplicationContext的方法

在这里我们忽略ApplicationContext所继承的接口，主要分析ApplicationContext自己提供的方法：

```java
//获得当前上下文的唯一id
String getId();

//获得应用名称
String getApplicationName();

//获得上下文的名称
String getDisplayName();

//获得上下文进行首次加载的时间戳
long getStartupDate();

//获得父ApplicationContext
ApplicationContext getParent();

//获取当前上下文的beanFactory(包装成AutowireCapableBeanFactory)
AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException;
```

接下来继续分析ApplicationContext所继承的一些接口。

### ResourceLoader

ApplicationContext继承了ResourcePatternResolver接口，而ResourcePatternResolver是对ResourceLoader的一个拓展，它是一个加载资源的策略接口，ResourcePatternResolver本身提供了一个可以通过正则表达式加载资源的方法`Resource[] getResources(String locationPattern)` , 在ResourceLoader接口中又提供了这两个方法：

```java
//通过一个确定路径获得资源
Resource getResource(String location);

//获得ClassLoader用于加载资源
ClassLoader getClassLoader();
```

在AbstractApplicationContext中，实际使用了一个叫做PathMatchingResourcePatternResolver独立的类作为资源加载器，在这个类中实现了`Resource[] getResources(String locationPattern)` 这个方法， 此外这个类又对ResourceLoader进行包装，并提供一个参数为ResourceLoader构造函数，通过策略模式来代理作为参数传入的ResourceLoader实现类。在实例化PathMatchingResourcePatternResolver类时，AAC又将本身作为参数，构造资源加载器。

```
/**
 * Return the ResourcePatternResolver to use for resolving location patterns
 * into Resource instances. Default is a
 * {@link org.springframework.core.io.support.PathMatchingResourcePatternResolver},
 * supporting Ant-style location patterns.
 * <p>Can be overridden in subclasses, for extended resolution strategies,
 * for example in a web environment.
 * <p><b>Do not call this when needing to resolve a location pattern.</b>
 * Call the context's {@code getResources} method instead, which
 * will delegate to the ResourcePatternResolver.
 * @return the ResourcePatternResolver for this context
 * @see #getResources
 * @see org.springframework.core.io.support.PathMatchingResourcePatternResolver
 */
protected ResourcePatternResolver getResourcePatternResolver() {
   return new PathMatchingResourcePatternResolver(this);
}
```

而AAC又是继承`DefaultResourceLoader`类的，所以ResourceLoader的声明的2个方法是在`DefaultResourceLoader`中实现的。在此先不展开，以后再进行分析。



### ApplicationEventPublisher

声明用于发布应用事件消息方法的接口, 它声明了两个方法：

```Java
//一个默认的方法 用于发布框架事件
default void publishEvent(ApplicationEvent event) {
   publishEvent((Object) event);
}

//发布事件，在AAC中实现
void publishEvent(Object event);
```



### MessageSource

一个用于国际化的接口，AAC默认使用一个DelegatingMessageSource来实现这个接口， 但DelegatingMessageSource只是一个空实现，如果需要实现国际化的操作需要使用者自定义messageSource来实现。spring为MessageSource提供了`ResourceBundleMessageSource` 和 `ReloadableResourceBundleMessageSource` 两个实现类。



### EnvironmentCapable

这个只声明了一个方法`getEnvironment()` 用于获得spring应用的运行环境。在AAC中，默认使用了StandardEnvironment类在作为这个接口的实现类。



## ConfigurableApplicationContext

该接口提供了一些配置和初始化应用上下文的方法：

```java
//设置当前上下文的唯一id
void setId(String id);

//设置父上下文
void setParent(@Nullable ApplicationContext parent);

//设置环境
void setEnvironment(ConfigurableEnvironment environment);

//获得环境
ConfigurableEnvironment getEnvironment();

//添加一个bean工厂后处理器
void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor);

//添加一个应用监听器
void addApplicationListener(ApplicationListener<?> listener);

//添加一个协议解决器
void addProtocolResolver(ProtocolResolver resolver);

//刷新应用上下文， AAC也用这个方法来初始化上下文
void refresh() throws BeansException, IllegalStateException;

//注册一个jvm关闭钩子，可以被调用多次， 但最多只有一个hook会被注册
void registerShutdownHook();

//关闭应用上下文，释放所有资源，包括缓存的单例bean
void close();

//判断上下文是否可用
boolean isActive();

//获得一个可配置的bean工厂
ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
```



## 应用上下文的初始化

先看一下AAC的构造方法,他提供了2个构造方法：

```
//默认构造函数
public AbstractApplicationContext() {
   this.resourcePatternResolver = getResourcePatternResolver();
}

//给定父上下文，进行构造
public AbstractApplicationContext(@Nullable ApplicationContext parent) {
   this();
   setParent(parent);
}
```

在实例化AAC时，会调用`getResourcePatternResolver()`方法，这个方法实例化了一个`PathMatchingResourcePatternResolver`对象，作为AAC的资源加载器，当指定父上下文时，还会把父上下文的环境配置和当前上下文的环境进行合并。

```java
    public void setParent(@Nullable ApplicationContext parent) {
       this.parent = parent;
       if (parent != null) {
          Environment parentEnvironment = parent.getEnvironment();
          if (parentEnvironment instanceof ConfigurableEnvironment) {
             getEnvironment().merge((ConfigurableEnvironment) parentEnvironment);
          }
       }
    }

	/**
	 * Return the {@code Environment} for this application context in configurable
	 * form, allowing for further customization.
	 * <p>If none specified, a default environment will be initialized via
	 * {@link #createEnvironment()}.
	 */
	@Override
	public ConfigurableEnvironment getEnvironment() {
		if (this.environment == null) {
			this.environment = createEnvironment(); //new StandardEnvironment()
		}
		return this.environment;
	}
```

getEnvironment()是获得当前上下文的配置环境，AAC实例化配置环境的策略是懒加载的，只有第一次调用getEnvironment方法是才会为当前上下文指定一个配置环境。

实例化了一个应用上下文后，会调用`refresh()` 方法对上下文进行初始化。

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      // Prepare this context for refreshing.
      prepareRefresh();

      // Tell the subclass to refresh the internal bean factory.
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();    //重建一个DefaultListableBeanFactory

      // Prepare the bean factory for use in this context.
      prepareBeanFactory(beanFactory);   //构造BeanFactory

      try {
         // Allows post-processing of the bean factory in context subclasses.
         postProcessBeanFactory(beanFactory);

         // Invoke factory processors registered as beans in the context.
         invokeBeanFactoryPostProcessors(beanFactory);

         // Register bean processors that intercept bean creation.
         registerBeanPostProcessors(beanFactory);

         // Initialize message source for this context.
         initMessageSource();

         // Initialize event multicaster for this context.
         initApplicationEventMulticaster();

         // Initialize other special beans in specific context subclasses.
         onRefresh();

         // Check for listener beans and register them.
         registerListeners();

         // Instantiate all remaining (non-lazy-init) singletons.
         finishBeanFactoryInitialization(beanFactory);

         // Last step: publish corresponding event.
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);

         // Propagate exception to caller.
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
      }
   }
}
```

`prepareRefresh()` 方法会做一些刷新之前的操作：

* 重置上下文的启动时间为当前时间
* 重置上下文的运行状态
* 验证环境中的properties
* 清空上下文事件列表

做完了上述操作后，接着会调用`obtainFreshBeanFactory()`方法，这个方法需要子类进行重写，他的主要功能是为应用上下文新建一个Bean工厂, 加载BeanDefinition ，此处先不展开。

接着调用的是`prepareBeanFactory()`方法来对BeanFactory做一些配置。此时BeanFactory已经准备完毕了，单做的都是标准配置，AAC提供了一个可以让子类修改BeanFactory配置的方法，子类通过重写这个方法可以修改BeanFactory。

接下来，符合要求的BeanFactory才算真正的配置完成，这个时候就会向所有实现`BeanFactoryPostProcessor` 的Bean发布消息，通过调用`invokeBeanFactoryPostProcessors()`方法来调用这些bean实现的postProcessBeanFactory()方法。

然后是`registerBeanPostProcessors()`方法，用来注册bean处理器。

接着是初始化MessageSource,初始化应用事件分发器，初测监听类。

随后调用`finishBeanFactoryInitialization()`完成Bean的初始化，实例化所有非懒加载的单例bean.

最后在`finishRefresh()`方法中完成相关事件的发布。







