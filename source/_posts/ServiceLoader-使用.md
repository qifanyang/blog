---
title: ServiceLoader-使用
date: 2016-08-15 14:16:58
tags: java
---

## SPI
service provider implementation , java JDBC驱动是最好的例子  
DriverManager获取连接时,会首先加载所有驱动(loadInitialDrivers),JDBC需要定义好java.sql.Driver接口,在spi接口中需要定义  
功能接口,版本信息,对于特定参数是否能够提供服务,spi实现类需要放在META-INF/services/java.sql.Driver文件中,可以是多个类  

当classpath中有多个jar包多个实现时,会遍历所有实现直到getConnection(url)返回不是空为止,所以如果有多个实现在classpath中  
是无法确定使用哪一个的(可以控制在classpath中顺序实现)

## 自定义SPI
1.参照JDBC实现spi接口  
2.编写META-INF/services文件  
3.编写实现类  
4.使用ServiceLoader.load()获取实现类对象,完成工作  
5.替换实现,重复以上步骤,替换jar即可  

## LazyIterator
遍历各种实现时并不是一次性加载所有实现,而是一个文件一个文件的遍历,防止加载不必要的类  

## 其它
spring.handler文件也使用类似方式,定义了很多默认实现,当打算采用新的实现,可以使用优先级列表来控制    