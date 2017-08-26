## 环境搭建
+ $GOROOT为安装目录
+ $GOPATH为工作目录
+ 目录结构
  - src 源文件
  - bin 可执行文件
  - pkg 编译后的包

## 编译
    go build,除非在源文件目录下,否则要指定目录.默认源文件目录为$GOPATH/src,所以只需要指定src下  
的包文件即可,对main包执行go build会生成可执行文件,对非main包执行go build不会产生文件  

    go install,对非main执行会在pkg目录下产生中间文件,类似c语言.o目标文件, 对main执行会在bin  
目录下生成可执行文件,如果有依赖会自动build和install  

    执行go build/install 要注意当前目录,否则会提示no buildable Go source files in    /Users/yangqifan/go, 提示在目录XXX中没有可构建的源文件  

## demo
~~~
src/myutil
package mymath

//字段名首字母大写,表示暴露字段, 可通过mymath.X访问
var X int = 1

//函数名首字母大写,表示暴露方法,类似Node.js的exports, 可通过mymath.Add()调用
func Add(x int, y int) int {
    return x+y
}

src/test
package main

import (
    "fmt"
    "os"
    "myutil"
)

func main(){
    fmt.Printf("x+y=%d\n", mymath.Add(44,55))
}
~~~

## 包
用于模块化,封装,代码重用.  
类似c++命名空间+函数  
类似java类名+静态方法  
类似Node.js用法,方法名大写就如同exports了方法  