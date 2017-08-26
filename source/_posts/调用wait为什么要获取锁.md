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

2.wait方法的作用?  
调用对象的wait方法,导致当前对象被jvm调度,放弃cpu控制权,并且释放对象锁    

3.wait方法需要获取对象锁么,为什么一定要获取对象锁呢?
菜鸟:如果当前线程如果没有获取对象锁,调用对象wait方法的对象锁会抛出IllegalMonitorStateException   

wait,notify,notifyAll --->简写WNN,
老鸟:应该从wait工作方式来分析,Object对象提供WNN三个方法用于线程之间协调执行,  
比如一个方法不允许并发执行,一个线程在执行其他线程就不能执行,java通过对象锁(可以理解为多线程共享内存通信)  
来实现,如果某个线程获取到了对象锁就可以执行该方法或代码块,没获取到锁的就需要加入对象所的waitset中,当获  
取到锁的线程执行方法完毕,释放锁并唤醒所有在waitset中的线程.当使用synchronized修饰方法或代码块时,就是  
这种工作方式.其中获取锁/检查对象锁状态/挂起没获取到对象锁的线程/唤起被挂起的线程,这些操作都是jvm完成的  
    synchronized可以理解为检查一个状态来决定当前线程是否继续执行/是否被挂起.如果这个状态不是对象锁?  
使用普通变量代替对象锁并结合WNN就可以实现可编程synchronized,(注:ReentrantLock比WNN方便)  
    实现synchronized语义:
    1.检查对象锁状态并获取, 检查一个变量状态(注:取名锁状态)并决定挂起当前线程/继续执行
    2.假如已经存在一个执行线程/一个被挂起线程  
    3.持有锁的线程执行完毕,将要执行synchronized退出同步块的语义,修改锁状态/唤醒被挂起的线程  
    被唤起的线程会再次去获取对象锁.被挂起的线程会再去检查锁状态

    分析1:如果检查状态和挂起线程两步操作被打断,中间执行notifyAll,那么被挂起的线程将丢失唤醒信号,  
    所以检查状态和挂起线程需要原子执行.要让原子执行可以用synchronized,WNN不就是为了实现可编程  
    synchronized的么?但没办法,使用WNN至少锁状态不再是对象锁,可以是应用程序中有业务含义的状态  
    所以称为可编程锁状态更为合适  

    
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
    this.notifyAll(); //d
}

//如果线程1不用获取锁直接执行a,这时候线程2执行c,d,线程1执行wait, 那么线程1就失去了这次被唤醒的机会  
//如果a b执行时获取了锁,那么 c d如果也去获取同一个锁就需要等待该锁,然后 d唤醒 b, 就不会丢失唤醒信号

~~~  
 
