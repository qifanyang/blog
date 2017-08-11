---
title: threadLocal用法
date: 2016-01-26 11:03:01
tags: java
---

# 介绍
    ThreadLocal通常叫线程局部变量,认为换种叫法更好理解:线程内的全局变量,意思就是在一个  
线程中可以像访问全局变量一样访问      

##用法
~~~
public class Service{
	private static ThreadLocal<InterestService> instance = new ThreadLocal() {

    @Override
    protected InterestService initialValue() {
      return new InterestService();
    }
  }

  public void businessLogic(){
    InterestService service = instance.get();
  }
 
}
~~~

##相关类和方法
ThreadLocal.入口,创建实例后,使用set(),get()即可  
ThreadLocal.ThreadLocalMap 用于存储线程本地变量的map,不同于HashMap,采用开放地址法解决冲突     
Thread 线程,包含属性ThreadLocalMap实例,ThreadLocal的set和get从这里获取值     
Thread.currentThread() 本地方法,获取当前线程  
WeakRefrence ThreadLocalMap中Entry继承该类,创建指向ThreadLocal的弱引用    

##如果采用全局map实现
1.多线程访问需要同步,高并发同步开销大,因为没有线程间通信所以不需要全局map    
2.线程异常退出无法从map中移除造成内存泄漏,当然开个定时器检查线程是否存活也可以避免内存泄漏   

##ThreadLocal引入
既要避免线程间通信开销,又要实现全局变量的效果,于是就产生了线程内的全局变量,方法中任意地方可以随时  
访问,不用担心线程安全问题  


## ThreadLocalMap
1.线程局部变量存储在Thread中的一个map中,一个线程可以存储多个线程局部变量,访问方式就是通过ThreadLocal来作为key,如果要存储多个变量到一个线程中,那么就要创建多个ThreadLocal实例来或得多个key,然后通过该值来  
获取线程局部变量  

2.ThreadLocalMap使用weakRefrence包装作为key的ThreadLocal.  

##线程退出,那么会内存泄露么?
测试代码,需要设置虚拟机内存 -Xms256m -Xmx256m
~~~
public class WeakRefrenceTest{
    public static void main(String[] args) throws InterruptedException {

        showFreeMemory("初始化空闲内存");

        Service service = new Service();

        WorkThread workThread = new WorkThread(service);
        workThread.start();

        System.out.println("主线程暂停3s,等待工作线程执行");
        Thread.sleep(3000L);

        showFreeMemory("工作线程存储大数据对象后空闲内存");

        workThread.interrupt();//这一步表示线程异常退出
//      workThread = null;//线程异常退出需要将线程引用置null, gc工作.线程池会移除线程引用
        System.gc();

        showFreeMemory("线程异常退出后空闲内存");
        Thread.sleep(3000L);
        System.gc();
        showFreeMemory("暂停3s,再次gc后空闲内存");
    }

    /**
     * 在workThread中执行的service
     */
    static class Service{
        private static ThreadLocal<byte[]> key = new ThreadLocal(){
            @Override
            protected byte[] initialValue() {
                int size = 100*1024*1024;
                return new byte[size];
            }
        };

        public void doSomeThing(){
            byte[] bytes = key.get();
            System.out.println(bytes.length);
        }

    }

    /**
     * 工作线程,存储一个测试大数据对象
     */
    static class WorkThread extends Thread{
        private Service service;

        public WorkThread(Service service){
            this.service = service;
        }

        @Override
        public void run() {
            while (true){
                try {
                    service.doSomeThing();
                    sleep(100000000L);
                } catch (InterruptedException e) {
                    throw new RuntimeException("test");
                }
            }
        }
    }


    static void showFreeMemory(String desc){

        System.out.println(desc + " : " + Runtime.getRuntime().freeMemory()/1024/1024+"M");
    }
}
~~~ 

输出  
~~~
初始化空闲内存 : 239M
主线程暂停3s,等待工作线程执行
工作线程存储大数据对象后空闲内存 : 137M
线程异常退出后空闲内存 : 143M
暂停3s,再次gc后空闲内存 : 143M
~~~
当调用Thread.interrupt()之后,睡眠的线程被打断,然后抛出运行时异常,线程退出.  
存储在ThreadLocal中的大对象不gc回收,但是main方法中仍然持有workThread引用  
按可达性分析ThreadLocal中的ThreadLocalMap是可以到达,所以不可以回收?  

debug分析,当调用了Thread.interrupt()之后,workThread中的threadLocals指向null  
线程退出,jvm自动修改了workThread对象内容?  
查看threadLocals访问的地方,果然Thread.exit()方法会在线程退出时,被jvm调用  
~~~
/**
     * This method is called by the system to give a Thread
     * a chance to clean up before it actually exits.
     */
    private void exit() {
        if (group != null) {
            group.threadTerminated(this);
            group = null;
        }
        /* Aggressively null out all reference fields: see bug 4006245 */
        target = null;
        /* Speed the release of some of these resources */
        threadLocals = null;
        inheritableThreadLocals = null;
        inheritedAccessControlContext = null;
        blocker = null;
        uncaughtExceptionHandler = null;
    }
~~~
threadLocals = null;所以gc时,可以回收掉放在线程局部变量内容  

##ThreadLocal为什么要包装成弱引用?
如果有100线程局部变量,当要从100线程中移除,该怎么做?  
可以在每个线程的业务代码中调用threadLocal.remove();但是没法获取每个线程  
如果是有个全局变量存储了所有线程,在每个线程中执行threadLocal.remove()就可以移除全部.  

这样做引入新问题,如何维护全局变量,如果线程异常退出,全局变量中的thread可能就没法移除了,当然可以  
使用LRU避免部分问题  

从以上分析要移除存储在Thread中的局部变量会有很多问题  
java的解决方法是能够访问线程就显示threadLocal.remove()移除,不能访问到线程那么可以使用弱引用  
将ThreadLocal指向null,那么当ThreadLocal被回收后,其它访问ThreadLocalMap会检测ThreadLocal  
被gc回收则移除线程局部变量  

问题:如果没有访问线程局部变量那么会存在内存泄漏
  
##应用
spring事务控制  
aop开启事务,将事务对象放入ThreadLocal中,方法调用时根据事务传播属性新建/挂起/加入 事务等等  

框架集成  
不用侵入业务代码,结合aop可以透明化实现很多功能,例如spring事务控制  

## tips
1.软引用,当内存不足时会被垃圾回收,用于做缓存  
2.弱引用,当只有弱引用存在,就会被垃圾回收  
3.引用队列,当垃圾回收时,会被放入refrence queue, 还有机会被重新使用,告诉GC不要回收  
4.WeakHashMap, Entry继承WeakRefrence,put时key会被作为WeakRefrence,如果是HashMap如果不手动remove那么value一直存在,而对于WeakRefrence如果没有其它地方使用key  
(没有强引用),那么该key就可以被回收,何时回收value呢?在get,put时都会自动去检查key是否被回收,如果回收了则回收value和entry,缺点是要等到手动调用get,set时才会回收value  



http://blog.smartbear.com/programming/how-and-when-to-use-javas-threadlocal-object/