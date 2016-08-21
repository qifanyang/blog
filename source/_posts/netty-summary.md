---
title: netty-summary
date: 2016-08-19 17:38:32
tags: netty
---

## 相关类
NioSocketChannel--->代表一个网络连接,发送数据也通过该类  
ChannelOutboundHandler--->用于发送消息缓冲,如果发送缓冲区满了,缓冲在这里
Channel(Unsafe)  
AbstractChannel(AbstractUnSafe)--->EventLoop | ServerChannel(对应一套服务端channel继承)
AbstractNioChannel(AbstractNioUnsafe) 
AbstractNioByteChannel(AbstractNioByteUnsafe)
NioSocketChannel(NioSocketChannelUnsafe)


,DefaultChannelPipeline,AbstractChannelHandlerContext,ChannelOutboundHandler  
ChannelOutboundBuffer,NioSocketChannel,AbstractChannel,NioByteUnsafe
com.mysql.jdbc.authentication.MysqlNativePasswordPlugin mysql插件认证类

## 发送数据过快
netty底层调用发送,当发送缓冲区满,netty会将数据累积在ChannelOutboundBuffer中,采用链表  

channel代表一个网络连接,发送数据是线程安全的  











