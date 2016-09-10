## binlog
biglog是一系列包含mysql数据更新信息的文件和一个包含所有二进制日志文件名的索引文件构成      
二进制日志用于mysql奔溃做数据恢复,复制也是通过二进制日志实现  
二进制日志默认没有开启,使用配置log-bin=xxx来开启,binlong要在事务执行提交才会执行写入biglog文件   

show binary logs; 查看二进制日志
show variables like 'datadir%'; 查看数据目录,不然要从配置文件中查看
show variables like 'log_bin%'; 查看二进制日志配置

## 主备配置(主备都没有数据)
1.在主库和备库上创建帐号
GRANT replication SLAVE , replication client ON *.* TO repl@'192.168.2.%' IDENTIFIED by '123456';
GRANT replication SLAVE , replication client ON *.* TO repl@'192.168.2.%' IDENTIFIED by '123456' WITH GRANT OPTION;
WITH GRANT OPTION 这个选项表示该用户可以将自己拥有的权限授权给别人

2.主库配置修改
修改my.ini配置,添加:
log_bin=mysql-bin  //启用二进制日志,日志文件名前缀为mysql-bin
server_id=10 //服务器id,主备中该值不能重复

3.备库配置修改
log_bin=mysql-bin //二进制日志名字默认采用机器名字,这里明确指出比较好
server_id=2
relay_log=D:\Program Files\MySQL\mysql-relay-bin
log_slave_updates=1 //备库重放的事件也将记录到自己的二进制日志中
read_only=1 //备库一般都是只读
replicate_do_db=test //从库只同步指定的数据库数据
#replicate_ignore_db=mysql //还可以忽略不同步数据库

上面的配置其实只有server_id是必须的,其它都有默认配置值

3.告诉备库如何从主库重放其二进制日志
change master to master_host='192.168.2.103',master_user='repl',master_password='123456',master_log_file='mysql-bin.000001',master_log_pos=0;
start slave;//开始复制
stop slave;//停止复制

## 主备配置(主库有数据,备库没有数据)
1.主库执行FLUSH TABLES WITH READ LOCK,阻止innodb提交数据(要保持session,断开的话会释放锁)

2.记录主库二进制日志文件名和在二进制日志中的偏移量(slave同步时从该位置开始读取)

3.继续持有锁,阻止数据被改变,然后备份数据
3.1 使用innodb的话,建议是用mysqldump, 还可以通过拷贝数据文件,但是手册中建议innodb不这么做,我们一般都是用innodb,所以还是采用mysqldump

4.数据快照已经拿到,同步的二进制日志文件名和位置也已经拿到,释放主库读锁, unlock tables

5.mysql -h master < fulldb.dump 将数据导入slave中

6.告诉备库如何从主库重放其二进制日志
change master to master_host='192.168.2.103',master_user='repl',master_password='123456',master_log_file='mysql-bin.000001',master_log_pos=0;
start slave;

## binlog格式
There are two types of binary logging:  
Statement-based logging: Events contain SQL statements that produce data changes (inserts, updates, deletes)  
Row-based logging: Events describe changes to individual rows  

log event 类型  
第一个event叫FORMAT_DESCRIPTION_EVENT,描述文件格式版本  
中间所有event归为一类,数据更新等    
最后一个evnet叫ROTATE_EVENT,指明下一个binlog file文件  

binlog是二进制格式的,可以使用mysqlbinlog转换为可以阅读的形式  

file strcuture=magic number(4 byte)+log event(data modification)  
magic nunber = 0xfe 0x62 0x69 0x6e(0xfe 'b''i''n')  
log event = header bytes + data bytes
The first event is a descriptor event that describes the format version of the file (the format used to write events in the file).  
The remaining events are interpreted according to the version.  
The final event is a log-rotation event that specifies the next binary log filename.  
log event中第一个event和最后一个event不是数据modification,第一个event叫做format description event  

v4 formart description event,阅读采用(offset+length),注意mysql协议中多字节采用小端读取(高位高字节)    
+=====================================+
| event  | timestamp         0 : 4    |
| header +----------------------------+
|        | type_code         4 : 1    | = FORMAT_DESCRIPTION_EVENT = 15
|        +----------------------------+
|        | server_id         5 : 4    |
|        +----------------------------+
|        | event_length      9 : 4    | >= 91  
|        +----------------------------+
|        | next_position    13 : 4    |
|        +----------------------------+
|        | flags            17 : 2    |
|        +----------------------------+
|        | extra_headers   19 : x-19  | Note: The extra_headers field does not appear in the FORMAT_DESCRIPTION_EVENT or ROTATE_EVENT header
+=====================================+
| event  | binlog_version   19 : 2    | = 4  
| data   +----------------------------+
|        | server_version   21 : 50   |
|        +----------------------------+
|        | create_timestamp 71 : 4    |
|        +----------------------------+
|        | header_length    75 : 1    |
|        +----------------------------+
|        | post-header      76 : n    | = array of n bytes, one byte per event  
|        | lengths for all            |   type that the server knows about  
|        | event types                |
+=====================================+

不同版本event structure不一样(有v1,v3,v4),所以这里有风险的,binlog文件解析器会随着结构改变而改变,所以解析器要指明mysql版本  


1.查看mysql binlog  
mysqlbinlog --no-defaults -v ../data/mysql-bin.000001 > xxx.sql  
```sql
```

## binlog文件增删改测试
1.更新数据,index最大的binlog文件被修改,说明更新数据的binlog被写入到index最大的文件中  




## 资料
http://dev.mysql.com/doc/internals/en/binary-log-overview.html  
http://dev.mysql.com/doc/internals/en/binary-log-versions.html  
http://dev.mysql.com/doc/internals/en/binlog-event.html  
https://dev.mysql.com/doc/internals/en/event-data-for-specific-event-types.html  






		 


