# 1 Spring
## 1.1 IoC
### 1.1.1 基于 xml 的 IoC
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
- @Component
	- 用于描述 `Spring` 中的 `Bean`，可以用在任何层次，如 `Service` 层，`Dao` 层
- @Controller
	- 用于控制层，功能与 `@Component` 相同
- @Service
	- 用于业务层，功能与 `@Component` 相同
- @Repository
	- 用于数据访问层，功能与 `@Component` 相同
#### 1.1.2.2 配置类注解
- @Configuration
	- 声明当前类为配置类，相当于 `XML` 形式的 `Spring` 配置
	- 加载注解格式的上下文对象需要使用 `AnnotationConfigApplicationContext`
- @ComponentScan: 类似 `XML` 中的 `ComponentScan`
- @Bean
	- 用于方法，声明当前方法的返回值为一个 `Bean`
#### 1.1.2.3 配置 Bean
- @Scope
	- 设置 Spring 容器如何新建 `Bean` 实例
		- Singe