---
title: mysql-mvcc
date: 2016-08-07 12:42:08
tags: mysql
---
## mysql多版本并发控制(MVCC)
高性能mysql这本书讲到,mysql并不只是实现了行级锁,为了提高并发性还实现了MVCC,MVCC在很多情况下避免加锁  
MVCC通过保存数数据在某个时间点的快照(snapshot)来实现(第一个select statement),每个事务看到的数据是  
一致的,根据事务的开始时间不同,每个事物对同一张表同一时刻看到的数据是不一样的(consistent read)  

### MVCC实现
![](../img/mysql-mvcc.png)


在使用以上规则测试时,要注意mysql consistent read机制的存在,不然select 返回删除标识小于当前事务ID的记录没法解释  
http://dev.mysql.com/doc/refman/5.7/en/glossary.html#glos_consistent_read  
http://dev.mysql.com/doc/refman/5.7/en/innodb-consistent-read.html  

## consistent read
对于repeatable_read isolate level, the first select create a snapshot of the database state  
对于read_committed isolate level, the snapshot is reset to the time of each consistent read operation.  
读一致性的存在,在事务中不用对其访问的表加锁,其它的事务可以同时更新数据  

当一个事务A创建一个读一致性(执行select),事务B删除一个数据,事务A然后查询仍然能够查询到事务B删除的数据,就像没有被B删除一样  
实现原理就是事务A创建读一致性时,创建了一个当前时间点的数据库快照,而事务B删除数据,会将老的数据写入到undo log中,事务A查询  
数据是返回的undo log中的数据,这类似copyOnWrite,如同linux实现数据快照一样,只有当修改时才修改快照数据,并拷贝原始数据  

## redo log(文件ib_logfile0,ib_logfile1)
A disk-based data structure used during crash recovery, to correct data written by incomplete transactions.  
During normal operation, it encodes requests to change InnoDB table data, which result from SQL statements   
or low-level API calls through NoSQL interfaces. Modifications that did not finish updating the data files  
before an unexpected shutdown are replayed automatically.

The redo log is physically represented as a set of files, typically named ib_logfile0 and ib_logfile1.  
The data in the redo log is encoded in terms of records affected; this data is collectively referred to as redo.  
The passage of data through the redo logs is represented by the ever-increasing LSN value. The original 4GB   
limit on maximum size for the redo log is raised to 512GB in MySQL 5.6.3.

The disk layout of the redo log is influenced by the configuration options innodb_log_file_size,   
innodb_log_group_home_dir, and (rarely) innodb_log_files_in_group. The performance of redo log operations is also   
affected by the log buffer, which is controlled by the innodb_log_buffer_size configuration option.


## undo log
A storage area that holds copies of data modified by active transactions. If another transaction needs to see the  
original data (as part of a consistent read operation), the unmodified data is retrieved from this storage area.  

By default, this area is physically part of the system tablespace. In MySQL 5.6 and higher, you can use the   
innodb_undo_tablespaces and innodb_undo_directory configuration options to split it into one or more separate   
tablespace files, the undo tablespaces, optionally stored on another storage device such as an SSD.  

The undo log is split into separate portions, the insert undo buffer and the update undo buffer.  


## binary log
用于主从,slave重做数据,master也可以根据binary log在恢复数据,但是事务没有提交的话binary log中没有记录  

mysql server崩溃重启,如果事务已经写入binary log则会提交事务,如果没有提交或者处于XA PREPARE状态,则回滚,分布式事务如果有参与者提交了,则PREPARE状态事务应该提交  

## 2PC异常恢复

在crash recover之后，外部应用程序可能会遇到以下几种情况：  
情况一：分布式事务对应的MySQL实例，部分完成prepare，部分未完成prepare。此时直接回滚完成prepare的实例即可。n_prepared <Total Nodes (处于prepare状态的节点数量要小于参与分布式事务的所有节点总数)。  
 

情况二：分布式事务对应的MySQL实例，全部完成prepare，未开始进行commit。此时即可提交此事务，也可回滚此事务(根据分布式事务原理，所有节点都完成prepare，应该提交)。n_prepared = Total Nodes。  
 

情况三：分布式事务对应的MySQL实例，全部完成prepare，并且部分节点已经完成commit。此时应该提交该事务处于prepare状态的节点。n_prepared 小于 Total Nodes。对比情况三与情况一，仅仅通过prepare节点的数量无法区分，  
因此应用程序需要在prepare完成之后记录日志(此时，应用程序起着事务协调者(Transcaction Coordinator)的角色，而根据MariaDB WorkLog#132[5]的说法，TC角色是可以进行”middle engine”优化的，不需要prepare过程，  
所有MySQL节点xa prepare返回之后，应用程序直接写commit标识即可，然后再对每个MySQL节点进行xa commit操作。)，从而用于区分情况一与情况三。
 
情况四：分布式事务对应的MySQL实例，全部完成commit。此时事务已经提交成功，xid不会出现在执行xa recover的任一个节点。不需要特殊处理。
情况五：未记录任何prepare日志。那么所有的事务，在各个存储引擎的crash recover时，都会被回滚，不需要外部特殊处理。  

## 分布式事务
企业级java开发使用JTA实现分布式事务,j2ee application server实现了JTATransactionManager,可以直接使用应用服务器的实现  
也可以单独使用JTA实现,比如atomikos等,http://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-jta.html  

JMS也实现了XA协议,也可以参数分布式事务  


## JTA
j2ee事务规范,可以实现分布式事务,jta提供相应接口,atomikos实现了独立于appllication server的JTA实现  
Java Transaction API (JTA) specifies standard Java interfaces between a transaction manager and the parties involved in a distributed transaction system: the resource manager,  
the application server, and the transactional applications.  

## JPA
The Java Persistence API provides a POJO persistence model for object-relational mapping. The Java Persistence API was developed by the EJB 3.0 software expert group as part of JSR 220,  
but its use is not limited to EJB software components. It can also be used directly by web applications and application clients, and even outside the Java EE platform, for example, in Java SE applications  

JPA为实现ORM映射提供了一个POJO持久化实体模型,JPA在EJB3.0中开发,但是不受限与EJB组件,能够直接在web应用程序中使用  

spring data JPA 提供更方便的方式使用jpa,只需要写出接口,spring 创建动态代理对象,根据方法名字来自动生成查询语句,这和mybatis mapper接口类似  
spring data JPA处理复查查询采用自动生成代理类并不现实,可以实现接口,然后在实现类中书写复杂sql,对于一个复杂sql,处理结果集映射就比较麻烦了,  
所以引入mybatis吧,所以根据项目情况决定采用JPA还是mybatis吧  

JPA是一种规范,为了统一Hibernate,TopLink,JDO各种ORM,

https://www.ibm.com/developerworks/cn/opensource/os-cn-spring-jpa/  









