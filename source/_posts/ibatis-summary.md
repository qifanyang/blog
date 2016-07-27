---
title: ibatis-summary
date: 2016-07-25 17:33:44
tags: ibatis
---

## 和apache dbutil比较
1.更加强大的结果集映射(关联映射,嵌套查询,嵌套查询结果)  
2.动态SQL(使用OGNL表达式)  
3.更加友好,更加细粒度的控制,比如TypeHandler,aliasName  

## 核心类
Configuration(配置中心),Executor,SqlSession,MapperProxy(helper class),MappedStatement

### MappedStatement
MappedStatement对应mapper.xml中的一个select或者其它声明,一个声明主要包含以下属性:  
1.声明id,全局唯一  
2.sqlCommandType,查询类型select,update...  
3.parameterMap,已经废弃  
4.resultMaps,  用来描述如何从数据库结果集中来加载对象  
5.sqlSource,要执行的sql,有StaticSqlSource(没有<if test="id != null"/>),有动态DynamicSqlSource(包含<if ...),用于创建BoundSql    

### SqlSession
默认实现DefaultSqlSession,主要包含属性Configuration和Executor,负责执行各种数据库操作  

### Executor
执行MappedStatement,执行缓存策略(如果是CachingExetutor),具体数据库操作有delegateExecutor完成  

### StatementHandler
Executor根据ms创建StatementHandler,然后逻辑交给改handler处理  
1.StatementHandler , 传统的JDBC代码在这里实现,创建Statement,设置超时,设置参数值(typeHandler在这里处理)  
2.BoundSql包含执行的sql语句,参数信息(parameterMappings),参数值(parameterObject),这些信息执行数据库操作  
3.Statement设置完毕后,ps.execute(),执行sql  
4.结果集映射,DefaultResultSetHandler负责结果集映射,ibatis最强大的结果集映射在这里实现  

### ResultSetHandler
spring jdbc执行完sql之后,用得比较多是交由RowMapper处理ResultSet,一般将结果集映射到具体的java对象,如果对象属性有嵌套
关系需要使用硬编码从结果集中取数据构建包含嵌套属性的对象,而ibatis结果集映射将硬编码抽象成配置  

ResultSetHandler使用ms的ResultMap来处理结果集,  
一个需要关联多张表的查询,查询结果对象如果包含一个引用类型属性,association可以准确的将结果集中的值设置到对应的引用属性中  
<association>用来查询嵌套属性,关联元素处理“有一个”类型的关系.  
嵌套查询,单独使用一个查询  
    <association property="author" column="author_id" javaType="Author" select="selectAuthor"/>  
嵌套查询结果,查询使用连接  
    <association property="author" column="blog_author_id" javaType="Author" resultMap="authorResult"/>  
collection类似association,是用来关联元素处理“有多个”类型的关系.嵌套查询和嵌套结果查询同上  

有时一个单独的数据库查询也许返回很多不同 (但是希望有些关联) 数据类型的结果集,鉴别器根绝返回值选择resultMap  

    <discriminator javaType="int" column="vehicle_type">
        <case value="1" resultMap="carResult"/>
        <case value="2" resultMap="truckResult"/>
        <case value="3" resultMap="vanResult"/>
        <case value="4" resultMap="suvResult"/>
    </discriminator>  


### ResultMap
负责ResultSet到java对象的映射,指定数据库列属性如何映射到java对象,ResultMap中type属性指定要返回的对象类型,  
当mappedStatement使用ResultType,ibatis默认创建的ResultMap,id为namespace-Inline,type为resultType类型,自动创建映射  
所以ResultType只是一种更加自动化的ResultMap实现,具体实现都是通过ResultMap.  

### TypeHandler
当ResultSet中具体一列的值映射到java属性时,需要用TypeHandler负责转换,同理写java到数据库时也要转换,

## ibatis中用到的设计模式
1.静态代理  
Cache实现中大量使用静态代理,比如LruCache,实际数据存储代理给PerpetualCache,LruCache包装PerpetualCache,结合java lib  
提供的LinkedHashMap实现Lru功能,多重Cahce可以串在一起具备每个Cahce的功能,当然要注意包装层次    

2.动态代理  
在使用Mapper接口查找MappedStatement,并执行查询时动态创建sqlSession.select(steamentId...),当然接口的包名+类名+方法名  
要和mapper.xml中namespace id和steatment id对应,jdk动态代理会缓存Class所以开销不大  

3.effective java builder模式  
因配置文件参数较多,构建配置相关的对象,比如MappedStatement使用Builder模式,方便校验哪些参数必须配置的,不干扰业务对象  

  






