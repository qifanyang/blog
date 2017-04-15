#dubbo

## 线程模型
dubbo Container就如同游戏服务器,玩家每次请求如同customer rpc调用,container(游戏服务器)需要根据  
请求参数(玩家使用道具)来执行具体逻辑.  

tomcat是采用线程池来处理http请求,调用Servlet  

dubbo则和游戏服务器开发类似,使用netty/mina作为transpoter,netty/mima处理网络传输有自己的IO线程,container也用线程池  
来处理业务逻辑.

~~~
事件处理线程说明
如果事件处理的逻辑能迅速完成，并且不会发起新的IO请求，比如只是在内存中记个标识，则直接在IO线程上处理更快，因为减少了线程池调度。
但如果事件处理逻辑较慢，或者需要发起新的IO请求，比如需要查询数据库，则必须派发到线程池，否则IO线程阻塞，将导致不能接收其它请求。
如果用IO线程处理事件，又在事件处理过程中发起新的IO请求，比如在连接事件中发起登录请求，会报“可能引发死锁”异常，但不会真死锁。
~~~

Dispatcher指定请求在什么线程中执行:  
游戏开发中:  
1.可以根据玩家ID取模到固定线程中执行  
2.根据玩家所在地图分配到对应线程中执行  
3.玩家操作不需要与其它玩家操作同步,可以把玩家操作入队然后提交到线程池中执行  

dubbo Dispatcher:  
all 所有消息都派发到线程池，包括请求，响应，连接事件，断开事件，心跳等。默认采用该策略  
direct 所有消息都不派发到线程池，全部在IO线程上直接执行。  
message 只有请求响应消息派发到线程池，其它连接断开事件，心跳等消息，直接在IO线程上执行。 
execution 只请求消息派发到线程池，不含响应，响应和其它连接断开事件，心跳等消息，直接在IO线程上执行。 
connection 在IO线程上，将连接断开事件放入队列，有序逐个执行，其它消息派发到线程池。  
总的来说要么全在cotainer 线程池中执行,要么全在io线程中执行,要么

## 序列化
hessian,json,java
## 协议
dubbo(默认值采用该协议,端口为20880,序列化hessian2)  
rmi(端口1099,java序列化)  
http(端口80,json序列化)

## 集群容错
http://dubbo.io/User+Guide-zh.htm#UserGuide-zh-FailoverCluster  

dubbo调用service cluster,如果被调用的服务失败,缺省为failover,自动调用集群中其它service  
如果被调用的service写入数据成功,只是返回时失败,这时customer再调用其它service,如果接口不  
幂等那就会出现脏数据,so非幂等接口应该使用failfast模式

在做平台开发时,比如支付完成需要通知第三方接入平台,需要保证通知一定能到达第三方,可是发送通知时  
第三方可能不可用,就需要使用failback模式,后台记录失败通知请求,定时重发  


## 并发控制
某一个service资源暂用比较严重,所以可能存在该方法被并发执行数量限制,so 一个方法最多被多少个线程调用执行.  
使用Semaphore可以实现这个功能.  

限制com.foo.BarService的每个方法，服务器端并发执行（或占用线程池线程数）不能超过10个：  
~~~
<dubbo:service interface="com.foo.BarService" executes="10" />
~~~

下面限制每个客户端,所以最大并发执行数量为clientCount*10
限制com.foo.BarService的每个方法，每客户端并发执行（或占用连接的请求数）不能超过10个：  
~~~
<dubbo:service interface="com.foo.BarService" actives="10" />
~~~

## 令牌验证
防止消费者绕过注册中心访问提供者  
在注册中心控制权限，以决定要不要下发令牌给消费者  
注册中心可灵活改变授权方式，而不需修改或升级提供者  
可以全局设置开启令牌验证：  
~~~
<!--随机token令牌，使用UUID生成-->  
<dubbo:provider interface="com.foo.BarService" token="true" />
~~~

## 隐式传参

服务消费方
~~~
RpcContext.getContext().setAttachment("index", "1"); // 隐式传参，后面的远程调用都会隐式将这些参数发送到服务器端，类似cookie，用于框架集成，不建议常规业务使用
xxxService.xxx(); // 远程调用
// ...
~~~
服务提供方
~~~
public class XxxServiceImpl implements XxxService {
 
    public void xxx() { // 服务方法实现
        String index = RpcContext.getContext().getAttachment("index"); // 获取客户端隐式传入的参数，用于框架集成，不建议常规业务使用
        // ...
    }
 
}
~~~
spring事务管理使用ThreadLocal来隐式传参,只用于框架集成,常规业务不应该使用,把ThreadLocal比作一道门,门打开就表示可以让人进去,可以把们隐藏,但找到了也是可以进门的
所以不要在业务代码中使用ThreadLocal传参