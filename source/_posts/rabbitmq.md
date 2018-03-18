# rabbitmq

## 常用参数
channel.queueDeclare(QUEUE_NAME, false, false, false, null);

  durable: 是否持久化(重复声明queue改变配置并不会生效,因为queue已经存在),  
mqserver重启后message仍然存在,如果突然宕机可能会丢失最新的message,mq-server并不会  
在每次收到message时都调用fsync同步数据到磁盘,如果不允许数据丢失使用publisher-comfirm机制  

  autoAck=true,当consumer收到消息后自动删除,如果worker业务逻辑没有处理完,  
可能会丢失消息,所以需要使用autoAck=false,然后手动channel.basicAck(envelope.getDeliveryTag(), false);  

int prefetchCount = 1;
channel.basicQos(prefetchCount);
mq-server收到消息直接使用round-robin策略分发给consumer,不会考虑consumer是否来得及  
处理,通过设置qos告诉mq-server需要等到consumer ack才分发下一个消息

## exchange , queue
  在RabbitMQ中,producer不直接发送消息到queue中,producer也不知道message将会被发送  
到哪个queue中,RabbitMQ引入了exchange,用于接收消息并分发到对应的queue中.  
  exchange不同类型有不同分发规则,类型有:direct,topic,headers,fanout  
  fanout:使用广播,忽略routeKey  
  direct:使用routKey分发,所有routKey一样的话,效果同fanout    
  topic:类似direct的routeKey,可以使用通配符,更加强大,长度最多255bytes.使用dot隔开的单词  
  作为routeKey,consumer绑定queue时通配符*代表一个单词,#代表0个或多个单词  
  
  创建exchange,channel.exchangeDeclare("logs", "fanout");  
  exchange于queue使用channel.queueBind(queueName, "logs", "");建立关系  





