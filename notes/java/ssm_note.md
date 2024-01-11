# 1 Spring
## 1.1 IoC
### 1.1.1 基于 XML 的 IoC
### 1.1.2 基于注解的 IoC
- `Spring` 默认不使用注解装配 `Bean`，需要在 `Spring` 的 `XML` 配置中开启 `Spring Beans` 的自动扫描功能
```xml
<context:component-scan base-package="pka_name1,pkg_name2"/>
<!-- 还可以指定排除某些注解或类，以及指定扫描哪些注解或类-->
<context:exclude-filter type="annotation" expression="FQCN of annotation"/>
<context:exclude-filter type="assignable" expression="FQCN of class"/>

<context:include-filter type="annotation" expression="FQCN of annotation"/>
<context:include-filter type="assignable" expression="FQCN of class"/>
```
- 在进行包扫描时，会对配置的包及其子包中所有文件进行扫描，多个包采用 ', ' 隔开
- 扫描过程仅读取合法的 `Java` 文件
- 扫描时仅读取 `Spring` 可识别的注解
- 扫描结束后会将可识别的有效注解转化为 `Spring` 对应的资源加入 `IoC` 容器
#### 1.1.2.1 定义 Bean 的注解
- @Component (c)
	- 用于描述 `Spring` 中的 `Bean`，可以用在任何层次，如 `Service` 层，`Dao` 层
- @Controller (c)
	- 用于控制层，功能与 `@Component` 相同
- @Service (c)
	- 用于业务层，功能与 `@Component` 相同
- @Repository (c)
	- 用于数据访问层，功能与 `@Component` 相同
#### 1.1.2.2 配置类注解
- @Configuration (c)
	- 声明当前类为配置类，相当于 `XML` 形式的 `Spring` 配置
	- 加载注解格式的上下文对象需要使用 `AnnotationConfigApplicationContext`
- @ComponentScan (c): 类似 `XML` 中的 `ComponentScan`
```java
@Configuration  
@ComponentScan("org.ranran.spring_ioc_annotations") // 开启扫描  
public class MyConfig {}
```
- @Bean (m)
	- 用于方法，声明当前方法的返回值为一个 `Spring` 管理的 `Bean`
	- 第三方 `Bean` 无法在其源码上进行修改，因此使用该注解解决第三方 `Bean` 的引入问题
#### 1.1.2.3 配置 Bean
- @Scope (c)
	- 设置 Spring 容器如何新建 `Bean` 实例
		- 默认为 `Singleton`
		- `Prototype`，非单例
- @PostConstruct (m)
	- 设置该方法为 `Bean` 对应的生命周期方法
	- 构造器执行完后执行
- @PreDestory (m)
	- 设置为生命周期方法
	- 在 Bean 销毁前执行
#### 1.1.2.4 注入
- **@Autowired**
```java
// Target 元注解表明 @Autowired 可以用于五个位置
@Target({ElementType.CONSTRUCTOR, ElementType.METHOD, ElementType.PARAMETER, ElementType.FIELD, ElementType.ANNOTATION_TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
public @interface Autowired {  
	// 要求被注入的对象要提前存在
	boolean required() default true;  
}
```
- `Controller` 注入 `Service`，`Service` 注入 `Dao`，均写在实现类中
- 注入方式
- 属性注入：在类中声明抽象类作为属性
```java
@Service  
public class MyServiceImpl implements MyService{  
    @Autowired    
    private MyDao dao;
}
```
- set 注入：在类中声明抽象类作为属性，定义其 `set` 方法后加注解
```java
@Autowired  
public void setDao(MyDao dao) {  
    this.dao = dao;  
}
```
- 构造器注入：在类的构造器上加注解
```java
@Autowired  
public MyController(MyService service) {  
    this.service = service;  
}
```
- 形参注入：在类的构造器参数列表中加注解
```java
public MyController(@Autowired MyService service) {  
    this.service = service;  
}
```
- <font color='red'>声明抽象类作为属性，只有一个有参构造的情况下不需要注入</font>
- @Autowired 和 @Qualifier 联合注入
	- @Autowired 按类型进行注入，如果一个接口有多个实现类时会报错
	- 因此配合 @Qualifier 匹配名称进行注入
```java
@Autowired
@Qualifier(value = "MyDaoImpl")
private MyDao dao;
```
- **@Resource**
	- `@Resource` 是 `JDK` 自带，而 `@Autowired` 是 `Spring` 中的
	- `@Resource` 默认根据 `name` 进行匹配，未指定 `name` 时通过类型进行匹配
	- `@Resource` 只能用于属性和 `Setter` 方法上
- 引入依赖
```xml
<dependency>  
    <groupId>jakarta.annotation</groupId>  
    <artifactId>jakarta.annotation-api</artifactId>  
    <version>2.1.1</version>  
</dependency>
```
- 根据名称注入
```java
@Service("service_name")

@Resource(name = "service_name")
private MyService service;
```
- 不写 `name` 时会根据变量名进行匹配
### 1.1.3 反射
- 获取 `Class` 对象的方式
	- `类名.class`
	- `对象.getClass()`
	- `Class.forName("FQCN")`
- 获取构造器
```java
clazz.getConstructors() // 只能获取 public 的构造器
clazz.getDeclaredConstructors() // 可以获取全部构造器

// getConstructor 检查构造器是否为 public
public Constructor<T> getConstructor(Class<?>... parameterTypes)  
    throws NoSuchMethodException, SecurityException  
{  
    @SuppressWarnings("removal")  
    SecurityManager sm = System.getSecurityManager();  
    if (sm != null) {  
        checkMemberAccess(sm, Member.PUBLIC, Reflection.getCallerClass(), true);  
    }  
    return getReflectionFactory().copyConstructor(  
        getConstructor0(parameterTypes, Member.PUBLIC));  
}

// getDeclaredConstructor 检查构造器是否不为 public
public Constructor<T> getDeclaredConstructor(Class<?>... parameterTypes)  
    throws NoSuchMethodException, SecurityException  
{  
    @SuppressWarnings("removal")  
    SecurityManager sm = System.getSecurityManager();  
    if (sm != null) {  
        checkMemberAccess(sm, Member.DECLARED, Reflection.getCallerClass(), true);  
    }  
  
    return getReflectionFactory().copyConstructor(  
        getConstructor0(parameterTypes, Member.DECLARED));  
}
```
- 通过有参构造器创建对象
```java
// public 的构造器
clazz.getConstructor(String.class, int.class, ...);

// private 的构造器
Constructor cons = clazz.getDeclaredConstructor(String.class, int.class, ...);
cons.setAccessible(true);
cons.newInstance(...);
```
- 获取属性
```java
clazz.getFields();
clazz.getDeclaredFields();
// 得到 Field 对象后，通过 getName() 方法获取属性名称
```
- 调用方法
```java
Method m = clazz.getMethod();
// m.invoke(obj, 参数列表)
```
### 1.1.4 创建流程
#### 1.1.4.1 入口
```java
new AnnotationConfigApplicationContext("pkg_name");

public AnnotationConfigApplicationContext(String... basePackages) {  
    this(); // 创建工厂组件
    scan(basePackages); // 扫描配置
    refresh(); // 初始化容器
}

public AnnotationConfigApplicationContext() { 
	// 创建工厂需要的组件
    this.reader = new AnnotatedBeanDefinitionReader(this);  
    this.scanner = new ClassPathBeanDefinitionScanner(this);  
}

```
#### 1.1.4.2 扫描 Bean 配置
```java
public void scan(String... basePackages) {  
    // 调用 ClassPathBeanDefinitionScanner 扫描 bean 配置信息
    this.scanner.scan(basePackages);  
}

// ClassPathBeanDefinitionScanner.java
public int scan(String... basePackages) { 
    // bean 配置信息的个数
    int beanCountAtScanStart = this.registry.getBeanDefinitionCount();  
    doScan(basePackages);  
    return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);  
}

protected Set<BeanDefinitionHolder> doScan(String... basePackages) {  
    Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();  
    for (String basePackage : basePackages) {  
       // 通过 findCandidateComponents 扫描 @Component 注解
       Set<BeanDefinition> candidates = findCandidateComponents(basePackage);  
       for (BeanDefinition candidate : candidates) {  
	      // 解析 bean 的 scope 作用域
          ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);  
          candidate.setScope(scopeMetadata.getScopeName());  
          // 生成 bean 的名称
          String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);  
          //设置bean的一些默认属性，lazy,init,destory方法等
          if (candidate instanceof AnnotatedBeanDefinition) {  AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);  
          }  
          // 检查是否已在缓存中
          if (checkCandidate(beanName, candidate)) {  
             BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);  
             definitionHolder =  
                   AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);  
             beanDefinitions.add(definitionHolder);  
             // 注册 bean 的配置到容器
             // beanDefinitionMap, beanDefinitionNames，aliasMap 三个缓存
             registerBeanDefinition(definitionHolder, this.registry);  
          }  
       }  
    }  
    return beanDefinitions;  
}
```
##### 1.1.4.2.1 扫描 @Component 注解
```java
// ClassPathScanningCandidateComponentProvider.java
public Set<BeanDefinition> findCandidateComponents(String basePackage) {  
    if (this.componentsIndex != null && indexSupportsIncludeFilters()) {...}  
    else {  
       return scanCandidateComponents(basePackage);  
    }  
}

private Set<BeanDefinition> scanCandidateComponents(String basePackage) {  
    Set<BeanDefinition> candidates = new LinkedHashSet<>();  
    try {  
       String packageSearchPath = ResourcePatternResolver.CLASSPATH_ALL_URL_PREFIX +  
             resolveBasePackage(basePackage) + '/' + this.resourcePattern; 
       // 扫描到的文件转换为 Resource 对象 
       Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);  
       for (Resource resource : resources) {  
          String filename = resource.getFilename();   
          try {  
	         // 获取 MetadataReader
             MetadataReader metadataReader = getMetadataReaderFactory().getMetadataReader(resource);  
             // 判断是否包含 @Component 注解
             if (isCandidateComponent(metadataReader)) {  
                ScannedGenericBeanDefinition sbd = new ScannedGenericBeanDefinition(metadataReader);  
                sbd.setSource(resource);  
                if (isCandidateComponent(sbd)) {   
                   candidates.add(sbd);  
                }  
             }  
          }  
       }  
    }  
    return candidates;  
}

protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {  
    // 判断 TypeFilter 是否匹配
    for (TypeFilter tf : this.includeFilters) {  
       if (tf.match(metadataReader, getMetadataReaderFactory())) {  
          return isConditionMatch(metadataReader);  
       }  
    }  
    return false;  
}

// includeFilters 中默认包含了 Component, ManagedBean, Named
protected void registerDefaultFilters() {
	this.includeFilters.add(new AnnotationTypeFilter(Component.class));
	this.includeFilters.add(new AnnotationTypeFilter(  
       ((Class<? extends Annotation>) ClassUtils.forName("jakarta.annotation.ManagedBean", cl)), false));
    this.includeFilters.add(new AnnotationTypeFilter(  
   ((Class<? extends Annotation>) ClassUtils.forName("jakarta.inject.Named", cl)), false));
}

// 第二次 isCandidateComponent 继续判断该类是否符合要求
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {   
	// isIndependent: 是否为顶级类或静态内部类
	// isConcrete: 不是接口类且不是抽象类
	// hasAnnotatedMethods：是否有使用注解的方法
    AnnotationMetadata metadata = beanDefinition.getMetadata();  
    return (metadata.isIndependent() && (metadata.isConcrete() ||  
          (metadata.isAbstract() && metadata.hasAnnotatedMethods(Lookup.class.getName()))));  
}
```
##### 1.1.4.2.2 判断是否已缓存过 Bean
```java
protected boolean checkCandidate(String beanName, BeanDefinition beanDefinition) throws IllegalStateException {  
	// 注册表没有，返回 true，表示可以注册该 bean 定义
    if (!this.registry.containsBeanDefinition(beanName)) {  
       return true;  
    }  
    // 注册表中包含该 beanName，获取到定义
    BeanDefinition existingDef = this.registry.getBeanDefinition(beanName);  
    BeanDefinition originatingDef = existingDef.getOriginatingBeanDefinition();  
    // 说明使用了代理，用原始的定义
    if (originatingDef != null) {  
       existingDef = originatingDef;  
    }  
    // 否则检查新旧定义兼容性
    if (isCompatible(beanDefinition, existingDef)) {  
       return false;  
    }  
}
```
##### 1.1.4.2.3 注册 Bean 定义
```java
protected void registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry) {  
    BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, registry);  
}

public static void registerBeanDefinition(  
       BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry) 
       throws BeanDefinitionStoreException {  
  
    // Register bean definition under primary name.  
    // 将 Bean 定义 添加到 beanDefinitionMap 中
    String beanName = definitionHolder.getBeanName();  
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());  
  
    // Register aliases for bean name, if any.  
    String[] aliases = definitionHolder.getAliases();  
    if (aliases != null) {  
       for (String alias : aliases) {  
          registry.registerAlias(beanName, alias);  
       }  
    }  
}
```
#### 1.1.4.3 容器初始化
```java
// 1.1.4.1
// AbstractApplicationContext.java
public void refresh() throws BeansException, IllegalStateException {  
    synchronized (this.startupShutdownMonitor) {  
  
       // 预处理工作
       prepareRefresh();  
  
       // 获取 Bean 工厂
       ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();  
  
       // 设置工厂配置属性信息
       prepareBeanFactory(beanFactory);  
  
       try {  
          // 子类的进一步设置
          postProcessBeanFactory(beanFactory);  

          // 执行 BeanFactoryPostProcessor后置处理器逻辑
          invokeBeanFactoryPostProcessors(beanFactory);  
  
          // 将实现了 BeanPostProcessor 接口的类信息添加到容器
          registerBeanPostProcessors(beanFactory);  
  
          // 初始化消息资源解析器 
          initMessageSource();  
  
          // 初始化事件监听器
          initApplicationEventMulticaster();  
  
          // 子类实现容器初始化扩展方法
          onRefresh();  
  
          // 获取注册监听器，分发容器初始化事件
          registerListeners();  
  
          // 创建非懒加载的 Bean
          finishBeanFactoryInitialization(beanFactory);  
  
          // 初始化之后的操作
          finishRefresh();  
       }  
    }  
}
```
##### 1.1.4.3.1 预处理工作
```java
protected void prepareRefresh() {  
    // Switch to active.  
    this.active.set(true);  
    // 子类实现初始化容器的属性值
    // AbstractApplicationContext -> GenericApplicationContext -> AnnotationConfigApplicationContext
    initPropertySources();  
  
    // Validate that all properties marked as required are resolvable:  
    getEnvironment().validateRequiredProperties();  
  
    // Allow for the collection of early ApplicationEvents,  
    // to be published once the multicaster is available...    
    this.earlyApplicationEvents = new LinkedHashSet<>();  
}
```
##### 1.1.4.3.2 获取 BeanFactory
```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() { 
    refreshBeanFactory(); 
    // getter 
    return getBeanFactory();  
}

protected final void refreshBeanFactory() throws BeansException {  
	// 清空已存在的 BeanFactory
    if (hasBeanFactory()) {  
       destroyBeans();  
       closeBeanFactory();  
    }  
    try {
	  // 创建 DefaultListableBeanFactory
       DefaultListableBeanFactory beanFactory = createBeanFactory();  
       beanFactory.setSerializationId(getId());  
       // 定义工厂创建 Bean 的规则
       customizeBeanFactory(beanFactory);  
       // 加载配置文件，解析为 BeanDefinition
       loadBeanDefinitions(beanFactory);  
       this.beanFactory = beanFactory;  
    }  
}
```
##### 1.1.4.3.3 设置工厂的基本属性
```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {  
    // 类加载器
    beanFactory.setBeanClassLoader(getClassLoader()); 
    // 表达式解析器 
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));  
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));  
  
    // Configure the bean factory with context callbacks.  
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));  
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);  
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);  
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);  
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);  
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);  
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);  
    beanFactory.ignoreDependencyInterface(ApplicationStartupAware.class);  
  
    // BeanFactory interface not registered as resolvable type in a plain factory.  
    // MessageSource registered (and found for autowiring) as a bean.    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);  
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);  
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);  
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);  
  
    // Register early post-processor for detecting inner beans as ApplicationListeners.  
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));  
  
    // Detect a LoadTimeWeaver and prepare for weaving, if found.  
    if (!NativeDetector.inNativeImage() && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {  
       beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));  
       // Set a temporary ClassLoader for type matching.  
       beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));  
    }  
  
    // Register default environment beans.  
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {  
       beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());  
    }  
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {  
       beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());  
    }  
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {  
       beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());  
    }  
    if (!beanFactory.containsLocalBean(APPLICATION_STARTUP_BEAN_NAME)) {  
       beanFactory.registerSingleton(APPLICATION_STARTUP_BEAN_NAME, getApplicationStartup());  
    }  
}
```