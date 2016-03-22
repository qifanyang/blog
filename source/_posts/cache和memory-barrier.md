---
title: cache和memory barriers
date: 2016-03-19 15:01:04
tags: java
---
## cache
出现原因:
cpu速度远远快于内存,cpu每次读取内存,都会花费几百个机器周期的时间等待内存返回数据,白白浪费了机器周期,故引入cache,读取花费的时间少但价格更高,所以cache没有多大
带来问题:
在多核cpu中,如果只有一个cache,所有的core都访问一个cache,竞争会非常激烈,所以每个core都会有自己的cache.如果同一个内存地址被映射到两个cache中,两个core同时修改改值,会导致cache不一致,这有点类似分布式系统中的一致性问题
解决办法:
引入cache后,cpu不在直接从内存中读取数据,而是从cache中读取,cache再从内存中读取数据,cache从内存中读取数据并不是一次读取一个字长,一般一次性读取多个字节,一次性读取数据的大小叫cache line,一般是是64bytes(局部性原理),在linux下可以从命令查询(cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size).
一致性协议(core coherency),比较出名的是MESI协议,在MESI协议中，每个Cache line有4个状态，可用2个bit表示
状态			描述
M(Modified)	这行数据有效，数据被修改了，和内存中的数据不一致，数据只存在于本Cache中。
E(Exclusive)这行数据有效，数据和内存中的数据一致，数据只存在于本Cache中。
S(Shared)	这行数据有效，数据和内存中的数据一致，数据存在于很多Cache中。
I(Invalid)	这行数据无效。
M和E都表示数据只在一个cache中,S和I表示数据在多个cache中.
因为cache line的存在,如果多个线程同时修改同一cache line中的变量,一致性协议将保证数据一致性,会频繁刷新cache数据到内存中然后再读取,效率低下,所以有的程序为了避免一致性协议带来的问题,把数据大小填充为cache line大小,这样不同线程更新变量时就不会导致其它cache line失效,disruptor就是这么做的

一致性协议产生的问题以及解决办法:
一致性协议虽然能保证cache一致性,但是MESI协议工作起来效率并不高,这是因为发生写操作往往会带来很长时间的等候：首先需要写的 CPU 需要让别的 CPU 将状态转换到 invalid，收到 response 以后才能进行实际的写，为此硬件专家使用了 store buffer（Sutter 同志也说过，modern CPU 如果没有 store buffer 就不值得买，可见这个 feature 对整体性能的影响是不可忽略的）
store buffer:
core0写一个在core1中的数据,core0让core1处于invalid,然后等待改操作完成再继续执行,效率低了,所以引入store buffer,core写入到store buffer就不管了,由store buffer来完成到内存的写入操作,cpu继续做原来的事情.

invalidate queue:
store buffer一般都很小,对于连续的写操作力不从心,core修改状态为S的变量时,先要invalid另一个core中变量,等待respone然后再修改,invalidate queue用来存在invalid指令,然后快速返回让发起invalid的core快速执行接下来的指令.

引入新的机制解决老的问题总会产生新的问题.

store buffer产生问题和解决办法:
当引入store buffer后,core0更新一个值,该更新被存在store buffer中,直到改值被写入到cache中,其它core是不知道该值的,所以store buffer破坏了内存一致性协议,在单线程情况下store buffer的存在并没有什么问题,但是在多线程情况下store buffer缓存了数据,内存一致性协议被破坏了,所以会造成数据被覆盖,然后引入了snoop,cpu读取数据会从store buffer和cache两处读取数据,当然store buffer的优先级更高,这样就可以读取到最新的值,

两个store buffer问题例子:
	a = 1;
	b = a + 1;
	assert(b == 2);
假设初始时a和b的值都是0，a处于CPU1-cache中，b处于CPU0-cache中。如果按照下面流程执行这段代码：
1 CPU0执行a=1; 
2 因为a在CPU1-cache中，所以CPU0发送一个read invalidate消息来占有数据 
3 CPU0将a存入store buffer 
4 CPU1接收到read invalidate消息，于是它传递cache-line，并从自己的cache中移出该cache-line 
5 CPU0开始执行b=a+1; 
6 CPU0接收到了CPU1传递来的cache-line，即“a=0” 
7 CPU0从cache中读取a的值，即“0” 
8 CPU0更新cache-line，将store buffer中的数据写入，即“a=1” 
9 CPU0使用读取到的a的值“0”，执行加1操作，并将结果“1”写入b（b在CPU0-cache中，所以直接进行） 
10 CPU0执行assert(b == 2); 失败
出现问题的原因是我们有两份”a”的拷贝，一份在cache-line中，一份在store buffer中。硬件设计师的解决办法是“store forwarding”，当执行load操作时，会同时从cache和store buffer里读取。也就是说，当进行一次load操作，如果store-buffer里有该数据，则CPU会从store-buffer里直接取出数 据，而不经过cache。因为“store forwarding”是硬件实现，我们并不需要太关心。

	void foo(void)
	{
		a = 1;
		b = 1;
	}

	void bar(void)
	{
		while (b == 0) continue;
		assert(a == 1);
	}
假设变量a在CPU1-cache中，b在CPU0-cache中。CPU0执行foo()，CPU1执行bar()，程序执行的顺序如下：
1 CPU0执行 a = 1; 因为a不在CPU0-cache中，所以CPU0将a的值放到store-buffer里，然后发送read invalidate消息
2 CPU1执行while(b == 0) continue; 但是因为b不再CPU1-cache中，所以它会发送一个read消息 
3 CPU0执行 b = 1;因为b在CPU0-cache中，所以直接存储b的值到store-buffer中 
4 CPU0收到 read 消息，于是它将更新过的b的cache-line传递给CPU1，并标记为shared 
5 CPU1接收到包含b的cache-line，并安装到自己的cache中 
6 CPU1现在可以继续执行while(b == 0) continue;了，因为b=1所以循环结束 
7 CPU1执行assert(a == 1);因为a本来就在CPU1-cache中，而且值为0，所以断言为假 
8 CPU1收到read invalidate消息，将并将包含a的cache-line传递给CPU0，然后标记cache-line为invalid。但是已经太晚了
就是说，可能出现这类情况，b已经赋值了，但是a还没有，所以出现了b = 1, a = 0的情况。对于这类问题，硬件设计者也爱莫能助，因为CPU无法知道变量之间的关联关系。
{这段话就是说的上面那个例子,不然真的看不懂
http://blog.csdn.net/demianmeng/article/details/22898079
我们知道程序员书写两个写操作的时候，隐含的假定是如果能观察到后一个写的结果，那么前一个写的结果势必也会发生，这是一个非常符合人直觉的行为，但是由于 store buffer 的存在，这个结论可能并不正确：这是因为如果观察线程位于另一个 core，首先读取后一个写（该地址并不在 cache 内）需要向写入线程所在 core 要对应地址的值，由于该 core 从 store buffer 返回了新值的时候这个 buffer 里面的写操作可能尚未发生，所以观察线程在获取了后一个写的最新结果时，前一个写的结果依然无法观察到(read invalidate来晚了)，这违背了 sequential consistency 的假定，往往程序员更倾向于这个 consistency model 下的 reasoning
}
store buffer的存在导致了上面的问题,core0顺序写两个变量a(在core1中),b(在core0中),core1读取b看见b最新值,core1在没收到core0 read invalidate a信号是直接读取本地a,
所以在core0中b写入发生了,但是a写入就像没法生一样,从代码上看是不可能的,但是cpu store buffer机制的存在导致了这种问题,所以引入了memory barriers.



## memory barrier
作用1:就是强制刷新store buffer到cache中,当然这就失去了store buffer的作用了,而且开销比直接写cache更大,所以这种操作不能频繁使用

接着上面的例子
	void foo(void)
    {
    	a = 1;
    	smp_mb();//写栅栏,刷新store buffer
    	b = 1;
    }

smp_mb()指令可以迫使CPU在进行后续store操作前刷新store-buffer。以上面的程序为例，增加memory barrier之后，就可以保证在执行b=1的时候CPU0-store-buffer中的a已经刷新到cache中了，此时CPU1-cache中的a 必然已经标记为invalid。对于CPU1中执行的代码，则可以保证当b==0为假时，a已经不在CPU1-cache中，从而必须从CPU0- cache传递，得到新值“1”

invalidate queue:
store buffer一般都很小,对于连续的写操作力不从心,core修改状态为S的变量时,先要invalid另一个core中变量,等待respone然后再修改,invalidate queue用来存在invalid指令,然后快速返回让发起invalid的core快速执行接下来的指令.所以invalidate queue是针对接受invalidate消息的core来说的

store buffer一般很小，所以CPU执行几个store操作就会填满。这时候CPU必须等待invalidation ACK消息，来释放缓冲区空间——得到invalidation ACK消息的记录会同步到cache中，并从store buffer中移除。同样的情形发生在memory barrier执行以后，这时候所有后续的store操作都必须等待invalidation完成，不论这些操作是否导致cache-miss。解决办法 很简单，即使用“Invalidate Queues”将invalidate消息排队，然后马上返回invalidate ACK消息。不过这种方法有问题。
考虑下面的情况：
   1: void foo(void)
   2: {
   3: a = 1;
   4: smp_mb();
   5: b = 1;
   6: }
   7:  
   8: void bar(void)
   9: {
  10: while (b == 0) continue;
  11: assert(a == 1);
  12: }
a处于shared状态，b在CPU0-cache内。CPU0执行foo()，CPU1执行函数bar()。执行操作如下：
1 CPU0执行a=1。因为cache-line是shared状态，所以新值放到store-buffer里，并传递invalidate消息来通知CPU1
2 CPU1执行 while(b==0) continue;但是b不再CPU1-cache中，所以发送read消息 
3 CPU1接受到CPU0的invalidate消息，将其排队，然后返回ACK消息 
4 CPU0接收到来自CPU1的ACK消息，然后执行smp_mb()，将a从store-buffer移到cache-line中 
5 CPU0执行b=1;因为已经包含了该cache-line，所以将b的新值写入cache-line 
6 CPU0接收到了read消息，于是传递包含b新值的cache-line给CPU1，并标记为shared状态 
7 CPU1接收到包含b的cache-line 
8 CPU1继续执行while(b==0) continue;因为为假所以进行下一个语句 
9 CPU1执行assert(a==1)，因为a的旧值依然在CPU1-cache中，断言失败 
10 尽管断言失败了，但是CPU1还是处理了队列中的invalidate消息，并真的invalidate了包含a的cache-line，但是为时已晚
可以看出出现问题的原因是，当CPU排队某个invalidate消息后，在它还没有处理这个消息之前，就再次读取该消息对应的数据了，该数据此时本应该已经失效的。
解决方法是在bar()中也增加一个memory barrier：
    void bar(void)
    {
    	while (b == 0) continue;
    	smp_mb();
    	assert(a == 1);
    }
此处smp_mb()的作用是处理“Invalidate Queues”中的消息，于是在执行assert(a==1)时，CPU1中的包含a的cache-line已经无效了，新的值要重新从CPU0-cache中读取。
memory bariier还可以细分为“write memory barrier(wmb)”和“read memory barrier(rmb)”。rmb只处理Invalidate Queues，wmb只处理store buffer。
读栅栏就是执行invalidate queue中的validate指令,因为接下来的指令对validate queue中的指令有依赖
可以使用rmb和wmb重写上面的例子：
    void foo(void)
    {
    	a = 1;
    	smp_wmb();
		b = 1;
    }
      
    void bar(void)
    {
    	while (b == 0) continue;
    	smp_rmb();
		assert(a == 1);
    }
最后提一下x86的mb。x86CPU会自动处理store顺序，所以smp_wmb()原语什么也不做，但是load有可能乱序，smp_rmb()和smp_mb()展开为lock;addl。


# 总结
因为cpu和内存速度不匹配引入了cache, 因为多核会导致cache不一致,所以引入了cache一致性协议,又因为一致性协议效率问题引入了store buffer,因为store buffer的存在导致了可见性问题,所以引入了内存写栅栏,强制刷新store buffer,因为store buffer太小,满了之后需要等待invalidate ack来释放store buffer,为了让接受invalidate消息的core快速响应,然后引入了invalidate queue,收到invalidate消息后直接放入queue,但是core读取在本cache的变量,但是变量已经在invalidate queue中了,所以脏读了,所以引入读栅栏,强制先执行invalidate queue,使本cache中的变量invalidate,读取时从发出invalidate的那个core的cache读取

# 在程序中使用内存栅栏
java的volatile关键字实现了读栅栏和写栅栏,用该关键字声明变量时编译器对动态插入软件栅栏指令,在字节码中看不出来效果,但是synchronied和volatile实现JMM定义的语义,

The Java Memory Model defines how threads interact through memory



# 参考

主要的处理器构成:参考(https://docs.oracle.com/cd/E19455-01/806-5257/guide-14/index.html)
The major multiprocessor components are:
1.The processors themselves
2.Store buffers, which connect the processors to their caches
3.Caches, which hold the contents of recently accessed or modified storage locations
4.memory, which is the primary storage (and is shared by all processors).



http://blog.csdn.net/chen19870707/article/details/39896655
[1] http://www.rdrop.com/users/paulmck/scalability/paper/whymb.2010.06.07c.pdf 
[2] http://en.wikipedia.org/wiki/Memory_barrier 
[3] http://www.mjmwired.net/kernel/Documentation/memory-barriers.txt
[4]http://sstompkins.wordpress.com/2011/04/12/why-memory-barrier%EF%BC%9F/