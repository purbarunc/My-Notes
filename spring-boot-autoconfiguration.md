# Spring Boot Autoconfiguration - Complete Guide

---

## Table of Contents

1. Overview
2. The Big Picture - Entry Points
   - @SpringBootApplication
   - @EnableAutoConfiguration
3. Step-by-Step Internal Flow
   - Step 1: AutoConfigurationImportSelector runs
   - Step 2: Classpath Scanning via spring.factories / .imports
   - Step 3: Conditional Filtering
   - Step 4: Ordering annotations
   - Step 5: Beans are registered
4. Concrete Example: DataSourceAutoConfiguration
5. The Condition Evaluation Engine
6. Writing Your Own Autoconfiguration
7. Debugging Autoconfiguration
8. Summary Flow Diagram
9. Key Takeaways (Spring Boot 3.x)
10. Spring Boot 4.x / Spring Framework 7 - What Changed and What Did Not
    - What Is Exactly The Same
    - What Changed
    - The Updated Mental Model
    - Removed in Spring Boot 4
    - Bottom Line

---

## Overview

Spring Boot Autoconfiguration is the mechanism that automatically configures your Spring
application based on the JARs present on the classpath, the beans you have defined, and various
property settings. It eliminates the need for most boilerplate XML or Java configuration.

---

## The Big Picture - Entry Points

### 1. @SpringBootApplication

Every Spring Boot app starts here:

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

@SpringBootApplication is a composed annotation of three annotations:

```java
@SpringBootConfiguration        // = @Configuration
@EnableAutoConfiguration        // <-- THE KEY
@ComponentScan                  // scans your packages
```

---

### 2. @EnableAutoConfiguration

This is the trigger. It imports AutoConfigurationImportSelector:

```java
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration { ... }
```

---

## Step-by-Step Internal Flow

### Step 1: AutoConfigurationImportSelector runs

When the Spring context starts, AutoConfigurationImportSelector.selectImports() is called.
Its job is to discover all autoconfiguration candidates.

```
AutoConfigurationImportSelector
    +-- getCandidateConfigurations()
            +-- SpringFactoriesLoader.loadFactoryNames()
                    +-- reads: META-INF/spring/
                               org.springframework.boot.autoconfigure.AutoConfiguration.imports
                               (or META-INF/spring.factories in older versions)
```

### Step 2: Classpath Scanning via spring.factories / .imports

Spring Boot ships with a file inside spring-boot-autoconfigure.jar.
File: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports

```
org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration
... (150+ entries)
```

All these class names are loaded as candidates.

### Step 3: Conditional Filtering

Each autoconfiguration class is annotated with @Conditional* annotations. Spring evaluates each
condition and only applies the configuration if conditions pass.

Key conditional annotations:

| Annotation                   | Meaning                              |
|------------------------------|--------------------------------------|
| @ConditionalOnClass          | Only if class is on classpath        |
| @ConditionalOnMissingClass   | Only if class is NOT on classpath    |
| @ConditionalOnBean           | Only if a bean of that type exists   |
| @ConditionalOnMissingBean    | Only if NO bean of that type exists  |
| @ConditionalOnProperty       | Only if property is set to value     |
| @ConditionalOnResource       | Only if resource file exists         |
| @ConditionalOnWebApplication | Only if it is a web application      |

### Step 4: @AutoConfigureOrder / @AutoConfigureBefore / @AutoConfigureAfter

Among the candidates that pass conditions, Spring respects ordering annotations to ensure
configurations load in the right sequence (e.g., DataSource before JPA).

### Step 5: Beans are registered

The surviving autoconfiguration classes act as @Configuration classes. Their @Bean methods
execute and register beans into the Spring Application Context.

---

## Concrete Example: DataSourceAutoConfiguration

Let's trace what happens when you add spring-boot-starter-data-jpa to your project.

### pom.xml:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

### application.properties:

```properties
# No datasource config needed -- autoconfiguration handles it
```

### What Spring Boot does internally:

DataSourceAutoConfiguration.java (simplified from actual source):

```java
@AutoConfiguration(before = SqlInitializationAutoConfiguration.class)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })  // H2 jar present? YES
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class,
          DataSourceCheckpointRestoreConfiguration.class })
public class DataSourceAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @Conditional(EmbeddedDatabaseCondition.class)                       // H2 is embedded? YES
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class }) // no custom DS? YES
    @Import(EmbeddedDataSourceConfiguration.class)
    protected static class EmbeddedDatabaseConfiguration { }

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @ConditionalOnProperty(name = "spring.datasource.type")
    static class PooledDataSourceAvailableConfiguration {
        // Hikari, DBCP2, etc.
    }
}
```

DataSourceProperties.java -- binds your properties:

```java
@ConfigurationProperties(prefix = "spring.datasource")
public class DataSourceProperties {
    private String url;
    private String username;
    private String password;
    private String driverClassName;
    // getters/setters...
}
```

EmbeddedDataSourceConfiguration.java -- creates the actual bean:

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(DataSourceProperties.class)
public class EmbeddedDataSourceConfiguration implements BeanClassLoaderAware {

    @Bean(destroyMethod = "shutdown")
    public EmbeddedDatabase dataSource(DataSourceProperties properties) {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)   // detected from classpath
            .build();
    }
}
```

A fully working DataSource bean is created without you writing a single line of configuration.

---

## The Condition Evaluation Engine

@ConditionalOnMissingBean is especially important -- it allows you to override autoconfiguration:

```java
// Your custom config -- PREVENTS DataSourceAutoConfiguration from creating a DataSource
@Configuration
public class MyDataSourceConfig {

    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        ds.setUsername("user");
        ds.setPassword("pass");
        return ds;
    }
}
```

Now DataSourceAutoConfiguration sees @ConditionalOnMissingBean(DataSource.class) -- a DataSource
bean exists -- so it skips autoconfiguration. Your bean wins.

---

## Writing Your Own Autoconfiguration

You can create custom autoconfiguration for your own libraries.

### Step 1: Create the autoconfiguration class

```java
@AutoConfiguration
@ConditionalOnClass(MyService.class)           // only if MyService.class is on classpath
@ConditionalOnMissingBean(MyService.class)     // only if user has not defined their own
@EnableConfigurationProperties(MyServiceProperties.class)
public class MyServiceAutoConfiguration {

    @Bean
    public MyService myService(MyServiceProperties props) {
        return new MyService(props.getEndpoint(), props.getTimeout());
    }
}
```

### Step 2: Bind properties

```java
@ConfigurationProperties(prefix = "myservice")
public class MyServiceProperties {
    private String endpoint = "http://localhost:8080";  // default
    private int timeout = 30;
    // getters/setters...
}
```

### Step 3: Register it

In src/main/resources/META-INF/spring/
org.springframework.boot.autoconfigure.AutoConfiguration.imports:

```
com.example.mylib.MyServiceAutoConfiguration
```

Any project that adds your JAR gets MyService automatically configured. Tune it via
application.properties:

```properties
myservice.endpoint=https://api.production.com
myservice.timeout=60
```

---

## Debugging Autoconfiguration

Spring Boot provides built-in tooling to see what was applied and why.

### 1. --debug flag or property

In application.properties:

```properties
debug=true
```

This prints the Conditions Evaluation Report to the console:

```
============================
CONDITIONS EVALUATION REPORT
============================

Positive matches (applied):
--------------------------
   DataSourceAutoConfiguration matched:
      - @ConditionalOnClass found required classes 'javax.sql.DataSource',
        'org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType'
        (OnClassCondition)

Negative matches (not applied):
-------------------------------
   RedisAutoConfiguration:
      - @ConditionalOnClass did not find required class
        'org.springframework.data.redis.core.RedisOperations' (OnClassCondition)
```

### 2. Actuator /actuator/conditions endpoint

Add the actuator dependency:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

Then hit `GET /actuator/conditions` for a full JSON report of all autoconfiguration decisions.

### 3. Exclude specific autoconfiguration

```java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
```

---

## Summary Flow Diagram

```
@SpringBootApplication
        |
        v
@EnableAutoConfiguration
        |
        v
AutoConfigurationImportSelector.selectImports()
        |
        v
SpringFactoriesLoader reads:
META-INF/spring/...AutoConfiguration.imports
        |
        v
150+ candidate classes loaded
        |
        v
Conditions evaluated (@ConditionalOnClass, @ConditionalOnMissingBean, ...)
        |
        v
Surviving classes ordered (@AutoConfigureAfter / @AutoConfigureBefore)
        |
        v
@Bean methods execute -> beans registered in ApplicationContext
        |
        v
Your app runs with zero boilerplate config
```

---

## Key Takeaways (Spring Boot 3.x)

1. **Discovery**    : Autoconfiguration classes listed in META-INF/spring/...AutoConfiguration.imports
2. **Filtering**    : @Conditional* annotations gate each class; only classpath-relevant configs apply
3. **Override**     : @ConditionalOnMissingBean means your beans always win over autoconfigured ones
4. **Transparency** : Use debug=true or the Actuator to see what was applied and why
5. **Extensible**   : You can write your own autoconfiguration for any library using the same mechanism

---

## Spring Boot 4.x / Spring Framework 7 - What Changed and What Did Not

Spring Boot 4 (4.0.x, 4.1.x, 4.2.x as of late 2026) is built on Spring Framework 7.0.
Both are released and stable. Spring Framework 7.1 is scheduled for November 2026.

### The Short Answer

The core autoconfiguration mechanism is fundamentally the same. The discovery-filter-apply pipeline
described above still works identically. However, Spring Boot 4 introduces meaningful refinements
and additions -- especially around AOT, modular design, and API enforcement.

---

### What Is Exactly The Same

| Aspect                                   | Spring Boot 3.x                        | Spring Boot 4.x |
|------------------------------------------|----------------------------------------|-----------------|
| Discovery file                           | META-INF/spring/...AutoConfiguration.imports | Same      |
| @EnableAutoConfiguration                 | Yes                                    | Yes             |
| AutoConfigurationImportSelector          | Yes                                    | Yes             |
| @ConditionalOn* annotations              | Yes                                    | Yes             |
| @AutoConfigureBefore/After               | Yes                                    | Yes             |
| @ConditionalOnMissingBean override       | Yes                                    | Yes             |
| spring-autoconfigure-metadata.properties | Yes (eager pre-filter)                 | Yes             |

---

### What Changed in Spring Boot 4 / Spring Framework 7

#### 1. Autoconfiguration Classes Are No Longer Public API (Enforced)

In Spring Boot 3, autoconfiguration classes were public but documented as not public API.
In Spring Boot 4, this is **enforced at the Java language level**:

> "Public members (aside from constants) have been removed from auto-configuration classes."

This means you can no longer inject or subclass autoconfiguration beans directly in your code --
which was always unsupported but previously possible.

#### 2. A New Modular Architecture

Spring Boot 4.0 ships **smaller, focused modules** instead of a few large monolithic JARs.
Autoconfiguration is split across more targeted modules. For example:

- MongoDB health auto-configuration moved from spring-boot-data-mongodb to spring-boot-mongodb
- Each feature area has its own module with its own autoconfiguration

This does not change how autoconfiguration works, but it changes which JAR a given
@AutoConfiguration class lives in.

#### 3. AutoConfiguration.replacements -- A New Companion File

To handle deprecating and renaming autoconfiguration classes (more common due to the modular
redesign), Spring Boot 4 introduced:

```
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.replacements
```

This tells Spring Boot how to handle before/after ordering and exclude references when a class has
been renamed or moved, without breaking existing users.

Note: The AutoConfiguration.imports file should also be updated to only reference the replacement
class.

#### 4. AOT (Ahead-of-Time) Processing -- The Biggest Shift

This is the most significant evolution. Spring Framework 6 introduced AOT; Spring Boot 4 /
Framework 7 matures and enforces it for native image builds.

**How it changes the autoconfiguration model:**

In a **JVM (classic) run**, everything works as before -- conditions are evaluated at runtime.

In a **Native Image (GraalVM) run**, Spring performs AOT processing at build time:

```
mvn spring-boot:process-aot    <-- new mandatory build step for native images
```

During AOT processing:
- The ApplicationContext is started partially: bean definitions are created, but NOT instantiated
- All @Conditional* conditions are **evaluated at BUILD TIME**
- The result is code-generated ApplicationContextInitializer source files that replace runtime
  reflection and condition evaluation

This means certain conditions have **limitations in native image mode**:

```java
// This works fine on JVM but has limitations in native image:
@ConditionalOnProperty(name = "feature.enabled", matchIfMissing = false)
```

Official note: "Properties that change if a bean is created are not supported
(e.g., @ConditionalOnProperty and .enabled properties)" -- Spring Boot 4 Native Image docs

The AOT engine generates code like this at build time to replace runtime condition evaluation:

```java
// AOT-generated (conceptual representation)
public class MyApp__ApplicationContextInitializer
    implements ApplicationContextInitializer<GenericApplicationContext> {

    @Override
    public void initialize(GenericApplicationContext context) {
        // All bean definitions pre-resolved - no reflection, no condition eval at runtime
        context.registerBean("dataSource", DataSource.class,
            () -> new EmbeddedDataSourceConfiguration().dataSource(props));
    }
}
```

#### 5. Spring Framework 7 Baseline Requirements

Spring Boot 4 requires Spring Framework 7, which sets:

| Requirement      | Spring Framework 6 (Boot 3) | Spring Framework 7 (Boot 4) |
|------------------|-----------------------------|-----------------------------|
| Minimum JDK      | JDK 17                      | JDK 17 (same)               |
| Recommended JDK  | JDK 21                      | JDK 25 (LTS)                |
| Jakarta EE       | Jakarta EE 10               | Jakarta EE 11               |
| Kotlin           | 1.9+                        | 2.2+                        |
| GraalVM          | 22+                         | 25                          |

#### 6. New Built-in Autoconfiguration Coverage

Spring Boot 4 added new autoconfiguration for things that previously required manual setup:

- **HTTP Service Clients** -- auto-configure @HttpExchange-annotated interfaces:

  ```java
  @HttpExchange(url = "https://echo.zuplo.io")
  public interface EchoService {
      @PostExchange
      Map<?, ?> echo(@RequestBody Map<String, String> message);
  }
  ```
  Configured via spring.http.client.* properties.

- **API Versioning** -- spring.mvc.apiversion.* / spring.webflux.apiversion.* with beans of
  type ApiVersionResolver, ApiVersionParser, and ApiVersionDeprecationHandler

- **OpenTelemetry SDK** -- new spring-boot-starter-opentelemetry starter, auto-configures OTLP
  export for metrics and traces

- **Redis Static Master/Replica** -- spring.data.redis.masterreplica.nodes (Lettuce only)

- **JmsClient** -- alongside the existing JmsTemplate and JmsMessagingTemplate

- **Multiple TaskDecorator beans** -- task scheduling/execution now supports multiple decorators
  via CompositeTaskDecorator, ordered by @Order / Ordered

- **Kotlin Serialization** -- new spring-boot-kotlinx-serialization-json module and
  spring-boot-starter-kotlin-serialization

- **RestTestClient** -- support in @SpringBootTest and @AutoConfigureMockMvc

#### 7. Jackson 3 by Default

Spring Boot 4 uses **Jackson 3** as its default JSON library, with new group IDs (tools.jackson
instead of com.fasterxml.jackson). The JacksonAutoConfiguration was updated accordingly.
The jackson-annotations module continues to use the com.fasterxml.jackson.core group ID.

#### 8. Nullability Annotations (JSpecify)

Spring Framework 7 migrated from JSR 305 nullability annotations to **JSpecify annotations**
throughout the codebase, including autoconfiguration classes. This can cause compilation failures
in Kotlin projects or projects using null checkers due to now-explicit nullable/non-nullable types.

---

### Summary: The Updated Mental Model

```
Classic JVM Run (unchanged from Boot 3):
-----------------------------------------
@SpringBootApplication
  -> AutoConfigurationImportSelector
  -> reads AutoConfiguration.imports
  -> evaluates @Conditional* at RUNTIME
  -> registers beans

Native Image Run (new in Boot 4 / Framework 7):
-------------------------------------------------
mvn spring-boot:process-aot  <-- build step
  -> starts partial ApplicationContext
  -> evaluates @Conditional* at BUILD TIME
  -> generates ApplicationContextInitializer source
  -> GraalVM compiles everything to native binary

At native runtime:
  -> no reflection, no condition evaluation
  -> pre-generated initializer runs directly
  -> much faster startup, lower memory
```

---

### Removed in Spring Boot 4

| Removed Feature                         | Reason                                           |
|-----------------------------------------|--------------------------------------------------|
| Undertow embedded server                | Not yet compatible with Servlet 6.1              |
| Spock test integration                  | Groovy 5 incompatibility                         |
| Spring Pulsar Reactive autoconfiguration| Reactor support removed from Spring Pulsar       |
| Embedded launch scripts                 | Conflicts with efficient deployment practices    |
| Spring Retry dependency management      | Replaced by new core features in Spring Framework|
| All deprecated Spring Boot 3.x APIs    | Standard major-version cleanup                   |

---

### Bottom Line

If you are running a standard JVM application, Spring Boot 4 autoconfiguration feels **identical**
to Boot 3. The real divergence comes when you target **GraalVM native images**, where the AOT
engine fundamentally changes when and how conditions are evaluated -- shifting the work from startup
time to build time. This is the biggest conceptual change in the autoconfiguration model across
these versions.

For **migration from Spring Boot 3.5 to 4.0**, the Spring team provides a dedicated migration guide
and a spring-boot-properties-migrator module that analyzes and flags renamed/removed configuration
properties at runtime.
