---
title: spring-summary
date: 2016-05-27 15:46:32
tags: spring
---

## IoC/DI
BeanDifinition定义对象的依赖,bean通过beanDifition来自己控制实例化自己和它的依赖,不需要硬编码
来设置依赖的对象,控制权交给了bean自己,所以叫控制反转(也叫依赖注入).


## Bean difinition Overview
spring container manages bean defition, 可以通过xml定义,注解定义,java编码定义
bean definition:

	class-->bean实例化时创建的对象的类
	name-->bean的名字,还可以有别名
	scope-->bean实例化行为,主要关注单例(single)和非单例(prototype)
	constructor arguments-->bean实例化时传入的参数,注意循环依赖注入
	properties-->bean依赖的对象,spring container会注入
	autowiring mode-->自动装配依赖的属性,不适用构造参数和属性设置,而且新加入依赖也不用更改配置,效率高.通过名字或者类型向容器要依赖的对象
	lazy-initialization mode-->默认spring初始化容器后就会创建bean实例,lazy-init=true就不会实例化,当非lazy single依赖lazy的bean,会实例化lazy bean依赖的对象
	initialization method-->spring bean声明周期,创建bean后调用指定方法完成初始化(C),初始化还可以通过@PostConstructor(A),InitializingBean(B)等
	destruction method-->类似initialization method, 容器写在bean时完成清理工作

## spring container 初始化过程
AbstractApplicationContext.refresh()完成spring初始化
1.ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();-->创建BeanFactory(默认是DefaultListAbleBeanFactory),加载bean difinition
2.prepareBeanFactory(beanFactory);-->配置beanFactory,默认加入一些BeanPostProcessor,忽略一些依赖,按类型依赖的对象
3.postProcessBeanFactory(beanFactory);-->默认空实现,可以在这里修改beanfactory中bean difinition,这时候没有实例化任何bean
4.invokeBeanFactoryPostProcessors(beanFactory);-->从BeanFactory按类型找出所有BeanFactoryPostProcessor,实例化BeanFactoryPostProcessor,可以调用他们来修改beanfactory中的bean difinition
5.registerBeanPostProcessors(beanFactory);-->同BeanFactoryPostProcessor类似,这里查找所有的BeanPostProcessor,并放入AbstractBeanFactory中的一个ArrayList(注意processor顺序)中,稍后实例化时用来处理bean对象
6.initMessageSource();-->估计是国际化
7.initApplicationEventMulticaster();-->spring事件处理系统,可以发布事件然后处理事件,或者关注特定的事件(观察者模式)
8.onRefresh();-->Initialize other special beans in specific context subclasses. 模板方法,默认没有实现
9.registerListeners()-->对应上面的事件系统,从BeanFactory查找ApplicationListener,并记录到Multicaster, spring定义了几个容器启动,关闭,刷新事件
10.finishBeanFactoryInitialization(beanFactory);-->初始化所有非lazy的单例bean,先初始化LoadTimeWeaverAware bean(加载tranformers),到这里设置bean定义不能再修改, 然后开始单例实例化
11.finishRefresh;
11_1.initLifecycleProcessor;-->向BeanFactory注册DefaultLifecycleProcessor,beanFactory.registerSingleton(name,obj)
11_2.getLifecycleProcessor().onRefresh();-->从BeanFactory查找Lifecycle bean,调用对应的start方法
11_3.publishEvent(new ContextRefreshedEvent(this));-->发布spring事件,先前的ApplicationListener得到通知
12.一些其它后续操作
到此为止,spring初始化完毕,其中1和10内部进行的工作最多

## bean difiniiton解析
1.创建DefaultListableBeanFactory实例,作为BeanFactory默认实现,为加载bean definition做准备
2.创建XmlBeanDefinitionReader(持有bean factory引用,用于bean注册, 因为该reader只具备注册bean definition功能,所以只可以持有BeanDefinitionRegistry引用, 不应该持有BeanFactory引用,好的设计),,开始解析配置的配置文件路径 application.xml
3.创建BeanDefinitionDocumentReader(只有XmlBeanDefinitonReader引用,用于访问bean factory等)开始解析, 创建上下文(XmlReaderContext,自定义bean definiiton parser必要参数,包含namespcehander,BeanReader)
4.BeanDefinitionDocumentReader开始解析xml信息,委托代理BeanDefinitionParserDelegate解析并返回bean definition
5.BeanDefinitionParserDelegate代理解析详细xml配置,创建GenericBeanDefinition实例并配置其属性并返回
6.BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry()) 注册bean定义
7.上面是解析默认标签(Beans下,bean, import,alias等),解析自定义标签,需要根据标签名从readcontext的namespaceHandlerResover中获取对应的NamespaceHandler,这里可以扩展自己的bean 定义解析器
8.hander.parse(Element,ParserContext),开始解析元素,ParserContext包含了readerContext(同解析默认标签,可以间接访问BeanFactory),实际的BeanDefinitionParser


从xml加载类似下面这种xml定义
<bean id="name" class="com.user" scope="prototype" ...> //同bean definition
XmlBeanDefinitionReader解析xml配置创建bean difinition并注册, AbstractBeanDefinitionReader主要包含 BeanDifinitionRegistry,resourceLoader
BeanDefinitionDocumentReader完成具体解析,
BeanDefinitionParserDelegate实际完成具体解析,还会解析default-init-method等等,
在解析过程中,可能会产生bean name重复等错误,所以需要提供错误反馈机制,包含错误消息,错位位置,方便定位,默认ProblemReporter实现是fail-fast抛出异常
错误报告器抽象的原因是,当产生错误时可以采取不同的策略,可以是fail-fast,可以使记录日志,可以是忽略等等
						BeanDefinition
							|
							|
				AbstractBeanDefinition
		|					|						|
RootBeanDefinition GenericBeanDefinition ChildBeanDefinition

根据bean xml创建bean definition, 默认是GenericBeanDefinition并设置beanClass或者className
然后设置一些列bean definition属性
parseMetaElements(ele, bd);
parseLookupOverrideSubElements(ele, bd.getMethodOverrides());//bean引用另外一个bean method, 但是另外的bean还没实例化,使用类似占位符方式
parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
parseConstructorArgElements(ele, bd);
parsePropertyElements(ele, bd);
parseQualifierElements(ele, bd);
然后注册bean difinition到 BeanDifinitionRegistry中, 实现类是DefaultListableBeanFactory, 用beanDefinitionMap存储

bean名字:
默认为id, 如果没有id则使用name(如果有多个name使用第一个,所以使用name和id效果一样),如果还没有name则使用默认名字生成策略
<bean id="xyx" name="abc,def" .. />// name指定多个别名
<alias name="xyx" alias="pqr"/>//很少使用,如果没法修改bean定义, 可以使用这种方式新加别名,spring提供了别名和raw name循环引用检查
aliasMap key为alias name, value为raw name, 

2.扫描包加载
<context:component-scan base-package="com.test"> //会扫描指定的包名下的所有类,包含子目录下的类(通配符 **/*.class),使用类ClassPathBeanDefinitionScanner实现
包扫描返回的bean difinition类是ScannedGenericBeanDefinition, 因为没有xml定义,需要知道类名字,是否抽象,父接口等需要信息,spring使用asm classReader来访问字节码
使用了访问者模式获取类信息,遍历方法,字段等
扫描包的时候,通过filter决定是否加载class为bean difinition, 默认会添加AnnotationTypeFilter(Component.class),当使用Controller注解也会加载,因为Controller注解使用了Componet作为元注解,还可以使用(javax.inject.Named),所以必须要使用Componet注解    
扫描加载了所有bean difinition之后,读取@Scope,@ScopedProxyMode来配置,没有则采用默认配置,然后确定bean名字,默认类首字母小写
然后设置beanDefinitionDefaults (在<beans ...> 中设置的default属性),在经历了和加载xml类似的过程之后,注册创建好的bean difinition到registry中,就完成了bean difinition扫描


3.java编码配置
spring boot 和 spring cloud使用该方式

## spring内部存放bean实例
	1.DefaultSingletonBeanRegistry.singletonObjects-->单例对象map,Cache of singleton objects: bean name --> bean instance
	2.DefaultSingletonBeanRegistry.singletonFactories-->提前曝光单例,但是单例属性还未设置完毕,Cache of singleton factories: bean name --> ObjectFactory
	3.DefaultSingletonBeanRegistry.earlySingletonObjects-->ObjectFactory创建的对象,属性还未设置,Cache of early singleton objects: bean name --> bean instance
	4.DefaultSingletonBeanRegistry.registeredSingletons-->Set of registered singletons, containing the bean names in registration order

其中2和3互斥,意思就是如果bean实例在2中,3中就不包含. 在3中就不在2中, 在[bean实例化](#bean实例化)过程会创建对象放入上面的数据结构中

## bean实例化
实例化触发点:
	1.初始化spring container时,单例提前实例化
	2.beanFactory.getBean(...)获取bean时实例化,实例化单例时有依赖自动调用getBean(name)

提前初始化单例时,遍历已经加载的bean definition,如果是非抽象,非lazy-init的单例,就执行实例化,通过执行方法getBean(name)
如果name以&开头(factoryBean),去掉&. 如果name是alias name,则需要获取raw name

1.在spring加载了所有bean definition之后,提前初始化非抽象,非lazy-init的单例,实现是遍历已经加载的bean definition, 执行方法getBean(name), 和手动获取bean实例一致

获取bean实例主要过程:

	1.doGetBean(name, requireType, args, typeCheckOnly)-->无论getBean(name),getBean(type),getBean(name,type)最终都会执行这里
	2.getSingleton(beanName)-->从缓存的singletonObjects查找,返回的可能是ObjectFactory创建的bean实例(属性没有被设置,在doCrateBean()方法中放入的),可能是FactoryBean
	3.if (isPrototypeCurrentlyInCreation(beanName)) {throw new BeanCurrentlyInCreationException(beanName);}-->只有在单例情况下
	才会尝试解决循环依赖,原型模式下有循环依赖直接抛出异常
	4.RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);-->将加载xml产生的GenericBeanDifition转换成RootBeanDifition
	因为xml定义的bean可能有父子关系,如果是子bean,则会合并父bean的相关属性,并使用子bean重写已经存在的属性,这里又有个递归向上查找
	5.String[] dependsOn = mbd.getDependsOn();-->检查循环依赖,如果没有循环依赖则实例化依赖的bean,<property name="a" ref="b"/>这种使用PropertValue保存
	6.mbd.isSingleton(),getSingleton(String, ObjectFactory<?>)-->创建单例,创建过程太复杂,默认通过mbd查找默认构造器创建实例,然后使用BeanWrapper包装,使用mbd初始化
	7.最复杂的是createBean(beanName, mbd, args)方法,doCreateBean()完成实例创建,BeanWrapper包装mbd中class创建的实例,如果实例是earlySingletonExposure
	则将该bean实例addSingletonFactory,this.singletonFactories.put(beanName, singletonFactory);这时候bean实例属性还没有填充.如果使用了AOP(<aop:aspectj-autoproxy>)(AnnotationAwareAspectJAutoProxyCreator)
	则调用InstantiationAwareBeanPostProcessorc,如果直接创建了代理对象则直接返回实例对象
	8.populateBean(beanName, mbd, instanceWrapper);-->填充bean属性,autowireByName(),根据mbd和bw获取bean中没有设置的属性,bw.getPropertyDescriptors()所有属性pd
	mbd.getPropertyValues();存在的属性值, 两者交集的补集就是需要注入的属性,然后根据属性调用getBean(name),又递归到第一步.如果有循环依赖,会使用ObjectFactory来创建对象(返回的对象是属性没填充的对象)
	不会使用createBean()方法(里面有populateBean()),

## spring如何解决循环依赖
A依赖B,B依赖A, 如何解决循环依赖,首先对于scope为prototype的bean,因为spring不会缓存这类bean的实例,所以无法解决循环依赖(创建时缓存了呢?)
用一次创建A和B为例子,spring创建bean A单例实例,不会直接创建bean实例,而是创建ObjectFactory,在需要访问bean A时,使用回调返回bean A(这里的bean A属性没有填充完毕), 当A populateBean属性是实例化B, 而B的populateBean又需要A, 这时返回的A是通过
ObjectFactory返回的,其属性没有被填充,但是可以用于设置B属性,  当A的依赖填充完毕后, B中引用的A也同时被填充完毕...


## spring AOP
1.使用@Aspectj

	使用该注解,会注册AnnotationAwareAspectJAutoProxyCreator.class到容器中,改类是一个InstanstionAwareBeeanPostProcessor,在每次从容器获取bean实例时,会
	自动调用该BeanPostProcessor创建代理类,

static {
		APC_PRIORITY_LIST.add(InfrastructureAdvisorAutoProxyCreator.class);//使用<tx:annotation-driven>会注入
		APC_PRIORITY_LIST.add(AspectJAwareAdvisorAutoProxyCreator.class);//使用<aop:config>会注入 <tx:advice>
		APC_PRIORITY_LIST.add(AnnotationAwareAspectJAutoProxyCreator.class);//使用<aop:aspectj-autoproxy>会注入
	}
三个自动代理创建器,list索引代表了优先级,spring容器中只能注册一个,重复注册优先级高的会替换优先级低的
AnnotationAwareAspectJAutoProxyCreator是一个BeanPostProcessor,在从spring容器获取实例对象时,会检查容器中使用@Aspectj注解的bean, 然后创建一个Advisor.
*AutoProxyCreator有个抽象方法getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, TargetSource customTargetSource)用于查找beanClass是否被代理,
被代理的话返回advice或者advisor. 其内部实现是先查找容器中所有Advisor,然后遍历advisor并使用pointcut验证beanclass是否可以被代理,可以被代理的话加入返回列表中,只要被代理
class有一个方法匹配都应该返回, 实际方法调用的时候还要再次验证目标方式是否可以被拦截


2.使用<aop:config>
除了使用@Aspectj注解之外,还可以使用<aop:config>,标签解析时spring注册AspectJAwareAdvisorAutoProxyCreator.class, 不过如果使用了@Aspectj标签的话会使用AnnotationAwareAspectJAutoProxyCreator
替代,不会使用AspectJAwareAdvisorAutoProxyCreator, 因为AnnotationAwareAspectJAutoProxyCreator是AspectJAwareAdvisorAutoProxyCreator的子类,查找advisor时会查找把使用@Aspectj注解的类作为advisor返回

例如下面添加一个advistor, 解析时会向容器中注册advisor, pointcut,aspectj
  <aop:config proxy-target-class="true"> //代理目标类,意思就是使用Cglib创建代理类,不适用JDK对接口创建的动态代理
        <aop:advisor advice-ref="transactionMethodInterceptor" //拦截器也是一个Advice
                     pointcut-ref="transactionMethodInterceptorPointcut"/>//指定pointcut,符合拦截规则将使用advice-ref执行的advice
 </aop:config>
 
拦截器有多种类型,Advisor,MethodInterceptor,Advice  但是最终都会包装成Advisor


## spring声明式事务配置
1.使用TransactionProxyFactoryBean来代理dao bean

2.使用拦截器TransactionInterceptor, 将手动使用TransanctionTemplate的方式自动化,控制粒度比较粗,可以使用<aop:config>将该拦截器用于service,也可以使用BeanNameAutoProxyCreator
<bean id="transactionInterceptor" class="org.springframework.transaction.interceptor.TransactionInterceptor">
        <property name="transactionManager" ref="transactionManager" />
        <!-- 配置事务属性 -->
        <property name="transactionAttributes">
            <props>
                <prop key="*">PROPAGATION_REQUIRED</prop>
            </props>
        </property>
    </bean>

3.使用tx标签配置的拦截器,和使用事务拦截器效果差不多,内部实现也是使用TransactionInterceptor作为advice实现,见TxAdviceBeanDefinitionParser.getBeanClass()
	<tx:advice id="txAdvice" transaction-manager="transactionManager">
        <tx:attributes>
            <tx:method name="*" propagation="REQUIRED" />
        </tx:attributes>
    </tx:advice>

    <aop:config>
        <aop:pointcut id="interceptorPointCuts" expression="execution(* com.bluesky.spring.dao.*.*(..))" />
        <aop:advisor advice-ref="txAdvice" pointcut-ref="interceptorPointCuts" />
    </aop:config>

4.使用事务注解@Transaction来使用事务,相对于<aop:config>配置事务来说没增加多少配置,但是控制粒度更细
<tx:annotation-driven transaction-manager="transactionManager" order="2"/>
AnnotationDrivenBeanDefinitionParser也是使用TransactionInterceptor作为advice,BeanFactoryTransactionAttributeSourceAdvisor作为advisor,默认使用InfrastructureAdvisorAutoProxyCreator作为自动代理创建器,具体使用哪个
AutoProxyCreator主要还是在于使用了哪些AOP方式,遵循AopConfigUtils中的APC_PRIORITY_LIST列表. BeanFactoryTransactionAttributeSourceAdvisor是一个advisor,使用了TransactionAttributeSourcePointcut作为pointcut,实例化bean时会调用该pointcut来执行匹配,
BeanFactoryTransactionAttributeSourceAdvisor的pointcut为TransactionAttributeSourcePointcut,改pointcut的classFileter为ClassFilter.TRUE,所以不在class上使用@Transaction注解也可以,  
然后该pointcut的method match使用TransactionAttributeSource.getTransactionAttribute(method, targetClass) != null来判断是否满足切面,在使用<annotation-driver >时注入proxyCreator设置的TransactionAttributeSource为AnnotationTransactionAttributeSource  
会首先判断方法上是否存在@Transactional注解,没有再判断类上是否有@Transactional注解,    
如果方法上或者类上使用了@Transactional注解,就说明match成功,说明该类可以用于事务增强,并创建RuleBasedTransactionAttribute作为事务属性实例(方法调用时使用),使用class+method作为key缓存在transactionAttributeSource中,当方法调用时,transactionInterceptor
会根据class+method去查找执行匹配时创建的事务属性来执行事务



## spring事务管理
分为编程式事务(类似JDBC事务)和声明式事务

### 编程式事务
1.jdbc事务-->1.设置不自动提交事务setAutoCommit(false)  2.执行sql 3.commit()有异常则rollBack()
2.hibernate事务-->1.beginTransaction() 2.执行sql 3.提交或者回滚  和jdbc类似, 只是用方法封装多做了框架相关的工作
3.spring编程式事务-->使用模板方法将开启事务,提交事务和回滚事务提取出来, 用户代码在回调中执行sql操作. TransanctionTemplate.execute(...)

在spring中使用编程式事务使用的对象:
1.TransactionDefinition(默认实现DefaultTransanctionDefinition),包含了事务隔离性,传播特性(如果在开始当前事务之前，一个事务上下文已经存在),是否只读,超时时间等事务相关的属性
2.PlatformTransactionManager(有多种实现,DataSourceTransactionManager,JtaTransactionManager)等
3.TransactionStatus代表事务状态对象(包含事务对象,挂起的事务等),使用该对象执行事务相关操作,提交,回滚,挂起以及事务状态查询

编程式事务使用方式,先创建事务定义(可以使用默认),然后从事务管理器中获取一个事务对象,然后使用该事务对象进行数据库操作
需要解决的问题:
1.创建数据库连接,设置事务属性-->这个简单,获取数据库连接,根据TransactionDefinition设置即可,spring还进行了包装ConnectionHolder->DataSourceTransactionObject->TransactionStatus
2.事务传播行为-->业务逻辑中数据库操作都放在方法当中,一般不可能在一个方法当中修改事务传播行为,所以已方法为单位来应用传播行为.当处理一个请求时,服务器可能会有一个或者多个方法调用,形成一个方法栈链.在这一次请求中
需要一个事务上下文环境,ThreadLocal用来存储. 当每方法调用时就会查看当前事务上下文事务已经有事务存在,然后根据传播行为来决定是否新建事务,加入事务还是挂起事务. 当然执行事务的前提是该类和改方法进行了事务增强

3.为什么用DataSource作为ThreadLocal的key,因为JDBCTemplate执行sql,需要获取数据库连接(自然需要注入DataSource),创建事务和执行sql的datasource都应该是同一个,在单个jvm中这个值是唯一的,所以用来作为ThreadLocal的key比较合理,如果不采用该值
那么在存储线程上下文到ThreadLocal时需要确定一个key, 而且要保证spring jdbc template代码中能够拿到存储在事务上下文中的数据库连接(或者没有使用事务就直接从数据源获取数据库连接),如果使用一个常量字符串作为key,就需要一个全局常量也可以满足, 如果有两个数据库那么使用字符串就行不通了.



### 声明式事务
事务方法调用,增强都会用到TransactionInterceptpr,这是一个MethodInterceptor,调用事务方法都会执行改拦截器,执行和TransactionTemplate差不多的事务逻辑


## spring依赖属性注入是调用set方法(Bean WriteMethod),如果在Bean difinition的property map中包含属性键值对,即使没有对应属性名字也会调用set方法

## spring扩展
1.自定义标签解析,添加BeanDefinitionParser,NamespaceHandler,xsd, spring.handlers, spring.schemas文件
解析时用的对象:
ReaderContext-->上下文
	|
	XmlBeanDefinitionReader-->包含BeanDefinitionRegistry,namespaceHandlerResolver
	|
	NamespaceHandlerResolver-->获取标签对应的parser




2.注册对象(将对象放入singletonObjects map中)或注册bean definition(将定义放入Beandefinition map中)
3.AutoProxyCreator实现


## Autowired
最常用的Autowired,可以作用于字段,方法, 使用AutowiredAnnotationBeanPostProcessor处理,从bean factory中去寻找指定类型的依赖,如果该类型有多个实例,在依赖的实例上使用@Primary,表明注入这个实例
也可以使用@Autowired @Qualifier明确指出依赖注入哪一个



