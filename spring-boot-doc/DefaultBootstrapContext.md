# DefaultBootstrapContext

基于 springboot3.2 和 springframework6.1.4

```java
private DefaultBootstrapContext createBootstrapContext() {
    DefaultBootstrapContext bootstrapContext = new DefaultBootstrapContext();
    this.bootstrapRegistryInitializers.forEach((initializer) -> initializer.initialize(bootstrapContext));
    return bootstrapContext;
}
```

该类是`ConfigurableBootstrapContext`接口的实现。它在Spring Boot应用程序的引导阶段使用，提供了一种注册和检索在引导过程中需要的对象实例的方式。

该类维护了两个`HashMap`实例，分别是`instanceSuppliers`和`instances`。`instanceSuppliers`映射将`InstanceSupplier`
对象与它们对应的类类型存储在一起。`InstanceSupplier`是一个功能接口，用于提供特定类型的实例。`instances`映射存储了对象的实际实例。

```java
private final Map<Class<?>, InstanceSupplier<?>> instanceSuppliers = new HashMap<>();
private final Map<Class<?>, Object> instances = new HashMap<>();
```

`register`和`registerIfAbsent`方法用于为特定类型注册`InstanceSupplier`。`register`
方法将替换已经为该类型注册的`InstanceSupplier`，而`registerIfAbsent`仅在尚未注册`InstanceSupplier`时才注册。

```java
public <T> void register(Class<T> type, InstanceSupplier<T> instanceSupplier) {...}

public <T> void registerIfAbsent(Class<T> type, InstanceSupplier<T> instanceSupplier) {...}
```

`isRegistered`方法检查是否为特定类型注册了`InstanceSupplier`。

```java
public <T> boolean isRegistered(Class<T> type) {...}
```

`get`、`getOrElse`和`getOrElseSupply`方法用于检索特定类型的实例。如果实例不存在，这些方法将使用已注册的`InstanceSupplier`
来创建一个。`get`方法如果未为该类型注册`InstanceSupplier`，将抛出异常，而`getOrElse`和`getOrElseSupply`
则分别返回默认值或使用`Supplier`提供默认值。

```java
public <T> T get(Class<T> type) throws IllegalStateException {...}

public <T> T getOrElse(Class<T> type, T other) {...}

public <T> T getOrElseSupply(Class<T> type, Supplier<T> other) {...}
```

`close`方法在`BootstrapContext`关闭且`ApplicationContext`
准备就绪时被调用。它向所有已注册的监听器广播`BootstrapContextClosedEvent`。

```java
public void close(ConfigurableApplicationContext applicationContext) {...}
```

总之，该类提供了一种在Spring Boot应用程序的引导阶段注册和检索对象实例的机制，允许进行灵活且可配置的引导过程。

一旦你在 `DefaultBootstrapContext` 中注册了自定义的初始化器，它将在应用程序引导（bootstrap）阶段执行。注册的初始化器会影响
Spring Boot 启动过程中的一些关键组件，如 `ConversionService`、`FormatterRegistry`
等。在初始化器中，你可以进行一些自定义的配置或注册一些特定的组件，以满足应用程序的需求。

以下是一个简单的例子，演示了如何在初始化器中注册自定义的转换器（`Converter`）：

```java
import org.springframework.boot.BootstrapContext;
import org.springframework.boot.DefaultBootstrapContext;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.WebApplicationType;
import org.springframework.core.convert.converter.Converter;
import org.springframework.format.FormatterRegistry;

public class CustomBootstrapExample {

    public static void main(String[] args) {
        // 创建 DefaultBootstrapContext 对象
        BootstrapContext bootstrapContext = new DefaultBootstrapContext();

        // 添加自定义的初始化器
        bootstrapContext.addBootstrapRegistryInitializer(registry -> {
            // 在这里可以注册自定义的转换器、格式化器等
            registry.register(MyCustomConverter.class);
        });

        // 创建 SpringApplication 对象并设置引导上下文
        SpringApplication application = new SpringApplication();
        application.setBootstrapContext(bootstrapContext);

        // 设置 WebApplicationType（例如，Reactive、Servlet）
        application.setWebApplicationType(WebApplicationType.SERVLET);

        // 运行应用程序
        SpringApplication.run(MySpringBootApplication.class, args);
    }

    static class MyCustomConverter implements Converter<String, Integer> {
        @Override
        public Integer convert(String source) {
            // 自定义转换逻辑
            return Integer.parseInt(source);
        }
    }

    static class MySpringBootApplication {
        // Spring Boot 应用程序的主类
    }
}
```

在这个例子中，`MyCustomConverter` 类实现了 Spring 的 `Converter` 接口，负责将 `String` 类型转换为 `Integer`
类型。通过在 `DefaultBootstrapContext` 中注册了一个初始化器，我们成功地将这个自定义的转换器注册到了引导过程中，从而影响了应用程序的启动行为。

请注意，这只是一个简单的示例，实际中你可以在初始化器中执行更复杂的自定义逻辑，包括注册其他类型的组件、修改配置等。自定义初始化器提供了一个扩展点，使得在应用程序引导阶段进行更多的自定义成为可能。