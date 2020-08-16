# @SpringBootApplication

@EnableAutoConfiguration：SpringBoot根据应用所声明的依赖来对Spring框架进行自动配置。简单概括一下就是，是借助@Import的帮助，将所有符合自动配置条件的bean定义加载到IoC容器。

@Configuration：它就是JavaConfig形式的Spring Ioc容器的配置类。被标注的类等于在spring的XML配置文件中(applicationContext.xml)，装配所有bean事务，提供了一个spring的上下文环境。

@ComponentScan：组件扫描，可自动发现和装配Bean，功能其实就是自动扫描并加载符合条件的组件或者bean定义，最终将这些bean定义加载到IoC容器中。可以通过basePackages等属性来细粒度的定制@ComponentScan自动扫描的范围，如果不指定，则默认Spring框架实现会从声明@ComponentScan所在类的package进行扫描。默认扫描SpringApplication的run方法里的Booter.class所在的包路径下文件，所以最好将该启动类放到根包路径下。

# 初始化

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.sources = new LinkedHashSet();
    this.bannerMode = Mode.CONSOLE;
    this.logStartupInfo = true;
    this.addCommandLineProperties = true;
    this.addConversionService = true;
    this.headless = true;
    this.registerShutdownHook = true;
    this.additionalProfiles = new HashSet();
    this.isCustomEnvironment = false;
    this.lazyInitialization = false;
    this.resourceLoader = resourceLoader;	//null
    Assert.notNull(primarySources, "PrimarySources must not be null");
    //主类的完整类名
    this.primarySources = new LinkedHashSet(Arrays.asList(primarySources));
    //配置环境是否为Web，"SERVLET"
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    //创建初始化构造器，扫描resources路径下META-INF/spring.factories文件并加载
    this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
    //创建应用监听器，扫描resources路径下META-INF/spring.factories文件并加载
    this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
    //配置应用主方法所在类
    this.mainApplicationClass = this.deduceMainApplicationClass();
}
```

```java
//Reactive：Spring团队推出的Reactor编程模型的非阻塞异步Web编程框架WebFlux
if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) 
    && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null) 
    && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
    return WebApplicationType.REACTIVE;
}
//None：非Web应用程序
for (String className : SERVLET_INDICATOR_CLASSES) {
    if (!ClassUtils.isPresent(className, null)) {
        return WebApplicationType.NONE;
    }
}
//Servlet：基于J2EE Servlet API的编程模型，运行在Servlet容器上
return WebApplicationType.SERVLET;

//通过类路径中是否存在WebFlux中的Dispatcherhandler，SpringMVC中的DispatcherServlet、Servlet、ConfigurableWebApplicationContext来推断Web应用程序类型
```

```java
//根据前面的type创建不同的Environment对象
switch(this.webApplicationType) {
case SERVLET:
    return new StandardServletEnvironment();
case REACTIVE:
    return new StandardReactiveWebEnvironment();
default:
    return new StandardEnvironment();
}
```

```java
//应用程序如果有命令行参数，则在Environment中添加一个与这个命令行参数相关的PropertySource 
//根据命令行参数中spring.profiles.active属性配置Environment对象中的activeProfile
if (this.addCommandLineProperties && args.length > 0) {
    String name = "commandLineArgs";
    if (sources.contains(name)) {
        PropertySource<?> source = sources.get(name);
        CompositePropertySource composite = new CompositePropertySource(name);
        composite.addPropertySource(new SimpleCommandLinePropertySource("springApplicationCommandLineArgs", args));
        composite.addPropertySource(source);
        sources.replace(name, composite);
    } else {
        sources.addFirst(new SimpleCommandLinePropertySource(args));
    }
}
```

1. New SpringApplication()构造一个Spring应用
2. 配置SpringApplication的相关参数：
3. 配置环境是否为Web，"SERVLET"
4. 创建初始化构造器，主要是创建各个Spring工厂实例
   1. org.springframework.boot.devtools.restart.RestartScopeInitializer
   2. org.springframework.boot.context.ConfigurationWarningsApplicationContextInitializer
   3. org.springframework.boot.context.ContextIdApplicationContextInitializer
   4. org.springframework.boot.context.config.DelegatingApplicationContextInitializer
   5. org.springframework.boot.rsocket.context.RSocketPortInfoApplicationContextInitializer
   6. org.springframework.boot.web.context.ServerPortInfoApplicationContextInitializer
   7. org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer
   8. org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener
5. 创建应用监听器
   1. org.springframework.boot.devtools.restart.RestartApplicationListener
   2. org.springframework.boot.devtools.logger.DevToolsLogFactory.Listener
   3. org.springframework.boot.ClearCachesApplicationListener
   4. org.springframework.boot.builder.ParentContextCloserApplicationListener
   5. org.springframework.boot.cloud.CloudFoundryVcapEnvironmentPostProcessor
   6. org.springframework.boot.context.FileEncodingApplicationListener
   7. org.springframework.boot.context.config.AnsiOutputApplicationListener
   8. org.springframework.boot.context.config.ConfigFileApplicationListener
   9. org.springframework.boot.context.config.DelegatingApplicationListener
   10. org.springframework.boot.context.logging.ClasspathLoggingApplicationListener
   11. org.springframework.boot.context.logging.LoggingApplicationListener
   12. org.springframework.boot.liquibase.LiquibaseServiceLocatorApplicationListener
   13. org.springframework.boot.autoconfigure.BackgroundPreinitializer
6. 配置应用主方法所在类

# 运行

```java
public ConfigurableApplicationContext run(String... args) {
    // 构造一个StopWatch，观察SpringApplication的执行
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
    // 设置了一些环境变量
    this.configureHeadlessProperty();
    // 获取事件监听器SpringApplicationRunListeners类型，并执行starting()方法
    SpringApplicationRunListeners listeners = this.getRunListeners(args);
    listeners.starting();
    
    Collection exceptionReporters;
    try {
        // 将args封装成ApplicationArgument对象
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 准备配置环境，并将配置环境与spring上下文绑定，执行environmentPrepared()方法
        ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
        // 设置一些环境的值
        this.configureIgnoreBeanInfo(environment);
        // 打印Banner
        Banner printedBanner = this.printBanner(environment);
        // 根据项目类型创建上下文，AnnotationConfigServletWebServerApplicationContext
        context = this.createApplicationContext();
        // 获取异常报告事件监听
        exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
        // 准备上下文，执行contextPrepared()方法、contextLoaded()方法
        this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        // 重点！spring启动代码，启动内置Tomcat，扫描并初始化Bean
        this.refreshContext(context);
        // 什么都没做
        this.afterRefresh(context, applicationArguments);
        stopWatch.stop();
        if (this.logStartupInfo) {
            (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
        }
        // 执行ApplicationRunListeners中的started()方法
        listeners.started(context);
        // 执行Runner(ApplicationRunner和CommandLineRunner)
        this.callRunners(context, applicationArguments);
    } catch (Throwable var10) {
        this.handleRunFailure(context, var10, exceptionReporters, listeners);
        throw new IllegalStateException(var10);
    }

    try {
        listeners.running(context);
        return context;
    } catch (Throwable var9) {
        this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
        throw new IllegalStateException(var9);
    }
}
```



## 创建应用上下文

```java
context = this.createApplicationContext();

protected ConfigurableApplicationContext createApplicationContext() {
        Class<?> contextClass = this.applicationContextClass;
        if (contextClass == null) {
            try {
                // 根据不同的webApplicationType创建不同的上下文对象
                switch(this.webApplicationType) {
                case SERVLET:
                    contextClass = Class.forName("org.springframework.boot.web.servlet.context.AnnotationConfigServletWebServerApplicationContext");
                    break;
                case REACTIVE:
                    contextClass = Class.forName("org.springframework.boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext");
                    break;
                default:
                    contextClass = Class.forName("org.springframework.context.annotation.AnnotationConfigApplicationContext");
                }
            } catch (ClassNotFoundException var3) {
                throw new IllegalStateException("Unable create a default ApplicationContext, please specify an ApplicationContextClass", var3);
            }
        }

        return (ConfigurableApplicationContext)BeanUtils.instantiateClass(contextClass);
    }
```

## 配置应用上下文

```java
this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);

private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
    // 设置context的Environment对象
    context.setEnvironment(environment);
    this.postProcessApplicationContext(context);
    // 执行所有ApplicationContextInitializer的initialize方法
    // 这些ApplicationContextInitializer是在SpringApplication中的构造函数中加载的（通过读取spring.factories加载）
    this.applyInitializers(context);
    // 发布ApplicationPreparedEvent（上下文已准备）事件
    listeners.contextPrepared(context);
    
    // ...
    
    Set<Object> sources = this.getAllSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    this.load(context, sources.toArray(new Object[0]));
    // 发布ApplicationLoadedEvent（上下文已加载）事件
    listeners.contextLoaded(context);
}
```

## 刷新应用上下文

```java
this.refreshContext(context);

private void refreshContext(ConfigurableApplicationContext context) {
    this.refresh(context);
    if (this.registerShutdownHook) {
        try {
            context.registerShutdownHook();
        } catch (AccessControlException var3) {
        }
    }
}

protected void refresh(ApplicationContext applicationContext) {
    Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    ((AbstractApplicationContext)applicationContext).refresh();
}

//spring容器启动代码
public void refresh() throws BeansException, IllegalStateException {
    synchronized(this.startupShutdownMonitor) {
        // 准备工作
        this.prepareRefresh();
        // 获取上下文的BeanFactory
        ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
        // 对BeanFactory做一些预备工作
        this.prepareBeanFactory(beanFactory);

        try {	
            // 增加一个后置处理器，忽略一个依赖接口
            this.postProcessBeanFactory(beanFactory);
            // 执行beanFactory的所有后置处理器
            this.invokeBeanFactoryPostProcessors(beanFactory);
            // 注册后置处理器
            this.registerBeanPostProcessors(beanFactory);
            // 初始化MessageSource(国际化相关)//忽略
            this.initMessageSource();
            // 构造了一个SimpleApplicationEventMulticaster当成默认的事件广播器
            this.initApplicationEventMulticaster();
            // 初始化ThemeSource(跟国际化相关的接口);调用createWebServer()创建WebServer
            this.onRefresh();
            // 将所有监听器注册到前两步创建的事件广播器中
            this.registerListeners();
            // 结束BeanFactory的初始化工作(这一步主要用来将所有的单例BeanDefinition实例化)
            this.finishBeanFactoryInitialization(beanFactory);
            // 上下文刷新完毕，启动WebServer
            this.finishRefresh();
        } catch (BeansException var9) {
            if (this.logger.isWarnEnabled()) {
                this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
            }

            this.destroyBeans();
            this.cancelRefresh(var9);
            throw var9;
        } finally {
            this.resetCommonCaches();
        }

    }
}

protected void prepareRefresh() {
    // ...
    
	// 初始化PropertySource
    this.initPropertySources();
    // 验证Environment中的必要属性
    this.getEnvironment().validateRequiredProperties();
    
    // ...
}

protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    //设置类加载器
    beanFactory.setBeanClassLoader(this.getClassLoader());
    //设置表达式解析器
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    //设置属性编辑器
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, this.getEnvironment()));
    //添加后置处理器
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
    //自动装配时忽略一些接口类型
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
    //注册可解析的依赖
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);
    //添加后置处理器
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
    //注册一些单例对象
    if (beanFactory.containsBean("loadTimeWeaver")) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    if (!beanFactory.containsLocalBean("environment")) {
        beanFactory.registerSingleton("environment", this.getEnvironment());
    }

    if (!beanFactory.containsLocalBean("systemProperties")) {
        beanFactory.registerSingleton("systemProperties", this.getEnvironment().getSystemProperties());
    }

    if (!beanFactory.containsLocalBean("systemEnvironment")) {
        beanFactory.registerSingleton("systemEnvironment", this.getEnvironment().getSystemEnvironment());
    }
}

protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 添加后置处理器
    beanFactory.addBeanPostProcessor(new WebApplicationContextServletContextAwareProcessor(this));
    // 忽略依赖接口
    beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    this.registerWebApplicationScopes();
}

public static void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
    // 从beanFactory中获取所有postProcessor
    String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
    int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
    // 添加后置处理器
    beanFactory.addBeanPostProcessor(new PostProcessorRegistrationDelegate.BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));
    
    // 将所有BeanPostProcessor按照实现了PriorityOrdered、Ordered、没有实现排序接口的顺序注册所有BeanPostProcessor到BeanFactory
    
    // ...
    
    // 在BeanFactory中注册一个类型为ApplicationListenerDetector的BeanPostProcessor
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}

// 由于AbstractApplicationContext是抽象类，具体实现在ServletWebServerApplicationContext中
protected void onRefresh() {
    super.onRefresh();
    try {
        // 创建WebServer
        this.createWebServer();
    } catch (Throwable var2) {
        throw new ApplicationContextException("Unable to start web server", var2);
    }
}

private void createWebServer() {
    WebServer webServer = this.webServer;
    ServletContext servletContext = this.getServletContext();
    if (webServer == null && servletContext == null) {
        // 获取webServerFactory
        ServletWebServerFactory factory = this.getWebServerFactory();
        // 创建webServer
        this.webServer = factory.getWebServer(new ServletContextInitializer[]{this.getSelfInitializer()});
    } else if (servletContext != null) {
        try {
            this.getSelfInitializer().onStartup(servletContext);
        } catch (ServletException var4) {
            throw new ApplicationContextException("Cannot initialize servlet context", var4);
        }
    }

    this.initPropertySources();
}

protected ServletWebServerFactory getWebServerFactory() {
    //  tomcatServletWebServerFactory
    String[] beanNames = this.getBeanFactory().getBeanNamesForType(ServletWebServerFactory.class);
    // ... 根据beanNames创建对应的ServletWebServerFactory
}

public WebServer getWebServer(ServletContextInitializer... initializers) {
    if (this.disableMBeanRegistry) {
        Registry.disableRegistry();
    }

    Tomcat tomcat = new Tomcat();
    File baseDir = this.baseDirectory != null ? this.baseDirectory : this.createTempDir("tomcat");
    tomcat.setBaseDir(baseDir.getAbsolutePath());
    Connector connector = new Connector(this.protocol);
    connector.setThrowOnFailure(true);
    tomcat.getService().addConnector(connector);
    this.customizeConnector(connector);
    tomcat.setConnector(connector);
    tomcat.getHost().setAutoDeploy(false);
    this.configureEngine(tomcat.getEngine());
    Iterator var5 = this.additionalTomcatConnectors.iterator();

    while(var5.hasNext()) {
        Connector additionalConnector = (Connector)var5.next();
        tomcat.getService().addConnector(additionalConnector);
    }

    this.prepareContext(tomcat.getHost(), initializers);
    return this.getTomcatWebServer(tomcat);
}

protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // ...
    
    // 
    beanFactory.preInstantiateSingletons();
}

protected void finishRefresh() {
    super.finishRefresh();
    // 启动WebServer
    WebServer webServer = this.startWebServer();
    if (webServer != null) {
        // 发布ServletWebServerInitializedEvent事件
        this.publishEvent(new ServletWebServerInitializedEvent(webServer, this));
    }

}

protected void finishRefresh() {
    this.clearResourceCaches();
    // 初始化生命周期，向BeanFactory注册一个DefaultLifecycleProcessor 
    this.initLifecycleProcessor();
    // 调用LifecycleProcessor的onRefresh()方法（找到所有已注册的SmartLifecycle，根据isRunning和isAutoStartup的条件判断，执行SmartLifecycle的start方法）
    this.getLifecycleProcessor().onRefresh();
    this.publishEvent((ApplicationEvent)(new ContextRefreshedEvent(this)));
    LiveBeansView.registerApplicationContext(this);
}
```

# Tomcat的启动

- 创建SpringApplication对象并初始化
- run()
  - this.refreshContext(context)
    - this.refresh(context)
      - ((AbstractApplicationContext)applicationContext).refresh();
        - this.onRefresh();
          - this.createWebServer();	在ServletWebServerApplicationContext中实现
            - ServletWebServerFactory factory = this.getWebServerFactory();
              - 获取到beanNames，值为"tomcatServletWebServerFactory"
            - this.webServer = factory.getWebServer(new ServletContextInitializer[]{this.getSelfInitializer()});
        - this.finishRefresh();
          - this.startWebServer();
            - webServer.start();

# 启动总结

- 创建SpringApplication对象并初始化
  - 设置各种参数
  - 设置类型：web下类型为SERVLET
  - 从resources路径下的META-INF/spring.factories中加载配置信息，并创建初始化构造器：8个
  - 从resources路径下的META-INF/spring.factories中加载配置信息，并创建事件监听器：12个
  - 设置应用主方法所在的完整类名
- 调用SpringApplication.run()方法，注册所有的Bean，并启动内置Tomcat
  - 构造一个StopWatch对象，观察SpringApplication的执行
  - 将所有的SpringApplicationRunListener封装到SpringApplicationRunListeners中，用于监听run()方法的执行
  - 创建并配置Environment(PropertySource、Profile)
  - 构造ApplicationContext对象
    - 根据web类型，构造不同的对象，SERVLET下为**AnnotationConfigServletWebServerApplicationContext**
  - 准备ApplicationContext
    - 将context中的environment替换成SpringApplication中创建的environment
    - 将SpringApplication中的initializers应用到context中
    - 加载两个单例bean到beanFactory中
      - springApplicationArguments
      - springBootBanner
    - 加载bean定义资源
    - 将SpringApplication中的listeners注册到context中，并广播ApplicationPreparedEvent事件
  - 刷新ApplicationContext
    - 对刷新进行准备,包括设置开始时间,设置激活状态,初始化Context中的占位符,子类根据其需求执行具体准备工作,而后再由父类验证必要参数
    - 刷新并获取内部的BeanFactory对象
    - 对BeanFactory进行准备工作,包括设置类加载器和后置处理器,配置不能自动装配的类型,注册默认的环境Bean
    - 为Context的子类提供后置处理BeanFactory的扩展能力,如想在bean定义加载完成后,开始初始化上下文之前进行逻辑操作,可重写这个方法
    - 执行Context中注册的BeanFactory后置处理器,有两种处理器,一种是可以注册Bean的后置处理器,一种的针对BeanFactory的后置处理器,执行顺序是先按优先级执行注册Bean的后置处理器,而后再按优先级执行针对BeanFactory的后置处理器
    - 按优先级顺序在BeanFactory中注册Bean的后置处理器,Bean处理器可在Bean的初始化前后处理
    - 初始化消息源,消息源用于支持消息的国际化
    - 初始化应用事件广播器,用于向ApplicationListener通知各种应用产生的事件,标准的观察者模型
    - 用于子类的扩展步骤,用于特定的Context子类初始化其他的Bean，实际上调用了ServletWebServerApplicationContext中onRefresh()方法，**创建了WebServer，**也就是Tomcat
    - 把实现了ApplicationListener的类注册到广播器,并对广播其中早期没有广播的事件进行通知
    - 冻结所有Bean描述信息的修改,**实例化非延迟加载的单例Bean**
    - 完成上下文的刷新工作,调用LifecycleProcessor.onRefresh(),**启动了WebServer**



# 初始化策略

- 初始化多部份(文件)解析器
- 初始化语言环境解析器
- 初始化主题解析器
- 初始化HandlerMappings
  - requestMappingHandlerMapping
  - welcomePageHandlerMapping
  - beanNameHandlerMapping
  - routerFunctionMapping
  - resourceHandlerMapping
- 初始化HandlerAdapters
  - requestMappingHandlerAdapter
  - handlerFunctionAdapter
  - httpRequestHandlerAdapter
  - simpleControllerHandlerAdapter
- 初始化Handler异常解析器
- 初始化请求到视图名称转换器
- 初始化视图解析器
- 初始化Flash Map Manager

# 请求处理流程

- 容器调用DispatcherServlet中的service方法
- 根据请求类型调用doGet()，doPost()等方法
- 执行processRequest()方法，执行请求上下文的从初始化过程
- 调用doService()方法
  - 给request设置一些属性
  - 执行核心doDispatch()方法
- 调用doDispatch()方法
- 获取MappedHandler
- 获取HandlerAdapter
- 利用反射执行方法
- 通过新建一个ServletWebRequest来渲染返回数据

# 自动配置

- @SpringBootApplication
  - @SpringBootConfiguration
  - @ComponentScan
  - @EnableAutoConfiguration
    - @AutoConfigurationPackage
      - @Import({**Registrar.class**})
        - AutoConfigurationPackages.register(registry, (new AutoConfigurationPackages.PackageImport(metadata)).getPackageName());
          - 其中：metadata是@SpringBootApplication注解上的类，也就是主配置类，说白了就是将主配置类（即@SpringBootApplication标注的类）的所在包及子包里面所有组件扫描加载到Spring容器。所以包名一定要注意。
    - @Import({**AutoConfigurationImportSelector.class**})
      - public String[] **selectImports**(AnnotationMetadata annotationMetadata)
      - public void **process**(AnnotationMetadata annotationMetadata, DeferredImportSelector deferredImportSelector)
      - private AutoConfigurationMetadata **getAutoConfigurationMetadata**()
        - static AutoConfigurationMetadata **loadMetadata**(ClassLoader classLoader)
          - 属于类**AutoConfigurationMetadataLoader**的静态方法
        - static AutoConfigurationMetadata **loadMetadata**(ClassLoader classLoader, String path)
          - 被传入的参数：path="META-INF/spring-autoconfigure-metadata.properties"
          - 从META-INF/spring-autoconfigure-metadata.properties中获取资源，通过Properties加载资源
          - 获取到了**493**个配置项
      - protected AutoConfigurationImportSelector.AutoConfigurationEntry **getAutoConfigurationEntry**(AutoConfigurationMetadata autoConfigurationMetadata, AnnotationMetadata annotationMetadata)
        - protected List<String> **getCandidateConfigurations**(AnnotationMetadata metadata, AnnotationAttributes attributes)
        - public static List<String> **loadFactoryNames**(Class<?> factoryType, @Nullable ClassLoader classLoader)
          - 属于类**SpringFactoriesLoader**的静态方法
        - private static Map<String, List<String>> **loadSpringFactories**(@Nullable ClassLoader classLoader)
          - 判断缓存
        - Enumeration<URL> urls = classLoader != null ? classLoader.getResources("**META-INF/spring.factories**") : ClassLoader.getSystemResources("META-INF/spring.factories");
          - 从META-INF/spring.factories中获取资源，通过Properties加载资源
          - 获取到**127**个配置类

# 自动配置生效

参考文档：https://blog.csdn.net/u014745069/article/details/83820511?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task

每一个XxxxAutoConfiguration自动配置类都是在某些条件之下才会生效的，这些条件的限制在Spring Boot中以注解的形式体现，常见的**条件注解**有如下几项：

> @ConditionalOnBean：当容器里有指定的bean的条件下。
>
> @ConditionalOnMissingBean：当容器里不存在指定bean的条件下。
>
> @ConditionalOnClass：当类路径下有指定类的条件下。
>
> @ConditionalOnMissingClass：当类路径下不存在指定类的条件下。
>
> @ConditionalOnProperty：指定的属性是否有指定的值，比如@ConditionalOnProperties(prefix=”xxx.xxx”, value=”enable”, matchIfMissing=true)，代表当xxx.xxx为enable时条件的布尔值为true，如果没有设置的情况下也为true。

以RedisAutoConfiguration配置类为例，解释一下全局配置文件中的属性如何生效，比如：spring.redis.host=127.0.0.1，是如何生效的。

```java
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnClass({RedisOperations.class})
@EnableConfigurationProperties({RedisProperties.class})
@Import({LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class})
public class RedisAutoConfiguration {
    // 内容省略，重点放在注解部分
}
```

在ServletWebServerFactoryAutoConfiguration类上，有一个@EnableConfigurationProperties注解：**开启配置属性**，而它后面的参数是一个RedisProperties类，这就是习惯优于配置的最终落地点。

```java
package org.springframework.boot.autoconfigure.data.redis;

import java.time.Duration;
import java.util.List;
import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(
    prefix = "spring.redis"
)
public class RedisProperties {
    private int database = 0;
    private String url;
    private String host = "localhost";
    private String password;
    private int port = 6379;
    private boolean ssl;
    private Duration timeout;
    private String clientName;
    private RedisProperties.Sentinel sentinel;
    private RedisProperties.Cluster cluster;
    private final RedisProperties.Jedis jedis = new RedisProperties.Jedis();
    private final RedisProperties.Lettuce lettuce = new RedisProperties.Lettuce();

    // 内容省略
}

```

在这个类上，我们看到了一个非常熟悉的注解：@ConfigurationProperties，它的作用就是从配置文件中绑定属性到对应的bean上，而@EnableConfigurationProperties负责导入这个已经绑定了属性的bean到spring容器中。那么所有其他的和这个类相关的属性都可以在全局配置文件中定义，也就是说，真正“限制”我们可以在全局配置文件中配置哪些属性的类就是这些XxxxProperties类，它与配置文件中定义的prefix关键字开头的一组属性是唯一对应的。

至此，我们大致可以了解。在全局配置的属性如：spring.redis.host等，通过@ConfigurationProperties注解，绑定到对应的XxxxProperties配置实体类上封装为一个bean，然后再通过@EnableConfigurationProperties注解导入到Spring容器中。

而诸多的XxxxAutoConfiguration自动配置类，就是Spring容器的JavaConfig形式，作用就是为Spring 容器导入bean，而所有导入的bean所需要的属性都通过xxxxProperties的bean来获得。

总结一下就是：

> Spring Boot**启动的时候**会通过**@EnableAutoConfiguration**注解找到**META-INF/spring.factories**配置文件中的所有自动配置类，并对其进行加载，而这些自动配置类都是以**AutoConfiguration**结尾来命名的，它实际上就是一个JavaConfig形式的**Spring容器配置类**，它能通过以**Properties结尾**命名的类中取得在全局配置文件中配置的属性如：spring.redis.host，而XxxxProperties类是通过**@ConfigurationProperties**注解与全局配置文件中对应的属性进行绑定的。