# SpringApplication 初始化

基于 springboot3.2 和 springframework6.1.4

## 初始化代码解释

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    this.bootstrapRegistryInitializers = new ArrayList<>(
            getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

这是来自Spring Boot框架的`SpringApplication`类的代码片段。该构造函数用于创建`SpringApplication`
的新实例，这是一个关键的类，负责引导和启动Spring应用程序。

让我们分解一下这个构造函数的关键方面：

1. **资源加载器（Resource Loader）**：`resourceLoader`参数表示要由应用程序上下文使用的资源加载器。它有助于从不同来源加载资源。

2. **主要来源（Primary Sources）**：`primarySources`参数表示主要的bean来源。这些类被视为配置Spring应用程序上下文的主要来源。

3. **Web应用程序类型（Web Application Type）**：`webApplicationType`是从类路径中推断出来的。它表示Web应用程序的类型（例如，Servlet、Reactive、None）。

4. **引导注册表初始化器（Bootstrap Registry Initializers）**：`bootstrapRegistryInitializers`
   是`BootstrapRegistryInitializer`实例的列表。它们负责在应用程序引导阶段自定义`BootstrapRegistry`。

5. **初始化器和监听器（Initializers and Listeners）**：构造函数通过使用`getSpringFactoriesInstances`
   从Spring工厂获取实例来设置初始化器和监听器。

```properties
    org.springframework.context.ApplicationContextInitializer=\
    org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer,\
    org.springframework.boot.context.ContextIdApplicationContextInitializer,\
    org.springframework.boot.context.config.DelegatingApplicationContextInitializer,\
    org.springframework.boot.rsocket.context.RSocketPortInfoApplicationContextInitializer,\
    org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
org.springframework.context.ApplicationListener=\
    org.springframework.boot.ClearCachesApplicationListener,\
    org.springframework.boot.builder.ParentContextCloserApplicationListener,\
    org.springframework.boot.context.FileEncodingApplicationListener,\
    org.springframework.boot.context.config.AnsiOutputApplicationListener,\
    org.springframework.boot.context.config.DelegatingApplicationListener,\
    org.springframework.boot.context.logging.LoggingApplicationListener,\
    org.springframework.boot.env.EnvironmentPostProcessorApplicationListener
```

6. **主应用程序类（Main Application Class）**：`mainApplicationClass`是通过推断得出的，表示应用程序的主类。

这个构造函数设置了`SpringApplication`实例的初始配置。在创建实例后，开发人员可以在调用`run`方法启动Spring应用程序之前进一步定制它。

## WebApplicationType

```java
private static final String[] SERVLET_INDICATOR_CLASSES = {"jakarta.servlet.Servlet", "org.springframework.web.context.ConfigurableWebApplicationContext"};

private static final String WEBMVC_INDICATOR_CLASS = "org.springframework.web.servlet.DispatcherServlet";

private static final String WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler";

private static final String JERSEY_INDICATOR_CLASS = "org.glassfish.jersey.servlet.ServletContainer";

static WebApplicationType deduceFromClasspath() {
    if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
            && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
        return WebApplicationType.REACTIVE;
    }
    for (String className : SERVLET_INDICATOR_CLASSES) {
        if (!ClassUtils.isPresent(className, null)) {
            return WebApplicationType.NONE;
        }
    }
    return WebApplicationType.SERVLET;
}
```

## getSpringFactoriesInstances

loadFactoriesResource 读取 public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories"
资源文件内容，以Map<String, List<String>>返回。
static final Map<ClassLoader, Map<String, SpringFactoriesLoader>> cache = new ConcurrentReferenceHashMap<>();
classLoader -> resourceLocation -> SpringFactoriesLoader。
然后通过SpringFactoriesLoader里面的Map<String, List<String>> factories获取接口的实现类。接口的 className —>
接口实现类的className。

```java

private <T> List<T> getSpringFactoriesInstances(Class<T> type, ArgumentResolver argumentResolver) {
    return SpringFactoriesLoader.forDefaultResourceLocation(getClassLoader()).load(type, argumentResolver);
}

public static SpringFactoriesLoader forDefaultResourceLocation(@Nullable ClassLoader classLoader) {
    return forResourceLocation(FACTORIES_RESOURCE_LOCATION, classLoader);
}

public static SpringFactoriesLoader forResourceLocation(String resourceLocation, @Nullable ClassLoader classLoader) {
    Assert.hasText(resourceLocation, "'resourceLocation' must not be empty");
    ClassLoader resourceClassLoader = (classLoader != null ? classLoader : SpringFactoriesLoader.class.getClassLoader());
    Map<String, SpringFactoriesLoader> loaders = cache.computeIfAbsent(resourceClassLoader, key -> new ConcurrentReferenceHashMap<>());
    return loaders.computeIfAbsent(resourceLocation, key -> new SpringFactoriesLoader(classLoader, loadFactoriesResource(resourceClassLoader, resourceLocation)));
}
```

`loaders.computeIfAbsent(resourceLocation, key -> new SpringFactoriesLoader(classLoader, loadFactoriesResource(resourceClassLoader, resourceLocation)))`
这段代码使用了Java 8的`computeIfAbsent`方法，它是`ConcurrentMap`接口的一个方法，用于根据给定的键计算值并将其插入到Map中（如果缺少该键的映射）。

具体来说，这段代码的作用是在`loaders`这个`ConcurrentMap`中，根据`resourceLocation`
这个键，计算并插入一个值。这个值是通过使用lambda表达式 `key -> new SpringFactoriesLoader(classLoader, loadFactoriesResource(resourceClassLoader, resourceLocation))`
计算得到的。

解释一下lambda表达式的部分：

- `key` 是 `resourceLocation`，即在`loaders`中要查找或插入的键。
- `new SpringFactoriesLoader(classLoader, loadFactoriesResource(resourceClassLoader, resourceLocation))`
  创建了一个`SpringFactoriesLoader`对象。这个对象的构造函数接受两个参数：`classLoader`
  和`loadFactoriesResource(resourceClassLoader, resourceLocation)`。其中，`loadFactoriesResource`方法用于加载资源，返回一个表示资源的对象。

总体来说，这段代码的目的是利用`computeIfAbsent`方法，根据`resourceLocation`计算出一个`SpringFactoriesLoader`
对象，并将其插入到`loaders`中，如果`loaders`中已存在该键对应的值，则直接返回已有的值而不重新计算。

从`SpringFactoriesLoader`中 `Map<String, List<String>> factories`获取接口的实现类名,并实例化以后返回。

```java
public <T> List<T> load(Class<T> factoryType, @Nullable ArgumentResolver argumentResolver, @Nullable FailureHandler failureHandler) {

    Assert.notNull(factoryType, "'factoryType' must not be null");
    List<String> implementationNames = loadFactoryNames(factoryType);
    logger.trace(LogMessage.format("Loaded [%s] names: %s", factoryType.getName(), implementationNames));
    List<T> result = new ArrayList<>(implementationNames.size());
    FailureHandler failureHandlerToUse = (failureHandler != null) ? failureHandler : THROWING_FAILURE_HANDLER;
    for (String implementationName : implementationNames) {
        T factory = instantiateFactory(implementationName, factoryType, argumentResolver, failureHandlerToUse);
        if (factory != null) {
            result.add(factory);
        }
    }
    AnnotationAwareOrderComparator.sort(result);
    return result;
}
```

## 补充知识

### Lambda表达式

从Java 8的角度解释Lambda表达式，Lambda表达式是一种轻量级的语法，用于在需要函数接口（Functional
Interface）的地方提供一个简洁的方式来定义匿名函数。

以下是Lambda表达式的一般语法：

```java
(parameters)->expression
```

或者

```java
(parameters)->{statements; }
```

其中：

- `parameters` 指的是Lambda表达式的参数列表，类似于方法的参数列表。
- `->` 是Lambda运算符，分隔参数列表和Lambda表达式的主体。
- `expression` 或 `{ statements; }` 是Lambda表达式的主体，可以是单个表达式或一个代码块。

举例来说，考虑一个简单的Lambda表达式，用于对两个数进行相加：

```java
(int a, int b)->a +b
```

在这里：

- `(int a, int b)` 是参数列表，表明Lambda表达式接受两个整数参数。
- `->` 是Lambda运算符。
- `a + b` 是表达式，表示对两个整数进行相加的操作。

Lambda表达式的主要用途是在函数接口的上下文中提供一种简洁的方式来创建匿名函数。函数接口是只包含一个抽象方法的接口。例如，`Runnable`、`Callable`
等都是函数接口。在Lambda表达式中，可以直接传递行为（函数）而不是实例化一个匿名类。

下面是一个例子，使用Lambda表达式创建一个线程：

```java
// 使用匿名内部类
Runnable runnable1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello from anonymous class!");
    }
};

// 使用Lambda表达式
Runnable runnable2 = () -> {
    System.out.println("Hello from Lambda!");
};
```

在这个例子中，`runnable2`使用Lambda表达式实现了`Runnable`接口，取代了传统的匿名内部类的写法。Lambda表达式的引入使得代码更为简洁和易读。

### computeIfAbsent

`computeIfAbsent` 是Java 8中 `Map` 接口引入的一个方法，用于在映射中根据指定的键计算值并插入到映射中，如果该键尚未与值关联或与null关联。

```java
V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction)
```

- `key` 是要计算值的键。
- `mappingFunction` 是一个用于计算值的函数接口，接收键作为参数，返回与键相关联的值。

该方法的行为如下：

- 如果指定的键在映射中不存在（或与null关联），则计算值并将其与键关联。
- 如果计算的值不为null，则将其与键关联。
- 如果计算的值为null，则不会将其与键关联，并且不会更改映射。

以下是一个简单的示例，演示了如何使用 `computeIfAbsent` 方法：

```java
import java.util.HashMap;
import java.util.Map;

public class ComputeIfAbsentExample {
    public static void main(String[] args) {
        Map<String, Integer> map = new HashMap<>();

        // 使用 computeIfAbsent 插入键值对
        map.computeIfAbsent("one", key -> 1);
        map.computeIfAbsent("two", key -> 2);

        System.out.println(map); // Output: {one=1, two=2}

        // 如果键已存在，computeIfAbsent 不会改变映射
        map.computeIfAbsent("one", key -> 10);

        System.out.println(map); // Output: {one=1, two=2}
    }
}
```

在上述示例中，`computeIfAbsent` 方法被用于向映射中插入键值对。如果键已经存在，该方法不会改变映射。这是一种方便的方式，以便在映射中插入新的键值对，而无需手动检查键是否存在。