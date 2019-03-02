---
title: （二）环境资源管理
date: 2018/12/10 17:48:25  # 文章发表时间
toc: true
tags:
- Java
- Spring 
categories:
- Java
- Spring源码
thumbnail: http://pic.jingl.wang/2019-03-02-DSC_0678.jpg
---


spring 中的运行环境通过Environment进行管理。环境分为2种：profile和property。

profile的作用是，在实际开发中，开发环境和实际线上环境的配置是不同的，但是如何能不修改代码就可以让代码在2个不同的环境中运行呢，profile就是用来解决这个问题的，可以为一个bean指定一个或多个profile，当代码运行在不同环境中时，通过配置profile，spring可以知道是否应该加载这个bean，以此来区分开发环境和正式环境的配置。

property 即常用的资源配置，这里不多赘述。

<!--more-->

![](http://pic.jingl.wang/2018-12-10-140919.png)

Environment继承自PropertyResolver, PropertyResolver 声明了解析property部分的方法，而Environment本身则在PropertyResolver的基础上，增加了profile部分的方法。

在Spring中，用Environment主要实现在AbstractEnvironment中，在这里，我将主要对这个类进行分析。在此之前，先来看下Environment和PropertyResolver所声明的方法。

### PropertyResolver

```Java
//property是否存在
boolean containsProperty(String key);

String getProperty(String key);
String getProperty(String key, String defaultValue);
<T> T getProperty(String key, Class<T> targetType);
<T> T getProperty(String key, Class<T> targetType, T defaultValue);

//当不存在property时会抛出异常
String getRequiredProperty(String key) throws IllegalStateException;
<T> T getRequiredProperty(String key, Class<T> targetType) throws IllegalStateException;

//解析字符串中的占位符
String resolvePlaceholders(String text);
String resolveRequiredPlaceholders(String text) throws IllegalArgumentException;
```

### Environment

```Java
//获得所有活动的profile
String[] getActiveProfiles();

//当未指定活动profile时 会使用默认的profile
String[] getDefaultProfiles();

//传入的profile是否是活动的
@Deprecated
boolean acceptsProfiles(String... profiles);
boolean acceptsProfiles(Profiles profiles);
```

## AbstractEnvironment

AbstractEnvironment是Environment主要的实现类以下简称AE，所有的实体类都会继承这个类。这个类中，使用了代理模式，将property部分的逻辑，交由`PropertySourcesPropertyResolver`来实现, 而AbstractEnvironment本身则实现了profile部分的逻辑。首先，先分析一下AbstractEnvironment完整的继承关系。

![](http://pic.jingl.wang/2018-12-10-144354.png)

从上图可以看出，AbstractEnvironment并非直接的实现Environment接口，而是继承了ConfigurableEnvironment和ConfigurablePropertyResolver接口。在这连个接口中，提供了一些用于配置的方法，具体如下：

### ConfigurablePropertyResolver

```Java
//获得/设置类型转换器 用于getProperty方法中的类型转换。
ConfigurableConversionService getConversionService();
void setConversionService(ConfigurableConversionService conversionService);

//设置占位符的前后缀 用于处理${...}
void setPlaceholderPrefix(String placeholderPrefix);
void setPlaceholderSuffix(String placeholderSuffix);

//设置占位符中的分隔符， spring默认的是":"，当需要默认值时，只需写成${key:defaultVal}即可
void setValueSeparator(@Nullable String valueSeparator);

//是否不处理嵌套的占位符
void setIgnoreUnresolvableNestedPlaceholders(boolean ignoreUnresolvableNestedPlaceholders);

//当进行property验证时，必须验证的property
void setRequiredProperties(String... requiredProperties);

//验证setRequiredProperties方法设置的property是否存在且合法
void validateRequiredProperties() throws MissingRequiredPropertiesException;
```

### ConfigurableEnvironment

```Java
//清空set重新添加活动的profile
void setActiveProfiles(String... profiles);

//在原有的基础上增加活动profile
void addActiveProfile(String profile);

//设置默认profile
void setDefaultProfiles(String... profiles);

//获得PropertySources, PropertySource是时间用来存储property的地方
MutablePropertySources getPropertySources();

//获取系统property
Map<String, Object> getSystemProperties();

//获取系统环境
Map<String, Object> getSystemEnvironment();

//合并其他Environment的profile 和 Environment
void merge(ConfigurableEnvironment parent);
```

以上就是AbstractEnvironment提供的所有方法，接下来，看一下他们是如何实现的。

首先，先看看AE的成员变量：

```
private final Set<String> activeProfiles = new LinkedHashSet<>();	//活动的profile set

//默认profile的set 如果指定则为{"default"}
private final Set<String> defaultProfiles = new LinkedHashSet<>(getReservedDefaultProfiles());

//一个用于存放的PropertySource的集合
private final MutablePropertySources propertySources = new MutablePropertySources();

//上面说到的代理的用于实现property部分逻辑的实现类
private final ConfigurablePropertyResolver propertyResolver =
      new PropertySourcesPropertyResolver(this.propertySources);
```

在这里需要简单提一下`MutablePropertySources`这个类。它实际上是一个PropertySource的集合，内部维护了一个List<PropertySource<?>>, 它实现PropertySources接口，而PropertySources又是继承自Iterable<PropertySource<?>>的。在上面代码中其实是建了一个空的PropertySource集合。那么PropertySource又是什么东西呢。从名字上看，就是property的源的意思，所有的property都是从这里获取，是用来存放property的地方。

AE只提供了一个无参的构造方法：

```java
public AbstractEnvironment() {
   customizePropertySources(this.propertySources);
}


/**
 * Customize the set of {@link PropertySource} objects to be searched by this
 * {@code Environment} during calls to {@link #getProperty(String)} and related
 * methods.
 *
 * <p>Subclasses that override this method are encouraged to add property
 * sources using {@link MutablePropertySources#addLast(PropertySource)} such that
 * further subclasses may call {@code super.customizePropertySources()} with
 * predictable results. For example:
 * <pre class="code">
 * public class Level1Environment extends AbstractEnvironment {
 *     &#064;Override
 *     protected void customizePropertySources(MutablePropertySources propertySources) {
 *         super.customizePropertySources(propertySources); // no-op from base class
 *         propertySources.addLast(new PropertySourceA(...));
 *         propertySources.addLast(new PropertySourceB(...));
 *     }
 * }
 *
 * public class Level2Environment extends Level1Environment {
 *     &#064;Override
 *     protected void customizePropertySources(MutablePropertySources propertySources) {
 *         super.customizePropertySources(propertySources); // add all from superclass
 *         propertySources.addLast(new PropertySourceC(...));
 *         propertySources.addLast(new PropertySourceD(...));
 *     }
 * }
 * </pre>
 * In this arrangement, properties will be resolved against sources A, B, C, D in that
 * order. That is to say that property source "A" has precedence over property source
 * "D". If the {@code Level2Environment} subclass wished to give property sources C
 * and D higher precedence than A and B, it could simply call
 * {@code super.customizePropertySources} after, rather than before adding its own:
 * <pre class="code">
 * public class Level2Environment extends Level1Environment {
 *     &#064;Override
 *     protected void customizePropertySources(MutablePropertySources propertySources) {
 *         propertySources.addLast(new PropertySourceC(...));
 *         propertySources.addLast(new PropertySourceD(...));
 *         super.customizePropertySources(propertySources); // add all from superclass
 *     }
 * }
 * </pre>
 * The search order is now C, D, A, B as desired.
 *
 * <p>Beyond these recommendations, subclasses may use any of the {@code add&#42;},
 * {@code remove}, or {@code replace} methods exposed by {@link MutablePropertySources}
 * in order to create the exact arrangement of property sources desired.
 *
 * <p>The base implementation registers no property sources.
 *
 * <p>Note that clients of any {@link ConfigurableEnvironment} may further customize
 * property sources via the {@link #getPropertySources()} accessor, typically within
 * an {@link org.springframework.context.ApplicationContextInitializer
 * ApplicationContextInitializer}. For example:
 * <pre class="code">
 * ConfigurableEnvironment env = new StandardEnvironment();
 * env.getPropertySources().addLast(new PropertySourceX(...));
 * </pre>
 *
 * <h2>A warning about instance variable access</h2>
 * Instance variables declared in subclasses and having default initial values should
 * <em>not</em> be accessed from within this method. Due to Java object creation
 * lifecycle constraints, any initial value will not yet be assigned when this
 * callback is invoked by the {@link #AbstractEnvironment()} constructor, which may
 * lead to a {@code NullPointerException} or other problems. If you need to access
 * default values of instance variables, leave this method as a no-op and perform
 * property source manipulation and instance variable access directly within the
 * subclass constructor. Note that <em>assigning</em> values to instance variables is
 * not problematic; it is only attempting to read default values that must be avoided.
 *
 * @see MutablePropertySources
 * @see PropertySourcesPropertyResolver
 * @see org.springframework.context.ApplicationContextInitializer
 */
protected void customizePropertySources(MutablePropertySources propertySources) {
}
```

在构造方法中，它将会调用customizePropertySources方法，在AE中，其实并没有对这个方法进行实现。需要在子类中重新这个类。而这个方法的主要目的是让子类在其中添加一些PropertySource, 如StandardEnvironment中是如下实现的。

```java
protected void customizePropertySources(MutablePropertySources propertySources) {
   propertySources.addLast(new MapPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));
   propertySources.addLast(new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
}
```

分析完了构造方法，接下来理一理成员方法的实现。同样的分为2个部分，profile和property，profile部分实现了单纯在Environment中和ConfigurableEnvironment中声明的方法。

```java
public String[] getActiveProfiles() {
   return StringUtils.toStringArray(doGetActiveProfiles());
}

/**
 * Return the set of active profiles as explicitly set through
 * {@link #setActiveProfiles} or if the current set of active profiles
 * is empty, check for the presence of the {@value #ACTIVE_PROFILES_PROPERTY_NAME}
 * property and assign its value to the set of active profiles.
 * @see #getActiveProfiles()
 * @see #ACTIVE_PROFILES_PROPERTY_NAME
 */
protected Set<String> doGetActiveProfiles() {
   synchronized (this.activeProfiles) {
      if (this.activeProfiles.isEmpty()) {
         String profiles = getProperty(ACTIVE_PROFILES_PROPERTY_NAME);
         if (StringUtils.hasText(profiles)) {
            setActiveProfiles(StringUtils.commaDelimitedListToStringArray(
                  StringUtils.trimAllWhitespace(profiles)));
         }
      }
      return this.activeProfiles;
   }
}
```

getActiveProfiles`方法是用来获得活动profile的，活动profile的获取是懒加载的，当第一次需要获得活动profile时，会通过`doGetActiveProfiles`方法，从property中进行加载，这个过程通过对activeProfiles进行加锁，保证了线程安全性。

```java
@Override
public void setActiveProfiles(String... profiles) {
   Assert.notNull(profiles, "Profile array must not be null");
   if (logger.isDebugEnabled()) {
      logger.debug("Activating profiles " + Arrays.asList(profiles));
   }
   synchronized (this.activeProfiles) {
      this.activeProfiles.clear();
      for (String profile : profiles) {
         validateProfile(profile); //判断profile是否合法 (不为空， 不以！开头)
         this.activeProfiles.add(profile);
      }
   }
}

@Override
public void addActiveProfile(String profile) {
   if (logger.isDebugEnabled()) {
      logger.debug("Activating profile '" + profile + "'");
   }
   validateProfile(profile);
   doGetActiveProfiles();
   synchronized (this.activeProfiles) {
      this.activeProfiles.add(profile);
   }
}
```

上面这两个方法是用来添加活动的profile的，区别是set方法是会清空原有的activeProfiles，而add方法只是追加，通样这两个方法都是线程安全的。

```java
@Override
public String[] getDefaultProfiles() {
   return StringUtils.toStringArray(doGetDefaultProfiles());
}

/**
 * Return the set of default profiles explicitly set via
 * {@link #setDefaultProfiles(String...)} or if the current set of default profiles
 * consists only of {@linkplain #getReservedDefaultProfiles() reserved default
 * profiles}, then check for the presence of the
 * {@value #DEFAULT_PROFILES_PROPERTY_NAME} property and assign its value (if any)
 * to the set of default profiles.
 * @see #AbstractEnvironment()
 * @see #getDefaultProfiles()
 * @see #DEFAULT_PROFILES_PROPERTY_NAME
 * @see #getReservedDefaultProfiles()
 */
protected Set<String> doGetDefaultProfiles() {
   synchronized (this.defaultProfiles) {
      if (this.defaultProfiles.equals(getReservedDefaultProfiles())) {
         //检查是否配置spring.profiles.default， 如果是则使用配置的profile作为默认profile
         String profiles = getProperty(DEFAULT_PROFILES_PROPERTY_NAME);
         if (StringUtils.hasText(profiles)) {
            setDefaultProfiles(StringUtils.commaDelimitedListToStringArray(
                  StringUtils.trimAllWhitespace(profiles)));
         }
      }
      return this.defaultProfiles;
   }
}

/**
* Specify the set of profiles to be made active by default if no other profiles
* are explicitly made active through {@link #setActiveProfiles}.
* <p>Calling this method removes overrides any reserved default profiles
* that may have been added during construction of the environment.
* @see #AbstractEnvironment()
* @see #getReservedDefaultProfiles()
*/
@Override
public void setDefaultProfiles(String... profiles) {
	Assert.notNull(profiles, "Profile array must not be null");
	synchronized (this.defaultProfiles) {
	this.defaultProfiles.clear();	//清除default
	validateProfile(profile);	//验证profile，如果为空或者以!开头则抛出异常IllegalArgumentException
	this.defaultProfiles.add(profile);
		}
	}
}
```

`getDefaultProfiles` 方法和`getActiveProfiles`方法类似，当用户没有自定义default profile时， 则会返回default作为默认profile。`setDefaultProfiles`方法也基本和`setActiveProfiles`类似，不多赘述。

```
@Override
@Deprecated
public boolean acceptsProfiles(String... profiles) {
   Assert.notEmpty(profiles, "Must specify at least one profile");
   for (String profile : profiles) {
      if (StringUtils.hasLength(profile) && profile.charAt(0) == '!') {
         if (!isProfileActive(profile.substring(1))) {
            return true;
         }
      }
      else if (isProfileActive(profile)) {
         return true;
      }
   }
   return false;
}

@Override
public boolean acceptsProfiles(Profiles profiles) {
   Assert.notNull(profiles, "Profiles must not be null");
   return profiles.matches(this::isProfileActive);
}

/**
 * Return whether the given profile is active, or if active profiles are empty
 * whether the profile should be active by default.
 * @throws IllegalArgumentException per {@link #validateProfile(String)}
 */
protected boolean isProfileActive(String profile) {
   validateProfile(profile);
   Set<String> currentActiveProfiles = doGetActiveProfiles();
   return (currentActiveProfiles.contains(profile) ||
         (currentActiveProfiles.isEmpty() && doGetDefaultProfiles().contains(profile)));
}
```

接下来是2个判断profile是否活跃的方法，前一个方法已经启用，但是2者实现的原理都是一样的，在activeProfiles和defaultProfiles中是否存在那个profile。

最后来看一下merge方法，merge方法用于将一个父环境中注册的propertySource和profile注册到当前环境中。同样这个方法是线程安全的。

```
@Override
public void merge(ConfigurableEnvironment parent) {
   for (PropertySource<?> ps : parent.getPropertySources()) {
      if (!this.propertySources.contains(ps.getName())) {
         this.propertySources.addLast(ps);
      }
   }
   String[] parentActiveProfiles = parent.getActiveProfiles();
   if (!ObjectUtils.isEmpty(parentActiveProfiles)) {
      synchronized (this.activeProfiles) {
         for (String profile : parentActiveProfiles) {
            this.activeProfiles.add(profile);
         }
      }
   }
   String[] parentDefaultProfiles = parent.getDefaultProfiles();
   if (!ObjectUtils.isEmpty(parentDefaultProfiles)) {
      synchronized (this.defaultProfiles) {
         this.defaultProfiles.remove(RESERVED_DEFAULT_PROFILE_NAME);
         for (String profile : parentDefaultProfiles) {
            this.defaultProfiles.add(profile);
         }
      }
   }
}
```

以上，profil部分的实现已经全部分析完了，接下来在对property部分的实现进行分析，之前说了AE是将这部分的逻辑代理到了`PropertySourcesPropertyResolver`这个类中，所以我将直接对这个类进行分析。

![](http://pic.jingl.wang/2018-12-24-142920.png)

对于`PropertyResovler`的逻辑，主要是放在了`AbstractPropertyResolver`中，PropertyResolver提供的方法，可以分为这么几类：

* property的获取
* 字符串中property占位符的解析替换
* 其他的set配置方法

所有的set配置方法，字符串解析的方法都在APR中实现，一部分property在APR中实现，但其最核心的获取操作实现在了`PropertySourcesPropertyResolver`中。

首先还是看一下APR的成员变量。

```java
@Nullable
private volatile ConfigurableConversionService conversionService;

@Nullable
private PropertyPlaceholderHelper nonStrictHelper;

@Nullable
private PropertyPlaceholderHelper strictHelper;

private boolean ignoreUnresolvableNestedPlaceholders = false;

private String placeholderPrefix = SystemPropertyUtils.PLACEHOLDER_PREFIX;

private String placeholderSuffix = SystemPropertyUtils.PLACEHOLDER_SUFFIX;

@Nullable
private String valueSeparator = SystemPropertyUtils.VALUE_SEPARATOR;

private final Set<String> requiredProperties = new LinkedHashSet<>();
```

第一个变量`conversionService` 是用来作类型转换的，可以将property通过这个服务把类型转换为目标类型；

第二第三个变量都是`PropertyPlaceholderHelper`类型的，他们是用来解析字符串中的property, 在这里有严格和非严格的资源占位符解析，如果是严格的，当无法找到相应资源时，将会抛出异常，如果是非严格的，则会将那个占位符替换为空字符串， 他们都是懒加载的。

第四个变量`ignoreUnresolvableNestedPlaceholders` 是一个布尔类型的配置变量，通过设置这个变量，可以选择是否忽略嵌套的无法解析的占位符；

第五第六个变量用来指定占位符的前缀和后缀，比如`${...}`;

第七个`valueSeparator`变量用来分隔占位符中，property的key和默认值，如`${key:defaultVal}`中的`:`;

最后一个变量是一个set容器，用来存放调用`validateRequiredProperties`时，必须存在的资源名称。

对于set方法，这里就不多做说明了，着重来看一下getProperty和resolvePlaceholder系列方法。

对于getProperty类方法，都是依赖`PropertySourcesPropertyResolver`中的`getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders)`方法，所以这里只需要分析这个方法即可。

```java
protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
   if (this.propertySources != null) {
      for (PropertySource<?> propertySource : this.propertySources) {
         //依次在注册的propertySource总寻找
         if (logger.isTraceEnabled()) {
            logger.trace("Searching for key '" + key + "' in PropertySource '" +
                  propertySource.getName() + "'");
         }
         Object value = propertySource.getProperty(key);
         if (value != null) {
            if (resolveNestedPlaceholders && value instanceof String) {
               //解析嵌套placeholder
               value = resolveNestedPlaceholders((String) value);
            }
            logKeyFound(key, propertySource, value);
            return convertValueIfNecessary(value, targetValueType);       //类型转换
         }
      }
   }
   if (logger.isTraceEnabled()) {
      logger.trace("Could not find key '" + key + "' in any property source");
   }
   return null;
}
```

他的逻辑其实非常的简单，依次在所有propertySource中寻找这个property，当找到这个property时，会通过resolveNestedPlaceholders这个方法处理value中嵌套的资源占位符，最后，调用`convertValueIfNecessary`方法通过ConversionService转换为目标类型。

对字符串中资源占位符的解析，是在`PropertyPlaceholderHelper` 中完成的，来看下这部分的逻辑：

```java
protected String parseStringValue(
      String value, PlaceholderResolver placeholderResolver, Set<String> visitedPlaceholders) {

   StringBuilder result = new StringBuilder(value);

   int startIndex = value.indexOf(this.placeholderPrefix); //寻找第一个${前缀
   while (startIndex != -1) { //存在占位符，则进行处理
      int endIndex = findPlaceholderEndIndex(result, startIndex);
      if (endIndex != -1) {
         String placeholder = result.substring(startIndex + this.placeholderPrefix.length(), endIndex); //拿出占位符key
         String originalPlaceholder = placeholder;
         if (!visitedPlaceholders.add(originalPlaceholder)) {   //判断是否存在循环引用的情况 即 placeholder=${placeholder}
            throw new IllegalArgumentException(
                  "Circular placeholder reference '" + originalPlaceholder + "' in property definitions");
         }
         // Recursive invocation, parsing placeholders contained in the placeholder key.
         placeholder = parseStringValue(placeholder, placeholderResolver, visitedPlaceholders); //处理placeholder中嵌套的占位符，获得真正的key 如#{#{key}}
         // Now obtain the value for the fully resolved key...
         String propVal = placeholderResolver.resolvePlaceholder(placeholder);  //尝试获得最终占位符锁对应的值
         if (propVal == null && this.valueSeparator != null) {  //没有找到placeholder对应的值，则可能出现存在默认值得情况 如#{key:defaultVal}
            int separatorIndex = placeholder.indexOf(this.valueSeparator); //查找字符串中的分隔符。
            if (separatorIndex != -1) {
               //存在分隔符，则在尝试用真实key寻找对应值，如果不存在则使用默认值
               String actualPlaceholder = placeholder.substring(0, separatorIndex);
               String  defaultValue = placeholder.substring(separatorIndex + this.valueSeparator.length());
               propVal = placeholderResolver.resolvePlaceholder(actualPlaceholder);
               if (propVal == null) {
                  propVal = defaultValue;
               }
            }
         }
         if (propVal != null) {
            // Recursive invocation, parsing placeholders contained in the
            // previously resolved placeholder value.
            //当找到placeholder对应的值后，需要在对这个值中嵌套的placeholder进行处理,进行递归调用
            propVal = parseStringValue(propVal, placeholderResolver, visitedPlaceholders);

            //替换placeholder
            result.replace(startIndex, endIndex + this.placeholderSuffix.length(), propVal);
            if (logger.isTraceEnabled()) {
               logger.trace("Resolved placeholder '" + placeholder + "'");
            }
            startIndex = result.indexOf(this.placeholderPrefix, startIndex + propVal.length());
         }
         else if (this.ignoreUnresolvablePlaceholders) {
            // Proceed with unprocessed value.
            //忽略不能解析的占位符，
            startIndex = result.indexOf(this.placeholderPrefix, endIndex + this.placeholderSuffix.length());
         }
         else {
            throw new IllegalArgumentException("Could not resolve placeholder '" +
                  placeholder + "'" + " in value \"" + value + "\"");
         }
         visitedPlaceholders.remove(originalPlaceholder);
      }
      else {
         //没有结束符则将前缀作为placeholder的一部分
         startIndex = -1;
      }
   }

   return result.toString();
}
```

这是一个递归的过程，依次对字符串中的占位符进行解析，如果找到的资源中还存在占位符，那么会先对资源中的占位符进行解析，然后在替换原字符串中的占位符，以此类推，当出现循环引用时,如"placeholder=${placeholder} 则会抛出异常。

