## binlog
二进制日志用于mysql奔溃做数据恢复,复制也是通过二进制日志实现
二进制日志默认没有开启,使用配置log-bin=xxx来开启

show binary logs; 查看二进制日志
show variables like 'datadir%'; 查看数据目录,不然要从配置文件中查看
show variables like 'log_bin%'; 查看二进制日志配置

## 主备配置
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
start slave;
stop slave;


		 


