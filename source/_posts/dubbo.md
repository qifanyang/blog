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

## dubbo filter
dubbo ServerConfig在spring注入service后会暴露服务,将Service实现使用动态代理创建统一包装的Invoker 
服务暴露通过Protocol.export实现,暴露服务会使用ProtocolFilterWrapper.buildInvokerChain()使用filter  
包装服务实现(filter chain模式),激活的filter可以根据url来获取,ExceptionFilter就是通过这种方式实现  
还有各种统计调用Filter都是这种模式实现,Filter模式可以实现对服务实现者透明,又可以对服务调用做监控  


## dubbo扩展
Protocol protocol = ExtensionLoader.getExtensionLoader(Protocol.class).getAdaptiveExtension();  
dubbo使用了很多这种方式来查找具体实现,通过接口名和Adaptive注解,或者方法中指定url中key来寻找扩展

扩展具体实现在配置文件中,类似spring namespace handler

## dubbo异常封装
使用动态生成的Wrapper对象调用对应serviceImpl方法,如果抛出异常,统一包装成InvocationTargetException  
如下:  
~~~
//        try {
//
//            if ("build".equals($2) && $3.length == 1) {
//                return ($w) w.build((java.lang.String) $4[0]);
//            }
//        } catch (Throwable e) {
//            throw new java.lang.reflect.InvocationTargetException(e);
//        }
~~~
然后在AbstractProxyInvoker的invoke中统一处理,捕获异常并返回带有异常的RpcResult,然后  
在ExceptionFilter就可以处理异常了    
~~~
       try {
            return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
        } catch (InvocationTargetException e) {
            return new RpcResult(e.getTargetException());
        } catch (Throwable e) {
            throw new RpcException("Failed to invoke remote proxy method " + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
        }
~~~
关于Wrapper的使用,如果直接调用被代理的方法,需要查找proxy的方法,每次调用都要使用反射查找,效率不高?  
所以使用类似字节码技术,生成一个方法的直接调用,但是java 反射调用超过一定次数也会创建字节码来优化  
反射调用,默认15次反射调用,所以这个优化没有多大必要,反而害得我找了半天在哪里将异常包装到RpcResult  

在字节码中将Throwable异常包装成了InvocationTargetException  

