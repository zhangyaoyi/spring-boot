# ConfigurationClassPostProcessor

基于 springboot3.2 和 springframework6.1.4

## 1. ConfigurationClassPostProcessor的注册过程

`ConfigurationClassPostProcessor` 是 Spring Framework 中的一个关键后置处理器,用于处理 `@Configuration` 注解的类,并生成相应的 Bean 定义。下面是 `ConfigurationClassPostProcessor` 的加载过程:

1. 注册 `ConfigurationClassPostProcessor`:
    - 通过调用 `AnnotationConfigUtils.registerAnnotationConfigProcessors(registry)` 方法,将 `ConfigurationClassPostProcessor` 注册到 `BeanDefinitionRegistry` 中。
    - 这通常在应用程序上下文的初始化过程中自动完成,例如在 `AnnotationConfigServletWebServerApplicationContext` 的构造函数中。

```java
    public AnnotationConfigServletWebServerApplicationContext() {
	this.reader = new AnnotatedBeanDefinitionReader(this);
	this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

```java
    public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
	Assert.notNull(environment, "Environment must not be null");
	this.registry = registry;
	this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
	AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```  

2. `BeanFactoryPostProcessor` 的执行:
    - `ConfigurationClassPostProcessor` 实现了 `BeanFactoryPostProcessor` 接口。
    - 在 Spring 容器的刷新过程中,会调用所有已注册的 `BeanFactoryPostProcessor` 的 `postProcessBeanFactory` 方法。
    - `ConfigurationClassPostProcessor` 的 `postProcessBeanFactory` 方法会被调用,开始处理配置类。

3. 解析配置类:
    - `ConfigurationClassPostProcessor` 通过调用 `ConfigurationClassParser` 来解析标记了 `@Configuration` 注解的类。
    - 解析过程会递归地处理配置类,包括其中的 `@Bean`、`@Import`、`@ComponentScan` 等注解。
    - 对于 `@ComponentScan` 注解,会根据指定的扫描路径和过滤条件扫描相应的类,并将其作为候选的 Bean 定义。

4. 处理 `@Import` 注解:
    - 对于配置类中的 `@Import` 注解,`ConfigurationClassPostProcessor` 会进一步处理导入的类。
    - 如果导入的类是另一个配置类,则递归地解析该配置类。
    - 如果导入的类是 `ImportSelector` 或 `ImportBeanDefinitionRegistrar` 的实现类,则调用相应的方法来注册额外的 Bean 定义。

5. 处理 `@Bean` 注解:
    - 对于配置类中的 `@Bean` 注解标记的方法,`ConfigurationClassPostProcessor` 会为每个方法生成一个 Bean 定义(`BeanDefinition`)。
    - Bean 定义包含了 Bean 的类型、作用域、依赖关系等信息,用于后续的 Bean 实例化和依赖注入。

6. 注册 Bean 定义:
    - 在解析和处理完所有的配置类后,`ConfigurationClassPostProcessor` 会将生成的 Bean 定义注册到 `BeanDefinitionRegistry` 中。
    - 这样,这些 Bean 定义就可以参与 Spring 容器的 Bean 创建和管理过程。

7. 后置处理 Bean 定义:
    - `ConfigurationClassPostProcessor` 还会对已注册的 Bean 定义进行后置处理。
    - 它会处理配置类中的 `@Bean` 方法中的特殊参数,如 `@Lazy`、`@Primary`、`@DependsOn` 等注解,并相应地修改 Bean 定义的属性。

8. 完成后置处理:
    - 在完成配置类的解析、Bean 定义的注册和后置处理后,`ConfigurationClassPostProcessor` 的工作就完成了。
    - Spring 容器会继续进行 Bean 的实例化、依赖注入和初始化等后续步骤。

通过 `ConfigurationClassPostProcessor` 的加载过程,Spring 实现了对 `@Configuration` 注解的类的处理,将其转换为 Bean 定义,并将其集成到 Spring 容器的管理中。这使得基于注解的配置成为 Spring
应用程序开发的主流方式之一,提供了更加灵活和可读性更好的配置方式。

## 2. ConfigurationClassParser

上述代码是 `ConfigurationClassParser` 类中的 `parse` 方法,用于解析配置类候选对象并生成相应的 Bean 定义。让我们逐步分析代码的作用:

1. 方法签名: `public void parse(Set<BeanDefinitionHolder> configCandidates)`
    - 该方法接受一个 `BeanDefinitionHolder` 对象的集合作为参数,表示配置类的候选对象。

2. 遍历配置类候选对象:
    - 使用 `for` 循环遍历 `configCandidates` 集合中的每个 `BeanDefinitionHolder` 对象。
    - `BeanDefinitionHolder` 是一个包装类,包含了 `BeanDefinition` 对象和相应的 Bean 名称。

3. 获取 `BeanDefinition` 对象:
    - 通过调用 `holder.getBeanDefinition()` 获取当前 `BeanDefinitionHolder` 对象关联的 `BeanDefinition` 对象。

4. 根据 `BeanDefinition` 的类型进行解析:
    - 如果 `BeanDefinition` 是 `AnnotatedBeanDefinition` 类型:
        - 调用 `parse(annotatedBeanDef.getMetadata(), holder.getBeanName())` 方法解析注解元数据。
        - `AnnotatedBeanDefinition` 表示通过注解定义的 Bean,可以访问其注解元数据。
    - 如果 `BeanDefinition` 是 `AbstractBeanDefinition` 类型并且具有 `beanClass`:
        - 调用 `parse(abstractBeanDef.getBeanClass(), holder.getBeanName())` 方法解析 Bean 的类型。
        - `AbstractBeanDefinition` 是 `BeanDefinition` 的抽象基类,`hasBeanClass()` 方法用于判断是否存在 Bean 的类型信息。
    - 否则,假设 `BeanDefinition` 具有 `beanClassName`:
        - 调用 `parse(bd.getBeanClassName(), holder.getBeanName())` 方法解析 Bean 的类名。

5. 处理解析过程中的异常:
    - 如果在解析过程中抛出了 `BeanDefinitionStoreException`,直接重新抛出该异常。
    - 如果抛出了其他异常,则包装成 `BeanDefinitionStoreException` 并抛出,提供更具体的错误信息,包括出错的配置类名称。

6. 处理延迟导入选择器:
    - 在解析完所有的配置类候选对象后,调用 `this.deferredImportSelectorHandler.process()` 方法处理延迟导入选择器。
    - 延迟导入选择器是一种特殊的导入选择器,它们的处理被延迟到所有常规配置类都处理完之后。

总的来说,这段代码的作用是遍历配置类候选对象,根据不同的 `BeanDefinition` 类型调用相应的解析方法,将配置类转换为 Bean 定义。在解析过程中,它还处理了可能出现的异常,并在最后处理延迟导入选择器。

这是 `ConfigurationClassParser` 类的核心逻辑之一,用于将注解配置类转换为 Spring 容器可以管理的 Bean 定义,为后续的 Bean 实例化和依赖注入做准备。

```java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
	for (BeanDefinitionHolder holder : configCandidates) {
		BeanDefinition bd = holder.getBeanDefinition();
		try {
			if (bd instanceof AnnotatedBeanDefinition annotatedBeanDef) {
				parse(annotatedBeanDef.getMetadata(), holder.getBeanName());
			}
			else if (bd instanceof AbstractBeanDefinition abstractBeanDef && abstractBeanDef.hasBeanClass()) {
				parse(abstractBeanDef.getBeanClass(), holder.getBeanName());
			}
			else {
				parse(bd.getBeanClassName(), holder.getBeanName());
			}
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(
					"Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
		}
	}

	this.deferredImportSelectorHandler.process();
}
```

## 3. AnnotationConfigUtils.registerAnnotationConfigProcessors

`AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry)` 是 Spring Framework 中的一个重要方法,用于在应用程序上下文中注册与注解配置相关的后置处理器和 Bean 工厂后置处理器。

这个方法的主要作用是确保在使用注解配置的 Spring 应用程序中,所有必要的后置处理器和 Bean 工厂后置处理器都被正确地注册,以支持注解驱动的配置和 Bean 定义。

以下是 `AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry)` 方法所执行的一些关键任务:

1. 注册 `ConfigurationClassPostProcessor`:
    - `ConfigurationClassPostProcessor` 是一个关键的后置处理器,用于处理 `@Configuration` 注解的类,并生成相应的 Bean 定义。
    - 它负责解析配置类、处理 `@Bean`、`@Import`、`@ComponentScan` 等注解,并将生成的 Bean 定义注册到容器中。

2. 注册 `AutowiredAnnotationBeanPostProcessor`:
    - `AutowiredAnnotationBeanPostProcessor` 是一个 Bean 后置处理器,用于处理 `@Autowired`、`@Value`、`@Inject` 等注解。
    - 它负责在 Bean 实例化后,对标记了这些注解的字段、方法和构造函数进行自动注入。

3. 注册 `CommonAnnotationBeanPostProcessor`:
    - `CommonAnnotationBeanPostProcessor` 是一个 Bean 后置处理器,用于处理 JSR-250 标准注解,如 `@PostConstruct`、`@PreDestroy`、`@Resource` 等。
    - 它负责在 Bean 的生命周期中调用标记了这些注解的方法,如在 Bean 初始化后调用 `@PostConstruct` 标记的方法。

4. 注册 `PersistenceAnnotationBeanPostProcessor`（如果存在 JPA）:
    - 如果应用程序的类路径中存在 JPA（Java Persistence API）相关的类,则会注册 `PersistenceAnnotationBeanPostProcessor`。
    - 这个后置处理器用于处理 JPA 相关的注解,如 `@PersistenceContext`、`@PersistenceUnit` 等,以支持 JPA 的集成。

5. 注册 `EventListenerMethodProcessor`:
    - `EventListenerMethodProcessor` 是一个后置处理器,用于处理 `@EventListener` 注解标记的方法。
    - 它负责将这些方法注册为事件监听器,以便在相应的事件发布时调用它们。

6. 注册 `DefaultEventListenerFactory`:
    - `DefaultEventListenerFactory` 是一个默认的事件监听器工厂,用于创建事件监听器的代理对象。
    - 它与 `EventListenerMethodProcessor` 协同工作,用于支持事件监听器的创建和调用。

通过调用 `AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry)`,Spring 确保了这些关键的后置处理器和 Bean 工厂后置处理器被正确注册,从而支持注解驱动的配置和 Bean 定义。这使得开发者可以使用注解来配置和定义
Bean,简化了 Spring 应用程序的开发和配置过程。

## 4. CachingMetadataReaderFactoryPostProcessor

**1. SharedMetadataReaderFactoryContextInitializer**

`spring-boot-project/spring-boot-autoconfigure/src/main/resources/META-INF/spring.factories`中定义了`ApplicationContextInitializer`。`SharedMetadataReaderFactoryContextInitializer`
实现了`ApplicationContextInitializer`接口。在`SpringApplication`构造器中，会被加入到`List<ApplicationContextInitializer<?>> initializers`中。当ApplicationContext初始化时，在`prepareConext`的`applyInitializers`
会调用`initialize`方法。执行完`initialize`方法，`CachingMetadataReaderFactoryPostProcessor`会被加入到`BeanFactoryPostProcessor`中。

```properties
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener
```

```java

@Override
public void initialize(ConfigurableApplicationContext applicationContext) {
	if (AotDetector.useGeneratedArtifacts()) {
		return;
	}
	BeanFactoryPostProcessor postProcessor = new CachingMetadataReaderFactoryPostProcessor(applicationContext);
	applicationContext.addBeanFactoryPostProcessor(postProcessor);
}
```

**2. CachingMetadataReaderFactoryPostProcessor**
在`AbtractApplicationContext`的`refresh`方法中，会调用`invokeBeanFactoryPostProcessors`方法。`CachingMetadataReaderFactoryPostProcessor`的`postProcessBeanDefinitionRegistry`会被执行。

`CachingMetadataReaderFactoryPostProcessor` 中的 `postProcessBeanDefinitionRegistry` 方法在 Bean 定义注册阶段被调用,其目的如下:

1. 创建 `CachingMetadataReaderFactory`:
    - `CachingMetadataReaderFactory` 是一个元数据读取器工厂,用于读取类的元数据信息,如类的注解、类名、接口等。
    - 它内部使用了缓存机制,可以提高元数据读取的性能,避免重复读取相同的类元数据。
      2将 `CachingMetadataReaderFactory` 设置到 `ConfigurationClassPostProcessor`:
    - `ConfigurationClassPostProcessor` 是一个重要的后置处理器,用于处理 `@Configuration` 类和执行配置类的增强。
    - 通过将 `CachingMetadataReaderFactory` 设置到 `ConfigurationClassPostProcessor` 中,可以在处理配置类时利用缓存的元数据读取器,加速配置类的解析和处理。

总的来说,`CachingMetadataReaderFactoryPostProcessor` 的 `postProcessBeanDefinitionRegistry` 方法的主要目的是创建并设置 `CachingMetadataReaderFactory`,以优化 Spring 容器在处理 Bean
定义和配置类时的性能。通过使用缓存的元数据读取器,可以避免重复读取相同的类元数据,从而加速容器的启动过程和 Bean 的实例化。

这个后置处理器是 Spring Framework 内部使用的一个优化手段,对于一般的 Spring 应用程序开发者来说,通常不需要直接接触或配置它。Spring 会自动应用这个后置处理器,以提供更好的性能和效率。
