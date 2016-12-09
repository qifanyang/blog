## Transaction注解使用
配置文件中添加:  
<tx:annotation-driven transaction-manager="transactionManager"/>  
加载时spring 自定义namespaceHandler注册BeanPostProcessor,Interceptor等  

1.AopNamespaceUtils.registerAutoProxyCreatorIfNecessary(parserContext, element);  
注册自动代理创建器,InfrastructureAdvisorAutoProxyCreator,该类是一个InstantiationAwareBeanPostProcessor  
从bean container中获取对象时会自动创建代理,使用BeanFactoryTransactionAttributeSourceAdvisor  

2.判断是否创建事务代理

3.TransactionInterceptor  ,AnnotationTransactionAttributeSource  


1.获取使用了Transaction注解的对象时,创建代理对象,添加的拦截器.  

2.添加的拦截器