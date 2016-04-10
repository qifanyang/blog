---
title: utf8的int-bytes转为byte-bytes
date: 2016-04-01 14:38:24
tags: 杂七杂八
---

## 场景
java的class文件存储cosntant_utf8_info结构时,使用了无符号bytes数组来存储utf8编码字节数组,使用java写解析器读取无符号byte时,采用的int存储,当需要构建string时需要将int数组转为bytes数组,担心出现强转导致数据有问题

## utf8编码格式

![utf8编码格式](/img/utf8.png)


## 结论:强转不会有问题

java的int转byte采用的是带符号扩展.

单字节:0XXX XXXX , 1扩展还是0.... 0000 0001
双字节:110X XXXX 10XX XXXX  , 符号为不为0,只是截取
三字节:同双字节

### java中窄化转换会导致值改变(截取)或者符号为改变(带符号扩展)

## 带符号扩展
	
		int i = 128;
        System.out.println(Integer.toBinaryString(i));//10000000
        System.out.println((byte)i);//强转后还是10000000,然后带符号扩展(补码)11111111111111111111111110000000
		输出:
		10000000
		-128

		i = 1;//自动转换不会有问题,java是带符号扩展的
        System.out.println(Integer.toBinaryString(i));//0.... 0000 0001
        System.out.println((byte)i);//强转后0000 0001, 然后带符号扩展00000000 ... 00000001, 还是1
		输出:
		1
		1
		
		i = -2;//带符号扩展,负数不会有问题
        System.out.println(Integer.toBinaryString(i));//11111111111111111111111111111110
        System.out.println((byte)i);//-2
		
		

## 数字在计算机中的存储的是补码

1.正数
原码,反码,补码一样	

2.负数
-2
1000...10(原码) 符号位(正数0, 负数1)+值二进制
1111...01(反码) 符号位不变+值二进制取反
1111...10(补码)	反码+1	