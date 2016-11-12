---
title: java异常
date: 2016-11-12 11:03:01
tags: java
---

# 简介
程序中方法调用,可以使用异常和返回值来处理异常情况,在C中没有异常,所以很多方法调用都要有一个返回值来表示方法调用是否正常  

如下  
~~~c
    //bind，成功返回0，出错返回-1
     if(bind(server_sockfd,(struct sockaddr *)&server_sockaddr,sizeof(server_sockaddr))==-1)
     {
         perror("bind");
         exit(1);
     }
~~~

C使用返回值,0表示正常,-1表示出错.在开发时只要有方法调用就需要通过这种方式,或者全局变量的方式来判定方法调用是否发生异常  
java中引入了异常,程序员每次方法调用可以不用关注调用是否发生异常,不用写if()判断,程序发生错误虚拟机会自动抛出一个异常,所以  
java程序员需要编写处理异常的代码,这是强制性的,正因为这样java才更加健壮.  

## java异常分类
### checked vs unchecked
1. checked exception 编译时异常强制程序员处理,例如IOException  
2. unchecked exception 不要求强制处理,RuntimeException和Error都是此类异常  

### Exception vs Error
1. Exception表示程序运行异常,需要开发人员编写异常处理器  
2. Error表示程序运行环境类异常,程序逻辑代码一般无法处理,比如OutOfMemory,StackOverFlow,程序中不要捕获Error,因为you can not recover     


## java异常继承结构

1. Throwable为所有异常的父类,直接子类有Exception和Error  
2. Exception子类有IOException,RuntimeException等  
2. Error子类有VirtualMachineError,AWTError等  

## java如何实现异常
javac编译带有异常处理的代码时会生成异常异常表,每一个catch匹配一条类似如下的记录  
~~~
from  to  target                    type  
pc1   pc2  exception_handler_pc     Class java/lang/Exception  
~~~
当在pc1和pc2之间的字节码发生异常,如果匹配type则goto到target的位置执行  

使用了try{}catch(){}会有异常表  
使用了synchronized方法也会有异常表,当发生unchecked exception时,需要执行monitorexit  

## Exception与Error的效率
1. 他们都是Throwable的子类默认都需要fillStackTrace(开销大概1ms),还有区别就是checked与unchecked区别

~~~
public class ExceptionVsErrorPerformanceTest {

    public static void main(String[] args) {
        int num = 10000000;
        long start = System.nanoTime();
        for (int i = 0; i < num; i++) {
            try {
                throwException();
            } catch (Exception e) {

            }
        }
        System.out.println(System.nanoTime() - start);
        start = System.nanoTime();
        for (int i = 0; i < num; i++) {
            try {
                throwError();
            } catch (Error e) {

            }
        }
        System.out.println(System.nanoTime() - start);
    }

    private static void throwException() throws Exception {
        throw new IOException();
    }

    private static void throwError() {
        throw new Error();
    }
}
~~~

运行时间:  
10747437503  
9844091406  

当交换方法调用顺序时,还是这个时间,所以这个测试例子看不出区别  

对于checked和unchecked,jvm会区别对待,checked要去遍历查找异常表,唯一的区别就是这了,当方法调用链深和异常catch也多时,可能有一些差别  


## 异常的开销
当JVM执行字节码时,检查到异常会创建一个异常对象,使用athrow指令向上抛出,jvm退出当前栈帧,进入上一个帧    
为了记录发生异常的位置,需要根据栈帧来填充异常路径,这个是创建异常的主要开销  


## 异常的用法
1. 优先使用java定义的异常  
2. 可恢复错误使用Exception,程序错误使用RuntimeException  
3. 不要直接定义Throwable子类的异常,就算定义也等同于Exception,没有什么用处  
4. 抛出异常给出异常提示,比如说金额不足等,Throwable.getMessage可以获取提示  



