---
title: "Dubbo服务暴露探究"
date: 2021-03-09T23:22:13+08:00
draft: false

tags: ["Dubbo", "Spring"]
categories: ["Dubbo"]
---

# Dubbo服务暴露探究
> **本文基于Dubbo源码版本号：eef6b450fc69f6670d19c050719a6766343a1729**
**编写时，master分支Dubbo版本为2.7.10-SNAPSHOT**
**IDE是Intellij IDEA 2020.3.1**

<font color=red>前方高能！包含大量源码与贴图！</font>

**Dubbo服务暴露流程起始于Spring容器启动后，** 通过监听Spring容器的ContextRefreshedEvent事件实现。
那么Dubbo是如何向Spring容器注册监听事件的呢？
### Dubbo服务暴露流程是如何融入Spring容器的？
阅读以下内容，会涉及的相关知识有：
1. Spring自定义命名空间
2. Spring容器启动刷新过程
3. Spring实例化Bean过程
4. Spring的BeanPostProcessor扩展点
5. Spring的ApplicationContextAware
6. Spring的ApplicationListener

> 从Spring2.0版本起，支持自定义XML Bean定义解析器，以及将此类解析器集成到Spring IoC容器中。
参考Spring官方文档：https://docs.spring.io/spring-framework/docs/5.2.x/spring-framework-reference/core.html#xml-custom

> Spring自定义命名空间的使用，涉及 META-INF/spring.handlers 文件。
参考Spring官方文档：https://docs.spring.io/spring-framework/docs/5.2.x/spring-framework-reference/core.html#xsd-custom-registration

在Dubbo工程内检索文件：spring.handlers
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210307170713.png)
文件内容如下：
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210307171346.png)

进入DubboNamespaceHandler类，查看类图，有实现NamespaceHandler接口，其中方法parse是实现自定义命名空间关键方法。
```java
BeanDefinition parse(Element element, ParserContext parserContext);
```
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210307222559.png)

在DubboNamespaceHandler类的parse(Element element, ParserContext parserContext)方法实现中，有一行
```java
registerCommonBeans(registry);
```
在此行打上断点，debug运行dubbo工程中的demo-xml模块的测试类Application的main方法，路径如下图：
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210307223423.png)

当程序第一次运行到断点处时，main线程栈信息如下图：
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210307222144.jpg)
从下往上分析main线程栈中的方法调用过程，大致得到Spring的自定义命名空间实现原理，其中使用了委派模式（委派模式不属于GOF23种设计模式）。
> 题外话：GOF23种设计模式，GOF是The Gang of Four，四人帮的意思，包括ErichGamma、Richard Helm、Ralph Johnson、John Vlissides。
1. Spring容器初次启动刷新流程中，obtainFreshBeanFactory()方法判断当前还没有创建BeanFactory实例；
2. 创建BeanFactory实例：
    ```java
    DefaultListableBeanFactory beanFactory = createBeanFactory();
    ```
3. 加载bean定义信息：
    ```java
    loadBeanDefinitions(beanFactory);
    ```
4. DefaultBeanDefinitionDocumentReader中判断出[dubbo:application: null]并非默认的命名空间，于是通过委派角色将具体的处理委派给对应的任务角色：
    ```java
    delegate.parseCustomElement(ele);
    ```
5. 解析得到DubboNamespaceHandler实例：
    ```java
    NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
    ```
6. 正式进入DubboNamespaceHandler的parse方法实现：
    ```java
    handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
    ```
---
进入registerCommonBeans(registry)的实现，可以看到有这么一行代码
```java
registerInfrastructureBean(registry, DubboApplicationListenerRegistrar.BEAN_NAME,
                DubboApplicationListenerRegistrar.class);
```
作用是向容器中注册org.apache.dubbo.config.spring.context.DubboApplicationListenerRegistrar类的Bean定义信息。

我们进入DubboApplicationListenerRegistrar看看这个类，
DubboApplicationListenerRegistrar类实现了ApplicationContextAware接口，其对应的方法：setApplicationContext(ApplicationContext applicationContext)；
具体的代码实现中，发现这么一行：
```java
addApplicationListeners((ConfigurableApplicationContext) applicationContext);
```
这个代码与Spring的事件扯上关系了。先在这一行打上断点，直接放行到此，看看这个流程的堆栈信息。
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210307232915.png)
依然从下往上分析main线程栈：
1. 在Spring容器启动刷新过程中，倒数第二步，实例化所有非延迟初始化的单例：
    ```java
    finishBeanFactoryInitialization(beanFactory);
    ```
2. 当执行到实例化DubboApplicationListenerRegistrar类的Bean时，进入实例化Bean的第三步，bean的初始化：
    ```java
    exposedObject = initializeBean(beanName, exposedObject, mbd);
    ```
3. 在执行初始化方法前，先处理BeanPostProcessor扩展点，其中一个实现是ApplicationContextAwareProcessor：
    ```java
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    ```
4. 最终调用了：
    ```java
    ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
    ```
    此处bean是DubboApplicationListenerRegistrar类的实例，由此来到了断点处。
---
接下来Dubbo正式向Spring容器添加自定义事件。
可以看到addApplicationListeners方法只做了两件事，向Spring容器添加了两个事件监听器。
```java
context.addApplicationListener(createDubboBootstrapApplicationListener(context));
context.addApplicationListener(createDubboLifecycleComponentApplicationListener(context));
```
createDubboBootstrapApplicationListener(context)方法中，创建了DubboBootstrapApplicationListener对象。
看看DubboBootstrapApplicationListener类结构：
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210308000316.png)

在DubboBootstrapApplicationListener类的onApplicationContextEvent方法实现中打个断点，然后放行。
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210308001326.png)
依然从下往上分析main线程栈：
1. Spring容器启动刷新的最后一步中，有个小步骤是：
    ```java
    publishEvent(new ContextRefreshedEvent(this));
    ```
    此处发布了Spring容器刷新事件；
2. 事件执行先到达OnceApplicationContextEventListener类的onApplicationEvent(ApplicationEvent event)实现，然后转到DubboBootstrapApplicationListener类的onApplicationContextEvent(ApplicationContextEvent event)方法。
---
如果是ContextRefreshedEvent事件，执行：
```java
onContextRefreshedEvent((ContextRefreshedEvent) event);
```
进入方法看到只做了一步：
```java
dubboBootstrap.start();
```
在DubboBootstrapApplicationListener的构造方法中可以看到关于dubboBootstrap属性的信息：
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210308002058.png)

> DubboBootstrap类使用了单例模式（饿汉式+双重检查锁）。

查看start()方法，发现Dubbo服务暴露的起点。
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210308002716.png)

### Dubbo服务暴露流程如何融入SpringBoot
说完了Spring xml配置形式的集成，再来看看在SpringBoot中，Dubbo是如何融入Spring大家庭的。

SpringBoot有一个知识点是自定义starter，实现关键在于META-INF/spring.factories文件,在dubbo-spring-boot-project工程源码中检索spring.factories文件，如下图：
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210308170638.png)
匹配到了多个文件，其中两个提及了autoconfigure，即自动配置的含义，最终在compatible module下，找到DubboAutoConfiguration类。
打开查看DubboAutoConfiguration类代码，存在注解@Configuration，可知这是一个Spring的配置类。
其中存在一个方法，serviceAnnotationBeanPostProcessor，定义了一个Bean，其类型为ServiceAnnotationBeanPostProcessor，
```java
@ConditionalOnProperty(prefix = DUBBO_SCAN_PREFIX, name = BASE_PACKAGES_PROPERTY_NAME)
@ConditionalOnBean(name = BASE_PACKAGES_PROPERTY_RESOLVER_BEAN_NAME)
@Bean
public ServiceAnnotationBeanPostProcessor serviceAnnotationBeanPostProcessor(
        @Qualifier(BASE_PACKAGES_PROPERTY_RESOLVER_BEAN_NAME) PropertyResolver propertyResolver) {
    Set<String> packagesToScan = propertyResolver.getProperty(BASE_PACKAGES_PROPERTY_NAME, Set.class, emptySet());
    return new ServiceAnnotationBeanPostProcessor(packagesToScan);
}
```
查看ServiceAnnotationBeanPostProcessor类结构，发现这是一个Spring的BeanFactoryPostProcessor扩展点实现类，还是个特殊的BeanDefinitionRegistryPostProcessor实现。
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210308172020.png)

直接来到ServiceAnnotationBeanPostProcessor类实现BeanDefinitionRegistryPostProcessor接口的方法：
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210308172515.png)

看到了熟悉的类DubboBootstrapApplicationListener，显然，在这里把这个类信息注册到Spring容器后，后续步骤与xml配置本质上是一样的。


### Dubbo的配置部分
回到Dubbo暴露的起点，即进入org.apache.dubbo.config.bootstrap.DubboBootstrap#exportServices.exportServices()方法内部：
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210308205349.png)
出现了一个新的角色configManager，通过getServices()方法查看类的代码，看到有个addConfig(AbstractConfig config)方法，打上断点，重新运行程序（终止当前运行，再重新开始运行，exportServices处已经是将各项配置都添加到configManager后的状态）到此处来查看此处的线程栈信息。
断点处如下图：
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210308205702.png)
线程栈信息如下：
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210308210310.png)
可以看到当前方法入参config变量的值为dubbo的application配置信息，而根据线程栈信息从下往上，可以得到程序执行的步骤：
1. Spring容器创建刷新的倒数第二步，初始化所有非延迟加载的单例Bean。
2. 初始化org.apache.dubbo.config.ApplicationConfig类的实例时，执行到了创建实例的第三步，执行Bean的初始化方法，
3. org.apache.dubbo.config.ApplicationConfig类继承了org.apache.dubbo.config.AbstractConfig类，而AbstractConfig类有如下方法，使用了@PostConstruct注解，会在Bean的初始化步骤时执行。
    ```java
    @PostConstruct
    public void addIntoConfigManager() {
        ApplicationModel.getConfigManager().addConfig(this);
    }
    ```
同理，重复运行程序到此断点处，还会看到
1. 注册中心配置：org.apache.dubbo.config.RegistryConfig
2. 协议配置：org.apache.dubbo.config.ProtocolConfig
3. 服务级服务提供配置：org.apache.dubbo.config.spring.ServiceBean
4. 配置中心：org.apache.dubbo.config.ConfigCenterConfig
5. 应用级别服务提供配置：org.apache.dubbo.config.ProviderConfig
6. 应用级别服务消费配置：org.apache.dubbo.config.ConsumerConfig
7. 服务级服务消费配置：org.apache.dubbo.config.spring.ReferenceBean
。。。

对应的实例依次被加入configManager中。
> 其余配置项可参照官方文档：
https://dubbo.apache.org/zh/docs/v2.7/user/references/xml/

### Dubbo服务正式暴露
> **以下部分包含大量源码，关键步骤的解释通过注释呈现。**

还是回到 Dubbo对Spring容器刷新完成事件进行处理的地方，继续exportServices的执行。
**org.apache.dubbo.config.bootstrap.DubboBootstrap#exportServices**
```java
private void exportServices() {

    // configManager.getServices()得到了所有ServiceBean配置
    // 循环处理每个ServiceBean。
    configManager.getServices().forEach(sc -> {
        // TODO, compatible with ServiceConfig.export()
        ServiceConfig serviceConfig = (ServiceConfig) sc;
        serviceConfig.setBootstrap(this);

        // exportAsync变量由：dubbo:provider.async 配置决定，
        // 默认为false，默认不开启异步暴露。
        if (exportAsync) {
            ExecutorService executor = executorRepository.getServiceExporterExecutor();
            Future<?> future = executor.submit(() -> {
                sc.export();
                exportedServices.add(sc);
            });
            asyncExportingFutures.add(future);
        } else {

            // 逐个对ServiceBean配置进行服务暴露
            // 接下来进入export()方法实现
            sc.export();
            exportedServices.add(sc);
        }
    });
}
```
通过一个服务配置ServiceBean，进行服务的暴露
**org.apache.dubbo.config.ServiceConfig#export**
```java
public synchronized void export() {
    // 判断是否应该暴露服务，涉及两项配置：
    // 应用级别服务提供配置：dubbo:provider.export；
    // SpringBoot可以在配置文件中配置dubbo.provider.export=false
    // 服务级别服务提供配置：dubbo:service.export；
    // 也可以通过注解配置：@DubboService(export = false)
    // 服务级别配置优先级高于应用级别
    if (!shouldExport()) {
        return;
    }

    if (bootstrap == null) {
        bootstrap = DubboBootstrap.getInstance();
        bootstrap.init();
    }

    checkAndUpdateSubConfigs();

    // 服务配置的元数据处理
    //init serviceMetadata
    serviceMetadata.setVersion(version);
    serviceMetadata.setGroup(group);
    serviceMetadata.setDefaultGroup(group);

    // 设置要暴露服务的接口类型
    serviceMetadata.setServiceType(getInterfaceClass());

    // 设置要暴露服务的接口名称
    serviceMetadata.setServiceInterfaceName(getInterface());

    // 设置要暴露服务的具体本地实现类的实例
    serviceMetadata.setTarget(getRef());

    // 是否需要延迟暴露，同export配置相似，也有服务级别与应用级别两层延时设置，
    // 服务级别优先级高于应用级别
    if (shouldDelay()) {
        // 延迟暴露
        DELAY_EXPORT_EXECUTOR.schedule(this::doExport, getDelay(), TimeUnit.MILLISECONDS);
    } else {
        // 不延迟，直接暴露
        // 接下来进入该方法实现
        doExport();
    }

    exported();
}
```
**org.apache.dubbo.config.ServiceConfig#doExport**
```java
protected synchronized void doExport() {
    // unexported变量标记服务被关闭，如果服务被关闭了，不再重新暴露，抛出异常
    if (unexported) {
        throw new IllegalStateException("The service " + interfaceClass.getName() + " has already unexported!");
    }
    // 如果服务已经暴露，跳过本次暴露
    if (exported) {
        return;
    }
    // 标记服务为已暴露
    exported = true;

    // dubbo:service.path配置，官方文档释义：提供者上下文路径，为服务path的前缀
    // 如果为空，取用接口名称
    if (StringUtils.isEmpty(path)) {
        path = interfaceName;
    }

    // 继续服务暴露流程
    // 接下来进入该方法实现
    doExportUrls();
}
```
**org.apache.dubbo.config.ServiceConfig#doExportUrls**
> 以下部分涉及参考资料，来自Dubbo官方文档：
**注册中心配置**。对应的配置类： **org.apache.dubbo.config.RegistryConfig**。同时如果有多个不同的注册中心，可以声明多个 <dubbo:registry> 标签，并在 <dubbo:service> 或 <dubbo:reference> 的 registry 属性指定使用的注册中心。
**registry.address属性**：注册中心服务器地址，如果地址没有端口缺省为9090，同一集群内的多个地址用逗号分隔，如：ip:port,ip:port，不同集群的注册中心，请配置多个<dubbo:registry>标签
**服务提供者协议配置**。对应的配置类： **org.apache.dubbo.config.ProtocolConfig**。同时，如果需要支持多协议，可以声明多个 <dubbo:protocol> 标签，并在 <dubbo:service> 中通过 protocol 属性指定使用的协议。
```java
private void doExportUrls() {
    //  获取「服务仓库」，暂且这么翻译吧
    // 这里使用了Dubbo SPI，
    // 关于Dubbo SPI机制，不清楚的可以参考我的另一篇博客
    // TODO:「这里是博客地址，现在还没写」
    ServiceRepository repository = ApplicationModel.getServiceRepository();
    // 根据当前实际要暴露的接口类，创建服务描述对象
    ServiceDescriptor serviceDescriptor = repository.registerService(getInterfaceClass());
    // 向服务仓库注册当前正在暴露的服务信息，
    // 包括服务的唯一名称，服务的实现类实例，服务描述，服务配置实例，服务元数据信息
    repository.registerProvider(
            getUniqueServiceName(),
            ref,
            serviceDescriptor,
            this,
            serviceMetadata
    );

    // 根据可能的多个注册中心配置，单个注册中心配置的address属性多个地址
    // 拼装得到所有的注册中心地址
    List<URL> registryURLs = ConfigValidationUtils.loadRegistries(this, true);

    // protocols是协议配置，允许初选多个协议配置
    // 遍历配置的所有协议，对每个协议都进行服务的暴露
    for (ProtocolConfig protocolConfig : protocols) {
        String pathKey = URL.buildKey(getContextPath(protocolConfig)
                .map(p -> p + "/" + path)
                .orElse(path), group, version);
        // In case user specified path, register service one more time to map it to path.

        // 服务配置若制定了分组group，版本version信息，向「服务仓库」中注册不同名称的服务
        repository.registerService(pathKey, interfaceClass);
        // TODO, uncomment this line once service key is unified
        serviceMetadata.setServiceKey(pathKey);

        // 对每种协议都进行暴露
        // 接下来进入该方法实现继续分析
        doExportUrlsFor1Protocol(protocolConfig, registryURLs);
    }
}
```
**org.apache.dubbo.config.ServiceConfig#doExportUrlsFor1Protocol**
这段代码很长，进行了一些删减，只保留了一些关键行
```java
private void doExportUrlsFor1Protocol(ProtocolConfig protocolConfig, List<URL> registryURLs) {
    String name = protocolConfig.getName();
    // 默认使用Dubbo协议
    if (StringUtils.isEmpty(name)) {
        name = DUBBO;
    }

    Map<String, String> map = new HashMap<String, String>();
    // 这里进行了灰常多的配置处理，把各种相关配置信息填充到map里面
    // 包括对接口类的方法的解析
    ...
    ...
    ...

    /**
    * Here the token value configured by the provider is used to assign the value to ServiceConfig#token
    */
    // service.token配置与provider.token配置
    // 依然是服务级别service的配置优先级高于应用级别provider的配置
    if(ConfigUtils.isEmpty(token) && provider != null) {
        token = provider.getToken();
    }
    // 令牌参数处理，添加到map
    if (!ConfigUtils.isEmpty(token)) {
        if (ConfigUtils.isDefault(token)) {
            map.put(TOKEN_KEY, UUID.randomUUID().toString());
        } else {
            map.put(TOKEN_KEY, token);
        }
    }
    //init serviceMetadata attachments
    serviceMetadata.getAttachments().putAll(map);

    // export service
    String host = findConfigedHosts(protocolConfig, registryURLs, map);
    Integer port = findConfigedPorts(protocolConfig, name, map);
    // 根据map收集的各种配置参数，生成URL对象，后续通过URL对象进行服务的暴露
    URL url = new URL(name, host, port, getContextPath(protocolConfig).map(p -> p + "/" + path).orElse(path), map);

    // You can customize Configurator to append extra parameters
    // 这儿又使用了Dubbo SPI，意思是可以通过自定义配置程序，来添加额外的参数
    // TODO: 表示没用过，等知道具体使用场景回头补充一下
    if (ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
            .hasExtension(url.getProtocol())) {
        url = ExtensionLoader.getExtensionLoader(ConfiguratorFactory.class)
                .getExtension(url.getProtocol()).getConfigurator(url).configure(url);
    }

    String scope = url.getParameter(SCOPE_KEY);
    // don't export when none is configured
    // Dubbo的scope属性，官方文档配置里面没找到解释
    // 大概逻辑是：scope配置的属性值不是none，就需要进行暴露
    // 不是remote，就需要进行本地的暴露，本地暴露是通过injvm协议进行暴露
    // 不是local，就需要进行远程的暴露
    if (!SCOPE_NONE.equalsIgnoreCase(scope)) {

        // export to local if the config is not remote (export to remote only when config is remote)
        if (!SCOPE_REMOTE.equalsIgnoreCase(scope)) {
            exportLocal(url);
        }
        // export to remote if the config is not local (export to local only when config is local)
        if (!SCOPE_LOCAL.equalsIgnoreCase(scope)) {
            if (CollectionUtils.isNotEmpty(registryURLs)) {
                // 远程暴露要对一堆配置中心地址挨个暴露
                for (URL registryURL : registryURLs) {
                    //if protocol is only injvm ,not register
                    if (LOCAL_PROTOCOL.equalsIgnoreCase(url.getProtocol())) {
                        continue;
                    }
                    //对URL的一些处理，好奇每一步细节逻辑可以自行翻代码debug查看
                    ...
                    ...
                    ...

                    // For providers, this is used to enable custom proxy to generate invoker
                    String proxy = url.getParameter(PROXY_KEY);
                    // 设置指定的生成动态代理方式，可选：jdk/javassist
                    if (StringUtils.isNotEmpty(proxy)) {
                        registryURL = registryURL.addParameter(PROXY_KEY, proxy);
                    }

                    // 重点，将服务的具体实现，通过动态代理包装一层，包装成一个invoker对象，便于调用
                    // PROXY_FACTORY变量使用了Dubbo SPI机制+自适应扩展,下面贴图展示了PROXY_FACTORY这个变量对应的类代码，是动态代理生成的
                    // 根据动态代理生成的代码中可以看到，url.getParameter("proxy", "javassist");
                    // 根据上面代码所示的proxy参数，是在动态代理类中，结合Dubbo SPI机制，达到自适应扩展
                    Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
                    // 将服务配置元数据与上面一行封装的invoker调用对象组合封装一层
                    DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                    // PROTOCOL变量也使用了Dubbo SPI机制+自适应扩展，可以参考PROXY_FACTORY的效果，此处不再赘述
                    // 根据配置对应的Protocol进行export()，默认为Dubbo协议，
                    // 接下来的从协议层继续暴露的分析需进入org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol#export方法
                    Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
                    exporters.add(exporter);
                }
            } else {
                if (logger.isInfoEnabled()) {
                    logger.info("Export dubbo service " + interfaceClass.getName() + " to url " + url);
                }
                // 如果没有出现注册中心配置，即配置的是直连模式，会走到这里
                // 其余逻辑同上
                Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, url);
                DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                Exporter<?> exporter = PROTOCOL.export(wrapperInvoker);
                exporters.add(exporter);
            }
            /**
                * @since 2.7.0
                * ServiceData Store
                */
            // 服务暴露后，将服务地址记录到本地内存
            // 这里也使用了Dubbo SPI，默认是local，
            // 即local=org.apache.dubbo.metadata.store.InMemoryWritableMetadataService实现
            WritableMetadataService metadataService = WritableMetadataService.getExtension(url.getParameter(METADATA_KEY, DEFAULT_METADATA_STORAGE_TYPE));
            if (metadataService != null) {
                metadataService.publishServiceDefinition(url);
            }
        }
    }
    this.urls.add(url);
}
```
PROXY_FACTORY变量的类型，动态代理生成的代码：
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210309153252.png)
这里再贴一下PROTOCOL变量类型的动态代理代码，因为这个protocol.export()方法执行后，有一个小插曲，有这个代码细节会更清晰一些。
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210309160106.png)
理想中代码应该执行到DubboProtocol类了，但是其实中间还有一个闯入者，根据下图分析：
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210309161026.png)
可以看到在执行export()方法时，传入的变量，getUrl().getProtocol()得到的结果是registry，所以PTOTOCOL变量对应的类的export()方法执行，通过Dubbo SPI机制+自适应，先来到了
**org.apache.dubbo.registry.integration.RegistryProtocol#export**
这一层是对注册中心协议的处理，如使用了zookeeper作为注册中心，则可以认为注册中心协议为zookeeper。这一块的逻辑不详细解析代码了，简单叙述下。
1. 这一行通过Dubbo SPI，执行了DubboProtocol.export(),将服务以dubbo协议暴露。
    ```java
    final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);
    ```
2. 把服务地址注册到zookeeper，本质就是在zookeeper上新建节点，存储如下格式的信息：
protocol://ip:port/interfaceName?serviceParams&providerParams
    ```java
    register(registryUrl, registeredProviderUrl);
    ```
还有涉及注册协议监听器，RegistryProtocolListener，通过实现自己的监听器来定制功能。
接下来就是协议层的暴露，进入代码方法
**org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol#export**
关键代码：
```java
// 打开服务
openServer(url);
```
后面的代码相对前面那些没有那么关键，就不再贴代码进行解释。
通过流程图简单理解下后续的流程。
![](https://cdn.jsdelivr.net/gh/yitulin/pictures/images/dubbo_20210309230612.png)
自此，Dubbo的服务暴露流程算是解析结束了，而在起点处的
```java
org.apache.dubbo.config.bootstrap.DubboBootstrap#exportServices
```
代码下面没几行，出现了一行代码：
```java
org.apache.dubbo.config.bootstrap.DubboBootstrap#referServices
```
显然，这里有去进行服务的引入。关于服务引入流程，请听下回分解。
### 流程图总结回顾
下次一定！
