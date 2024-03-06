# application.properties的读取过程

基于 springboot3.2 和 springframework6.1.4

## 调用的线程

![application.properties的读取过程1.png](resource%2Fapplication.properties%E7%9A%84%E8%AF%BB%E5%8F%96%E8%BF%87%E7%A8%8B1.png)

## 读取过程

在 SprigApplication 中，通过 `prepareEnvironment` 方法创建和配置环境，然后将环境传递给 `SpringApplicationRunListeners` 的 `environmentPrepared` 方法。
`EventPublishingRunListener`的`environmentPrepared` 方法中，并发布`ApplicationEnvironmentPreparedEvent`事件。

```java
// SpringApplication.java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
		DefaultBootstrapContext bootstrapContext, ApplicationArguments applicationArguments) {
	// Create and configure the environment
	ConfigurableEnvironment environment = getOrCreateEnvironment();
	configureEnvironment(environment, applicationArguments.getSourceArgs());
	ConfigurationPropertySources.attach(environment);
	listeners.environmentPrepared(bootstrapContext, environment);
	DefaultPropertiesPropertySource.moveToEnd(environment);
	Assert.state(!environment.containsProperty("spring.main.environment-prefix"),
			"Environment prefix cannot be set via properties.");
	bindToSpringApplication(environment);
	if (!this.isCustomEnvironment) {
		EnvironmentConverter environmentConverter = new EnvironmentConverter(getClassLoader());
		environment = environmentConverter.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
	}
	ConfigurationPropertySources.attach(environment);
	return environment;
}

// EventPublishingRunListener.java
public void environmentPrepared(ConfigurableBootstrapContext bootstrapContext, ConfigurableEnvironment environment) {
	multicastInitialEvent(new ApplicationEnvironmentPreparedEvent(bootstrapContext, this.application, this.args, environment));
}
```

`ApplicationEnvironmentPreparedEvent` 事件的监听器 `EnvironmentPostProcessorApplicationListener` 会监听该事件。

```java
// EnvironmentPostProcessorApplicationListener.java
@Override
public boolean supportsEventType(Class<? extends ApplicationEvent> eventType) {
	return ApplicationEnvironmentPreparedEvent.class.isAssignableFrom(eventType)
			|| ApplicationPreparedEvent.class.isAssignableFrom(eventType)
			|| ApplicationFailedEvent.class.isAssignableFrom(eventType);
}

@Override
public void onApplicationEvent(ApplicationEvent event) {
	if (event instanceof ApplicationEnvironmentPreparedEvent environmentPreparedEvent) {
		onApplicationEnvironmentPreparedEvent(environmentPreparedEvent);
	}
	if (event instanceof ApplicationPreparedEvent) {
		onApplicationPreparedEvent();
	}
	if (event instanceof ApplicationFailedEvent) {
		onApplicationFailedEvent();
	}
}
```

在`onApplicationEnvironmentPreparedEvent`方法中，会调用`EnvironmentPostProcessor`的`postProcessEnvironment`方法。

```java
private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
	ConfigurableEnvironment environment = event.getEnvironment();
	SpringApplication application = event.getSpringApplication();
	for (EnvironmentPostProcessor postProcessor : getEnvironmentPostProcessors(application.getResourceLoader(),
			event.getBootstrapContext())) {
		postProcessor.postProcessEnvironment(environment, application);
	}
}
```

获取在`spring.factory`中配置的`EnvironmentPostProcessor`。

```java

@Override
public List<EnvironmentPostProcessor> getEnvironmentPostProcessors(DeferredLogFactory logFactory,
		ConfigurableBootstrapContext bootstrapContext) {
	ArgumentResolver argumentResolver = ArgumentResolver.of(DeferredLogFactory.class, logFactory);
	argumentResolver = argumentResolver.and(ConfigurableBootstrapContext.class, bootstrapContext);
	argumentResolver = argumentResolver.and(BootstrapContext.class, bootstrapContext);
	argumentResolver = argumentResolver.and(BootstrapRegistry.class, bootstrapContext);
	return this.loader.load(EnvironmentPostProcessor.class, argumentResolver);
}
```

`spring.factory`中配置的`EnvironmentPostProcessor`。`ConfigDataEnvironmentPostProcessor`是对`application.properties`的处理。

```properties
# Environment Post Processors
org.springframework.boot.env.EnvironmentPostProcessor=\
org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor,\
org.springframework.boot.context.config.ConfigDataEnvironmentPostProcessor,\
org.springframework.boot.env.RandomValuePropertySourceEnvironmentPostProcessor,\
org.springframework.boot.env.SpringApplicationJsonEnvironmentPostProcessor,\
org.springframework.boot.env.SystemEnvironmentPropertySourceEnvironmentPostProcessor,\
org.springframework.boot.reactor.ReactorEnvironmentPostProcessor
```

在`ConfigDataEnvironmentPostProcessor`的`postProcessEnvironment`方法中，`ConfigDataEnvironment`会被实例化并调用`processAndApply`方法。

`postProcessEnvironment`->`processAndApply`->`processWithProfiles`->`withProcessedImports`->`resolveAndLoad`

```java
void postProcessEnvironment(ConfigurableEnvironment environment, ResourceLoader resourceLoader, Collection<String> additionalProfiles) {
	this.logger.trace("Post-processing environment to add config data");
	resourceLoader = (resourceLoader != null) ? resourceLoader : new DefaultResourceLoader();
	getConfigDataEnvironment(environment, resourceLoader, additionalProfiles).processAndApply();
}

void processAndApply() {
	ConfigDataImporter importer = new ConfigDataImporter(this.logFactory, this.notFoundAction, this.resolvers, this.loaders);
	registerBootstrapBinder(this.contributors, null, DENY_INACTIVE_BINDING);
	ConfigDataEnvironmentContributors contributors = processInitial(this.contributors, importer);
	ConfigDataActivationContext activationContext = createActivationContext(contributors.getBinder(null, BinderOption.FAIL_ON_BIND_TO_INACTIVE_SOURCE));
	contributors = processWithoutProfiles(contributors, importer, activationContext);
	activationContext = withProfiles(contributors, activationContext);
	contributors = processWithProfiles(contributors, importer, activationContext);
	applyToEnvironment(contributors, activationContext, importer.getLoadedLocations(), importer.getOptionalLocations());
}

private ConfigDataEnvironmentContributors processWithProfiles(ConfigDataEnvironmentContributors contributors,
		ConfigDataImporter importer, ConfigDataActivationContext activationContext) {
	this.logger.trace("Processing config data environment contributors with profile activation context");
	contributors = contributors.withProcessedImports(importer, activationContext);
	registerBootstrapBinder(contributors, activationContext, ALLOW_INACTIVE_BINDING);
	return contributors;
}

ConfigDataEnvironmentContributors withProcessedImports(ConfigDataImporter importer, ConfigDataActivationContext activationContext) {
	ImportPhase importPhase = ImportPhase.get(activationContext);
	this.logger.trace(LogMessage.format("Processing imports for phase %s. %s", importPhase, (activationContext != null) ? activationContext : "no activation context"));
	ConfigDataEnvironmentContributors result = this;
	int processed = 0;
	while (true) {
		ConfigDataEnvironmentContributor contributor = getNextToProcess(result, activationContext, importPhase);
		if (contributor == null) {
			this.logger.trace(LogMessage.format("Processed imports for of %d contributors", processed));
			return result;
		}
		if (contributor.getKind() == Kind.UNBOUND_IMPORT) {
			ConfigDataEnvironmentContributor bound = contributor.withBoundProperties(result, activationContext);
			result = new ConfigDataEnvironmentContributors(this.logger, this.bootstrapContext, result.getRoot().withReplacement(contributor, bound));
			continue;
		}
		ConfigDataLocationResolverContext locationResolverContext = new ContributorConfigDataLocationResolverContext(result, contributor, activationContext);
		ConfigDataLoaderContext loaderContext = new ContributorDataLoaderContext(this);
		List<ConfigDataLocation> imports = contributor.getImports();
		this.logger.trace(LogMessage.format("Processing imports %s", imports));
		Map<ConfigDataResolutionResult, ConfigData> imported = importer.resolveAndLoad(activationContext, locationResolverContext, loaderContext, imports);
		this.logger.trace(LogMessage.of(() -> getImportedMessage(imported.keySet())));
		ConfigDataEnvironmentContributor contributorAndChildren = contributor.withChildren(importPhase, asContributors(imported));
		result = new ConfigDataEnvironmentContributors(this.logger, this.bootstrapContext, result.getRoot().withReplacement(contributor, contributorAndChildren));
		processed++;
	}
}

Map<ConfigDataResolutionResult, ConfigData> resolveAndLoad(ConfigDataActivationContext activationContext,
		ConfigDataLocationResolverContext locationResolverContext, ConfigDataLoaderContext loaderContext,
		List<ConfigDataLocation> locations) {
	try {
		Profiles profiles = (activationContext != null) ? activationContext.getProfiles() : null;
		List<ConfigDataResolutionResult> resolved = resolve(locationResolverContext, profiles, locations);
		return load(loaderContext, resolved);
	}
	catch (IOException ex) {
		throw new IllegalStateException("IO error on loading imports from " + locations, ex);
	}
}
```

## resolveAndLoad

### ConfigDataLocationResolvers

`ConfigDataLocationResolver` 是一个接口，它在 Spring Boot 的配置数据加载机制中起到了关键的作用。这个接口定义了如何解析和处理配置数据位置。

`ConfigDataLocationResolver` 有两个主要的方法：

1. `isResolvable(ConfigDataLocationResolverContext context, ConfigDataLocation location)`：这个方法用于判断给定的 `ConfigDataLocation` 是否可以被这个解析器解析。

2. `resolve(ConfigDataLocationResolverContext context, ConfigDataLocation location)`：这个方法用于解析给定的 `ConfigDataLocation`，并返回一个或多个 `ConfigDataResource` 实例。这些实例代表了可以从中加载配置数据的资源。

`StandardConfigDataLocationResolver` 是 `ConfigDataLocationResolver` 的一个实现，它负责解析和处理标准的配置数据位置。这个类的主要职责包括解析配置数据位置、处理配置文件名、处理配置数据资源以及处理配置数据位置的可选性。

总的来说，`ConfigDataLocationResolver` 和它的实现类 `StandardConfigDataLocationResolver` 是 Spring Boot 配置数据加载机制的一部分，它们帮助 Spring Boot 知道从哪里加载配置数据，以及如何加载这些数据。

```java
//从`spring.factory`中获取`ConfigDataLocationResolver`。
ConfigDataLocationResolvers(DeferredLogFactory logFactory, ConfigurableBootstrapContext bootstrapContext, Binder binder, ResourceLoader resourceLoader, SpringFactoriesLoader springFactoriesLoader) {
	ArgumentResolver argumentResolver = ArgumentResolver.of(DeferredLogFactory.class, logFactory);
	argumentResolver = argumentResolver.and(Binder.class, binder);
	argumentResolver = argumentResolver.and(ResourceLoader.class, resourceLoader);
	argumentResolver = argumentResolver.and(ConfigurableBootstrapContext.class, bootstrapContext);
	argumentResolver = argumentResolver.and(BootstrapContext.class, bootstrapContext);
	argumentResolver = argumentResolver.and(BootstrapRegistry.class, bootstrapContext);
	argumentResolver = argumentResolver.andSupplied(Log.class, () -> {
		throw new IllegalArgumentException("Log types cannot be injected, please use DeferredLogFactory");
	});
	this.resolvers = reorder(springFactoriesLoader.load(ConfigDataLocationResolver.class, argumentResolver));
}
```

```properties
# ConfigData Location Resolvers
org.springframework.boot.context.config.ConfigDataLocationResolver=\
org.springframework.boot.context.config.ConfigTreeConfigDataLocationResolver,\
org.springframework.boot.context.config.StandardConfigDataLocationResolver
```

### StandardConfigDataLocationResolver

`StandardConfigDataLocationResolver` 是 Spring Boot 配置数据加载机制的一部分，它负责解析和处理标准的配置数据位置。

以下是这个类的主要职责：

1. 解析配置数据位置：这个类实现了 `ConfigDataLocationResolver` 接口，它的 `resolve` 方法用于解析给定的 `ConfigDataLocation`，并返回一个或多个 `StandardConfigDataResource` 实例。这些实例代表了可以从中加载配置数据的资源。

2. 处理配置文件名：在解析配置数据位置时，这个类会处理配置文件名。例如，它会检查配置文件名是否包含 "*"，因为这是不允许的。

3. 处理配置数据资源：这个类会处理配置数据资源，包括文件和目录。对于文件，它会检查文件的扩展名是否已知，如果未知，它会抛出异常。对于目录，它会查找目录中的所有子目录，并为每个子目录创建一个 `StandardConfigDataResource`
   实例。

4. 处理配置数据位置的可选性：这个类会处理配置数据位置的可选性。如果配置数据位置是可选的，但不存在，那么它会跳过这个位置，而不会抛出异常。

总的来说，`StandardConfigDataLocationResolver` 类是 Spring Boot 配置数据加载机制的一部分，它帮助 Spring Boot 知道从哪里加载配置数据，以及如何加载这些数据。

```java
//自定义的 config file name 或者 default config file name。
static final String CONFIG_NAME_PROPERTY = "spring.config.name";

static final String[] DEFAULT_CONFIG_NAMES = { "application" };
```

```java
//从`spring.factory`中获取`PropertySourceLoader`, 初始化为`PropertiesPropertySourceLoader`和`YamlPropertySourceLoader`。
private final List<PropertySourceLoader> propertySourceLoaders;

public StandardConfigDataLocationResolver(DeferredLogFactory logFactory, Binder binder, ResourceLoader resourceLoader) {
	this.logger = logFactory.getLog(StandardConfigDataLocationResolver.class);
	this.propertySourceLoaders = SpringFactoriesLoader.loadFactories(PropertySourceLoader.class, getClass().getClassLoader());
	this.configNames = getConfigNames(binder);
	this.resourceLoader = new LocationResourceLoader(resourceLoader);
}

```

```properties
# PropertySource Loaders
org.springframework.boot.env.PropertySourceLoader=\
org.springframework.boot.env.PropertiesPropertySourceLoader,\
org.springframework.boot.env.YamlPropertySourceLoader
```

在`PropertiesPropertySourceLoader`中，

```java
public String[] getFileExtensions() {
	return new String[] { "properties", "xml" };
}
```

在`YamlPropertySourceLoader`中，

```java
public String[] getFileExtensions() {
	return new String[] { "yml", "yaml" };
}
```

`getReferencesForConfigName` 是 `StandardConfigDataLocationResolver` 类中的一个私有方法，它的主要作用是为给定的配置名称生成一组 `StandardConfigDataReference` 引用。

以下是这个方法的主要步骤：

1. 创建一个空的 `Deque<StandardConfigDataReference>` 对象，用于存储生成的引用。

2. 遍历 `this.propertySourceLoaders`，这是一个 `PropertySourceLoader` 对象的列表，每个 `PropertySourceLoader` 都可以加载一种特定类型的配置数据。

3. 对于每个 `PropertySourceLoader`，遍历其支持的文件扩展名。

4. 对于每个文件扩展名，创建一个新的 `StandardConfigDataReference` 对象。这个对象包含了配置数据位置、目录、配置文件名、配置文件的激活配置文件（profile）、文件扩展名以及用于加载这个引用的 `PropertySourceLoader`。

5. 检查新创建的 `StandardConfigDataReference` 对象是否已经在 `references` 队列中。如果不在，就将其添加到队列的前面。

6. 最后，返回包含所有生成的 `StandardConfigDataReference` 对象的 `references` 队列。

总的来说，`getReferencesForConfigName` 方法的作用是为给定的配置名称生成一组 `StandardConfigDataReference` 引用，这些引用代表了可以从中加载配置数据的资源。

```java
private Deque<StandardConfigDataReference> getReferencesForConfigName(String name, ConfigDataLocation configDataLocation, String directory, String profile) {
	Deque<StandardConfigDataReference> references = new ArrayDeque<>();
	for (PropertySourceLoader propertySourceLoader : this.propertySourceLoaders) {
		for (String extension : propertySourceLoader.getFileExtensions()) {
			StandardConfigDataReference reference = new StandardConfigDataReference(configDataLocation, directory,
					directory + name, profile, extension, propertySourceLoader);
			if (!references.contains(reference)) {
				references.addFirst(reference);
			}
		}
	}
	return references;
}
```

注意：application 的配置文件是有循序。
![application.properties的读取过程 2.png](resource%2Fapplication.properties%E7%9A%84%E8%AF%BB%E5%8F%96%E8%BF%87%E7%A8%8B%202.png)

```java
private List<StandardConfigDataResource> resolve(Set<StandardConfigDataReference> references) {
	List<StandardConfigDataResource> resolved = new ArrayList<>();
	for (StandardConfigDataReference reference : references) {
		resolved.addAll(resolve(reference));
	}
	if (resolved.isEmpty()) {
		resolved.addAll(resolveEmptyDirectories(references));
	}
	return resolved;
}
```

### ConfigDataLoaders

`ConfigDataLoaders`的初始化是在`ConfigDataEnvironment`的构造函数中。

```java
    this.loaders =new

ConfigDataLoaders(logFactory, bootstrapContext, SpringFactoriesLoader.forDefaultResourceLocation(resourceLoader.getClassLoader()));
```

在`ConfigDataLoaders`中`List<ConfigDataLoader> loaders`被初始化。

```java
    this.loaders =springFactoriesLoader.

load(ConfigDataLoader .class, argumentResolver);
```

```properties
# ConfigData Loaders
org.springframework.boot.context.config.ConfigDataLoader=\
org.springframework.boot.context.config.ConfigTreeConfigDataLoader,\
org.springframework.boot.context.config.StandardConfigDataLoader
```

### StandardConfigDataLoader

选定的代码是 `StandardConfigDataLoader` 类中 `load` 方法的一部分，该类是 Spring Boot 中用于处理 `Resource` 支持的位置的 `ConfigDataLoader`。

```java
List<PropertySource<?>> propertySources = reference.getPropertySourceLoader().load(name, originTrackedResource);
```

这行代码负责从特定资源加载配置数据。

`reference.getPropertySourceLoader()` 调用从 `StandardConfigDataReference` 对象中检索一个 `PropertySourceLoader`。`PropertySourceLoader` 是 Spring 框架中的一个接口，定义了如何加载 `PropertySource`
的方法。`PropertySource` 表示一组键值对的源，可以用来填充 Spring 应用中的 `PropertyResolver`，如 `Environment`。

`load` 方法返回从提供的资源加载的一组 `PropertySource` 对象，这些对象被存储在 `propertySources` 变量中。列表中的每个 `PropertySource` 都代表从资源加载的一组配置属性。

```java
public ConfigData load(ConfigDataLoaderContext context, StandardConfigDataResource resource) throws IOException, ConfigDataNotFoundException {
	if (resource.isEmptyDirectory()) {
		return ConfigData.EMPTY;
	}
	ConfigDataResourceNotFoundException.throwIfDoesNotExist(resource, resource.getResource());
	StandardConfigDataReference reference = resource.getReference();
	Resource originTrackedResource = OriginTrackedResource.of(resource.getResource(),
			Origin.from(reference.getConfigDataLocation()));
	String name = String.format("Config resource '%s' via location '%s'", resource,
			reference.getConfigDataLocation());
	List<PropertySource<?>> propertySources = reference.getPropertySourceLoader().load(name, originTrackedResource);
	PropertySourceOptions options = (resource.getProfile() != null) ? PROFILE_SPECIFIC : NON_PROFILE_SPECIFIC;
	return new ConfigData(propertySources, options);
}
```