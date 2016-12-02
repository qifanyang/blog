---
title: javabeans set thread-safe
date: 2016-11-18
tags: effective-java
---

## 概要
在effective-java中第一张讲静态工厂方法时提到,基于javabeans set模式创建对象,会存在对象状态不一致问题,想下自己经常都是这么使用的,  
在方法内部创建的对象只要其它线程不可访问就不会存在javabean状态不一致问题,书上估计是详说如果通过javabeans set模式创建的对象,  
对外发布,让其它线程可以访问,set方法没有同步,会存在状态不一致.   

在开发业务时,比如存储一个数据对象到数据库中,先构造对象,然后设置属性,然后save到数据库中,这几个方法  
执行都在一个线程中,所以不会存在对象状态不一致问题.  

但是如果把一个javabean对象通过方法调用传递到外面

## 避免线程安全问题:  
1. 创建无状态对象  
2. 对象属性为常量  
3. 修改对象状态时进行必要的同步    

基于javabeans set模式的对象创建因为不满足第3条,所以在多线程程序中会存在状态不一致问题  


## 对象状态不一致


## race conditions

Race conditions are timing-dependent program flow scenarios that can lead to state (program data) corruption  
竞态条件是程序执行流程依赖时间的一种场景, 可能会导致对象状态被破坏,典型就是多线程同时修改一个对象的状态,如果没有同步  
则该对象可能处于不一致状态.数据库也有不一致状态,比如把A的钱转到B上,如果把A的钱减去了但是B没有加上,就叫数据不一致  

~~~
public void setSpot(Point point) {                 // 'spot' setter
    this.x = point.x;
    this.y = point.y;
}
~~~  

上面的方法在多线程中调用,线程T1执行this.x=11时,如果被操作系统调度,线程T2进来执行this.x=4,然后T1恢复执行,返回的对象  
状态就会存在不一致问题,这就是javabean set方式会造成对象状态不一致的原因, 所以基于javabeans set模式创建对象一定不能  
在多线程中使用,会出现对象状态不一直问题,如果一定要用必须要进行同步  

### note:
java基本数据类型赋值是原子的,但是long和double不是,要保证long和double也是原子的需要加上volatile关键字  









http://www.javaworld.com/article/2077029/java-concurrency/javabeans--properties--events--and-thread-safety.html






