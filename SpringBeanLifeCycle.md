# Spring Bean Lifecycle & Initialization Hooks

A comprehensive guide explaining the complete lifecycle of a Spring Bean in the Spring ApplicationContext, from instantiation to destruction, including Aware interfaces, `BeanPostProcessor` execution, `@PostConstruct`, `@PreDestroy`, and custom callback methods.

---

## 1. High-Level Lifecycle Flow Diagram

```text
Context Start Phase
 └─> Load BeanDefinitions
      └─> Execute BeanFactoryPostProcessors (e.g. PropertyPlaceholderConfigurer)
           └─> Iterate over BeanDefinitions:
                ├─ 1. Instantiate Bean (Constructor / Reflection)
                ├─ 2. Populate Properties / Dependency Injection (@Autowired, setters)
                ├─ 3. Invoke Aware Callbacks (BeanNameAware, BeanFactoryAware, ApplicationContextAware)
                ├─ 4. BeanPostProcessor.postProcessBeforeInitialization()
                ├─ 5. Initialization Callbacks:
                │    ├─ @PostConstruct method
                │    ├─ InitializingBean.afterPropertiesSet()
                │    └─ Custom init-method defined in @Bean
                ├─ 6. BeanPostProcessor.postProcessAfterInitialization() (AOP Proxy Wrapping)
                └─ [Bean is Fully Initialized & Ready for Use]

Context Shutdown Phase
 └─> ApplicationContext.close()
      └─> For each singleton bean:
           ├─ 1. @PreDestroy method
           ├─ 2. DisposableBean.destroy()
           └─ 3. Custom destroy-method defined in @Bean
```

---

## 2. Detailed Lifecycle Stages Explained

### Stage 1: Bean Instantiation
The Spring IoC container reads `BeanDefinition` metadata and instantiates the bean class using constructor injection or default zero-arg constructors via reflection.

### Stage 2: Property Population & Dependency Injection
Spring injects required dependencies into the instantiated bean via field `@Autowired`, setter injection, or constructor parameter resolution.

### Stage 3: Aware Interface Callbacks
If the bean implements any Spring `Aware` marker interfaces, Spring passes internal framework components to the bean:
- `BeanNameAware.setBeanName(String name)`: Passes the configured bean name.
- `BeanFactoryAware.setBeanFactory(BeanFactory factory)`: Passes the owning `BeanFactory`.
- `ApplicationContextAware.setApplicationContext(ApplicationContext context)`: Passes the running `ApplicationContext`.

### Stage 4: `BeanPostProcessor` Before Initialization
Spring executes `postProcessBeforeInitialization(Object bean, String beanName)` for all registered `BeanPostProcessor` beans on the current bean. `@PostConstruct` annotations are processed during this phase by Spring's `InitDestroyAnnotationBeanPostProcessor`.

### Stage 5: Initialization Callbacks
Spring triggers custom initialization logic in the following sequence:
1. **`@PostConstruct`**: JSR-250 annotation executed first.
2. **`InitializingBean.afterPropertiesSet()`**: Interface callback method.
3. **Custom `init-method`**: Configured on `@Bean(initMethod = "customInit")`.

### Stage 6: `BeanPostProcessor` After Initialization
Spring executes `postProcessAfterInitialization(Object bean, String beanName)`. This is the exact phase where Spring AOP creates **dynamic JDK proxies or CGLIB proxies** for beans annotated with `@Transactional`, `@Async`, or custom aspects.

---

## 3. Production Java Example

```java
package com.example.demo.lifecycle;

import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.BeanNameAware;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.DisposableBean;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;

@Component
public class DatabaseConnectionPoolBean implements 
        BeanNameAware, 
        ApplicationContextAware, 
        InitializingBean, 
        DisposableBean {

    private String beanName;
    private ApplicationContext applicationContext;

    public DatabaseConnectionPoolBean() {
        System.out.println("1. Constructor: Bean Instantiated");
    }

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("2. BeanNameAware: setBeanName(" + name + ")");
    }

    @Override
    public void setApplicationContext(ApplicationContext context) throws BeansException {
        this.applicationContext = context;
        System.out.println("3. ApplicationContextAware: setApplicationContext(...)");
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("4. @PostConstruct: Executing initial warm-up tasks");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("5. InitializingBean: afterPropertiesSet() executed");
    }

    public void customInitMethod() {
        System.out.println("6. Custom initMethod executed");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("7. @PreDestroy: Releasing pool connections prior to context shutdown");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("8. DisposableBean: destroy() executed");
    }
}
```

---

## 4. Key Differences: `BeanFactoryPostProcessor` vs `BeanPostProcessor`

| Feature | `BeanFactoryPostProcessor` | `BeanPostProcessor` |
| :--- | :--- | :--- |
| **Execution Target** | Modifies **BeanDefinition metadata** before any bean objects are created. | Modifies **actual bean object instances** after instantiation. |
| **Common Use Cases** | Resolving property placeholders (`${db.url}`), overriding bean scope definitions. | Creating AOP proxies (`@Transactional`), processing annotations (`@Autowired`, `@PostConstruct`). |
| **Example Classes** | `PropertySourcesPlaceholderConfigurer` | `AutowiredAnnotationBeanPostProcessor`, `AnnotationAwareAspectJAutoProxyCreator` |
