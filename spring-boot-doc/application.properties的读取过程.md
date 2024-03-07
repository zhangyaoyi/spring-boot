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
```

`List<ConfigDataLocation> locations = new ArrayList<>();`: 创建一个 `ConfigDataLocation` 类型的 `ArrayList`,用于存储配置数据的位置。

1. `locations.add(ConfigDataLocation.of("optional:classpath:/;optional:classpath:/config/"));`: 向 `locations` 列表中添加一个 `ConfigDataLocation` 对象。`ConfigDataLocation.of()`
   方法用于创建 `ConfigDataLocation` 对象,传入的参数是一个字符串,表示配置数据的位置。这里的位置是 `optional:classpath:/ 和 optional:classpath:/config/`,表示在类路径的根目录和 config 目录下查找配置文件,如果找不到则忽略。

2. `locations.add(ConfigDataLocation.of("optional:file:./;optional:file:./config/;optional:file:./config/*/"));`: 向 `locations` 列表中再添加一个 `ConfigDataLocation`
   对象。这里的位置是 `optional:file:./、optional:file:./config/ 和 optional:file:./config/*/`,表示在当前目录、config 目录以及 config 目录的所有子目录下查找配置文件,如果找不到则忽略。

3. `DEFAULT_SEARCH_LOCATIONS = locations.toArray(new ConfigDataLocation[0]);`: 将 `locations` 列表转换为 `ConfigDataLocation` 类型的数组,并赋值给静态变量 `DEFAULT_SEARCH_LOCATIONS`
   。这样 `DEFAULT_SEARCH_LOCATIONS` 就包含了所有默认的配置数据位置。

总的来说,这段代码的作用是初始化一个名为 `DEFAULT_SEARCH_LOCATIONS` 的静态变量,它包含了一些默认的配置数据位置,用于在应用程序启动时加载配置文件。这些位置包括类路径的根目录、config 目录、当前目录、config 目录以及
config 目录的所有子目录。如果在这些位置找不到配置文件,则会忽略

```java
// ConfigDataEnvironment.java
static final ConfigDataLocation[] DEFAULT_SEARCH_LOCATIONS;

static {
	List<ConfigDataLocation> locations = new ArrayList<>();
	locations.add(ConfigDataLocation.of("optional:classpath:/;optional:classpath:/config/"));
	locations.add(ConfigDataLocation.of("optional:file:./;optional:file:./config/;optional:file:./config/*/"));
	DEFAULT_SEARCH_LOCATIONS = locations.toArray(new ConfigDataLocation[0]);
}
```
`ConfigDataEnvironmentContributors` 是 Spring Boot 中的一个类,它的作用是在应用程序启动时为环境贡献配置数据。换句话说,它负责从各种来源(如配置文件、环境变量、命令行参数等)收集配置数据,并将其添加到 Spring
的环境中,以便在应用程序中使用。

下面是 `ConfigDataEnvironmentContributors` 的一些主要功能:

1. 加载外部配置: `ConfigDataEnvironmentContributors` 会根据配置的位置(如 `application.properties` 或 `application.yml` 文件)加载外部配置数据,并将其添加到环境中。

2. 处理配置文件: 它能够处理不同格式的配置文件,如 properties 文件和 YAML 文件,并将其转换为 Spring 的 `PropertySource` 对象。

3. 合并配置数据: 如果存在多个配置源(如多个配置文件),`ConfigDataEnvironmentContributors` 会将它们合并为一个统一的配置视图,并解决任何潜在的冲突。

4. 支持配置文件 profiles: 它支持 Spring 的 profile 功能,可以根据激活的 profile 加载特定的配置文件。

5. 绑定配置数据: `ConfigDataEnvironmentContributors` 将配置数据绑定到 Spring 的 `Environment` 对象,这使得加载的配置数据在整个应用程序中可用,并可以通过 `@Value` 注解或 `Environment` 接口访问。。

总之,`ConfigDataEnvironmentContributors` 在 Spring Boot 应用程序的启动过程中扮演着重要的角色,它负责收集和处理各种来源的配置数据,并将其提供给应用程序使用。这使得 Spring Boot 应用程序能够方便地外部化配置,并支持灵活的配置管理。

```java
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

```java
//`ConfigDataLoaders`的初始化是在`ConfigDataEnvironment`的构造函数中。    
this.loaders =new

ConfigDataLoaders(logFactory, bootstrapContext, SpringFactoriesLoader.forDefaultResourceLocation(resourceLoader.getClassLoader()));
```

```java
//在`ConfigDataLoaders`中`List<ConfigDataLoader> loaders`被初始化。
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

## 补充说明

### ConfigDataImporter

`ConfigDataImporter` 是 Spring Boot 中负责导入配置数据的核心组件。它的主要作用是从各种来源加载配置数据,并将其转换为 Spring 的 `PropertySource` 对象,以便在应用程序中使用。

以下是 `ConfigDataImporter` 的一些关键职责:

1. 处理配置数据位置: `ConfigDataImporter` 接收一组 `ConfigDataLocationResolvers`,用于解析配置数据的位置。这些位置可以是类路径上的文件、目录、URL 或特定的配置文件格式,如 `application.properties`
   或 `application.yml`。

2. 加载配置数据: 给定配置数据的位置,`ConfigDataImporter` 使用 `ConfigDataLoaders` 来加载实际的配置数据。每个 `ConfigDataLoader` 都知道如何从特定的源(如文件或 URL)加载配置数据。

3. 创建 `PropertySource` 对象: 一旦配置数据被加载,`ConfigDataImporter` 会将其转换为一个或多个 `PropertySource` 对象。`PropertySource` 是一个键值对的容器,表示一组配置属性。

总的来说,`ConfigDataImporter` 在 Spring Boot 的外部化配置处理过程中扮演着核心角色。它协调不同的组件,如位置解析器和数据加载器,以从各种源加载配置数据,并将其集成到 Spring 环境中,从而实现了 Spring Boot
的强大和灵活的配置管理功能。

### ConfigDataEnvironmentContributors

#### 例子说明

以一个具体的例子来说明 `ConfigDataEnvironmentContributors` 的作用。

假设我们有一个 Spring Boot 应用程序,其目录结构如下:

```
myapp/
  ├── src/
  │   ├── main/
  │   │   ├── java/
  │   │   │   └── com/
  │   │   │       └── example/
  │   │   │           └── MyApplication.java
  │   │   └── resources/
  │   │       ├── application.properties
  │   │       └── application-dev.properties
  │   └── test/
  └── config/
      └── application-prod.properties
```

在这个例子中,我们有三个配置文件:

1. `src/main/resources/application.properties`: 主配置文件,包含通用的配置属性。
2. `src/main/resources/application-dev.properties`: 开发环境特定的配置文件。
3. `config/application-prod.properties`: 生产环境特定的配置文件,位于外部的 `config` 目录中。

现在,当我们启动这个 Spring Boot 应用程序时,`ConfigDataEnvironmentContributors` 会执行以下步骤:

1. 加载主配置文件 `application.properties`,并将其中的配置属性添加到环境中。

2. 如果激活了 `dev` profile,那么 `ConfigDataEnvironmentContributors` 会加载 `application-dev.properties` 文件,并将其中的配置属性添加到环境中,覆盖任何同名的属性。

3. 如果激活了 `prod` profile,那么 `ConfigDataEnvironmentContributors` 会在外部的 `config` 目录中查找 `application-prod.properties` 文件,并将其中的配置属性添加到环境中,再次覆盖任何同名的属性。

4. 最后,`ConfigDataEnvironmentContributors` 会将所有收集到的配置属性绑定到 Spring 的 `Environment` 对象,以便在应用程序中使用。

在这个例子中,`ConfigDataEnvironmentContributors` 负责根据不同的 profile 和配置文件的位置,自动加载和合并配置属性,并将它们提供给 Spring 应用程序使用。这使得我们可以轻松地管理不同环境下的配置,而无需在代码中进行复杂的配置加载逻辑。

#### 加载过程

好的,让我们通过分析 Spring Boot 的源代码来了解 `ConfigDataEnvironmentContributors` 的加载过程。

在 Spring Boot 应用程序启动时,会调用 `SpringApplication` 的 `run` 方法。在这个方法中,会创建一个 `ConfigurableEnvironment` 对象,并调用 `ConfigDataEnvironmentPostProcessor` 的 `postProcessEnvironment` 方法来处理配置数据。

以下是 `ConfigDataEnvironmentPostProcessor` 的 `postProcessEnvironment` 方法的简化版本:

```java
public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
	ConfigDataEnvironmentContributors contributors = getContributors(environment);
	ConfigDataEnvironmentContributor contributor = contributors.getContributor(environment);
	contributor.contribute(environment);
}
```

这个方法首先通过 `getContributors` 方法获取 `ConfigDataEnvironmentContributors` 对象。然后,它根据当前环境调用 `getContributor` 方法获取一个 `ConfigDataEnvironmentContributor` 对象,并调用其 `contribute`
方法来贡献配置数据到环境中。

`ConfigDataEnvironmentContributors` 的 `getContributor` 方法如下:

```java
public ConfigDataEnvironmentContributor getContributor(ConfigurableEnvironment environment) {
	ConfigDataLocationResolvers resolvers = createConfigDataLocationResolvers(environment);
	ConfigDataLoaders loaders = createConfigDataLoaders(environment);
	return new ConfigDataEnvironmentContributor(environment, resolvers, loaders);
}
```

这个方法创建了两个重要的对象:

1. `ConfigDataLocationResolvers`: 用于解析配置数据的位置。
2. `ConfigDataLoaders`: 用于加载配置数据。

然后,它创建一个新的 `ConfigDataEnvironmentContributor` 对象,并将环境、解析器和加载器传递给它。

`ConfigDataEnvironmentContributor` 的 `contribute` 方法负责实际的配置数据加载过程:

```java
public void contribute(ConfigurableEnvironment environment) {
	ConfigDataImporter importer = new ConfigDataImporter(this.resolvers, this.loaders);
	ConfigDataEnvironmentContributors.importConfigData(environment, importer);
}
```

这个方法创建一个 `ConfigDataImporter` 对象,并将解析器和加载器传递给它。然后,它调用 `ConfigDataEnvironmentContributors` 的 `importConfigData` 方法来导入配置数据。

`importConfigData` 方法会使用 `ConfigDataImporter` 加载配置数据,并将其绑定到环境中。它会处理配置文件的加载、合并和覆盖等逻辑。

总的来说,`ConfigDataEnvironmentContributors` 通过协调 `ConfigDataLocationResolvers`、`ConfigDataLoaders` 和 `ConfigDataImporter` 的工作,实现了从各种来源加载配置数据并将其贡献到 Spring 环境中的功能。这个过程是
Spring Boot 自动配置的重要组成部分,使得开发者可以方便地外部化配置,并根据不同的环境和需求进行配置管理。

