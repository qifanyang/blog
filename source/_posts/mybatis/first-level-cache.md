# mybatis一级缓存

## 场景
  Trade表为主表,TradeItem为子表, 为一对多关系,当操作TradeItem时,需要验证Trade状态等.  
这时在验证方法中会有一次查询Trade的操作,使用spring jdbc为了减少一次查询,总是在程序入口  
查询trade,把trade作为参数传给验证方法.当使用mybatis时,就不用传参了,需要时直接调用查询  
一级缓存会根据sql语句和条件自动缓存,方法会更简洁,验证时不必先去写查询了