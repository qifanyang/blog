---
title: ServiceLoader-使用
date: 2016-08-15 14:16:58
tags: java
---

## SPI
service provider implementation , java JDBC驱动是最好的例子,可以达到解耦的目的    
DriverManager获取连接时,会首先加载所有驱动(loadInitialDrivers),JDBC需要定义好java.sql.Driver接口,在spi接口中需要定义  
功能接口,版本信息,对于特定参数是否能够提供服务,spi实现类需要放在META-INF/services/java.sql.Driver文件中,可以是多个类  

当classpath中有多个jar包多个实现时,会遍历所有实现直到getConnection(url)返回不是空为止,所以如果有多个实现在classpath中  
是无法确定使用哪一个的(可以控制在classpath中顺序实现)

## service provider framework component
1.service interface -->connection  
2.service provider interface -->Driver  
3.provider registration api-->DriverManager.register()  
4.service access api-->DriverManager.getConnection()  

## 自定义SPI
1.参照JDBC实现spi接口(Driver),服务接口si(Connection)  
2.编写META-INF/services文件  
3.编写实现类  
4.使用ServiceLoader.load()获取实现类对象,完成工作  
5.替换实现,重复以上步骤,替换jar即可  

## JDBC 4改进
在jdbc 4.0之前,使用jdbc需要显示使用Class.forname("com.mysql.jdbc.Driver")来初始化驱动类,但是在JDBC 4.0之后使用了ServiceLoader,驱动需要编写  
META-INF/services/java.sql.Driver文件就不用再显示加载驱动了    

## LazyIterator
JDBC遍历各种实现时并不是一次性加载所有实现,而是一个文件一个文件的遍历,防止加载不必要的类  

## 其它
spring.handler文件也使用类似方式,定义了很多默认实现,当打算采用新的实现,可以使用优先级列表来控制    

## 场景
SPI实现了以jar包提供服务的方式,应用代码只需要依赖SPI接口,当想要替换实现只需要更新jar包即可  
SPI提供了一种解耦的方法,是面向接口编程的实践,当更改实现无需在代码中显示设置新的实现,只需要  
将jar包放入classpath,当使用接口时,在类加载时创建具体实现以供使用,为了确定使用哪个实现在数据库  
连接url中指定了一个标识,比如jdbc:mysql, jdbc:oracle 等等  

在spring中使用面向接口编程,service中属性也使用接口,spring默认按类型注入,如果一个接口有两个实现  
那么spring不知道注入哪一个然后抛出异常,可以使用qualifer明确使用哪个接口实现,也可以让spring只  
管理一个接口实现(去掉一个类的@Component),如果很多地方都在使用显然使用去掉注解方式改动最小  
当有大量代码都依赖接口来编程,想替换实现时优势就明显了,改动的代码很少,只需提供新的实现  
