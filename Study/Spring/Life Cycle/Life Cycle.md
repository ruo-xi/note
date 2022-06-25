## Life Cycle

### Beanpost Processor

1.  InstantiationAwareBeanPostProcessor.**postProcessBeforeInstantiation**(Class<?> beanClass, String beanName)
2.  MergedBeanDefinitionPostProcessor.**postProcessMergedBeanDefinition**(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName)
3.  InstantiationAwareBeanPostProcessor.**postProcessAfterInstantiation**(Object bean, String beanName)
4.  InstantiationAwareBeanPostProcessor.**postProcessProperties**(PropertyValues pvs, Object bean, String beanName)
5.  BeanPostProcessor.**postProcessBeforeInitialization**(Object bean, String beanName)
6.  BeanPostProcessor.**postProcessAfterInitialization**(Object bean, String beanName)



