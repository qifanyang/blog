# 引用
    java中持有对象的变量叫引用,没有指针,无法通过引用释放对象的内存空间,GC自动管理对象内存释放.  

#术语与Reference
    Reference对象 -> instance
    Reference对象持有的引用对象 -> referent
    instance有几个内部状态:
    a.Active, 新创建的instance处于该状态,当referent可达性发生变化时,instance状态发生变化,如果创建  
    instance时指定了ReferenceQueue,那么instance状态变化为Pending,并加入Pending列表,否则处于InActive,状态变化都是VM自动触发完成.
    b.Pending: 如果创建instance,指定了ReferenceQueue,那么referent可达性改变时,instance处于  
    该状态,等待Reference-Handler Thread enqueue
    c.Enqueued. 被加入引用队列,通过instance.get()可以获取referent.但是还是处于该状态
    d.InActive:死亡状态,从ReferenceQueue移除后,处于该状态

    Reference-Handler Thread:  
    该线程一般处于睡眠状态,当有instance加入到Pending列表时,该线程被唤醒. (注意: Pending是静态属性  所以是全局共享的,不属于某个reference instance, Reference-Handler Thread是处理所有Reference),当pending不为null时:  
    a.如果pending不是Cleaner对象,那么进行enqueued操作. 
    b.如果pending是Cleaner对象,说明已经被垃圾回收了,执行Cleaner run,达到通知的目的 
# 强引用,弱引用,软引用,虚引用
  java对引用进行了扩展,可以给GC一些信息,影响垃圾回收过程.
1.强引用: 
    Object o = new Object(), o叫做强引用,在线程栈中o指向的对象是无法回收的
2.弱引用: 
    Object o = new Object();
    WeakReference<Object> ref = new WeakReference<Object>(o);
    o是强引用, WeakReference将o包装, 现在有两条可达性分析路径
    o -> 业务对象
    ref -> 弱引用对象 -> 业务对象
    当方法退出时,在线程栈中的强引用o被销毁, 现在只可以通过弱引用ref访问业务对象  
    
    在垃圾回收时,referent只有instance持有,那么该referent可以被回收,等同于提示GC可以忽略该可达性  
    分析
3.软引用 
    类似弱引用,但GC时不一定垃圾回收,只有在GC的时候存在内存资源紧张时,才进行垃圾回收.软引用一般用于  
    实现cache,类似操作系统的cache file,在需要的时候用于其它用途.如同:内存你(应用程序)可以任意使用,  
    但是在我(JVM)需要内存的时候,你必须归还给我,当然如果你正在使用,那么我等你这一次使用完.  

4.虚引用
    GC回收时提示作用,无其它功能.实现GC对象观察者模式

#WeakHashMap
    曾经有次面试,Naver China面试官说在什么情况下HashMap存在内存泄漏问题,我一头雾水?  
    我的理解是Map需要手动管理,如果放入后因为某些原因一直没有移除就叫内存泄漏. 
    后面仔细问了下,面试官是想表达如果key无法找到了,比如是new Object(),那么处了clear真还没法移除了.  

    WeakHashMap就不会存在这个问题,它的key是weak keys,弱引用. 当只有弱引用时是可以回收的.但在工作中,  
    实际应用场景很少能用到该种特性,在web程序中都是无状态的,如果要做成有状态,这个状态在每次GC时又  
    都被回收掉,很少有这种场景.在做游戏开发时,每帧渲染使用了WeakHashMap,因为在下一帧很多数据都需要  
    被移除,但是偶尔需要重用某些数据(被回收了就算了,存在的就更好了),所以就在回收队列中查找  

    WeakHashMap如果移除value, ,该map再次被访问时会检查.通过方法expungeStaleEntries(...)检查  
    ,当key被加入到ReferenceQueue中,则检查,合理会造成线性遍历,如果弱引用对象很多,效率不高  
    如果key在队列中则释放对应的value值. 

#ThreadLocalMap  
    经常面试会有人问threadLocal使用过?会有内存泄漏问题么?  
    它的内部使用ThreadLocal作为key, ThreadLocalMap存数据,在Thead对象中  
    所以这里是使用Map,存在的内存泄漏问题类似WeakHashMap,就是关于key的问题  
    ThreadLocal key一般作为静态属性,应用程序只是使用set,不使用remove,并且key一直存在,那么  
    value一直存在的,对于服务器程序,几百个线程,大对象没及时移除,会造成内存OOM的. 

    如果ThreadLocalMap没有使用ReferenceQueue,那么如何清除value的呢?
    ThreadLocal key作文静态属性,所以一般总会有一个强引用指向,所以该key不会被回收  
    不同于WeakHashMap在put时检查queue, 检查ThreadLocal 作为key被回收没有, 回收了则回收value  

    所以需要在set 和 get的时候才会赋值value为null,GC才可以回收. 当线程退出时threadLocals=null  
    ,而threadLocal作为weak key只是提示什么时候可以回收value,不能阻止线程退出时回收value .

    所以讲threadlocal key赋值为null,可以让weak key被回收, 提示所有线程ThreadLocals可以清除  
    线程该key的value了,但是到底何时清除还要等到ThreadLoalMap再次被访问时才会触发  

    ThreadLoalMap的set和get会触发清除threadlocal变量, 但清除操作并不是一定会触发.

    set时: 
    1.如果map中对应的entry位置,那么直接替换并返回,不执行清除操作,所以快速返回.
    2.当没有对应的entry,那么新建entry并放入,从该key对应得index检查后面的entry是否需要清除  
    而且是检查log(map.size)个位置, 也是为了快速返回.

    get时:
    如果没有快速查找到对应的entry,执行hash冲突查找,并将key为null的value清除掉

    所以ThreadLocal的value不是及时释放的,会存在一定的内存浪费,所以要即使remove,还是要手动管理map


    
    