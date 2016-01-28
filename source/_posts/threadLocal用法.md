---
title: threadLocal用法
date: 2016-01-26 11:03:01
tags: java核心
---

# 介绍
	ThreadLocal是一个很少人知道但比较核心的java对象,应用相当广泛,在多线程程序中大量用到
	实现绑定一个对象到当前线程,类似一条流水线,各个环节可以查看流水线上的进度然后选择是否该
	自己作业了,在程序中的效果就是,不用在方法之间传送某个值,在用的时候从当前线程取就是了,当然
	使用全局变量也可以实现该效果,但是在多线程程序中,多线程访问共享变量需要同步,所以效率不高

	也可以使用局部变量避免线程安全问题,但是这样子每次都要重复创建对象,如果创建对象的开销比较大,那么就不可取了,这时候就要使用ThreadLocal了.

	当使用了线程池时,线程使用完了要归还到线程池,这时候要注意了,可能需要清楚线程局部变量,这
	时候,ThreadLocal的作用就是在方法调用链上扮演一个线程内部的全局变量,而且还要不停的重置

	ThreadLocal局部变量的生命周期和线程一样,而且使用了弱引用,当线程挂掉了,线程对象被回收
	了,只有弱引用指向线程局部变量,垃圾回收是可以回收该对象的.

# 用法

	private static ThreadLocal<InterestService> instance = new ThreadLocal() {

    @Override

    protected InterestService initialValue() {

      return new InterestService();

    }
  	}

	一般使用静态变量引用ThreadLocal,使用initialValue()或set()方法来完成初始化










http://blog.smartbear.com/programming/how-and-when-to-use-javas-threadlocal-object/