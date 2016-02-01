---
title: spring核心接口
date: 2016-01-28 15:51:18
tags: spring
---

### BeanFactory
	最顶层的接口,也是最简单的接口,但是一般都不直接使用该接口,根据接口隔离原则,增加了很多子接口
	用来扩展接口的功能,没个子接口都定义了一些符合接口名的功能,我们一般通过使用ApplicattonContex
	来使用spring提供的功能.

### BeanDefinition
	spring ioc容器管理的实际是BeanDefinition,包含了class,name,autowire-mode,init-method
	constructor arguments,property等属性这些属性让spring容器知道该如何创建bean,以及管理bean

### InitializingBean
	当某个bean的属性被设置后,spring会调用实现了该接口的bean的afterPropertiesSet(), 用来完成一些
	初始化工作,或者一些检查工作..., 当然在bean的生命周期中,bean初始化也可以使用init-method来完成

### BeanPostProcessor
	Factory hook that allows for custom modification of new bean instances,
    e.g. checking for marker interfaces or wrapping them with proxies.
	
	ApplicationContexts can autodetect BeanPostProcessor beans in their
    bean definitions and apply them to any beans subsequently created.
	Plain bean factories allow for programmatic registration of post-processors,
	applying to all beans created through this factory.
	spring的注释最清楚,该接口就是个工厂钩子,实现了该接口的bean,spring容器通过检查接口,可以发现该类bean是特殊
	bean,然后在实例化bean先调用一个方法,实例化完后再调用一个方法,会自动调用该类特殊bean来处理新创建的bean,
	比如创建代理...,这里需要注意postProcessBeforeInitialization,是在InitializingBean钩子方法,以及init-method
	之前,实例化后置方法在那两个方法之后

### FactoryBean
	实现给接口的bean可以自定义创建返回的bean,当从spring容器中获取bean的时候,spring会检查bean定义,如果bean实现了
	该接口,spring调用getObject()返回bean实例,有点像工厂方法,但是这里可以进行一些编码操作,比如ibatis返回mapper
	就使用了FactoryBean,虽然bean定义加载时候是加载接口,但是可以修改具体的className,实例化就用FactoryBean来创建
	,这就是一种使用场景


### BeanFactoryPostProcessor
	A BeanFactoryPostProcessor may interact with and modify bean definitions
	api doc写到该接口主要是来更新bean定义的,但是不会实例化bean,回调钩子方法时,所有
	bean都加载了,但是都还没有实例化
	比如PropertyPlaceholderConfigurer就是实现该了接口,用于替换属性中的${name},在
	加载bean配置文件后,创建了一个bean definition, 但是里面的${name}还没有替换,然后
	bean都加载完了,spring容器回调实现了该bean的实例来处理${name}

### ApplicationContext
	开发中一般都使用该接口,不但具有beanFactory的功能,还有resource,message...

### Ordered
	bean实例化优先级,值小的优先级高,值大得优先级低,控制bean实例化顺序时用到

## 小结
	spring是一个对象容器,在加载,实例化时都提供了很多扩展方法,使用户代码可以参与到定制
	bean的过程中去,一般都是通过实现特定接口,spring容器回调接口完成一些扩展,这些扩展要么
	用户提供,要么spring自带的(比如${name}替换, AOP)
	
	spring源代码中用了很多设计模式,为了实现开闭原则,代码复杂度也提高了,各种类包装只为
	更好的扩展性,比如当看到方法调用的时候还new className(args)之类的,一般这里都是接受
	接口类型的参数,如果你需要加自定义操作就可以用自己的实现来调用,也许这就是作为一个通用
	框架必须做的事情吧,需求太多...
	


	

	
	
