title: 'SpringBoot 原理分析'
author: chenxi
tags: 
  - spring-boot
categories: 
  - spring-boot
date: 2018-11-30 15:34:00
---

## SpringBoot 简介

- 约定大于配置, 开箱即用, 最小化配置（包括 maven/gradle 和代码里配置）

- 快速搭建服务，如微服务实战


## SpringBoot 特性

- 内嵌servlet容器: jar包启动
  - 命令行参数

- 简化构建工具的使用，如 maven
    - 依赖管理 `parent pom "dependencyManagement"`

- AutoConfiguration，提供各种 starter

- `@ConfigurationProperties`

- logging: 默认输出到控制台，简化日志配置，可以直接在 `application.properties` 文件里配置日志
	- log pattern, color output (if terminal support ANSI)

- Actuator (devOps)
    - many endpoints: `info, health, metrics, configprops, beans, logs ...`
	- custom health indicator
    - 结合 maven git-commit-id plugin

- spring-boot-devtools 热部署工具


## 实现原理

### 1. 简化构建工具的使用 (以 maven 为例)

+ **spring-boot-dependencies:**
    - 声明所有 dependency 和 plugin 的版本: 包括各种第三方类库和 starter 的版本

![](2020-07-28-19-57-17.png)

+ **spring-boot-starter-parent:**
    - common properties;
    - resource filtering: [maven-resources-plugin](https://maven.apache.org/plugins/maven-resources-plugin/examples/filter.html)
    - plugin configuration

![](2020-07-28-19-53-27.png)

---

### 2. 自动配置原理分析

#### 2.1 现象
- 添加`@EnableAutoConfiguration`注解;

- 在pom.xml中引入相应的starter;

- (可选) 在配置文件里添加对应的配置项;

- (可选) 自定义Bean，替换预定义对默认Bean;

---

#### 2.2 AutoConfiguration

"xxAutoConfiguration"本质就是一个`@Configuration`类, 特殊之处在于：
1. 使用一系列条件化注解进行**推断**：是否加载当前Configuration；是否加载某个Bean.
    对比"手动配置"：预先定义好许多configuration和bean，通过条件触发.
    e.g.: `WebMvcAutoConfiguration`, `DataSourceAutoConfiguration`, `MybatisAutoConfiguration`

2. 加载时机：最后加载，**优先级**用户定义的Bean;

3. `@AutoConfigureOrder`注解

如果这些注解都去掉, 就和我们在Spring应用中自定义的Configuration类没有区别了。


**三个问题**:

- 条件化触发是如何实现的？

- 为什么只是在 pom 里声明一个 starter, SpringBoot 就会进行对应的自动配置?
AutoConfiguration具体是如何被框架调用的?

- 为什么自定义的bean可以覆盖默认的bean?

#### 2.3 条件化注解

> 条件化加载是在Spring4中引入的新特性

- @ConditionalOnWebApplication

- @ConditionalOnMissingBean

- @ConditionalOnClass

- @ConditionalOnProperty
    - havingValue
    - matchIfMissing

##### 自定义条件化注解:
```java
@Conditional(xxCondition.class)
public @interface ConditionalOnXXX {
}

// xxCondition extends SpringBootCondition extends Condition
```

##### SpringBootCondition

> Provides sensible logging to help the user diagnose what classes are loaded.

1. 提供了基本的实现(template method design pattern), 子类只需要实现'getMatchOutcome'方法
    ` public abstract ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata);`

2. 提供两个子类可能用到的方法 -- anyMatches, matchers
    e.g.: EmbeddedDatabaseCondition

#### 2.4 自动配置日志报告

##### 生成内容
SpringBootCondition
```java
private void recordEvaluation(ConditionContext context, String classOrMethodName, ConditionOutcome outcome) {
    if (context.getBeanFactory() != null) {
        ConditionEvaluationReport.get(context.getBeanFactory())
                .recordConditionEvaluation(classOrMethodName, this, outcome);
    }
}
```

core classes of 'conditional evaluation reporting': `ConditionMessage`, `ConditionOutcome`, `ConditionEvaluationReport`.

##### 输出内容
ConditionEvaluationReportLoggingListener.ConditionEvaluationReportListener
监听`ContextRefreshedEvent`和`ApplicationFailedEvent`这两个事件, 输出自动报告的配置：
```java
public void logAutoConfigurationReport(boolean isCrashReport) {
    if (this.report == null) {
        if (this.applicationContext == null) {
            this.logger.info("Unable to provide the conditions report "
                    + "due to missing ApplicationContext");
            return;
        }
        this.report = ConditionEvaluationReport
                .get(this.applicationContext.getBeanFactory());
    }
    if (!this.report.getConditionAndOutcomesBySource().isEmpty()) {
        if (isCrashReport && this.logger.isInfoEnabled()
                && !this.logger.isDebugEnabled()) {
            this.logger.info(String
                    .format("%n%nError starting ApplicationContext. To display the "
                            + "conditions report re-run your application with "
                            + "'debug' enabled."));
        }
        if (this.logger.isDebugEnabled()) {
            this.logger.debug(new ConditionEvaluationReportMessage(this.report));
        }
    }
}
```
`ConditionEvaluationReportMessage.toString()`输出的内容就是我们在控制台看到的自动配置报告了。


---

#### 2.5 AutoConfiguration加载原理

##### SpringFactoriesLoader (since 3.2)

java doc:
> General purpose factory loading mechanism for internal use within the framework.
SpringFactoriesLoader loads and instantiates factories of a given type from "META-INF/spring.factories" files
which may be present in multiple JAR files in the classpath.
The spring.factories file must be in Properties format, where the key is the fully qualified name of the interface or abstract class,
and the value is a comma-separated list of implementation class names.

```java
public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader)
```

之所以把这个放在前面介绍是因为SpringBoot里大量地使用里这个机制，加载AutoConfiguration只是其中的一个运用, 后面会提到其他的使用场景。

通过`SpringFactoriesLoader`寻找类路径下的所有的'META-INF/spring.factories'文件，
找到所有key为`org.springframework.boot.autoconfigure.EnableAutoConfiguration`所对应的value，
得到一个列表：所有AutoConfiguration的全限定类名。

SpringBoot预定义的'autoconfiguration': spring-boot-autoconfigure-xxx.jar: META-INF/spring.factories

对于自带的AutoConfiguration和三方的AutoConfiguration, SpringBoot采取完全相同的加载方式.

> SpringFactoriesLoader的实现类似于SPI（Service Provider Interface)
java SPI提供一种服务发现机制，为某个接口寻找服务实现的机制。
有点类似IOC的思想，将装配的控制权移到程序之外.


##### Java SPI (JDBC 4.0)
`java.util.ServiceLoader` [SPI](https://blog.csdn.net/u013679744/article/details/80009878)
before JDBC 4.0:
```java
// Register JDBC driver
Class.forName("com.mysql.jdbc.Driver");
// Open a connection
Connection conn = DriverManager.getConnection(DB_URL, USER, PASS);
```

JDBC 4.0:
```java
// Open a connection
Connection conn = DriverManager.getConnection(DB_URL, USER, PASS);
```


##### DeferredImportSelector

`@EnableAutoConfiguration`注解:
```java
// @Import(EnableAutoConfigurationImportSelector.class) // if the version is 1.x
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {}
```

`AutoConfigurationImportSelector --> DeferredImportSelector (since 4.0) --> ImportSelector (since 3.1)`

ImportSelector:
> Interface to be implemented by types that determine which @Configuration class(es) should be imported
based on a given selection criteria, usually one or more annotation attributes.

可以理解为是动态地加载Configuration类


##### AutoConfigurationImportSelector.selectImports:

1. Find all possible auto configuration classes, filtering duplicates
    ```
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    ```

2. Remove those specifically disabled
    1. @EnableAutoConfiguration exclude
    2. spring.autoconfigure.exclude

3. Check excluded classes

4. *Filter auto-config* by `AutoConfigurationImportFilter` (spring.factories)
    *拓展点*: 过滤掉先前的"AutoConfiguration"

5. fireAutoConfigurationImportEvents, `AutoConfigurationImportListener` (spring.factories)
    ```java
    ConditionEvaluationReportAutoConfigurationImportListener

    report.recordEvaluationCandidates(event.getCandidateConfigurations());
    report.recordExclusions(event.getExclusions());
    ```

##### OnClassCondition (implements AutoConfigurationImportFilter)

`AutoConfigurationImportFilter`:
> Filter that can be registered in {@code spring.factories} to limit the  auto-configuration classes considered.
This interface is designed to allow fast removal  of auto-configuration classes before their bytecode is even read.

spring-boot-autoconfigure-xx.jar  META-INF/spring-autoconfigure-metadata.properties

`AutoConfigurationMetadata`
```java
AutoConfigurationImportSelector.selectImports:
    AutoConfigurationMetadata autoConfigurationMetadata = AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);

OnClassCondition.StandardOutcomesResolver.getOutcomes:
    Set<String> candidates = autoConfigurationMetadata.getSet(autoConfigurationClass, "ConditionalOnClass");
```


##### ImportSelector是如何被Spring框架调用的

PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors:
- ConfigurationClassParser.parse
    - parse
        - processConfigurationClass
            - doProcessConfigurationClass
                - processImports
    - processDeferredImportSelectors

#### 2.6 @ImportAutoConfiguration (useful for testing)
> Variant of AutoConfigurationImportSelector for ImportAutoConfiguration.

override method - `getCandidateConfigurations`:
```java
private void collectAnnotations(Class<?> source,
        MultiValueMap<Class<?>, Annotation> annotations, HashSet<Class<?>> seen) {
    if (source != null && seen.add(source)) {
        for (Annotation annotation : source.getDeclaredAnnotations()) {
            if (!AnnotationUtils.isInJavaLangAnnotationPackage(annotation)) {
                if (ANNOTATION_NAMES
                        .contains(annotation.annotationType().getName())) {
                    annotations.add(source, annotation);
                }
                collectAnnotations(annotation.annotationType(), annotations, seen);
            }
        }
        collectAnnotations(source.getSuperclass(), annotations, seen);
    }
}

private Collection<String> getConfigurationsForAnnotation(Class<?> source,
        Annotation annotation) {
    String[] classes = (String[]) AnnotationUtils
            .getAnnotationAttributes(annotation, true).get("classes");
    if (classes.length > 0) {
        return Arrays.asList(classes);
    }
    return loadFactoryNames(source);
}
```
e.g.: `DataJpaTest`

---

## 参考资料
[简书-Spring Boot自动配置原理](https://www.jianshu.com/p/346cac67bfcc)

[Spring Cloud AutoConfiguration 简介](http://www.scienjus.com/spring-cloud-autoconfiguration/)

[详解Spring的ImportSelector接口](https://www.jianshu.com/p/23d4e853b15b)



