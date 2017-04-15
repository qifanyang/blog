# 定时任务

## 场景
定时任务使用的地方非常多,比如报表定时执行,定时重置某些数据,分布式补偿任务定时执行以保证最终一致性  

## java中的定时任务

### timerTask
1.TimerTask.run方法如果抛出异常,则TimerThread退出循环,定时任务不再执行,so run方法中需要catch所有异常  
2.任务执行时间太长会阻塞调度,需要等到任务执行完毕才能继续执行定时任务
~~~
public class TimerTaskTest {
    public static void main(String[] args) {
        //创建TimerThread(Thread线程子类),并启动线程,while(true)中调度任务
        //TimerThread包含一个TaskQueue,队列为优先队列,使用二叉堆实现,任务下一次执行时间作为优先条件
        Timer timer = new Timer();
        TimerTask task = new TimerTask() {
            long lastExecuteTime = System.currentTimeMillis();
            @Override
            public void run() {
                System.out.println("i am timer task, i am working!!!, last run time = " + new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date(lastExecuteTime)));
                lastExecuteTime = System.currentTimeMillis();
                Thread.interrupted();
            }
        };

        timer.schedule(task, 5000,5000);//任务调度基于java Object.wait(time)利用虚拟机的线程调度
//        timer.schedule(task, 5000,5000);//会判断任务状态,抛出异常


    }
}
~~~

## schedulerExecutor
多线程执行定时任务,一个任务超时不会导致串行执行  
~~~
        ScheduledThreadPoolExecutor s = new ScheduledThreadPoolExecutor(4);

//        for(int i = 0; i < 10; i++)
        s.scheduleAtFixedRate(()->{
            System.out.println(Thread.currentThread().getName());
//            throw new RuntimeException("exp");

        },1, 1 , TimeUnit.SECONDS);
    }
~~~

## Quartz
核心组件:  
scheduler:调度器(包含常规调度线程和misfired trigger线程)  
job:具体任务  
trigger:调度规则  
misfire:超时任务调度  
调度器负责任务调度,有任务需要执行则从线程池中取出一个空闲线程执行该任务,如果没有空闲线程,那么任务会超时.  
这时misfired Thread扫描超时的trigger,根据misfired policy修改trigger下次执行时间  

无状态Job,可以并发执行,StatefulJob没法并发调度

## Quartz集群
1.使用可以持久化的JobStore,Quartz使用数据库表来感知集群其它节点的存在  
http://tech.meituan.com/mt-crm-quartz.html  

~~~
Quartz数据库核心表如下：

Table Name	Description
QRTZ_CALENDARS	存储Quartz的Calendar信息
QRTZ_CRON_TRIGGERS	存储CronTrigger，包括Cron表达式和时区信息
QRTZ_FIRED_TRIGGERS	存储与已触发的Trigger相关的状态信息，以及相联Job的执行信息
QRTZ_PAUSED_TRIGGER_GRPS	存储已暂停的Trigger组的信息
QRTZ_SCHEDULER_STATE	存储少量的有关Scheduler的状态信息，和别的Scheduler实例
QRTZ_LOCKS	存储程序的悲观锁的信息
QRTZ_JOB_DETAILS	存储每一个已配置的Job的详细信息
QRTZ_JOB_LISTENERS	存储有关已配置的JobListener的信息
QRTZ_SIMPLE_TRIGGERS	存储简单的Trigger，包括重复次数、间隔、以及已触的次数
QRTZ_BLOG_TRIGGERS	Trigger作为Blob类型存储
QRTZ_TRIGGER_LISTENERS	存储已配置的TriggerListener的信息
QRTZ_TRIGGERS	存储已配置的Trigger的信息
~~~
调度任务并发执行时,先获取数据库行锁,然后执行任务,集群中其它机器无法获取到行锁则阻塞,当获取到行锁的机器执行成功  
后更新jobdetail,被阻塞的机器获取到锁后继续等待  

