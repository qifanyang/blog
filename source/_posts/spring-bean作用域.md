---
title: spring bean作用域
date: 2016-02-02 16:20:48
tags: spring
---

### scope 作用域
	
	默认为singleton(单例),prototype(每次都创建),[requeset,session,application](web相关)
	重点为singleton和prototype


### singleton依赖prototype
	因singleton的bean在spring容器加载后会先实例化,依赖的prototype bean也需要注入,所以会导致
	prototype bean也实例化,但是这只在实例化singleton bean时发生一次,调用单例bean上的方法返回
	的prototype都是同一个

	实现singleton bean上实现返回prototype bean每次都是新的对象
	a. singleton bean 实现ApplicationContextAware,在返回prototype bean方法内部调用applicationContex.getBean()
		但是这样子对spring api产生依赖

	b. 将单例类抽象,返回bean的方法为抽象方法, 定义bean的时候使用<lookup-method>指定抽象方法包含返回值,spring会使用
		cglib实现该类,重写抽象方法,根据返回值从当前spring容器中查找bean

	c. 方法替换,自己实现一个方法替换器,接口MethodReplacer,在需要替换bean定义的地方使用<replaced-method>指定要被替换的方法
### singleton依赖singleton

### prototype依赖prototype
	每次都会创建新的对象,所依赖的对象也是每次都创建

	
### prototype依赖singleton
