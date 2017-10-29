# jdk9模块系统

## 新建module-info.java
在模块目录下新建模块描述文件,定义依赖模块和导出模块.如下:
~~~
module com.greetings { //模块名称为com.greetings
    requires java.base; //依赖的模块

    exports com.greetings; //该模块导出的包,其它模块引用该模块时才可以访问.只有模块化时才会有作用
}
~~~

## 目录结构
src/main/java/com/greetings/Main.java
src/main/java/module-info.java

不使用jmod maven插件,使用命令打包

mkdir -p mods/com.greetings //创建目录
javac -d mods/com.greetings src/main/java/module-info.java src/main/java/com/greetings/Main.java //编译源文件,输出到指定目录 mods/com.greetings

java --module-path mods -m com.greetings/com.greetings.Main //执行模块程序, 没有打包的

mkdir mlib //创建模块目录
jar --create --file=mlib/com.greetings.jar --main-class=com.greetings.Main -C mods/com.greetings .  //创建模块jar 使用mods/com.greetings下的class

/Library/Java/JavaVirtualMachines/jdk-9.jdk/Contents/Home/bin/jlink  --module-path /Library/Java/JavaVirtualMachines/jdk-9.jdk/Contents/Home/jmods:mlib --add-modules com.greetings --output greetingsapp

使用jlink创建运行时镜像, 输出到目录greetingsapp
--module-path 指定模块目录,包含jdk模块和自己开发的目录
--add-module 创建镜像需要模块
--output 输出目录

## 镜像目录
greetingsapp/bin //包含java命令程序
greetingsapp/lib //镜像运行时的库
greetingsapp/legal
greetingsapp/include
greetingsapp/conf 
greetingsapp/release //可以查看包含的非jdk模块

查看完成模块命令  greetingsapp/bin/java --list-modules
yangqifandeMacBook-Pro:module yangqifan$ greetingsapp/bin/java --list-modules
com.greetings
java.base@9

只包含java.base和com.greetings模块, 所以镜像可以很小, mac上镜像为35M

运行镜像程序   bin/java com.greetings.Main运行一个程序  
一个镜像中可以包含多个模块,多个程序入口,可以运行多种程序,镜像是一个完整的运行环境  

## jmod
jmod和jar文件差不多,在顶层目录下多了module-info.java,可以包含本地命令文件和配置文件

/Library/Java/JavaVirtualMachines/jdk-9.jdk/Contents/Home/bin/jmod create --class-path mods/com.greetings  --main-class com.greetings.Main mlib/greetings.jmod










