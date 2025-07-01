```java
Context Start
 └─> Load BeanDefinitions
      └─> BeanFactoryPostProcessors
           └─> For each BeanDefinition:
                └─> Instantiate Bean
                     └─> Populate Properties / DI
                          └─> Aware callbacks
                               └─> BeanPostProcessor.postProcessBeforeInitialization
                                    └─> afterPropertiesSet / @PostConstruct / init-method
                                         └─> BeanPostProcessor.postProcessAfterInitialization
                                              └─> (bean is ready)

Context Shutdown
 └─> For each singleton bean:
      └─> @PreDestroy
           └─> DisposableBean.destroy
                └─> custom destroy-method
                     └─> DestructionAwareBeanPostProcessor

```
