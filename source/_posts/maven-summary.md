---
title: maven-summary
date: 2016-08-17 10:56:38
tags:
---

## maven 结构

```xml
    <groupId>com.alibaba.otter</groupId>
    <artifactId>canal</artifactId>
    <packaging>pom</packaging>
    <name>canal module for otter ${project.version}</name>
    <version>1.0.23-SNAPSHOT</version>
```

groupId+artifactId+packaging+version 叫唯一坐标,用于标识一个maven 项目  

groupId可以是公司域名+项目组  
artifactId可是项目名  
packaging是打包执行结果,默认是jar, 可以是war等,如果是pom表示该项目有子项目  

## maven继承
```xml
    <parent>
        <groupId>org.sonatype.oss</groupId>
        <artifactId>oss-parent</artifactId>
        <version>7</version>
    </parent>
```
maven继承,可以在父pom中设置各种属性,依赖等等,继承达到复用目的  

## 属性设置
```xml
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <!--maven properties-->
        <maven.test.skip>true</maven.test.skip>
        <downloadSources>true</downloadSources>
        <!-- compiler settings properties -->
        <java_source_version>1.6</java_source_version>
        <java_target_version>1.6</java_target_version>
        <file_encoding>UTF-8</file_encoding>
    </properties>
```
各种属性设置,其它地方可以通过${name}应用,比如版本信息等,方便修改  

## 子模块
```xml
        <modules>
            <module>common</module>
            <module>meta</module>
            <module>dbsync</module>
        </modules>
``` 
  
## 依赖管理
```xml
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring</artifactId>
                <version>2.5.6</version>
                <exclusions>  
                <exclusion>      
                    <groupId>commons-logging</groupId>          
                    <artifactId>commons-logging</artifactId>  
                </exclusion>  
                </exclusions>  
            </dependency>
        </dependencies>
    </dependencyManagement>
```
注意maven有传递依赖的特性,会自动把依赖的依赖包含进来,这样会引入版本冲突,如果依赖的依赖冲突而且高版本是兼容低版本  
那么,可以把低版本的依赖排除掉,使用exclusion.如果依赖的依赖版本不兼容,那么需要引入classloader来隔离各自的依赖了  

## 构建配置
```xml
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.7</source>
                    <target>1.7</target>
                </configuration>
            </plugin>
            <plugin>
                <artifactId>maven-clean-plugin</artifactId>
                <version>3.0.0</version>
                <executions>
                <execution>
                    <id>auto-clean</id>
                    <phase>initialize</phase>
                    <goals>
                    <goal>clean</goal>
                    </goals>
                </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```
可是使用很多插件,自定义构建过程,为了配置默认的插件行为,需要在项目pom中明确配置插件属性,上面配置了每次default lifecycle  
的initialize phase 都执行clean goal    

## 使用
maven强调约定大于配置,定义了几种声明周期(lifecycle),default, clean and site, 每种声明周期有不同数量的phase,而我们  
使用maven都是直接使用命令(也叫goal),比如compile,clean,install, 这些命令会被绑定到生命周期的phase,因pacakging值的  
不同,默认绑定值的数量也不一样,例如:packaging值为pom 绑定如下:  

```
phase--->goal
package	site:attach-descriptor
install	install:install
deploy	deploy:deploy
```

执行一个goal,对应生命周期phase之前的phase都会先执行,比如complie,那么会先执行resources:resources  

所以关键几个概念:lifecycle,phase,goal,packaging  




