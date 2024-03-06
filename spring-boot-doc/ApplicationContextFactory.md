# ApplicationContextFactory

基于 springboot3.2 和 springframework6.1.4

## ApplicationContextFactory初始化

在`SpringApplication`中初始化`ApplicationContextFactory`

```java
private ApplicationContextFactory applicationContextFactory = ApplicationContextFactory.DEFAULT;
```

在`ApplicationContextFactory`中创建了`DefaultApplicationContextFactory`实例

```java
ApplicationContextFactory DEFAULT = new DefaultApplicationContextFactory();
```

## DefaultApplicationContextFactory

这个类的主要功能是根据不同的 `WebApplicationType` 创建相应的 `ConfigurableApplicationContext`实例，并提供了一些方法用于获取环境类型和创建环境。同时，它使用了 `SpringFactoriesLoader` 来加载 `ApplicationContextFactory`实例，并在一系列实例中执行特定的操作。

1. **类的方法：**
    - **`getEnvironmentType` 方法：**
      ```java
      @Override
      public Class<? extends ConfigurableEnvironment> getEnvironmentType(WebApplicationType webApplicationType) {
          return getFromSpringFactories(webApplicationType, ApplicationContextFactory::getEnvironmentType, null);
      }
      ```
      该方法根据 `WebApplicationType` 获取环境类型，并通过 `getFromSpringFactories`方法执行 `ApplicationContextFactory::getEnvironmentType` 操作。

    - **`createEnvironment` 方法：**
      ```java
      @Override
      public ConfigurableEnvironment createEnvironment(WebApplicationType webApplicationType) {
          return getFromSpringFactories(webApplicationType, ApplicationContextFactory::createEnvironment, null);
      }
      ```
      该方法根据 `WebApplicationType` 创建环境，并通过 `getFromSpringFactories`方法执行 `ApplicationContextFactory::createEnvironment` 操作。

    - **`create` 方法：**
      ```java
      @Override
      public ConfigurableApplicationContext create(WebApplicationType webApplicationType) {
          try {
              return getFromSpringFactories(webApplicationType, ApplicationContextFactory::create,
                      this::createDefaultApplicationContext);
          }
          catch (Exception ex) {
              throw new IllegalStateException("Unable create a default ApplicationContext instance, "
                      + "you may need a custom ApplicationContextFactory", ex);
          }
      }
      ```
      该方法根据 `WebApplicationType` 创建应用程序上下文，并通过 `getFromSpringFactories`方法执行 `ApplicationContextFactory::create` 操作。如果发生异常，则抛出 `IllegalStateException`。

    - **`createDefaultApplicationContext` 方法：**
      ```java
      private ConfigurableApplicationContext createDefaultApplicationContext() {
          if (!AotDetector.useGeneratedArtifacts()) {
              return new AnnotationConfigApplicationContext();
          }
          return new GenericApplicationContext();
      }
      ```
      私有方法，根据AOT（Ahead-of-Time）检测决定创建 `AnnotationConfigApplicationContext` 或 `GenericApplicationContext`。

    - **`getFromSpringFactories` 方法：**
      ```java
      private <T> T getFromSpringFactories(WebApplicationType webApplicationType,
              BiFunction<ApplicationContextFactory, WebApplicationType, T> action, Supplier<T> defaultResult) {
          for (ApplicationContextFactory candidate : SpringFactoriesLoader.loadFactories(ApplicationContextFactory.class,
                  getClass().getClassLoader())) {
              T result = action.apply(candidate, webApplicationType);
              if (result != null) {
                  return result;
              }
          }
          return (defaultResult != null) ? defaultResult.get() : null;
      }
      ```
      私有方法，根据 `SpringFactoriesLoader` 加载的 `ApplicationContextFactory`实例执行操作。首先尝试执行传入的操作，如果操作返回非空结果，则返回该结果。如果没有找到匹配的实例，则返回默认结果。

2. **ApplicationContext的实例**

   - 如果 webApplicationType 是 `WebApplicationType.NONE`，则创建 `AnnotationConfigApplicationContext`
     实例或创建 `GenericApplicationContext` 实例。
   - 如果 webApplicationType 是 `WebApplicationType.SERVLET`，则创建 `AnnotationConfigServletWebServerApplicationContext`
     实例或创建`ServletWebServerApplicationContext`。
   - 如果 webApplicationType 是 `WebApplicationType.REACTIVE`，则创建 `AnnotationConfigReactiveWebServerApplicationContext`
     实例或创建`ReactiveWebServerApplicationContext`实例。

## BiFunction<ApplicationContextFactory, WebApplicationType, T>类型

1. `BiFunction`:
   这是一个Java函数式接口，表示接受两个输入参数并产生一个结果的函数。在这里，它接受两个参数：`ApplicationContextFactory`和 `WebApplicationType`。

2. `<ApplicationContextFactory, WebApplicationType, T>`: 这是 `BiFunction`的泛型参数。它表示该函数接受一个 `ApplicationContextFactory` 类型的参数（第一个参数），一个 `WebApplicationType`类型的参数（第二个参数），并返回一个泛型类型 `T` 的结果。

在方法的实际使用中，`action` 参数代表了一个操作，由调用方提供。这个操作将在 `SpringFactoriesLoader`加载的每个 `ApplicationContextFactory` 实例上执行，以及传递的 `WebApplicationType` 参数。

例如，调用方可以通过提供一个 lambda 表达式或方法引用来定义具体的操作。下面是一个示例，展示了如何使用该方法并提供一个简单的 lambda 表达式：

```java
WebApplicationType webApplicationType = ...;  // 某个Web应用程序类型
Supplier<T> defaultResult = ...;  // 默认结果的提供者

T result = getFromSpringFactories(webApplicationType,
        (factory, type) -> {
            // 具体的操作逻辑，可能使用 factory 和 type 进行一些操作
            return factory.createApplicationContext(type);
        },
        defaultResult
);
```

在这个例子中，`action` 是一个 lambda 表达式，接受 `ApplicationContextFactory` 和 `WebApplicationType` 作为参数，并返回一个结果。这个 lambda 表达式定义了一个操作，可能是创建一个应用程序上下文（`ApplicationContext`）的实例，具体逻辑取决于调用方的需求。

总的来说，`action` 是一个允许调用方定义具体操作的函数，这样 `getFromSpringFactories`方法可以执行这个操作，并返回结果。这种设计提供了灵活性，使得方法可以用于不同的场景，并允许调用方根据需要自定义操作。