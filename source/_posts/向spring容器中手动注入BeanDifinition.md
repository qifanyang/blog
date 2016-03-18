---
title: 向spring容器中手动注入BeanDifinition
date: 2016-02-02 09:50:54
tags: spring
---

### 硬编码注入对象

	有时候在某些情况下需要硬编码注入对象可以使用DefaultListableBeanFactory.registerSingleton(),DefaultListableBeanFactory
	可以通过applicationContex.getBeanFactory()或得,注册实例对象,DefaultListableBeanFactory.autowireBean()完成对象依赖的注入.
	
	因为硬编码注入的是对象,所以没有Bean difinition,当要autowireBean的时候,就需要构造一个默认BeanDifinition
	并注册到spring容器中




### ibatis注入mapper实现

	在spring中使用ibatis时, service层需要注入ibatis mapper, @Autowired private AccountMapper accountMapper;
	而在ibatis用法当中mapper一般都是接口,spring自动扫描机制默认是不会加载接口作为bean difinition的,所以就需要自己
	实现mapper扫描并注册到spring容器中
	
	MapperScannerConfigurer完成该功能,该类继承BeanDefinitionRegistryPostProcessor,而BeanDefinitionRegistryPostProcessor
	是一个BeanFactoryPostProcessor,可以完成向spring容器中注入bean difiniton的功能

	ClassPathMapperScanner完成ibatis mapper的扫描,该类继承ClassPathBeanDefinitionScanner,新加了过滤条件,可以加载接口
	重写了doScan()方法,修改了返回Bean difinition的className为MapperFactoryBean,也新加了几个属性,mapperInterface值为
	mapper的className,spring可以自动根据类名去查找class并注入, 扫描完成后每个mapper都有一个bean difinition,而且bean difinition
	都是Autowired byType,spring容器会根据类型在spring容器中去查找依赖完成属性注入,当在service层注入mapper的时候,实例化bean并返回,
	因为bean difinition的BeanClass是一个FactoryBean,调用getObject(),然后到这里和单独使用ibatis就差不多了


	
### 自定义一个Bean完成bean difinition注册
	继承BeanDefinitionRegistryPostProcessor,实现接口方法,结合ClassPathBeanDefinitionScanner扫描bean difinition可以完成bean difinition
	注册,实现可参考ibatis MapperScannerConfigurer
	
	