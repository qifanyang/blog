---
title: mysql-client-protocol-yy
date: 2016-08-21 16:47:50
tags: mysql protocol
---

## 协议
编写网络程序都需要定义协议,网络程序中定义的协议位于TCP/IP 四层协议中应用层,比如http  
编写数据库程序通过JDBC驱动访问mysql server也需要使用协议,JDBC驱动中已经实现网络交互协议(类MysqlIO完成通讯)  
mysql和服务器通信采用单双工模式,发送消息后必须等待server返回数据后再进行下一步操作,读数据的时候不会发送数据  

## mysql-client-server-protocol
用于MySQL客户端和服务器通信,可以用来写:  
* driver(connector)
* MySQL Proxy
* communication between master and slave replacation server

## 协议数据类型
|Type|	Description  |
|-------|----------|
|int<1> |	1 byte Protocol::FixedLengthInteger  |
|int<2>	|2 byte Protocol::FixedLengthInteger  |
|int<3>	|3 byte Protocol::FixedLengthInteger  |
|int<4>	|4 byte Protocol::FixedLengthInteger  |
|int<6>	|6 byte Protocol::FixedLengthInteger  |
|int<8>	|8 byte Protocol::FixedLengthInteger  |
|int<lenenc>|	Protocol::LengthEncodedInteger | 
|string<lenenc>|	Protocol::LengthEncodedString | 
|string<fix>|	Protocol::FixedLengthString  |
|string<var>|	Protocol::VariableLengthString: | 
|string<EOF>|	Protocol::RestOfPacketString  |
|string<NUL>|	Protocol::NulTerminatedString  |
### Integer
整形数据分为固定长度整形和长度被编码的整形  
* 固定长度整形,比如占用一个字节的整形,使用符号int<1>表示,同理还有int<2>,int<3>
* 长度被编码的整形int<lenenc>,表示整形占用的字节数量不固定,根据第一个字节的值来确定数据的字节数量,小于251表示单字节,等于252表示三字节(第一个字节标识,后两个字节为数值内容)
    * if value < 251 , it is stored as a 1-byte Integer, 对于小于251的值,相对于int节约了3字节
    * if value >= 251 and < 2^16 , it is stored as fc(252)+2-byte Integer, 对于小于2^16的数值还是比int节约一个字节
    * if value >= 2^16 and < 2^24 , it is stored as fd+3-byte integer, 使用了4字节,但是存值只用了三字节
    * If the value is ≥ (224) and < (264) it is stored as fe + 8-byte integer.  
        *fa       -- 250 单字节
        *fc fb 00 -- 251 两字节

对于长度被编码的整形而言,如果值很多大于2^24小于2^32,那么需要采用8字节就很浪费了,所以对于很多长度小于251,偶尔有大数据的协议,这种方式可以节约带宽3字节  

需要注意mysql使用小端字节序(低位低字节),java默认采用大端字节序(低位高字节-->存储中低位地址数据在值中的高位),比如int<3> 01 00 00 读取后要反转 值为1  
java Integer类中有方法Integer.reverseBytes()来反转字节,注意Integer.reverse()是反转bit,不要使用错了  
### String
string类型是一个字节数组序列,有如下格式:  
* 固定长度string<fix>,表示字节数组长度为fix
* 空结束符字符串string<NULL>,使用[00]表示字符串的结束
* 变长长度字符串string<var>,前面一个int<?>表示后面的string数组长度
* 长度编码字符串string<lenenc>,字符串长度使用int<lenenc>
* string<EOF>,这个用在数据的末尾,表示从当前位置到结束都是该string的数据


## mysql协议包结构
header(4 byte)=packageLength(3 byte)+multiPackageSeq(1 byte)  
packageLength用于指明包大小
multiPackageSeq用于标识数据包序号,分包序列号,每次发送新命令时都重置该值    

根据header可以长度最大值为2^24-1,所以包最大可以为16M,更大的数据需要通过拆包发送  

首先从socket.inputstream中阻塞读取4字节,把头部读取出来,然后再解析包长度,再次阻塞读取包长度指明的数据,然后将一个完整的数据包交给上层处理  

比如发送16 777 215 (224−1) bytes looks like:  
*  ff ff ff 00 ... //第一个个包长度为0xffffff, 序列为0
*  00 00 00 01     //数据刚好被第一个包发送完毕,所以第二个包长度为0x000000, 序列号为1

## Generic Respone Packets
对于大多数客户端发送给服务器的命令,服务器返回OK_PACKET , ERR_PACKET , EOF_PACKET  
详细介绍:http://dev.mysql.com/doc/internals/en/generic-response-packets.html  

## 客户端和服务器连接阶段
http://dev.mysql.com/doc/internals/en/connection-phase.html  
* 交换各自支持的功能,比如ssl,压缩等
* 认证

当客户端connect服务器时,服务器可能返回ERR_PACKET,或者返回HANDSHAKE_PACKET,客户端使用Handshake_Respone_Packet回应  
Initial_Handshake_packet有v9和v10两个版本,v10兼容v9, 在v9的基础上新增返回数据  
Handshake_response也分为 Protocol::HandshakeResponse41 or Protocol::HandshakeResponse320. 新的server用41,老server用320  

握手阶段比较重要就是确定服务器和客户端具备的能力capability,和认证方式,mysql5.5以后支持插件认真,需要指出插件名  

## text protocol command
文本协议有很多,COM_PING,COM_QUIT,等
### COM_QUERY
用于向服务器发送基于文本的查询,这个查询会立即被执行,payload:
* 1              [03] COM_QUERY  
* string[EOF] the query the server shall execute

响应:lenenc-int     number of columns in the resultset  
1. 如果列的数量为0,表示该包为OK_PACKET  
2. 如果列的数量值是无效的(-1),该包可能是ERR_PACKET或者LOAD_INFILE_Request
3. text resultset, 包含column type, column define, text resultset row  这里有点类似binlog 中 row event和table map event合体




## msyql handshake握手过程
### 读取响应
当一个网络程序连接到mysql server,服务器首先返回自身信息,比如:
protocolVersion:协议版本,服务器功能会越发复杂,新版本的客户端可以兼容所以老的server端,协议版本可以告诉客户端做响应处理,  
serverVersion:服务器版本(5.6.28-log),服务器版本可能变化比较快(功能变化)    
threadId:在server端的线程id  
seed:用于验证账号密码,sever端随机生成种子(使用种子防止重放),SHA(SHA(SHA(password))+seed),使用两次hash是应为数据库中  
使用hash存储密码,验证密码也是传送的hash结果,更加安全  
serverCharsetIndex:服务器编码  
serverStatus:服务器状态  
serverCapabilities:服务器功能,这个比较重要了,该字段指出服务器具备哪些功能,这个字段应该设置为可扩展的,应为功能只会越来越多  
比如把最高位保留给扩展,1表示下一个字段还是为serverCapabilities,0表示没有下一字段作为serverCapabilities了.  
authPluginDataLength:插件数据,返回插件名字,告诉客户端使用what plugin进行接下来的处理  

### 发送请求
sendBuf.writeInt(clientParam);客户端根据服务器serverCapabilities值和connection属性设置,创建clientParam,发送必要的信息  
sendBuf.writeInt(16777215);//writeLong(this.maxThreeBytes);
sendBuf.writeByte(33);//CharsetMapping.MYSQL_COLLATION_INDEX_utf8;
sendBuf.writeBytes(new byte[23]);
 //写user
 sendBuf.writeBytes(user.getBytes());
 sendBuf.writeByte(0);
//wite toserver length
sendBuf.writeByte(0);
 //write database
sendBuf.writeBytes(database.getBytes());
sendBuf.writeByte(0);
sendBuf.writeBytes("mysql_native_password".getBytes());
sendBuf.writeByte(0);
发送数据到服务器完成认证(这里没有密码验证),完成握手


## 执行select
发送sql语句,然后读取响应结果,创建结果集,很烦躁的一个步骤,构建结果集先根据返回列数构建Field,然后读取每行数据构建ResultSetRow,  
从结果集中resultSet.getLong(1),就是从一个ByteArrayRow的byte[]数组中构建结果  


## tips
发送一个string,编码成byte[],先写字符串长度,再写bytes[], 长度值一般使用int,如果字符串很短那么浪费了3字节,如果很长那么不支持  
采用类似UTF-8可变字符编码的形式:
```java
    final void writeFieldLength(long length) throws SQLException {
        if (length < 251) {
            writeByte((byte) length);
        } else if (length < 65536L) {
            ensureCapacity(3);
            writeByte((byte) 252);
            writeInt((int) length);
        } else if (length < 16777216L) {
            ensureCapacity(4);
            writeByte((byte) 253);
            writeLongInt((int) length);
        } else {
            ensureCapacity(9);
            writeByte((byte) 254);
            writeLongLong(length);
        }
    }

final long readFieldLength() {
        int sw = this.byteBuffer[this.position++] & 0xff;

        switch (sw) {
            case 251:
                return NULL_LENGTH;//标识

            case 252:
                return readInt();

            case 253:
                return readLongInt();

            case 254:
                return readLongLong();

            default:
                return sw;
        }
    }
```
上面这种编码对于使用int作为长度,于字符串长度小于251节约空间3字节,对于小于65536长度节约1字节,当大于65535会更加浪费,如何使用根据具体情况


## 分包
在设计通信协议时,一般使用int(4 byte)作为包长度字段,如果一个消息数据大于int则需要分多次发送,这种需求可以在应用层处理,当知道发送数据很大  
在应用层将消息拆分,接收端然后再组装.更好的实现方法应该在网络层,根据发送数据数据的大小自动拆包,这里的拆包类似IP包分包,应用层数据包  
拆分,可以理解为应用层的报文单元.  

mysql如何判断当前数据包后面还有一个数据包,是根据最大包长度,如果当前数据包长度等于最大长度,则说明紧接着还有一个数据包,  
发送数据包也是根据最大长度拆分,有种特殊情况是发送的时候刚好按最大包发送完毕,会导致进接着的下一个包长度为0,但是为了读取  
时能正确处理,还是要发送一个包体长度为0的空包过去  

IP包拆分,是根据header中DF,MF









