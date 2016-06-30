---
title: redis-summary
date: 2016-06-21 10:37:41
tags:
---

## 摘要
差不多一年前用了下redis,很久不用之后又忘记,和使用mongodb一样,看了的东西一段时间不用就会忘记,so需要记录笔记,把要点和心得记录下来供以后翻阅,在使用的过程中加深理解再来完善笔记


## 安装
redis在github上提供源码,可以自己在linux,OSX,FreeBSB下编译,windows官方不提供,但可以在github下载到一个编译好的windows版本.地址:https://github.com/MSOpenTech/redis/releases

下载zip包后解压,执行服务器程序:redis-server.exe, 然后执行客户端程序redis-cli.exe, 就在可以在客户端执行命令了

例子:

    127.0.0.1:6379> set a 111
    OK
    127.0.0.1:6379> get a
    "111"
    127.0.0.1:6379>

## 值数据类型
redis是key-value内存数据库,key为字符串,value有多种数据类型,包含:字符串,列表(list),有序集合(zset),哈希表(hash),集合(set), 和java中的集合类型差不多
key太长浪费内存,浪费带宽,浪费查找时间,最大key长度512M
key太短可读性差,一般使用object-type:id 来作为key, 比如user:1000

每个redis命令能够操作的值数据类型是先定义好的,如同jvm字节码指令iadd,后面的操作数只能是整形, redis中lpush 操作的value只能是list
一般格式为:command + key + argument , 对于不同的command, argument值不一样 
http://redis.io/topics/data-types-intro

### 字符串:

    127.0.0.1:6379> set a 111
    OK
    127.0.0.1:6379> get a
    "111"
    127.0.0.1:6379> mset a 10 b 20 c 30  //设置多个key value值
    OK
    127.0.0.1:6379> mget a b c
    1) "10"
    2) "20"
    3) "30"
    127.0.0.1:6379>type a //查询类型
    string
    127.0.0.1:6379>exists a //查询是否存在
    (integer) 1
    127.0.0.1:6379>del a //删除key a


### 列表

    127.0.0.1:6379> lpush mylist A //列表左边添加A
    (integer) 1
    127.0.0.1:6379> lpush mylist B
    (integer) 2
    127.0.0.1:6379> lrange mylist 0 -1 //取值从0到结尾
    1) "B"
    2) "A"
    127.0.0.1:6379>
    //列表采用双端链表实现

### Hash

    127.0.0.1:6379> hmset user:100 age 30 sex m
    OK
    127.0.0.1:6379> hget user:100
    (error) ERR wrong number of arguments for 'hget' command
    127.0.0.1:6379> hget user:100 age
    "30"
    127.0.0.1:6379> hgetall user:100
    1) "age"
    2) "30"
    3) "sex"
    4) "m"
    127.0.0.1:6379>
    //hash来存储关系型数据库中的一条记录比较合适

### Set

    127.0.0.1:6379> sadd myset 1 2 3
    (integer) 3
    127.0.0.1:6379> sget myset
    (error) ERR unknown command 'sget'
    127.0.0.1:6379> smembers myset
    1) "1"
    2) "2"
    3) "3"

### Sorted sets

    127.0.0.1:6379> zadd leaderboard 10 yang
    (integer) 1
    127.0.0.1:6379> zadd leaderboard 29 li
    (integer) 1
    127.0.0.1:6379> zadd leaderboard 3 hua
    (integer) 1
    127.0.0.1:6379> zrange leaderboard 0 -1
    1) "hua"
    2) "yang"
    3) "li"
    127.0.0.1:6379> zrange leaderboard 0 1
    1) "hua"
    2) "yang"
    127.0.0.1:6379>
    127.0.0.1:6379> zrangebyscore leaderboard -inf 11 //按score查找, -inf表示负无穷
    1) "hua"
    2) "yang"
    127.0.0.1:6379>
    //采用跳跃表实现,每个节点还存储了更下一层节点的数量,加快查找. 主要用于排行榜,可以查看指定排名区间,也可以查看指定score区间

## redis键实现
key是string类型, value也有string类型, 所以string类型是很基础的数据类型, redis采用c实现,但是没有采用c中的char * ,使用字符数组
来实现string. 原因一:c在char数组中使用\0来表示string结束位置, 这种方式不是二进制安全的,因为在char数组中加入了特殊的结束符号,当特殊符号
属于需要的值时,就可能需要使用转义符号之类的处理.原因二:当value是string时可能有append操作,对于频繁的append操作每次都去malloc内存,增大
开销,所以采用类似java 中的ByteBuffer的方式,新建结构体sds,包含了已经占用长度,空闲长度,char[]. 其实key也可以使用c 的 char * , 因为key
长度是不会变的, redis为了二进制安全还是采用了自定义结构来实现,(jpg图片也可以作为key)

sds == simple dynamic struct

    typedef char *sds;
    struct sdshdr {
    // buf 已占用长度
    int len;
    // buf 剩余可用长度
    int free;
    // 实际保存字符串数据的地方
    char buf[];
    };

## redis值对象实现
redis不仅支持string值类型,还支持其他很多数据类型,list,set,hash,sorted set等等,为了支持这些值类型,redis实现了几个数据结构来完成对应的功能

### 双端链表
就是数据结构中的双端链表,用来支持list值类型,也用于事务模块命令保存,保存客户端等, redis中的链表为了方便,扩展数据结构中链表的一些功能,比如拷贝,
释放内存,保存链表长度等,在c结构体中保存函数指针,有点类似面向对象编程,当然结构体没有继承和多态特性

### 字典
字典又叫映射,关联数组, java中有多种map实现,字典的主要用途有以下两个：

    1. 实现数据库键空间（key space）；用于快速查找key是否存在和key对应的值
    2. 用作Hash 类型键的其中一种底层实现； 

hash实现可以使用java中的hashmap来理解,不同的是redis rehash不仅可以增大hash空间,而且还可以缩小,因为对于一个需要长期运行的系统来说,
删除很多元素后为了节约内存,应该缩小hash内部的数组

### 跳跃表
跳跃表和红黑树差不多,但实现更简单,估计改造也更简单,所以作者选择了跳跃表,由William Pugh 在论文《Skip lists: a probabilistic
alternative to balanced trees》中提出, 跳跃表在redis中只用于sorted set实现
为了适应自身的需求，Redis 基于William Pugh 论文中描述的跳跃表进行了修改，包括：

    1. score 值可重复。
    2. 对比一个元素需要同时检查它的score 和memeber 。
    3. 每个节点带有高度为1 层的后退指针，用于从表尾方向向表头方向迭代。


## redis类型系统
对于del key, key所指向的值对象不一样,其行为是不一样的.从OOP的角度来理解, del为调用接口函数, key为引用或指针, key.del(),redis首先
根据key找到对应的对象,然后根据对象encoding做出具体执行哪个函数的决定,不同的底层数据结构需要不同的函数来完成增删改查,在c++中是根据
对象内存中的虚函数表来查找函数入口,在java中是根据java对象的class来决定执行哪个函数.

redis对象定义:

    typedef struct redisObject {
        // 类型,可以是string,list,set,hash,sorted set
        unsigned type:4;
        // 编码方式,raw,int,ht,zipmap,linkedlist,ziplist,intset,skiplist
        unsigned encoding:4;
        // LRU 时间（相对于server.lruclock）
        unsigned lru:22;
        // 引用计数
        int refcount;
        // 指向对象的值,实际存储数据的地方,不同数据类型对应不同的数据结构
        void *ptr;
    } robj;

为了节约内存空间,ptr指向的数据结构可能为因为值数量的增加而改变,例如zset, 如果值个数少于128,并且值长度小于64,
redis采用ziplist来保存(更加节约内存,数据规模小查找也快),当数量越来越多超过128个则会转换成skiplist来存储(数据规模变大了,查找开始变得重要了).

所以针对每种数据类型,都有编码的选择(采用什么数据结构存储),编码的转换(随着规模扩大等原因,采用更适合的数据结构),

## redis事务
redis中的事务就是讲多个命令打包执行,中间不会穿插其它命令,所以隔离级别为serilazation
例子:

    127.0.0.1:6379> multi //开启事务
    OK
    127.0.0.1:6379> set a 5555 //命令放入事务队列
    QUEUED
    127.0.0.1:6379> set a 6555 //命令放入事务队列
    QUEUED
    127.0.0.1:6379> get a //命令放入事务队列
    QUEUED
    127.0.0.1:6379> exec //执行事务
    1) OK
    2) OK
    3) "6555"
    127.0.0.1:6379>

### 带watch的事务  
WATCH 命令用于在事务开始之前监视任意数量的键：当调用EXEC 命令执行事务时，如果任意一个被监视的键已经被其他客户端修改了，
那么整个事务不再执行，直接返回失败。

例子:

    127.0.0.1:6379> watch a
    OK
    127.0.0.1:6379> multi
    OK
    127.0.0.1:6379> get a
    QUEUED
    127.0.0.1:6379> exec //执行这个命令之前,开启另一个client 执行set a 99999
    (nil) //事务执行失败
    127.0.0.1:6379>

### redis事务ACID
Redis 事务保证了其中的一致性（C）和隔离性（I），但并不保证原子性（A）和持久性（D）

    原子性(A):单个命令是原子的,但是多个命令不是,如果执行事务时宕机,Redis不会进行任何的重试或者回滚动作
    一致性(C):RDB模式,AOF模式
    隔离性(I):通过单线程模型保证
    持久性(D):AOF采用一个后台线程保存事务数据,不能保证事务执行了就代表持久化了


## redis持久化
redis虽然是内存数据库,但也提供了持久化方案

1.RDB
配置:

    save 900 1
    save 300 10
    save 60 10000
保存内存快照到磁盘,间隔时间大,容易丢失数据,有点是启动加载快. 可以理解为拷贝mysql的数据文件

2.AOF
配置:

    # appendfsync always
    appendfsync everysec
    # appendfsync no
记录命令文本到日志中,粒度要小写,但是频繁的同步到磁盘效率也很低, 可以理解为mysql的binlog

在mongodb中开启journal日志,没说最性能有很大影响,磁盘是顺序写的.但是redis这里说影响很大,说明redis对响应要求很高


# 总结

    1.redis是一个Key-Value内存数据库,也提供持久化支持. 
    2.命令格式: command + key + xxx , key是用来查找值的, 或得值对象后还可以进行进一步操作,比如zrange等等



