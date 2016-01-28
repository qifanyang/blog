---
title: tomcat connector配置
date: 2016-01-27 10:16:43
tags: tomcat
---

## HTTP Connector配置

	文档有详细说明Java HTTP Connector: /docs/config/http.html (blocking & non-blocking)
	
	默认配置:
	<Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
	通过查看文档,有些默认属性比较重要
	*maxThreads default 200
	The maximum number of request processing threads to be created by this Connector, which therefore determines the maximum number of simultaneous requests that can be handled. If not specified, this attribute is set to 200. If an executor is associated with this connector, this attribute is ignored as the connector will execute tasks using the executor rather than an internal thread pool.
	
	*acceptCount default 100
	The maximum queue length for incoming connection requests when all possible request processing threads are in use. Any requests received when the queue is full will be refused. The default value is 100.

	当tomcat收到http请求后,如果当前处理线程没有达到maxThreads,则继续创建创建线程处理请求,如果达maxThreads
	则将请求放到一个队列中,如果队列大小超过acceptCount则直接返回拒绝请求给客户端

	*maxConnections	
	The maximum number of connections that the server will accept and process at any given time. When this number has been reached, the server will accept, but not process, one further connection. This additional connection be blocked until the number of connections being processed falls below maxConnections at which point the server will start accepting and processing new connections again. Note that once the limit has been reached, the operating system may still accept connections based on the acceptCount setting. The default value varies by connector type. For BIO the default is the value of maxThreads unless an Executor is used in which case the default will be the value of maxThreads from the executor. For NIO the default is 10000. For APR/native, the default is 8192.

	Note that for APR/native on Windows, the configured value will be reduced to the highest multiple of 1024 that is less than or equal to maxConnections. This is done for performance reasons.
	If set to a value of -1, the maxConnections feature is disabled and connections are not counted.

	对于BIO Connector该值为maxThreads,对于NIO Connector该值为10000,因为NIO使用的Select I/O复用模式
	一个线程可以处理多个请求,所以该值没有确定的值,10000只是一个合适的值
	对于NIO环境,达到10000后,也会接受请求,直到达到acceptCount,内部处理还是使用线程池处理

## tomcat Connector类型
	HHTP Connector, AJP Connector, AJP用于和web Connector通信,读取http 请求,然后应用程序服务器处理
	请求,再通过AJP返回数据到web server, 引入AJP还有几个好处,利用更高效的http server服务静态资源,做负载
	均衡,无缝升级等,动态的升级后台程序,只需在负载均衡哪里关掉流量,然后更新服务启动再次在负载均衡处加上即可

	Apache HTTP Server 与 Tomcat 的三种连接方式: JK, http_proxy, ajp_proxy

	https://www.ibm.com/developerworks/cn/opensource/os-lo-apache-tomcat/

## HTTP Connector
	处理http请求,读取请求并转发到,servlet container容器, 处理静态内容的速度比apache server还是要慢些

## AJP(Apache JServ Protocol) Connector
	采用AJP协议通信,比http更高效的二进制协议,如果使用了负载均衡,web服务器可以使用AJP和servlet container
	通信,在单个tomcat实例中,http Connector处理http请求并转发到ajp Connector
	Client <- http/s-> Proxy <- http/s -> App
	vs
	Client <- http/s-> Proxy <- AJP -> App
	下面这种方式更加高效,可以复用tcp连接,而且使用二进制协议比文本效益效率高