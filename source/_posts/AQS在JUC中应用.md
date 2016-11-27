## 简介
AQS是AbstractQueuedSynchronizer的简称,实现了内置锁wait set和monitor的功能  
AQS中的state字段代表了monitor功能,而且可以存在多种状态  
AQS中的Node链表实现了wait set的功能,该链表是线程安全的非阻塞数据结构  

## AQS api

### 1.acquire获取
~~~
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
~~~
类似进入synchronized方法,要先获取锁,这里要去检查AQS state状态是否为0,为0则获取成功,不为0这查看同步器被哪个  
线程拥有,是当前线程的话那么state++,实现了可重入.如果被其它线程获取那么这次获取锁失败,进入等待队列,tryAcquire  
是个抽象方法,这是子类ReentrantLock的实现    

tryAcquire有哪些实现  
- ReentrantLock
- ReentrantReaderWriteLock
- ThreadPoolExecutor.Worker  


acquire会被并发调用,所以这个状态检查和设置必须原子实现, 等待队列也要线程安全  

## 非阻塞算法
一个线程挂起或者失败不会导致其它线程也挂起或者失败,那么这个算法是非阻塞算法  
如果在算法的每个步骤都有一个线程能够执行下去,那么这个算法是无锁算法  

AQS中的入队列:  
~~~
 private Node enq(final Node node) {
        for (;;) {
            //初始化时,tail,head都为null,
            Node t = tail; 
            if (t == null) { // Must initialize   1
                if (compareAndSetHead(new Node())) 2
                    //初始化成功,把head和tail都设置为一个head,只有一个线程可以进入这里
                    //如果其它线程块执行到2了,会再次读取tail,然后就不会进入这里
                    tail = head;
            } else {
                //新插入的node应该在node之后,这里只会原子的在tail后添加node
                //如果在添加的时候被其它线程修改了,for循环会再次读取最新的tail,然后将node添加到tial后面
                //直到成功添加
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
~~~

AQS出队列:    
当调用release时,如果需要调度那么执行unparkSuccessor,这里只唤醒一个Node  
当有多个Node在wait set中时,doReleaseShared没有遍历head链表,而是只操作head,被唤醒的线程负责修改head的next  
doReleaseShared循环读取hean的next然后unpark it. 直到tail被移动到head为止  
doReleaseShared机制好蛋疼    
~~~
 private void unparkSuccessor(Node node) {
        //这里node一般是head 
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;//有等待node,那么s!=null, 如果有取消那么ws=1,需要遍历队列,更新s为没有被取消过的node
        //为什么要从尾部开始遍历,因为进入是head的下一个node已经被取消了,所以从tail开始了么?
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);//告诉虚拟机可以调度该线程了
    }
~~~


