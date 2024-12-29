在开发中，往往会遇到一些关于延时任务的需求。例如：
+ 生成订单 30 分钟未支付，则自动取消
+ 生成订单 60 秒后，给用户发短信

对上述的任务，我们给一个专业的名字来形容，那就是延时任务。那么这里就会产生一个问题，这个延时任务和定时任务的区别究竟在哪里呢？一共有如下几点区别：
+ 定时任务有明确的触发时间，延时任务没有
+ 定时任务有执行周期，而延时任务在某事件触发后一段时间内执行，没有执行周期
+ 定时任务一般执行的是批处理操作，是多个任务，而延时任务一般是单个任务

下面，我们以判断订单是否超时为例，进行方案分析

# 一：数据库轮询

该方案通常是在小型项目中使用，即通过一个线程定时的去扫描数据库，通过订单时间来判断是否有超时的订单，然后进行 update 或 delete 等操作。

优点
+ 简单易行，支持集群操作

缺点
+ 对服务器内存消耗大
+ 存在延迟，比如你每隔 3 分钟扫描一次，那最坏的延迟时间就是 3 分钟
+ 性能低，假设订单有几千万条，每隔几分钟这样扫描一次，数据库损耗极大

# 二：JDK延时队列

该方案是利用 JDK 自带的 DelayQueue 来实现，这是一个无界阻塞队列，该队列只有在延迟期满的时候才能从中获取元素。

优点
+ 效率高,任务触发时间延迟低。

缺点
+ 服务器重启后，数据全部消失，怕宕机
+ 集群扩展相当麻烦
+ 因为内存条件限制的原因，比如下单未付款的订单数太多，那么很容易就出现 OOM 异常
+ 代码复杂度较高

# 三：时间轮算法

先上一张时间轮的图:
![时间轮算法](http://oss.eyescode.top/eyeshunt/content/86773247-79b8-279d-0791-b03452c84647.png)

时间轮算法可以类比于时钟，如上图箭头（指针）按某一个方向按固定频率轮动，每一次跳动称为一个 tick。这样可以看出定时轮有个 3 个重要的属性参数：
+ ticksPerWheel（一轮的 tick 数）
+ tickDuration（一个 tick 的持续时间）
+ timeUnit（时间单位）

例如当 ticksPerWheel=60，tickDuration=1，timeUnit=秒，这就和现实中的始终的秒针走动完全类似了。如果当前指针指在 1 上面，我有一个任务需要 4 秒以后执行，那么这个执行的线程回调或者消息将会被放在 5 上。那如果需要在 20 秒之后执行怎么办，由于这个环形结构槽数只到 8，如果要 20 秒，指针需要多转 2 圈。位置是在 2 圈之后的 5 上面（20 % 8 + 1）

优点：
+ 效率高
+ 任务触发时间延迟时间比数据库轮询低
+ 代码复杂度比 delayQueue 低

缺点：
+ 服务器重启后，数据全部消失，怕宕机
+ 集群扩展相当麻烦
+ 因为内存条件限制的原因，比如下单未付款的订单数太多，那么很容易就出现 OOM 异常

# 四：redis的zset结构

zset是一个有序集合，每一个元素（member）都关联了一个 score，通过 score 排序来取集合中的值

添加元素：`ZADD key score member [[score member][score member] …]`
按顺序查询元素：`ZRANGE key start stop [WITHSCORES]`
查询元素：`score:ZSCORE key member`
移除元素：`ZREM key member [member …]`

语法测试如下：
```
添加单个元素
redis> ZADD page_rank 10 google.com
(integer) 1

添加多个元素
redis> ZADD page_rank 9 baidu.com 8 bing.com
(integer) 2

redis> ZRANGE page_rank 0 -1 WITHSCORES
1) "bing.com"
2) "8"
3) "baidu.com"
4) "9"
5) "google.com"
6) "10"

查询元素的score值
redis> ZSCORE page_rank bing.com
"8"

移除单个元素
redis> ZREM page_rank google.com
(integer) 1

redis> ZRANGE page_rank 0 -1 WITHSCORES
1) "bing.com"
2) "8"
3) "baidu.com"
4) "9"
```

那么如何实现呢？我们将订单超时时间戳与订单号分别设置为 score 和 member，系统扫描第一个元素判断是否超时，具体如下图所示：
![redis实现思路](http://oss.eyescode.top/eyeshunt/content/c762b7674ea845e464a3be0622473c6e.png)

然而，如果不做额外处理的话，在高并发条件下，多消费者可能会取到同一个订单号，解决方案有如下几种：
+ 加分布式锁，性能低
+ 对 ZREM 的返回值进行判断，只有大于 0 的时候，才消费数据

优点：
+ 由于使用 redis 作为消息通道。如果发送程序或者任务处理程序挂了，重启之后，还有重新处理数据的可能性
+ 做集群扩展相当方便
+ 时间准确度高

缺点：
+ 需要额外进行 redis 维护

# 五：redis的过期回调机制

redis 的 Keyspace Notifications，中文翻译就是键空间机制，就是利用该机制可以在 key 失效之后，提供一个回调，实际上是 redis 会给客户端发送一个消息。是需要 redis 版本 2.8 以上。

但这个方式有硬伤，Redis 的发布/订阅目前是即发即弃(fire and forget)模式的，因此无法实现事件的可靠通知。也就是说，如果发布/订阅的客户端断链之后又重连，则在客户端断链期间的所有事件都丢失了。因此，这个方案不太推荐。当然，如果对可靠性要求不高，那也可以使用。


# 六：消息队列

可以采用 rabbitMQ 的延时队列。RabbitMQ 具有以下两个特性，可以实现延迟队列：
+ RabbitMQ 可以针对 Queue 和 Message 设置 x-message-tt，来控制消息的生存时间，如果超时，则消息变为 dead letterl（死信）
+ RabbitMQ 的 Queue 可以配置 x-dead-letter-exchange 和 x-dead-letter-routing-key（可选）两个参数，如果控制队列内出现了 deadletter，则可以按照这两个参数重新路由。

结合以上两个特性，就可以模拟出延迟消息的功能。

优点：
+ 高效
+ 可以利用 rabbitmq 的分布式特性轻易的进行横向扩展
+ 消息支持持久化，增加了可靠性。

缺点：
+ 本身的易用度要依赖于 rabbitMq 的运维
+ 因为要引用 rabbitMq，所以复杂度和成本变高

------
原作者：大彬

原文链接：https://www.topjavaer.cn/advance/system-design/2-order-timeout-auto-cancel.html

站长略有修改