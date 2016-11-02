---
title: threadLocal用法
date: 2016-01-26 11:03:01
tags: java
---

# 介绍
ThreadLocal是一个很少人知道但比较核心的java对象,应用相当广泛,在多线程程序中大量用到实现绑定一个对象
到当前线程,类似一条流水线,各个环节可以查看流水线上的进度然后选择是否该自己作业了,在程序中的效果就是,
不用在方法之间传送某个值,在用的时候从当前线程取就是了,当然使用全局变量也可以实现该效果,但是在多线程程
序中,多线程访问共享变量需要同步,所以效率不高

也可以使用局部变量避免线程安全问题,但是这样子每次都要重复创建对象,如果创建对象的开销比较大,那么就不可
取了,这时候就要使用ThreadLocal了.

当使用了线程池时,线程使用完了要归还到线程池,这时候要注意了,可能需要清除线程局部变量,这时候,ThreadLocal
的作用就是在方法调用链上扮演一个线程内部的全局变量,而且还要不停的重置

ThreadLocal局部变量的生命周期和线程一样,而且使用了弱引用,当线程挂掉了,线程对象被回收了,只有弱引用指向
线程局部变量,垃圾回收是可以回收该对象的.

## 用法
	private static ThreadLocal<InterestService> instance = new ThreadLocal() {

    @Override

    protected InterestService initialValue() {

      return new InterestService();

    }
  	}

	一般使用静态变量引用ThreadLocal,使用initialValue()或set()方法来完成初始化


## ThreadLocalMap
1.线程局部变量存储在Thread中的一个map中,一个线程可以存储多个线程局部变量,访问方式就是通过ThreadLocal来作为key,如果要存储多个变量到一个线程中  
那么就要创建多个ThreadLocal实例来或得多个key,然后通过该值来获取线程局部变量  

2.ThreadLocalMap使用weakRefrence来作为map的entry,当只有弱引用指向对象时,该对象可被回收,也就是ThreadLocal被弱引用包装(被弱引用指向),如果外部没有强引用指向  
那么Thread.ThreadLocalMap中使用对应ThreadLocal作为key的value将被回收,防止ThreadLocal做为局部变量放入线程本地变量,然后没办法移除,导致线程泄露  

还有一种情况,线程退出,那么会内存泄露么?线程局部变量是存放在Thread.ThreadLocalMap中,当Thread退出,那么该Thread是可以被回收的,那么该Thread中的threadLocalMap  
就是可以被回收的(可达性分析),map中的对象再进行可达性分析,key是ThreadLocal,如果没有其它强引用指向(一般是全局静态变量),那么可以回收. value是线程局部变量值,  
一般外部没有强引用指向,所以该value是可以被回收  


3.在访问ThreadLocal get,set时,会自动去检查ThreadLocalMap的弱引用为被垃圾回收没有,如果被回收了,那么将线程局部变量value=null,让其可以被垃圾回收

## tips
1.软引用,当内存不足时会被垃圾回收,用于做缓存  
2.弱引用,当只有弱引用存在,就会被垃圾回收  
3.引用队列,当垃圾回收时,会被放入refrence queue, 还有机会被重新使用,告诉GC不要回收  



http://blog.smartbear.com/programming/how-and-when-to-use-javas-threadlocal-object/