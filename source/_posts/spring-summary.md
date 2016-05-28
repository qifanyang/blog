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
parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
parseConstructorArgElements(ele, bd);
parsePropertyElements(ele, bd);
parseQualifierElements(ele, bd);
然后注册bean difinition到 BeanDifinitionRegistry中, 实现类是DefaultListableBeanFactory, 用beanDefinitionMap存储


2.扫描包加载
<context:component-scan base-package="com.test"> //会扫描指定的包名下的所有类,包含子目录下的类(通配符 **/*.class),使用类ClassPathBeanDefinitionScanner实现
包扫描返回的bean difinition类是ScannedGenericBeanDefinition, 因为没有xml定义,需要知道类名字,是否抽象,父接口等需要信息,spring使用asm classReader来访问字节码
使用了访问者模式获取类信息,遍历方法,字段等
扫描包的时候,通过filter决定是否加载class为bean difinition, 默认会添加AnnotationTypeFilter(Component.class),当使用Controller注解也会加载,因为Controller注解使用了Componet作为元注解,还可以使用(javax.inject.Named),所以必须要使用Componet注解    
扫描加载了所有bean difinition之后,读取@Scope,@ScopedProxyMode来配置,没有则采用默认配置,然后确定bean名字,默认类首字母小写
然后设置beanDefinitionDefaults (在<beans ...> 中设置的default属性),在经历了和加载xml类似的过程之后,注册创建好的bean difinition到registry中,就完成了bean difinition扫描


3.java编码配置




## bean实例化
bean根据className或classType创建实例,并且需要解决依赖关系,2016-5-29分析下依赖如何解决...




