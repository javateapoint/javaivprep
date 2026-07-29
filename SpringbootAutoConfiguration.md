# Spring Boot Auto-Configuration Deep Dive & Startup Sequence

A detailed guide explaining how Spring Boot auto-configuration works under the hood, including Spring Boot 2.x vs. Spring Boot 3.x import mechanisms, conditional evaluation annotations, custom auto-configuration creation, and application startup phases.

---

## 1. Startup & Auto-Configuration Sequence

```text
1. JVM Launch & Main Entry
   └─ SpringApplication.run(Application.class, args)

2. Bootstrap & Environment Preparation
   ├─ Prepare ConfigurableEnvironment (application.properties / YAML, system env)
   └─ Fire ApplicationStartingEvent & ApplicationEnvironmentPreparedEvent

3. Bean Definition Scanning
   ├─ Component Scanning (@Component, @Service, @Repository, @Controller)
   └─ Parse user-defined @Configuration classes

4. Auto-Configuration Discovery & Filtering
   ├─ Read import configurations:
   │    • Spring Boot 3.x+: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
   │    • Spring Boot 2.x (Legacy): META-INF/spring.factories
   ├─ Select candidates via AutoConfigurationImportSelector
   ├─ Evaluate @ConditionalOn* filters:
   │    • @ConditionalOnClass: Active if targeted class exists on classpath (e.g. HikariDataSource)
   │    • @ConditionalOnMissingBean: Active only if no user-defined bean of that type exists
   │    • @ConditionalOnProperty: Active based on application property flags
   │    • @ConditionalOnBean: Active when a prerequisite bean exists in context
   │    • @ConditionalOnResource: Active if specified resource file exists
   │    • @ConditionalOnWebApplication: Active only for Servlet/Reactive web contexts
   └─ Register passing @AutoConfiguration beans into BeanDefinitionRegistry

5. Bean Instantiation & Lifecycle Hooks
   ├─ Instantiate eager singletons
   ├─ Inject dependencies (@Autowired, Constructor, @Value)
   ├─ Execute BeanPostProcessors (AOP proxying, @Transactional, Micrometer metrics)
   └─ Run @PostConstruct & init methods

6. Application Ready Event
   ├─ Fire ApplicationReadyEvent
   └─ Execute CommandLineRunner & ApplicationRunner beans
```

---

## 2. Spring Boot 2.x vs. Spring Boot 3.x Auto-Configuration Discovery

| Aspect | Spring Boot 2.x (Legacy) | Spring Boot 3.x (Modern) |
| :--- | :--- | :--- |
| **Declaration File** | `META-INF/spring.factories` | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |
| **File Format** | Key-value properties format (`EnableAutoConfiguration=com.example.Config`) | Plain list of fully-qualified class names (one per line). |
| **Annotation** | `@Configuration` | `@AutoConfiguration` (supports `@AutoConfigureBefore` / `@AutoConfigureAfter`). |

---

## 3. Creating a Custom Starter Auto-Configuration (Spring Boot 3)

### Step 1: Create the Auto-Configuration Class

```java
package com.example.custom.starter;

import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;

@AutoConfiguration
@ConditionalOnClass(CustomApiClient.class)
@EnableConfigurationProperties(CustomApiProperties.class)
@ConditionalOnProperty(prefix = "custom.api", name = "enabled", havingValue = "true", matchIfMissing = true)
public class CustomApiAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public CustomApiClient customApiClient(CustomApiProperties properties) {
        return new CustomApiClient(properties.getBaseUrl(), properties.getTimeoutMs());
    }
}
```

### Step 2: Declare in `AutoConfiguration.imports`

File: `src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

```text
com.example.custom.starter.CustomApiAutoConfiguration
```

---

## 4. Key Interview Summary Checklist

- **`@SpringBootApplication` Composition**:
  - `@SpringBootConfiguration` (specialized `@Configuration`)
  - `@EnableAutoConfiguration` (enables auto-configuration discovery)
  - `@ComponentScan` (scans package and sub-packages for Spring components)
- **Overriding Auto-Configuration Beans**:
  - Auto-configurations specify `@ConditionalOnMissingBean`. If a developer defines a `@Bean` in their code, Spring Boot silently skips creating the default auto-configured bean.
- **Disabling Specific Auto-Configurations**:
  - Via annotation: `@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})`
  - Via properties: `spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration`
