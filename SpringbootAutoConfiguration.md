## Spring Boot 3 Application Startup & Auto‑Configuration Flow

Below is a concise, step‑by‑step **visual summary** of what happens under the hood when you start a Spring Boot 3 application. 
It covers both the classic lifecycle and new Spring Boot 3 AOT/bootstrap phases.

```text
┌────────────────────────────────────────────────────────────────────────┐
│                         JVM Process Launch                          │
└────────────────────────────────────────────────────────────────────────┘
             ↓
┌────────────────────────────────────────────────────────────────────────┐
│                          Bootstrap Phase                            │
│  • `SpringApplication.run(...)`                                       │
│  • Create `BootstrapContext` & `Environment`                          │
│  • Invoke `EnvironmentPostProcessor`s (e.g. property sources)         │
│  • Notify `ApplicationListeners` (application starting event)         │
└────────────────────────────────────────────────────────────────────────┘
             ↓
┌────────────────────────────────────────────────────────────────────────┐
│                         Context Preparation                          │
│  • Create `SpringApplication` instance                                │
│  • Apply `ApplicationContextInitializers`                             │
│  • Build `ApplicationContext` (e.g. `AnnotationConfigApplicationContext`)  
│  • Register `ApplicationEnvironmentPreparedEvent`                    │
└────────────────────────────────────────────────────────────────────────┘
             ↓
┌────────────────────────────────────────────────────────────────────────┐
│                  Bean Definition Loading & Scanning                  │
│  • `@ComponentScan` → scan classpath for `@Component`‑stereotypes    │
│  • Register plain bean definitions                                   │
│  • Load user `@Configuration` classes                                 │
└────────────────────────────────────────────────────────────────────────┘
             ↓
┌────────────────────────────────────────────────────────────────────────┐
│                    Auto‑Configuration Import                       │
│  • `@EnableAutoConfiguration` (via `@SpringBootApplication`)         │
│    └─▶ `SpringFactoriesLoader` reads `META‑INF/spring.factories`    │
│    └─▶ `AutoConfigurationImportSelector`                              │
│         • Filters candidates with `@ConditionalOn*` annotations      │
│         • Honors `spring.autoconfigure.exclude` & `@EnableAutoConfig(exclude=…)`  
│  • Register filtered auto‑config classes as bean definitions         │
└────────────────────────────────────────────────────────────────────────┘
             ↓
┌────────────────────────────────────────────────────────────────────────┐
│                       BeanFactory Post‑Processing                    │
│  • Invoke all `BeanFactoryPostProcessor`s                             │
│    (e.g. `PropertySourcesPlaceholderConfigurer`, AOT optimizers)      │
└────────────────────────────────────────────────────────────────────────┘
             ↓
┌────────────────────────────────────────────────────────────────────────┐
│                        ApplicationContext Refresh                    │
│  • **Instantiation**: create bean instances (singletons eager)       │
│  • **Populate Properties**: DI via `@Autowired`, constructor, setters│
│  • **Aware Callbacks**: `BeanNameAware`, `ApplicationContextAware`, etc.  
│  • **postProcessBeforeInit**: `BeanPostProcessor`                     │
│  • **Init Callbacks**: `@PostConstruct`, `InitializingBean`, `initMethod`  
│  • **postProcessAfterInit**: AOP proxying, metrics, tracing            │
└────────────────────────────────────────────────────────────────────────┘
             ↓
┌────────────────────────────────────────────────────────────────────────┐
│                    Smart Lifecycle & AOT Hooks                       │
│  • `SmartInitializingSingleton.afterSingletonsInstantiated()`        │
│  • `SmartLifecycle` beans start in phase order                       │
│  • Spring Boot 3 AOT: native image runtime optimizations applied      │
└────────────────────────────────────────────────────────────────────────┘
             ↓
┌────────────────────────────────────────────────────────────────────────┐
│                          Application Ready                           │
│  • `ApplicationStartedEvent`                                          │
│  • `ApplicationReadyEvent`                                            │
│  • Your `CommandLineRunner` and `ApplicationRunner` beans execute     │
└────────────────────────────────────────────────────────────────────────┘
             ↓
┌────────────────────────────────────────────────────────────────────────┐
│                         Graceful Shutdown                            │
│  • On JVM exit or `context.close()`                                    │
│  • `@PreDestroy` / `DisposableBean.destroy()`                         │
│  • `stop()` on `SmartLifecycle` beans                                 │
│  • `ApplicationFailedEvent` / `ApplicationStoppedEvent`               │
└────────────────────────────────────────────────────────────────────────┘
```
