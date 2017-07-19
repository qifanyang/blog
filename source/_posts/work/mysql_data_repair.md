## why
    工作中会遇到使用sql修复数据,或者批量导入数据到新表中,如果需要执行很多语句,但是某些语句执行失败  
比如,不满足唯一索引,如果素有的操作在一个事务中那就不会出现一致性问题.

## test

~~~
-- ----------------------------
--  Table structure for `a`
-- ----------------------------
DROP TABLE IF EXISTS `a`;
CREATE TABLE `a` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `age` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;

-- ----------------------------
--  Records of `a`
-- ----------------------------
BEGIN;
INSERT INTO `a` VALUES ('1', '22');
INSERT INTO `a` VALUES ('1', '22');
COMMIT;
~~~

    插入数据,如果主键重复,第一条记录会被插入,而且主键自增浪费一个.  
    期望异常时回滚事务,使用存储过程实现  

~~~
delimiter //
DROP PROCEDURE
IF EXISTS sp_test ; CREATE PROCEDURE sp_test()
BEGIN

DECLARE EXIT HANDLER FOR SQLEXCEPTION ROLLBACK ;
DECLARE EXIT HANDLER FOR NOT found ROLLBACK ;
DECLARE EXIT HANDLER FOR SQLWARNING ROLLBACK ; 

START TRANSACTION ;  

INSERT INTO `a` VALUES ('1' , '232') ; 
INSERT INTO `a` VALUES ('1' , '33') ; 
COMMIT ;
END//
delimiter ;

CALL sp_test();
~~~
    因为需要在navicat中执行,上面使用delimiter重新定义了statement定界符  
    还定义了异常处理器,当异常时回滚事务.需要注意事务语句块不能包含删除表,改表结构等语句  
    因为导致提交事务,导致无法回滚  


## 参考链接
存储过程定义  
https://dev.mysql.com/doc/refman/5.7/en/stored-programs-defining.html  
handler语法  
https://dev.mysql.com/doc/refman/5.7/en/declare-handler.html  
导致implicitly提交事务  
https://dev.mysql.com/doc/refman/5.7/en/implicit-commit.html  
