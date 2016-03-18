---
title: api调用次数控制
date: 2016-02-14 14:41:33
tags: algorithm
---

## 使用场景
	在调用第三方接口时,比如微信接口,都有会有调用次数限制,比如企业用户每日调用次数更多,
	比如在AWS中各种服务api调用次数也有限制,所以在做接口开发,api接口次数调用控制很重要
	

## api调用次数控制
	
	//代码片段,每秒只能调用4次
	long time_marker = System.currentTimeMillis();
    int counter = 0;
    public boolean take1(){
        long now = System.currentTimeMillis();
        long pass_time = now - time_marker;
        if(pass_time > 1000){
            counter = 1;
            time_marker = now;
        }else{
            if(counter >= 4)return false;
            ++counter;
        }
        return true;
    }

	//不足之处,比如每秒最多调用4次,第1.1s,1.2s调用一次,1.5s调用一次,那么从1.5s-2.0s做多调用
	//一次,每次调用都是基于某一次调用的一个时间点和1s这个时间周期来控制的,不能是基于任意一次调用
	//时间点来做限制

## api调用次数控制---token bucket
	
	//google api rate limit , leaky bucket and token bucket
	//漏铜算法和令牌桶算法 
	public boolean take() {
        long now = System.currentTimeMillis();
        tokens += (int) ((now - timestamp) * tokensPerSeconds / 1000);
        if (tokens > capacity) tokens = capacity;
        timestamp = now;
        if (tokens < 1f) return false;
        tokens--;
        return true;
    }
	//和上一个实现的却别,take()动态增加计数,每次收到请求都会根据时间戳计算可以增加token数量
	//根据流逝的时间来增加token,就不会有上面算法的问题,在1.5s-2.5s之间也许可以有7次调用,就
	//违背了1s内最多4次调用的限制了,而采用token bucket算法,因为是基于流逝时间来增加bucket
	//所以不可能调用7次,在1.5s-2.5s之间间隔1s又可以有4次api调用,看上去调用次数的时间点又是
	//基于1.5s的了,而对于1.0s这个时间点来说,1.0s-2.0s这个时间段4次api调用任然成立,所以看上去
	//就像基于任何时间点来计算api调用次数

	//capacity作用,允许突发的大量请求,当capacity比tokensPerSeconds大时,时间流逝转换为token直到
	//token为capacity,突然大量请求来临时允许超过tokensPerSeconds限制,直到消耗完(capacity-tokensPerSeconds)
	//个令牌,然后开始实施tokensPerSeconds限制,这种算法适应性更好

	//算法问题,基于时间流逝转换为令牌数量,这里有时间损失,转换为整数是会忽略小数,不足以转换
	//为一个令牌的时间会被忽略掉,导致1s内的令牌数量不足4个,不考虑性能可以改为浮点型


## 两种算法比较
	第一种算法关注开始时间和计数器,在调用时根据当前时间判断是否增加计数器值还是是否重置
	第二种算法的令牌也类似计数器,但是思路有点不一样,根据流逝的时间来增加计数器,然后每次调用
	对计数器做减法,有点逆向思维的感觉

http://blog.gssxgss.me/not-a-simple-problem-rate-limiting/
	
http://www.cnblogs.com/zhengyun_ustc/archive/2012/11/17/topic1.html		
