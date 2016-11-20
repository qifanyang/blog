---
title: volatile与synchronized
date: 2016-11-20 11:03:01
tags: java
---

# 简介
多线程程序为了线程安全,引入互斥和可见性  

### volatile
被volatile修饰的基本变量,赋值时需要刷新结果到内存中,或者通知其它cpu cahce,或刷新store buffer或invalid queue    
这样其它线程就可以看见最新修改结果. 

volatile 读操作开销非常低 —— 几乎和非 volatile 读操作一样,具体要看jvm实现读取volatile变量加了什么内存栅栏指令  

### synchronized
synchronized为内置锁,实现互斥和可见性,当synchronized块退出时要保证下一个进入synchronized块能看到上一个退出所  
做的更改,该方式开销大,会带来伸缩性问题  

多线程程序为了正确读取一个有状态对象的值,读写方法都要加synchronized关键字
~~~
@ThreadSafe
public class Counter {
    private  int value;

    public synchronized int getValue() { return value; }

    public synchronized int increment() {
        return value++;
    }
}
~~~

结合synchronized和volatile实现,读方法不使用synchronized,声明变量时使用volatile    

~~~
@ThreadSafe
public class CheesyCounter {
    // Employs the cheap read-write lock trick
    // All mutative operations MUST be done with the 'this' lock held
    @GuardedBy("this") private volatile int value;

    public int getValue() { return value; }

    public synchronized int increment() {
        return value++;
    }
}
~~~

AtomicInteger无锁实现计数  
该方式使用cpu提供的CAS指令直线,线程虽然没有互斥,但是CPU一直在执行轮询,当并发很大时,可能造成失败次数很多,大部分线程得不能  
执行成功,CPU一直居高不下,可能吞吐量还不如synchronied实现的方式,至少互斥让cpu空出来去做其它的事情,而不是浪费在轮询上面  

如果要必要两种效率,真不好比较,因为synchronized有自旋锁,也就是当互斥时其并没有被挂起,而是也在做轮询   

## java bean线程安全
在effective java中提到java bean set模式有状态不一致问题  
在ibm developer上看到一篇文章:  

缺乏同步会导致无法实现可见性，这使得确定何时写入对象引用而不是原语值变得更加困难。在缺乏同步的情况下，可能会遇到某个对象引用的更新值
（由另一个线程写入）和该对象状态的旧值同时存在。（这就是造成著名的双重检查锁定（double-checked-locking）问题的根源，  
其中对象引用在没有同步的情况下进行读操作，产生的问题是您可能会看到一个更新的引用，但是仍然会通过该引用看到不完全构造的对象）。  
实现安全发布对象的一种技术就是将对象引用定义为 volatile 类型。  
清单 3 展示了一个示例，其中后台线程在启动阶段从数据库加载一些数据。其他代码在能够利用这些数据时，在使用之前将检查这些数据是否曾经发布过。  
~~~
public class BackgroundFloobleLoader {
    public volatile Flooble theFlooble;

    public void initInBackground() {
        // do lots of stuff
        theFlooble = new Flooble();  // this is the only write to theFlooble
    }
}

public class SomeOtherClass {
    public void doWork() {
        while (true) { 
            // do some stuff...
            // use the Flooble, but only if it is ready
            if (floobleLoader.theFlooble != null) 
                doSomething(floobleLoader.theFlooble);
        }
    }
}
~~~
这里可以应用happend-before,只要floobleLoader.theFlooble != null,说明初始化过,那么加载的数据就是可见的  

在java bean set模式中,如果bean被另外一个线程访问,因为没有同步,另外一个线程可能无法看见最新的bean属性值,  
当然在一个线程中,bean不会有状态不一致问题,即使另一个线程能看见新的引用,但是可能看不见最新的值,而且可能是  
部分看得见,部分看不见.java并发编程中讲解了很多    













http://www.ibm.com/developerworks/cn/java/j-jtp06197.html



