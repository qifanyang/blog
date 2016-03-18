---
title: spring aop实现
date: 2016-02-04 14:53:42
tags: spring
---
## AOP
aop是对oop的补充,面向对象编程强调的是封装,继承,多态,对于封装得很好的面向对象的程序,需求变化得很快,当需要在很多地方加入公共逻辑
在一个庞大的程序的各个地方去修改代码是不现实的,工作量太大.tomcat的filter实现了动态添加一些逻辑的功能,但是在应用开发的时候,不可
能在任意地方埋点(比如回调等),为了再任意地方可以插入自定义逻辑,AOP产生了.

Spring AOP实现采用了jdk proxy或cglib proxy,在执行被代理对象方法时先执行intercept list,在spring中,AOP主要用于:
1.declarative transaction management 声明式事务管理
2.实现自定义切面,完成一些自定义逻辑


## AOP术语
aspect 一个关注点的模块化，这个关注点可能会横切多个对象。事务管理是J2EE应用中一个关于横切关注点的很好的例子
joinpoint 对应一个方法的执行
advice 在切面的某个特定的连接点上执行的动作。其中包括了“around”、“before”和“after”等不同类型的通知
pointcut 匹配连接点的断言。通知和一个切入点表达式关联(就是advisor)，并在满足这个切入点的连接点上运行
introduction
target object 被一个或者多个切面所通知的对象。也被称做被通知（advised）对象
aop proxyAOP框架创建的对象，用来实现切面契约（例如通知方法执行等等）。在Spring中，AOP代理可以是JDK动态代理或者CGLIB代理。
weaving把切面连接到其它的应用程序类型或者对象上，并创建一个被通知的对象。这些可以在编译时（例如使用AspectJ编译器），
类加载时和运行时完成。Spring和其他纯Java AOP框架一样，在运行时完成织入。
advisor这个概念来自Spring1.2对AOP的支持,而在AspectJ中没有等价的概念。 advisor就像一个小的自包含的切面，这个切面只有一个通知

通过切入点匹配连接点的概念是AOP的关键，这使得AOP不同于其它仅仅提供拦截功能的旧技术。 切入点使得通知可以独立对应到面向对象的层次结构中。例如，一个提供声明式事务管理	的环绕通知可以被应用到一组横跨多个对象的方法上（例如服务层的所有业务操作）。

## AspectJ使用
@AspectJ使用了Java 5的注解，可以将切面声明为普通的Java类,AOP运行时依然使用的是spring aop.
在类上使用注解Aspect,方法上使用@PointCut,@Before
aspect表达式execution（modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern（param-pattern）throws-pattern?）
除了返回类型模式（上面代码片断中的ret-type-pattern），名字模式和参数模式以外， 所有的部分都是可选的
所以至少有三部分参数,其它参数可有可无
任意公共方法的执行：execution（public * *（..））
任何一个名字以“set”开始的方法的执行：execution（* set*（..））
AccountService接口定义的任意方法的执行：execution（* com.xyz.service.AccountService.*（..））


## <aop:config>
当在spring bean 定义文件中使用aop:config,NamespaceHandler注册AspectJAwareAdvisorAutoProxyCreator,这是一个BeanPostProcessor,
在回调方法中,会查找容器所有的advisor,并判断advisor是否可以应用于bean,AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
当有advisor可以应用于bean时,才会创建该bean的动态代理,在执行动态代理对象方法调用的时候,还会再次匹配pointcut来返回advice
proxyFactory.addAdvisor(advisor);然后在执行代理方法的时候判断advisor是否和调用的方法匹配,这里使用了缓存,不是每次调用都去判断
ConfigBeanDefinitionParser,向spring容器中注册DefaultBeanFactoryPointcutAdvisor,里面会读取xml配置的advice和pointcut,
前面AspectJAwareAdvisorAutoProxyCreator,就是根据这个advisor来创建动态代理的

简单来说,如果使用了aop:config,spring会自动向容器中添加一个BeanPostProcessor和一个Advisor来完成动态代理自动创建
aop:advisor,aop:aspect,@Aspect,这三种方法底层实现一样,所以可以兼容

### springAop手动创建

	以JDK动态代理为例子来跟踪流程,一下是spring doc中的例子

	ProxyFactory factory = new ProxyFactory(new SimplePojo());
    factory.addAdvice(new RetryAdvice());//advisor包装advice
    Pojo pojo = (Pojo) factory.getProxy();
    pojo.pojoName();
	
	Pojo为接口,SimplePojo为子类,RetryAdvice为一个MethodInterceptor

	factory.addAdvice(new RetryAdvice()),默认使用DefaultPointcutAdvisor包装advice,注意默认使用的PointCut.TRUE,表示匹配任意方法调用
	
	factory.getProxy()返回JdkDynamicAopProxy.getProxy()创建的动态代理,并且返回JdkDynamicAopProxy
	实现了借口InvocationHandler,所以当调用pojo.pojoName()的时候,调用返回JdkDynamicAopProxy.invoke()

	进入JdkDynamicAopProxy.invoke(),
	List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
	先根据前面添加的advice,这里返回匹配pointCut的methodInterceptor列表,然后先调用拦截器链在调用实际的方法
	
	invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
	// Proceed to the joinpoint through the interceptor chain.
	retVal = invocation.proceed();
	使用ReflectiveMethodInvocation包装,然后再ReflectiveMethodInvocation中完成调用,这里算是一种策略模式吧,
	注意这里使用了一个递归来完成拦截器列表的遍历执行,当递归次数达到拦截器列表长度的时候,表示拦截器全部执行完毕
	可以调用被代理的方法了.


### springAOP事务配置
	spring事务配置有很多种方式,习惯用tx标签和注解来实现
	
	tx标签
	<tx:advice id="txAdvice" transaction-manager="transactionManager">
        <tx:attributes>
            <tx:method name="*" propagation="REQUIRED" />
        </tx:attributes>
    </tx:advice>

	<aop:config>
        <aop:pointcut id="interceptorPointCuts" expression="execution(* com.bluesky.spring.dao.*.*(..))" />
        <aop:advisor advice-ref="txAdvice" pointcut-ref="interceptorPointCuts" />
    </aop:config>
	
	<tx:advice>解析的时候会向spring容器注入TransactionInterceptor,
	<aop:config>定义advisor(其中包含advice和pointcut),返回dao对象时手动构建代理对象变添加advisor


	注解方式
	<tx:annotation-driven transaction-manager="transactionManager" />
	对应的NamespaceHandler注入了一个InfrastructureAdvisorAutoProxyCreator,该类是一个BeanPostProcessor,,当从容器中获取对象
	时,自动判断是否创建代理,事务拦截器完成事务相关的功能,AbstractAutoProxyCreator自动完成proxy创建

### spring读取配置XmlBeanDifinitionReader
	spring读取xml配置文件对于每个标签都有对应的NamespaceHandler,在解析xml配置文件的时候


	