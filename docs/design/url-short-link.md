URL 短链，就是把原来较长的网址，转换成比较短的网址。比如这样一个链接：`http://space.eyescode.top/shuoshuo/details?id=UAS6CySag/0=`，使用短链服务后就能变成：`http://f6a.cn/p34Mv`

那么为什么要做这样的转换呢？来看看短链带来的好处：
+ 在微博， Twitter 这些限制字数的应用中，短链带来的好处不言而喻，网址短、美观、便于发布和传播，可以写更多有意义的文字
+ 在短信中，如果含长网址的短信内容超过 70 字，就会被拆成两条发送，而用短链则可能一条短信就搞定，如果短信量大也可以省下不少钱
+ 我们平常看到的二维码，本质上也是一串 URL，如果是长链，对应的二维码会密密麻麻，扫码的时候机器很难识别，而短链则不存在这个问题
+ 出于安全考虑，不想让有意图的人看到原始网址
+ 分析不同渠道的流量

短链工作的基本流程如下：
![短链工作流程](http://oss.eyescode.top/eyeshunt/content/1fcfdd7f-309b-f9dc-cee2-aad2caf1ee15.png)

设计短链功能的时候，应当有如下需求：
+ 功能性需求：
  + 给定原始的长 URL，短链服务能生成比它短且唯一的 URL
  + 用户点击短 URL，能跳转到长原始的长 URL
  + 短 URL 经过一定时间后，会过期
  + 接口需要设计成 REST API
+ 非功能性需求：
  + 高可用：服务不能存在单点故障
  + 高性能：生成短 URL 的过程，以及从短 URL 跳转到原始 URL 要近实时
  + 安全：短链不可被预测，否则简单的遍历就能把所有短链都遍历完，空耗系统资源

当然这里当然不能把完整的短链系统给说清楚，有兴趣的话可以看本文末尾的文章。本文只介绍短链生成方案。

设计短链生成算法，本质上是为了寻找一种映射关系，能将原始 URL 和生成的短链对应起来。下面我们来分析一下目前短链生成算法一些主流的方案。

# 一：Hash 算法

观察上例中的短链 `http://f6a.cn/p34Mv`，显然它是由固定短链域名加上长链映射成的一串字母组成，那么长链怎么才能映射成一串字母呢，那就可以用到哈希函数了。

问题就在于选取哪种哈希函数，相信肯定有很多人说用 MD5，SHA 等算法，其实这样做有点杀鸡用牛刀了，而且既然是加密就意味着性能上会有损失，我们其实不关心反向解密的难度，反而更关心的是哈希的运算速度和冲突概率。

能够满足这样的哈希算法有很多，这里推荐 Google 出品的 MurmurHash 算法，MurmurHash 是一种非加密型哈希函数，适用于一般的哈希检索操作。与其它流行的哈希函数相比，对于规律性较强的 key，MurmurHash 的随机分布特征表现更良好。非加密意味着着相比 MD5，SHA 这些函数它的性能肯定更高（实际上性能是 MD5 等加密算法的十倍以上），MurmurHash 提供了两种长度的哈希值，32 bit，128 bit，为了让网址尽可通地短，我们选择 32 bit 的哈希值，32 bit 能表示的最大值近 43 亿，对于中小型公司的业务而言绰绰有余。也正是由于它的这些优点，所以虽然它出现于 2008，但目前已经广泛应用到 Redis、MemCache、Cassandra、HBase、Lucene 等众多著名的软件中。

但既然是哈希函数，不可避免地会产生哈希冲突（尽管概率很低），该怎么解决呢。我们知道既然访问短链能跳转到长链，那么两者之前这种映射关系一定是要保存起来的，可以用 Redis 或 Mysql 等，这里我们选择用 Mysql 来存储。表结构应该如下所示：
```sql
CREATE TABLE `short_url_map` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `lurl` varchar(160) DEFAULT NULL COMMENT '长地址',
  `surl` varchar(10) DEFAULT NULL COMMENT '短地址',
  `gmt_create` int(11) DEFAULT NULL COMMENT '创建时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

于是我们有了以下设计思路。
1. 将长链（lurl）经过 MurmurHash 后得到短链
2. 再根据短链去 short_url_map 表中查找看是否存在相关记录，如果不存在，将长链与短链对应关系插入数据库中存储
3. 如果存在，说明已经有相关记录了，此时在长串上拼接一个自定义好的字段，比如「DUPLICATE」，然后再对对接的字段串「lurl + DUPLICATE」做第一步操作，如果最后还是重复呢，那再拼一个字段串，根据短链取出长链的时候把这些自定义好的字符串移除即可

以上步骤显然是要优化的，插入一条记录居然要经过两次 sql 查询（根据短链查记录，将长短链对应关系插入数据库中），如果在高并发下，显然会成为瓶颈。

**唯一索引优化**

首先我们需要给短链字段 surl 加上唯一索引，当长链经过 MurmurHash 得到短链后，直接将长短链对应关系插入 db 中，如果 db 里不含有此短链的记录，则插入，如果包含了，说明违反了唯一性索引，此时只要给长链再加上我们上文说的自定义字段「DUPLICATE」,重新 hash 再插入即可，看起来在违反唯一性索引的情况下是多执行了步骤，但我们要知道 MurmurHash 发生冲突的概率是非常低的，基本上不太可能发生，所以这种方案是可以接受的。

**布隆过滤器优化**

当然如果在数据量很大的情况下，冲突的概率会增大，此时我们可以加布隆过滤器来进行优化。用所有生成的短网址构建布隆过滤器，当一个新的长链生成短链后，先将此短链在布隆过滤器中进行查找，如果不存在，说明 db 里不存在此短网址，可以插入。但是由于布隆过滤器的特性，如果它返回存在，那它并不一定会存在，此时就需要二次查询或者利用唯一索引处理了。

# 二：自增序列算法

我们可以维护一个 ID 自增生成器，比如 1，2，3 这样的整数递增 ID，当收到一个长链转短链的请求时，ID 生成器为其分配一个 ID，再将其转化为 62 进制，拼接到短链域名后面就得到了最终的短网址，那么这样的 ID 自增生成器该如何设计呢。如果在低峰期发号还好，高并发下，ID 自增生成器的的 ID 生成可能会系统瓶颈，所以它的设计就显得尤为重要。

主要有以下四种获取 id 的方法
+ UUID：即全局唯一标识符，是指在一台机器上生成的数字，它保证对在同一时空中的所有机器都是唯一的，但这种方式生成的 id 比较长，且无序，在插入 db 时可能会频繁导致页分裂，影响插入性能。
+ Redis：用 Redis 是个不错的选择，性能好，单机可支撑 10 w+ 请求，足以应付大部分的业务场景，但有人说如果一台机器扛不住呢，可以设置多台嘛，比如我布置 10 台机器，每台机器分别只生成尾号0，1，2，... 9 的 ID, 每次加 10即可，只要设置一个 ID 生成器代理随机分配给发号器生成 ID 就行了。不过这种方案需要考虑持久化（短链 ID 总不能一样吧），灾备，成本有点高。
+ Snowflake：即雪花算法，这也是个不错的选择，不过 Snowflake 依赖于系统时钟的一致性。如果某台机器的系统时钟回拨，有可能造成 ID 冲突，或者 ID 乱序。
+ Mysql 自增主键：这种方式使用简单，扩展方便

当然，id 并非一定要实时生成，那样在高并发场景下系统压力会很大，可以预先生成一部分存好，需要时拿出来使用即可。

------
摘自：
+ [系统设计之路：如何设计一个URL短链服务](https://zhuanlan.zhihu.com/p/370475544)
+ [高性能短链设计](https://zhuanlan.zhihu.com/p/113528722)

站长略有修改