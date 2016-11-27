## 面试问题
1.Object中有哪些方法?  
- clone
- equals
- toString
- wait
- notify
- notifyall
- hashCode
- getClass

2.ok,wait方法的作用?  
调用对象的wait方法,导致当前对象被挂起,当前线程不会持有对象的锁  

3.wait方法需要获取对象锁么,为什么一定要获取对象锁呢?
a.jvm本地方法判断了,当前线程如果没有获取调用wait方法的对象锁会抛出IllegalMonitorStateException,  
因为wait和notify/notifyall是用来线程之间同步/协调,也就是让线程之间互相协调执行  

b.如果调用wait时不用获取锁,那么随时都可以执行wait方法,而且一般都是使用一个条件判断然后进入wait,如果程序  
更新这个条件(所有线程都不要再执行wait了),然后执行notifyall,如果这时候线程依然调用wait(因为不用获取锁)将会  
导致这个线程永远不会被唤醒,有的任务永远也不可能完成,也造成内存泄漏     
~~~
List list ...
private T  take(){
    while(list.isEmpty()){ //a
        this.wait(); //b
        ...
    }

        return list.get();
    }

}

private void put(T t){
    list.add(t);// c
    this.notifyall(); //d
}

//如果线程1不用获取锁直接执行a,这时候线程2执行c,d,线程1执行wait, 那么线程1就失去了这次被唤醒的机会  
//如果a b执行时获取了锁,那么 c d就会需要等待该锁,然后 d唤醒 b, 就不会丢失泛型信号

//wait / notify 这种编程模式必须获取锁才能正确实现程序功能,没准备好则等待,准备好则通知我
~~~  

c.如果调用wait不用获取锁,那么对象的wait set会存在并发访问,那么jvm需要支持实现并发的wait set数据结构,也增加  
了复杂度,底层越简单越好才是最好的,就算采用非阻塞算法/无锁算法实现也会存在一定开销  
