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
1.从xml加载
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
bean根据className或classType创建实例,并且需要解决依赖关系,2016-5-29分析下依赖如何解决...
实例化触发点:
	1.初始化spring container时,单例实例化
	2.beanFactory.getBean(...)获取bean时实例化,实例化时有依赖自动调用getBean(name)

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
		APC_PRIORITY_LIST.add(InfrastructureAdvisorAutoProxyCreator.class);
		APC_PRIORITY_LIST.add(AspectJAwareAdvisorAutoProxyCreator.class);
		APC_PRIORITY_LIST.add(AnnotationAwareAspectJAutoProxyCreator.class);	
		}
三个自动代理创建器,list索引代表了优先级,spring容器中只能注册一个,重复注册优先级高的会替换优先级低的
AnnotationAwareAspectJAutoProxyCreator是一个BeanPostProcessor,在从spring容器获取实例对象时,会检查容器中使用@Aspectj注解的bean, 然后创建一个Advisor.
*AutoProxyCreator有个抽象方法getAdvicesAndAdvisorsForBean(Class<?> beanClass, String beanName, TargetSource customTargetSource)用于查找beanClass是否被代理,
被代理的话返回advice或者advisor. 其内部实现是先查找容器中所有Advisor,然后遍历advisor并使用pointcut验证beanclass是否可以被代理,可以被代理的话加入返回列表中,只要被代理
class有一个方法匹配都应该返回, 实际方法调用的时候还要再次验证目标方式是否可以被拦截


2.使用<aop:config>
除了使用@Aspectj注解之外,还可以使用<aop:config>,标签解析时spring注册AspectJAwareAdvisorAutoProxyCreator.class, 不过如果使用了@Aspectj标签的话会使用AnnotationAwareAspectJAutoProxyCreator
替代,不会使用AspectJAwareAdvisorAutoProxyCreator, 因为AnnotationAwareAspectJAutoProxyCreator是AspectJAwareAdvisorAutoProxyCreator的子类,查找advisor时会把使用@Aspectj注解的类作为advisor返回


例如下面添加一个advistor, 解析时会向容器中注册advisor, pointcut,aspectj
  <aop:config proxy-target-class="true"> //代理目标类,意思就是使用Cglib创建代理类,不适用JDK对接口创建的动态代理
        <aop:advisor advice-ref="transactionMethodInterceptor" //拦截器也是一个Advice
                     pointcut-ref="transactionMethodInterceptorPointcut"/>//指定pointcut,符合拦截规则将使用advice-ref执行的advice
 </aop:config>
 
拦截器有多种类型,Advisor,MethodInterceptor,Advice  但是最终都会包装成Advisor

使用事务注解@Transaction来使用事务,相对于<aop:config>配置事务来说没增加多少配置,但是控制粒度更加细也方便修改
<tx:annotation-driven transaction-manager="transactionManager" order="2"/>






## spring依赖属性注入是调用set方法(Bean WriteMethod),如果在Bean difinition的property map中包含属性键值对,即使没有对应属性名字也会调用set方法