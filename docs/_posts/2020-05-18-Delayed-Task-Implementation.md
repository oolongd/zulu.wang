---
draft: true
category: algorithm
tags:
  - algorithm
  - netty
date: 2020-05-18
title: 延迟任务的实现
vssue-title: Delayed Task Implementation
---
## 缘起
在需求开发过程中，我么经常会遇到“某一段时间之后，执行某一个任务的需有”的需求。如：
1. 京东商城下单后未支付，24小时后，自动取消订单并释放库存
2. 外卖订单超过15分钟未支付，自动取消
3. 下单后未支付，1小时给用户发送付款提醒

针对这种业务场景，首先想到的解决方案是启一个定时任务，比如每分钟遍历一次库中的数据，将到期的数据取出来，统一做业务处理。这种方式业务代码实现起来相对简单，但也有不足之处：
1. 轮询效率比较低
2. 每次扫库，已经被执行过记录，仍然会被扫描（只是不会出现在结果集中），有重复计算的嫌疑
3. 时效性不够好，如果每小时轮询一次，最差的情况下，时间误差会达到1小时
4. 如果通过增加cron轮询频率来减少（3）中的时间误差，（1）中轮询低效和（2）中重复计算的问题会进一步凸显

## 高效延时消息设计与实现

### 时间轮算法实现
包含两个重要的数据结构：
* 环形队列，例如可以创建一个包含 n 个 bucket 的环形队列（本质是个数组）
* 任务集合，环上每一个 bucket 是一个 Set<Task>

1996 年 George Varghese 和 Tony Lauck 的论文[《Hashed and Hierarchical Timing Wheels: Data Structures for the Efficient Implementation of a Timer Facility》](http://cseweb.ucsd.edu/~varghese/PAPERS/twheel.ps.Z)中提出了一种时间轮管理 Timeout 事件的方式。其设计非常巧妙，并且类似时钟的运行。

假设时间轮上有3600个 bucket ，每一个 bucket 代表一秒，同时启动一个计时器，每隔一秒钟在时间轮上移动一次，currentBucketIndex 指针用来标志当前所检测的 bucket，现在有一个任务需要在2小时30分钟后执行，由于时间轮正好是3600格因此 currentBucketIndex 每移动一圈即表示1小时，2小时30分钟即需要移动2圈半，所有我们需要为每一个任务引入2个新的变量来判断当前是否应该执行任务，一个是 roundsCount，用来表示距离任务执行需要执行的轮数。一个是 bucketIndex 用来表示执行任务时应处于哪一个 bucket 中。计算方式也很简单，先统一时间单位，2小时30分钟即9000秒，相关变量计算方式如下：

``` 
roundsCount = 9000 / 3600 = 2
bucketIndex = （9000 % 3600）+ currentBucketIndex = 1800 + currentBucketIndex
``` 

当定时器检测到 roundsCount 为0，并且 bucketIndex 小于等于 currentBucketIndex 即可以执行当前任务。

时间轮算法被广泛用于各种语言的项目中，这里以 netty 为例。由于 netty 动辄管理100w+的连接，每一个连接都会有很多超时任务。比如发送超时、心跳检测间隔等，如果每一个定时任务都启动一个 Timer,不仅低效，而且会消耗大量的资源。

`HashedWheelTimer` 即为 netty 中实现的时间轮，它的构造函数主要有一下几个参数：
* tickDuration：时间轮中的每一个 bucket 表示多长时间 
* unit：tickDuration的时间单位
* ticksPerWheel：表示时间轮中有多少个 bucket

`HashedWheelBucket` 用来存放 `HashedWheelTimeout` 即定时任务，结构类似于LinkedList

`HashedWheelTimeout` 是一个定时任务的内部包装类，会保存定时任务到期执行的任务、deadline、round 等信息。

`Worker` 是时间轮的核心线程类。tick 的转动，过期任务的处理都是在这个线程中处理的。

具体的源码地址 [HashedWheelTimer](https://github.com/netty/netty/blob/4.1/common/src/main/java/io/netty/util/HashedWheelTimer.java)

实际项目中可以直接使用 netty 提供的方法，具体方式如下：
```java
package wang.zulu.delay;

import io.netty.util.HashedWheelTimer;

import java.time.LocalDateTime;
import java.util.concurrent.TimeUnit;

public class WheelTimerTest {
    public static void main(String[] args) {
        //设置每个格子是 100ms, 总共 256 个格子
        HashedWheelTimer hashedWheelTimer = new HashedWheelTimer(100, TimeUnit.MILLISECONDS, 256);
        //加入三个任务，依次设置超时时间是 10s 5s 20s
        System.out.println("加入一个任务，ID = 1, time= " + LocalDateTime.now());
        hashedWheelTimer.newTimeout(timeout -> {
            System.out.println("执行一个任务，ID = 1, time= " + LocalDateTime.now());
        }, 10, TimeUnit.SECONDS);
        System.out.println("加入一个任务，ID = 2, time= " + LocalDateTime.now());
        hashedWheelTimer.newTimeout(timeout -> {
            System.out.println("执行一个任务，ID = 2, time= " + LocalDateTime.now());
        }, 5, TimeUnit.SECONDS);
        System.out.println("加入一个任务，ID = 3, time= " + LocalDateTime.now());
        hashedWheelTimer.newTimeout(timeout -> {
            System.out.println("执行一个任务，ID = 3, time= " + LocalDateTime.now());
        }, 20, TimeUnit.SECONDS);
        System.out.println("等待任务执行===========");
    }
}

```
使用时间轮来实现延时任务虽然提高程序的执行效率但还是避免不了单点故障。

### Redis ZSet 实现
Redis 里有 5 种数据结构，最常用的是 String 和 Hash，而 ZSet 是一种支持按 score 排序的数据结构，每个元素都会关联一个 double 类型的分数。有序集合的成员是唯一的,但分数(score)却可以重复。Redis 通过分数来为集合中的成员进行从小到大的排序，借助这个特性我们可以把超时时间作为 score 来将任务进行排序。
具体的思路如下：
1. 使用 `ZADD key score member` 命令向 redis 中放入任务，任务的 ID 作为 member，任务需要被执行的时间为 score
2. 启动一个定时器，每隔一段时间使用`ZRANGEBYSCORE key start stop withscores count` 命令从 redis 中读取任务，其中 start 可以取固定值0，stop 为系统的当前时间。
3. 使用 `ZREM key member` 命令从 redis 中删除任务。

还是以上面的例子为例，首先启动一个定时器每隔一段时间在 Redis 执行如下命令

```
ZRANGEBYSCORE key 0 currentTimestamp 0 1
```

其中 `key` 为存放任务的 zset 名称，`currentTimestamp` 为当前系统的时间
现在假设有一个 ID 为1的任务需要2小时30分钟后执行，当前时间戳为 1589798086，在 Redis 中即执行

```
zadd key 1589807086 1
```


相关代码实现如下：

```java
package wang.zulu.delay;


import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;

import java.time.LocalDateTime;
import java.util.Set;
import java.util.concurrent.TimeUnit;

public class RedisTest {
    public static void main(String[] args) {
        DelayQueue delayQueue = new DelayQueue();
        //加入三个任务，依次设置超时时间是 10s 5s 20s
        delayQueue.produce(1, System.currentTimeMillis() + 10000);
        delayQueue.produce(2, System.currentTimeMillis() + 5000);
        delayQueue.produce(3, System.currentTimeMillis() + 20000);
        System.out.println("等待任务执行===========");
        delayQueue.consumer();
    }
}


class DelayQueue {
    private final JedisPool pool;
    private final String redisKey = "daily-queue";

    DelayQueue() {
        pool = new JedisPool(new JedisPoolConfig(),
                "10.10.10.211", 6379, 2000, "zeaho123", 5);
    }

    public void produce(Integer taskId, long exeTime) {
        try (Jedis jedis = pool.getResource()) {
            System.out.println("加入一个任务， ID = " + taskId + ", exeTime: " + exeTime + ", 当前时间：" + LocalDateTime.now());
            jedis.zadd(redisKey, exeTime, String.valueOf(taskId));
        }
    }

    public void consumer() {
        while (!Thread.interrupted()) {
            try (Jedis jedis = pool.getResource()) {
                Set<String> zrange = jedis.zrangeByScore(redisKey, 0, System.currentTimeMillis(), 0, 1);
                if (zrange.isEmpty()) {
                    //如果消息是空的，则休息 500 毫秒然后继续
                    try {
                        TimeUnit.MILLISECONDS.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    continue;
                }
                String next = zrange.iterator().next();
                if (jedis.zrem(redisKey, next) > 0) {
                    System.out.println("从延时队列中获取到任务，taskId:" + next + " , 当前时间：" + LocalDateTime.now());
                }
            }

        }
    }

}



```

相比时间轮的实现方式，使用 Redis 可以将数据持久化到磁盘，规避了数据丢失的风险，并且支持分布式，避免了单点故障。

### 思考
基于 Redis 的实现方式在分布式的部署过程中，极端情况下是否会出现同一个任务被执行多次的情况？


## 参考
* [延时任务的几种实现方式](https://xie.infoq.cn/article/857cea9c7e0a05a483a8d5c96)
* [netty源码解读之时间轮算法实现-HashedWheelTimer](https://zacard.net/2016/12/02/netty-hashedwheeltimer/)
* [1分钟实现“延迟消息”功能](https://cloud.tencent.com/developer/article/1048667)