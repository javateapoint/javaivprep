
## Spring Boot Application Startup & Auto‑Configuration (Simplified)

```text
1. JVM Launch
   └─ SpringApplication.run(...)

2. Bootstrap Phase
   ├─ Build Environment & Property Sources
   └─ Fire ApplicationStartingEvent

3. Bean Definition Loading
   ├─ Scan for @Component, @Service, @Repository, @Controller
   └─ Parse @Configuration classes

4. Auto‑Configuration
   ├─ Read META‑INF/spring.factories
   ├─ Select candidates via AutoConfigurationImportSelector
   ├─ Apply @ConditionalOn* filters:
   │    • @ConditionalOnClass: active if a class is on the classpath (e.g., HikariCP present ⇒ DataSource auto‑config)
   │    • @ConditionalOnMissingBean: active when no existing bean of that type is defined (override defaults)
   │    • @ConditionalOnProperty: active based on config property (e.g., spring.security.enabled=true ⇒ security auto‑config)
   │    • @ConditionalOnBean: active when another bean exists (e.g., MeterRegistry ⇒ metrics auto‑config)
   │    • @ConditionalOnResource: active if a resource is available (e.g., web.xml ⇒ servlet support)
   │    • @ConditionalOnExpression: active based on SpEL expression (e.g., custom condition)
   └─ Register matching @Configuration beans

5. BeanFactory Post‑Processing
   └─ Run all BeanFactoryPostProcessor (property placeholders, etc.)

6. Bean Instantiation & Injection
   ├─ Instantiate beans (eager singletons)
   ├─ Inject dependencies (@Autowired, constructor, @Value)
   ├─ Call Aware callbacks (BeanNameAware, ApplicationContextAware…)
   └─ Execute BeanPostProcessor before-init

7. Initialization & Proxies
   ├─ Run @PostConstruct & afterPropertiesSet()
   ├─ Call custom init-method
   └─ Execute BeanPostProcessor after-init (AOP proxies, metrics)

8. Application Ready
   ├─ Fire ApplicationReadyEvent
   └─ Run CommandLineRunner / ApplicationRunner

9. Shutdown (on close or JVM exit)
   ├─ Fire ContextClosedEvent
   ├─ Call @PreDestroy & DisposableBean.destroy()
   └─ Stop SmartLifecycle beans
````

**Tips for Interviews:**

* **Key Annotations:** `@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`.
* **Conditional Annotations:** Know the main ones and when to use:

  * `@ConditionalOnClass` (classpath checks)
  * `@ConditionalOnMissingBean` (override custom beans)
  * `@ConditionalOnProperty` (feature toggles)
  * `@ConditionalOnBean` (dependent features)
  * `@ConditionalOnResource` (resource-driven config)
  * `@ConditionalOnExpression` (complex conditions)
* **Lifecycle Hooks:** BeanFactoryPostProcessor → BeanPostProcessor → init/destroy callbacks → lifecycle interfaces.
* **Override & Disable:** Use `spring.autoconfigure.exclude` or `@EnableAutoConfiguration(exclude = …)`.

```

