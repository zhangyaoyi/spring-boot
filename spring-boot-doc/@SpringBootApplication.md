# @SpringBootApplication

基于 springboot3.2 和 springframework6.1.4

## @SpringBootApplication的三大功能

`@SpringBootApplication` 是 Spring Boot 提供的一个方便的组合注解,用于简化 Spring Boot 应用程序的配置和启动。它combines了以下三个注解的功能:

1. `@Configuration`:
    - 标识该类是一个配置类,可以包含 Bean 定义的方法。
    - 它是 Spring 框架的核心注解,用于定义配置类和 Bean 的创建。

2. `@EnableAutoConfiguration`:
    - 启用 Spring Boot 的自动配置机制。
    - Spring Boot 会根据类路径中的依赖和定义的 Bean 来自动配置应用程序的各个方面,如数据库连接、MVC 配置等。
    - 自动配置的目的是减少手动配置的工作量,提供开箱即用的功能。

3. `@ComponentScan`:
    - 启用组件扫描,自动检测和注册 Spring 组件。
    - 默认情况下,它会扫描当前包及其子包中的组件,如 `@Component`、`@Service`、`@Repository` 等注解标记的类。
    - 通过组件扫描,Spring 可以自动发现和实例化应用程序中的组件,并将它们注册为 Bean。

除了组合这三个注解外,`@SpringBootApplication` 还提供了一些额外的功能和定制选项:

- 可以通过 `scanBasePackages` 或 `scanBasePackageClasses` 属性来指定要扫描的包或类,覆盖默认的扫描行为。
- 可以通过 `exclude` 或 `excludeName` 属性来排除特定的自动配置类或 Bean,从而定制自动配置的行为。

使用 `@SpringBootApplication` 注解标记主类(通常命名为 `Application` 或 `MainApplication`)有以下好处:

- 简化配置: 通过一个注解即可启用多个 Spring Boot 功能,减少了显式配置的需求。
- 自动配置: Spring Boot 会根据类路径中的依赖自动配置各种功能,如嵌入式服务器、数据库连接池等,提供开箱即用的体验。
- 组件扫描: 自动发现和注册应用程序中的 Spring 组件,无需手动配置。
- 易于启动: 使用 `@SpringBootApplication` 标记的主类可以直接运行,无需额外的配置或部署步骤。

总之,`@SpringBootApplication` 注解简化了 Spring Boot 应用程序的配置和启动过程,提供了自动配置和组件扫描等便利功能,使得开发者可以快速创建和运行 Spring Boot 应用程序。

```java

@SpringBootApplication
public class Application {
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
	//..............
}
```

### 1. `@Configuration`

`@Configuration` 是 Spring Framework 提供的一个重要注解,用于定义配置类并声明 Bean。它是 Spring 基于 Java 的配置方式的核心组件。

以下是 `@Configuration` 注解的主要作用:

1. 声明配置类:
    - 标记了 `@Configuration` 注解的类表示这是一个配置类,用于定义 Bean 和应用程序的配置。
    - 配置类通常包含一个或多个使用 `@Bean` 注解标记的方法,这些方法用于声明和配置 Bean。
    - Spring 容器会识别并处理标记了 `@Configuration` 的类,并根据其中的 `@Bean` 方法创建和管理相应的 Bean。

2. 定义 Bean:
    - 在配置类中,可以使用 `@Bean` 注解标记方法来定义 Bean。
    - `@Bean` 注解告诉 Spring 容器该方法返回的对象应该被注册为一个 Bean,并由容器管理其生命周期。
    - 通过在配置类中使用 `@Bean` 方法,可以以编程方式创建和配置 Bean,灵活地控制 Bean 的创建过程和依赖关系。

3. 组合和模块化配置:
    - `@Configuration` 类可以相互引用和组合,以创建模块化和可重用的配置。
    - 一个配置类可以通过在方法上标记 `@Bean` 注解来引用另一个配置类中定义的 Bean。
    - 通过组合多个 `@Configuration` 类,可以将应用程序的配置划分为不同的模块,提高配置的可读性和可维护性。

4. 启用依赖注入:
    - 在 `@Configuration` 类中,可以使用 `@Autowired`、`@Inject` 或 `@Resource` 等注解来注入依赖。
    - Spring 容器会根据这些注解自动进行依赖注入,将所需的 Bean 注入到配置类中。
    - 通过依赖注入,可以轻松地组装和配置复杂的对象关系,无需手动创建和管理依赖对象。

5. 支持条件化配置:
    - `@Configuration` 类可以与条件化注解（如 `@Conditional`）结合使用,以根据特定条件有条件地创建和配置 Bean。
    - 通过使用条件化注解,可以根据运行时环境、类路径上的依赖项或自定义条件来动态地启用或禁用配置类或 Bean 的创建。

6. 集成测试支持:
    - `@Configuration` 类可以在集成测试中使用,以配置测试所需的 Bean 和依赖项。
    - 通过在测试类中使用 `@ContextConfiguration` 注解并指定 `@Configuration` 类,可以加载测试所需的应用程序上下文和 Bean。
    - 这种方式使得编写集成测试变得更加简单和可控,无需依赖外部的配置文件。

总之,`@Configuration` 注解是 Spring 基于 Java 的配置方式的核心组件。它允许开发者使用 Java 类来定义和配置 Bean,提供了更加灵活和强大的配置能力。通过使用 `@Configuration` 类,可以模块化和组合配置,启用依赖注入,并支持条件化配置和集成测试。

使用 `@Configuration` 注解,开发者可以以编程方式控制应用程序的配置和组装,提高了配置的可读性、可维护性和灵活性。它是 Spring 基于 Java 的配置方式的基石,与其他注解（如 `@Bean`、`@Autowired` 等）配合使用,构成了 Spring
应用程序配置的完整生态系统。

```java
public class ConfigurationClassPostProcessor {

	public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
		//获取所有的bean names
		String[] candidateNames = registry.getBeanDefinitionNames();
		//遍历candidateNames，找出所有被标注@Configuration配置类
		if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
			configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
		}
		//创建配置类解析器ConfigurationClassParser
		ConfigurationClassParser parser = new ConfigurationClassParser(this.metadataReaderFactory, this.problemReporter, this.environment, this.resourceLoader, this.componentScanBeanNameGenerator, registry);
		//解析配置类
		parser.parse(candidates);
		//获取解析后的配置类ConfigurationClass集合
		Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
		//创建ConfigurationClassBeanDefinitionReader
		ConfigurationClassBeanDefinitionReader reader = new ConfigurationClassBeanDefinitionReader(registry, this.sourceExtractor, this.resourceLoader, this.environment, this.importBeanNameGenerator, parser.getImportRegistry());
		//加载配置类中的BeanDefinition，并注册到BeanDefinitionRegistry中，这里的BeanDefinitionRegistry就是DefaultListableBeanFactory
		this.reader.loadBeanDefinitions(configClasses);
	}
}

public class ConfigurationClassParser {

	protected void processConfigurationClass(ConfigurationClass configClass, Predicate<String> filter) throws IOException {
		if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
			return;
		}

		ConfigurationClass existingClass = this.configurationClasses.get(configClass);
		if (existingClass != null) {
			if (configClass.isImported()) {
				if (existingClass.isImported()) {
					existingClass.mergeImportedBy(configClass);
				}
				// Otherwise ignore new imported config class; existing non-imported class overrides it.
				return;
			}
			else {
				// Explicit bean definition found, probably replacing an import.
				// Let's remove the old one and go with the new one.
				this.configurationClasses.remove(configClass);
				this.knownSuperclasses.values().removeIf(configClass::equals);
			}
		}

		// Recursively process the configuration class and its superclass hierarchy.
		SourceClass sourceClass = null;
		try {
			sourceClass = asSourceClass(configClass, filter);
			do {
				sourceClass = doProcessConfigurationClass(configClass, sourceClass, filter);
			}
			while (sourceClass != null);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"I/O failure while processing configuration class [" + sourceClass + "]", ex);
		}

		this.configurationClasses.put(configClass, configClass);
	}
}
```

```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter) throws IOException {

	if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
		// Recursively process any member (nested) classes first
		processMemberClasses(configClass, sourceClass, filter);
	}

	// Process any @PropertySource annotations
	for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
			sourceClass.getMetadata(), org.springframework.context.annotation.PropertySource.class,
			PropertySources.class, true)) {
		if (this.propertySourceRegistry != null) {
			this.propertySourceRegistry.processPropertySource(propertySource);
		}
		else {
			logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
					"]. Reason: Environment must implement ConfigurableEnvironment");
		}
	}

	// Search for locally declared @ComponentScan annotations first.
	Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
			sourceClass.getMetadata(), ComponentScan.class, ComponentScans.class,
			MergedAnnotation::isDirectlyPresent);

	// Fall back to searching for @ComponentScan meta-annotations (which indirectly
	// includes locally declared composed annotations).
	if (componentScans.isEmpty()) {
		componentScans = AnnotationConfigUtils.attributesForRepeatable(sourceClass.getMetadata(),
				ComponentScan.class, ComponentScans.class, MergedAnnotation::isMetaPresent);
	}

	if (!componentScans.isEmpty() &&
			!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
		for (AnnotationAttributes componentScan : componentScans) {
			// The config class is annotated with @ComponentScan -> perform the scan immediately
			Set<BeanDefinitionHolder> scannedBeanDefinitions =
					this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
			// Check the set of scanned definitions for any further config classes and parse recursively if needed
			for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
				BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
				if (bdCand == null) {
					bdCand = holder.getBeanDefinition();
				}
				if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
					parse(bdCand.getBeanClassName(), holder.getBeanName());
				}
			}
		}
	}

	// Process any @Import annotations
	processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

	// Process any @ImportResource annotations
	AnnotationAttributes importResource =
			AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
	if (importResource != null) {
		String[] resources = importResource.getStringArray("locations");
		Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
		for (String resource : resources) {
			String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
			configClass.addImportedResource(resolvedResource, readerClass);
		}
	}

	// Process individual @Bean methods
	Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
	for (MethodMetadata methodMetadata : beanMethods) {
		configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
	}

	// Process default methods on interfaces
	processInterfaces(configClass, sourceClass);

	// Process superclass, if any
	if (sourceClass.getMetadata().hasSuperClass()) {
		String superclass = sourceClass.getMetadata().getSuperClassName();
		if (superclass != null && !superclass.startsWith("java") &&
				!this.knownSuperclasses.containsKey(superclass)) {
			this.knownSuperclasses.put(superclass, configClass);
			// Superclass found, return its annotation metadata and recurse
			return sourceClass.getSuperClass();
		}
	}

	// No superclass -> processing is complete
	return null;
}
```

这段代码是 `ConfigurationClassParser` 类中的 `doProcessConfigurationClass` 方法,用于处理单个配置类。它接受一个 `ConfigurationClass` 对象、一个 `SourceClass` 对象和一个用于过滤的 `Predicate<String>`。

让我们逐步分析这个方法的主要逻辑:

1. 处理成员类:
    - 如果配置类被 `@Component` 注解标记,则首先递归处理其成员类。
    - 通过调用 `processMemberClasses` 方法处理配置类中的嵌套类。

2. 处理 `@PropertySource` 注解:
    - 获取配置类上的 `@PropertySource` 注解属性,并交给 `PropertySourceRegistry` 进行处理。
    - 如果 `PropertySourceRegistry` 不存在,则记录一条忽略 `@PropertySource` 注解的信息日志。

3. 搜索 `@ComponentScan` 注解:
    - 首先在配置类上搜索直接声明的 `@ComponentScan` 注解。
    - 如果没有找到直接声明的 `@ComponentScan` 注解,则搜索元注解中的 `@ComponentScan` 注解。

4. 处理 `@ComponentScan` 注解:
    - 如果找到了 `@ComponentScan` 注解,并且条件评估器允许在 `REGISTER_BEAN` 阶段处理该配置类,则进行以下操作:
        - 使用 `ComponentScanParser` 解析 `@ComponentScan` 注解,获取扫描到的 Bean 定义。
        - 对于每个扫描到的 Bean 定义,检查是否是配置类候选,如果是,则递归解析该配置类。

5. 处理 `@Import` 注解:
    - 调用 `processImports` 方法处理配置类上的 `@Import` 注解。
    - 获取配置类上的 `@Import` 注解,并递归处理导入的类。

6. 处理 `@ImportResource` 注解:
    - 获取配置类上的 `@ImportResource` 注解属性。
    - 对于每个 `@ImportResource` 注解指定的资源路径,将其添加到配置类的导入资源列表中。

7. 处理 `@Bean` 方法:
    - 检索配置类中的 `@Bean` 方法元数据。
    - 对于每个 `@Bean` 方法,将其封装为 `BeanMethod` 对象,并添加到配置类中。

8. 处理接口上的默认方法:
    - 调用 `processInterfaces` 方法处理配置类实现的接口上的默认方法。

9. 处理父类:
    - 如果配置类有父类,并且父类不是 `java` 包下的类,且未被处理过,则将父类添加到已知的父类映射中,并返回父类的注解元数据进行递归处理。

10. 如果没有父类,则处理完成,返回 `null`。

总的来说,`doProcessConfigurationClass` 方法的主要作用是处理单个配置类,包括处理成员类、`@PropertySource`、`@ComponentScan`、`@Import`、`@ImportResource`、`@Bean` 方法、接口默认方法以及父类。它使用递归的方式处理配置类的层次结构,确保所有相关的配置都被解析和处理。

这个方法是 `ConfigurationClassParser` 的核心逻辑之一,用于将配置类转换为 Spring 容器可以使用的 Bean 定义和配置信息。它与其他方法和组件协作,共同完成了 Spring 基于 Java 的配置类的解析和处理过程。

### 2. `@ComponentScan`

`@ComponentScan` 是 Spring Framework 提供的一个重要注解,用于启用组件扫描功能。它的主要作用是自动发现和注册 Spring 容器中的 Bean。

以下是 `@ComponentScan` 注解的几个主要作用:

1. 自动扫描组件:
    - `@ComponentScan` 会根据指定的扫描路径和过滤条件,自动扫描应用程序中的组件类。
    - 默认情况下,它会扫描标记了 `@Component`、`@Controller`、`@Service`、`@Repository` 和 `@Configuration` 注解的类。
    - 通过组件扫描,Spring 容器可以自动发现和实例化这些组件,并将它们注册为 Bean。

2. 简化配置:
    - 使用 `@ComponentScan` 可以显著简化 Spring 应用程序的配置。
    - 无需在 XML 配置文件或 Java 配置类中显式定义和注册每个 Bean,`@ComponentScan` 会自动完成这个任务。
    - 只需在配置类上标记 `@ComponentScan` 注解,并指定要扫描的包路径,Spring 容器就会自动扫描和注册组件。

3. 灵活的扫描路径和过滤条件:
    - `@ComponentScan` 提供了灵活的扫描路径和过滤条件配置选项。
    - 可以通过 `basePackages` 或 `basePackageClasses` 属性指定要扫描的包路径或类所在的包。
    - 可以使用 `includeFilters` 和 `excludeFilters` 属性来包含或排除特定的组件类型或名称模式。
    - 这种灵活性使得可以精确控制要扫描和注册的组件范围。

4. 与 `@Configuration` 类配合使用:
    - `@ComponentScan` 通常与标记了 `@Configuration` 的配置类一起使用。
    - 在配置类上标记 `@ComponentScan` 注解,可以指定要扫描的包路径,从而将组件扫描的配置集中在一个地方。
    - 这种方式使得配置类成为应用程序配置的中心,提供了清晰和集中的配置管理。

5. 支持多个 `@ComponentScan` 注解:
    - 可以在应用程序中使用多个 `@ComponentScan` 注解,以扫描不同的包路径或应用不同的过滤条件。
    - 多个 `@ComponentScan` 注解可以分布在不同的配置类中,以实现模块化和可重用的配置。

总之,`@ComponentScan` 注解通过自动扫描和注册组件,简化了 Spring 应用程序的配置过程。它提供了灵活的扫描路径和过滤条件,使得可以精确控制要扫描和注册的组件范围。与 `@Configuration` 类配合使用,`@ComponentScan`
成为了管理和组织应用程序配置的强大工具。

通过使用 `@ComponentScan`,开发者可以专注于编写业务逻辑和组件类,而无需手动配置和注册每个 Bean,提高了开发效率和可维护性。Spring 容器会自动扫描和管理这些组件,使应用程序的组装和配置变得更加简单和直观。

### 3. `@EnableAutoConfiguration`

```java

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
	//..............
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {
	//..............
}
```

`@EnableAutoConfiguration` 注解中使用的 `@Import` 是在标记了 `@Configuration` 的配置类中被处理的。

当 Spring Boot 应用程序启动时,Spring 容器会首先处理标记了 `@Configuration` 的配置类。在处理过程中,`ConfigurationClassParser` 会解析配置类,并处理其中的各种注解,包括 `@Import` 注解。

`@EnableAutoConfiguration` 注解本身也使用了 `@Import` 注解,它导入了 `AutoConfigurationImportSelector` 类:

```java

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
	// ...
}
```

当 `ConfigurationClassParser` 处理标记了 `@Configuration` 的配置类时,它会递归地处理其中的 `@Import` 注解。对于 `@EnableAutoConfiguration` 注解,它会处理 `@Import(AutoConfigurationImportSelector.class)`
,并调用 `AutoConfigurationImportSelector` 的 `selectImports` 方法来确定要导入的自动配置类。

`AutoConfigurationImportSelector` 会根据类路径中的依赖和条件来选择相应的自动配置类,并将它们作为要导入的配置类返回。这些自动配置类本身也通常标记了 `@Configuration` 注解,因此它们也会被 `ConfigurationClassParser`
处理和解析。

总结一下,`@EnableAutoConfiguration` 中的 `@Import` 注解是在标记了 `@Configuration` 的配置类的处理过程中被处理的。Spring Boot 利用了 Spring 框架的配置类解析机制,通过 `@Import` 注解将自动配置类导入到应用程序的配置中,从而实现自动配置的功能。

这种机制允许 Spring Boot 在应用程序启动时动态地发现和应用相关的自动配置,简化了开发者的配置工作,提供了开箱即用的功能。同时,开发者也可以通过定制自动配置类或使用条件注解来控制自动配置的行为,以满足特定的应用程序需求。