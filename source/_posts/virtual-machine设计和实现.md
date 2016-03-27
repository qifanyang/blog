---
title: virtual-machine设计和实现
date: 2016-03-24 11:02:59
tags: 杂七杂八
---
## 教程
https://en.wikibooks.org/wiki/Creating_a_Virtual_Machine

计划深入了解下java虚拟机,但是太过于庞大,所以找了一个教程学习,加深理解JVM

jvm规范指令集 https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html  

jvm规范有完整的介绍

hotspot主要采用c++实现,jdk采用c/c++实现,看了虚拟机之后,发现平时写的java代码,其实只是c/c++程序的输入而已,java程序本质上还是c/c++程序

比如无法理解的是java本地方法,java和c之间是如何调用的?对于java中的对象,数据类型,虚拟机都有对应的数据类型,java调用本地方法,对于虚拟机来说只是读取到一条指令,然后调用一个c方法而已,当然还有一个栈环境,这个栈也只是一个c/c++模拟出来的.

虚拟机是对一个抽象机器的实现,抽象机器在很多论文中出现,是一种对机器的抽象描述

一个简易虚拟机组成:
a.指令集(opcode),对应一个处理器的指令集,包含加载,存储,算术运算,跳转...
b.读取指令集的程序,完成指令集的读取,解码...
c.执行单元,负责执行读取到的指令集
d.类似游戏主循环一样,一个while(true)循环不停的加载指令,执行指令,一个while循环可以对应一个Thread来实现多线程效果(猜想)

指令集到机器码转换的过程叫做汇编,逆向叫反汇编

51单片机寻址模式:
立即寻址  加载立即数  iload #5; 
直接寻址  加载内存中的数据 iload @1000
间接寻址  加载内存中的数据所代表的地址的数据


一种指令编码实现:
16bit长度,4个16进制的数字0-9, A-F, 
第1个数字, 表示指令  ,  4个bit, 所以可以有16个指令
第2个数字, 寄存器编号, 不同指令可以默认使用特定的寄存器, 当执行了该指令后,对应寄存器中一定是该指令产生的结果
第3个和第4个数字,联合起来可以是立即数,可以是内存地址, 根据不同的指令,表示不同的含义



执行引擎简单实现:
每个opcode对应一个编码,使用一个二维数组存储opcode和对应的处理函数指针,解释执行只是一个根据数组下标找到函数指针,然后一个函数调用,速度还是很快的




ClassFile { 
u4 magic; 
u2 minor_version; 
u2 major_version; 
u2 constant_pool_count; 
cp_info constant_pool[constant_pool_count-1];
u2 access_flags; 
u2 this_class; 
u2 super_class; 
u2 interfaces_count; 
u2 interfaces[interfaces_count]; 
u2 fields_count; 
field_info fields[fields_count]; 
u2 methods_count; 
method_info methods[methods_count]; 
u2 attributes_count; 
attribute_info attributes[attributes_count]; 
}

解析class文件要点:
1.常量池位置0不存储,例如常量池有15个item, 则constant_pool_count=16, 常量池索引从1开始

2.u1,u2,u4都是无符号类型,不要和java的byte,short,int对应, 可以自定义类型U1,U2,U4

3.attribute属性根据attribute_name_index指向常量池的值来确定具体是什么属性,然后进入
具体的属性,比如Code,因为在attribute_info中已经知道attribute_name_index和attribute_length
所以在具体的属性中对应着两个值是知道的,解析时class不在重复,所以应该跳过这两个值

4.解析常量池,常量池中数据类型的tag已知,比如constant_utf8_info中tag为1,不应再从class中读取

5.使用DataInput读取应调用readUnsignedXxx()

6.code_attribute用于存储字节码

7.解析字节码,可以分析有哪些方法,不用使用类加载器加载,然后再使用反射分析.比如要加载包名和类名
都相同的一个类,只要知道他们的区别,通过字节码分析就可以确定,当然使用asm分析类不用字节维护解析器

