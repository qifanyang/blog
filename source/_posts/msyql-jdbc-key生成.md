---
title: msyql jdbc key生成
date: 2016-09-23 11:06:14
tags:
---

## jdbc获取主键
statement.execute(sql, java.sql.Statement.RETURN_GENERATED_KEYS)或者conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS)  
然后通stmt.getGeneratedKeys(),根据查询返回的结果集创建一个新的主键结果集,使用访问结果集的方式来遍历主键    

## 结果集生成
当执行sql后(select, insert, update, delete),根据返回columnCount来确定构造结果集  
如果columnCount == 0, 则是更新操作,读取更新产生的主键,或者被更新的id  
如果columnCount == -1, 表示sendFileToServer  
绕过columnCount > 0, 则表示查询,返回列数量   

## 单条数据插入获取主键
插入数据,返回结果columnCount数量为0, 读取updateId(插入数据的主键id),updateCount(插入条数)  

~~~
        Connection conn = getConnection();
        String sql = "INSERT INTO user (age) VALUES (2)";
        PreparedStatement ps = conn.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS);
    
        ps.executeUpdate();

        ResultSet generatedKeys = ps.getGeneratedKeys();
        while(generatedKeys.next()){
            System.out.println("生成主键 id = " + generatedKeys.getLong(1));
        }

~~~

## 批量插入数据

和插入单条数据类似,只是updateCount为多条,而updateId为插入的第一条数据主键id,接着其它的数据主键id依次递增  
所以mysql server要保证同一sql批量插入的数据是连续的,或者主键生成是连续的  

mysql server使用AUTO_INC LOCK 还保证批量插入数据是连续的,当一个事务t插入数据带有 AUTO_INCREMENT列的数据时  
其它事务插入数据必须等待事务t释放AUTO_INC LOCK, 所以mysql主键生成还是有效率瓶颈的,因为AUTO_INC LOCK是一个表  
级锁,所以有的实现采用多张表来生成主键,减少等待锁的时间  

https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-auto-inc-locks  

## 更新
jdbc更新没法返回被更新的主键值

## spring jdbc 获取主键
~~~
    public long save(final String insertSql, final Object[] args) {
        KeyHolder keyHolder = new GeneratedKeyHolder();
        jdbcTemplate.update(new PreparedStatementCreator() {
            public PreparedStatement createPreparedStatement(Connection conn) throws SQLException {
                PreparedStatement ps = conn.prepareStatement(insertSql, Statement.RETURN_GENERATED_KEYS);
                if (args != null && args.length > 0) {
                    for (int i = 1; i <= args.length; i++) {
                        ps.setObject(i, args[i - 1]);
                    }
                }
                return ps;
            }
        }, keyHolder);
        return keyHolder.getKey().longValue();
    }
~~~



## jdbc砍掉超时sql
执行查询时,在本地开启一个定时任务,如果超过设置时间没有结果集返回,则定时任务新建一个数据库连接,发送语句kill query connectionId干掉查询  
mysql server也可以使用事件来执行长时间任务检查  
https://help.aliyun.com/knowledge_detail/41723.html#5  
每60分钟执行一次超时任务检查  
~~~
create event my_long_running_trx_monitor
on schedule every 60 minute
starts '2015-09-15 11:00:00'
on completion preserve enable do
begin
  declare v_sql varchar(500);
  declare no_more_long_running_trx integer default 0; 
  declare c_tid cursor for
    select concat ('kill ',trx_mysql_thread_id,';') 
    from information_schema.innodb_trx 
    where timestampdiff(minute,trx_started,now()) >= 60;
  declare continue handler for not found
    set no_more_long_running_trx=1;
 
  open c_tid;
  repeat
    fetch c_tid into v_sql;
 set @v_sql=v_sql;
 prepare stmt from @v_sql;
 execute stmt;
 deallocate prepare stmt;
  until no_more_long_running_trx end repeat;
  close c_tid;
end;
~~~

## prepareStatement
服务端预处理conn.prepareStatement(...)向服务器发送COM_STMT_PREPARE命令,服务器创建预处理,返回声明id, fieldCount,parameterCount  



  
