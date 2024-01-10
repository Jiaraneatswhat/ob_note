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
- 不写 name 时会根据变量名进行匹配
### 1.1.3 反射
- 获取 `Class` 对象的方式
	- `类名.class`
	- `对象.getClass()`
	- `Class.forName("FQCN")`
	- 