---
title: mysql-client-protocol-yy
date: 2016-08-21 16:47:50
tags: mysql protocol
---

## 协议
编写网络程序都需要定义协议,网络程序中定义的协议位于TCP/IP 四层协议中应用层,比如http  
编写数据库程序通过JDBC驱动访问mysql server也需要使用协议,JDBC驱动中已经实现网络交互协议(类MysqlIO完成通讯)  
mysql和服务器通信采用单双工模式,发送消息后必须等待server返回数据后再进行下一步操作,读数据的时候不会发送数据  


## mysql协议包结构
header(4 byte)=packageLength(3 byte)+multiPackageSeq(1 byte)  
packageLength用于指明包大小
multiPackageSeq用于标识数据包序号,可以从来调试通信  

首先从socket.inputstream中阻塞读取4字节,把头部读取出来,然后再解析包长度,再次阻塞读取包长度指明的数据,  
然后将一个完整的数据包交给上层处理  

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










