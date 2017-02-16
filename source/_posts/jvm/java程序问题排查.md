---
title: java程序问题排查
date: 2017-02-16 23:14
tags: jvm
---

## 问题现象
  当服务器运行一段时间后响应速度变慢或者超时无响应时,很多时候重启服务器现象就会消失,但过了一段时间问题重现这时候  
就要注意了,可能程序或者服务器出现问题了,这时候需要排查问题  

## 排查问题使用工具
命令:top,jps,jmap,jstack等  

## 查看cpu使用率过高
1.使用top查看程序cpu,内存使用情况,如果程序相应缓慢,程序一般cpu使用率会很高,确定PID  
2.使用top -H -p 进程ID 查看java程序中每个线程的CPU使用情况,确定cpu使用高的线程ID  
3.使用jstack pid > /dump.txt 查看cpu使用率高的线程栈中执行的方法,以及线程状态,WAITING,BLOCKED,  
3.1检查cpu使用率高的线程当前执行的方法代码是否有问题,一般会有循环导致cpu使用率过高

# 查看内存问题
1.查看内存情况,garbage collector ,使用命令jmap -heap pid
2.使用jstat -gc pid查看垃圾收集时间,频率,以及收集后内存变化  
3.如果垃圾收集后内存效果变化不明显,则需要使用jmap -dump:format=b,file=文件名 [pid] dump内存,然后使用jvisual vm  
来分析对象个数,也可以使用jmap -histo pid输出统计信息  

4.启动程序时最好加上gc日志参数:  
JVM的GC日志的主要参数包括如下几个：
-XX:+PrintGC 输出GC日志  
-XX:+PrintGCDetails 输出GC的详细日志  
-XX:+PrintGCTimeStamps 输出GC的时间戳（以基准时间的形式）  
-XX:+PrintGCDateStamps 输出GC的时间戳（以日期的形式，如 2013-05-04T21:53:59.234+0800）  
-XX:+PrintHeapAtGC 在进行GC的前后打印出堆的信息  
-XX:+PrintGCApplicationStoppedTime // 输出GC造成应用暂停的时间  
-Xloggc:../logs/gc.log 日志文件的输出路径  

## 问题
遇到程序运行一段时间后相应变慢,最终确定是gc线程一直运行,cpu使用率很高,查看垃圾会后器是parallel GC,改为CMS后问题解决  



## 参考
http://www.cnblogs.com/mikevictor07/p/5024645.html
