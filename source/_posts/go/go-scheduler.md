## go调度
  go采用的M:N线程模型,不像java每个线程对应一个OSThread,go可以把任意数量的gorutines使用任意数量的  OSThreds运行,好处是可以快速的context switch和充分利用多核的性能.缺点是增加了调度器的复杂性  

## M:N线程模型的问题
  在看深入JVM线程模型时,M:N模型如果进行系统调用会阻塞当前线程,当使用java线程池时,如果有阻塞任务  
那么会把线程阻塞.因为java线程池使用的是一个全局任务队列,所以不会影响其它任务执行  

## go如何解决M:N的问题
  go调度器抽象出了3个主要实体:
  M -> 代表OSThread, machine简写
  G -> 代表goroutines, 包含了栈,指令指针和其它用于调度的信息
  P -> 代表调度的contexts,它可以被看做一个局部的单线程调度器,它是实现从N:1调度器到M:N调度器的重要  
  组成部分,P是Process简写  

  ![](/img/in-motion.jpg)  

  上图代表了2 threads(M), 每个线程包含一个context (P),每个线程运行一个goroutine(G),为了运行  
  goroutine,一个线程必须持有一个context  

  灰色的goroutines没有运行,但是准备好被调度,它们被放到一个叫做runqueues的调度队列中,当执行go  
  statement的时候,goroutine被加入到runqueues的尾部,等待调度  

  为了降低互斥冲突,每个context都有自己的runqueue,不像java线程池共享一个任务队列.在多核(32)竞争  
  会很激烈,会经常等待unlocked,所以java经常采用多个线程池执行不同的任务,叫做资源隔离,一方面也减少  
  全局任务队列的竞争

  go调度器一般情况下都像上图那样调度goroutine,但也会有些特殊情况,如下:

### syscall
  当goroutine执行syscall,导致block,也会导致当前执行goroutine的OSThread block,调度器会将当前  
  M的P交给另一个M执行, go调度器保证有足够的OSThread运行context. go调度器可能创建新的线程来执行  
  被block的M的context,也有可能从Thread Cache中获取空闲的OSThread. 

  java中如果每个线程都有自己的任务队列,相当于把任务队列交个其它线程执行.如果直接执行系统调用那么线程  
  会直接被阻塞,无法自己主动交出任务队列.可以通过类似AOP,在执行阻塞操作之前先将任务队列交出去,java协程  
  库可能就这么实现,如果java线程池实现任务调度,然后每个runnable对象就类似goroutine

  当syscall返回时,被挂起的M为了运行挂起的goroutine会尝试获取context,如果获取失败会将goroutine放入  
  全局的runqueue,并且把当前线程放入Thread Cache中.
  其它的context会从定时检查global runqueue获取goroutine运行,所以即使GOMAXPROCS值为1,当进行系统  
  调用时,go程序是多线程的.

### stealing work
  当每个context的runqueue不平衡的时候,运行完毕的context会从其它context窃取goroutine运行,充分利用  
机器多核性能  
  java中的fork join,当自己的队列运行完毕,会从其它任务队列中窃取任务执行,这样最大效率利用多核cpu,不过  
如果本身cpu已经比较繁忙,窃取任务执行也可能不会有效率提升.就如同你明确知道一个人没事做然后分配事情给他做  
这样效率才会提高,如果一个人比较繁忙就算他主动申请做更过的事,结果也会影响他做其它的事,干活的人只有这么多  
cpu都是不停歇的工作,所以stealing work需要有一定的应用场景.
  就如同java toturial中讲到,Fork/Join的目的是使用所有可用的处理器增强程序的性能.比如4核cpu就如同4  
个人,有的人完成任务快一点,然后就没事做了,这时再去主动帮助别人做事.整体就可以做得更多.程序整体性能就提高  

## 链接

http://morsmachine.dk/go-scheduler 
https://docs.oracle.com/javase/tutorial/essential/concurrency/forkjoin.html 
http://gee.cs.oswego.edu/dl/papers/fj.pdf 
