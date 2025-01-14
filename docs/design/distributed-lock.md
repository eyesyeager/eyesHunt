在单机环境下，当存在多个线程可以同时改变某个变量（可变共享变量）时，就会出现线程安全问题。这个问题可以通过 JAVA 提供的 volatile、ReentrantLock、synchronized 以及 concurrent 并发包下一些线程安全的类等来避免。

而在多机部署环境，需要在多进程下保证线程的安全性，Java 提供的这些 API 仅能保证在单个 JVM 进程内对多线程访问共享资源的线程安全，已经不满足需求了。这时候就需要使用分布式锁来保证线程安全。通过分布式锁，可以保证在分布式部署的应用集群中，同一个方法在同一时间只能被一台机器上的一个线程执行。

在分析分布式锁的三种实现方式之前，先了解一下分布式锁应该具备哪些条件：
+ 互斥性：在任意时刻，只有一个客户端能持有锁
+ 不会死锁：具备锁失效机制，防止死锁，即使有客户端在持有锁的期间崩溃而没有主动解锁，也要保证后续其他客户端能加锁
+ 加锁和解锁必须是同一个客户端：客户端 a 不能将客户端 b 的锁解开，即不能误解锁
+ 高性能、高可用的获取锁与释放锁
+ 具备可重入特性
+ 具备非阻塞锁特性，即没有获取到锁将直接返回获取锁失败

# 基于数据库实现

**悲观锁**

创建一张锁表，然后通过操作该表中的数据来实现加锁和解锁。当要锁住某个方法或资源时，就向该表插入一条记录，表中设置方法名为唯一键，这样多个请求同时提交数据库时，只有一个操作可以成功，判定操作成功的线程获得该方法的锁，可以执行方法内容；想要释放锁的时候就删除这条记录，其他线程就可以继续往数据库中插入数据获取锁。

**乐观锁**

先在数据库建一张资源表，包含资源名称 resource_name、版本号 version（可以是随机数、时间戳，与锁是否过期有关）等信息，将资源名称建立唯一索引。

执行逻辑如下：
1. 在程序尝试上锁时，根据资源名称查询数据，获取锁的版本号也就是时间戳
2. 将当前时间与版本号作比较，如果当前时间大于等于版本号的时间，就以资源名称和查出来的版本号作为条件，更新数据库版本号为自己的版本号。自己的版本号就是当前时间戳加上超时时间
3. 如果更新返回的影响行数大于 0 就是更新成功，该线程获取了锁
4. 如果更新返回的影响行数不大于 0 就是更新失败，该线程无法获取锁，然后睡眠一定时间再重试。我们可以设置重试时间，如果超过时间就不再获取锁了
5. 如果版本号已经被更新了的话，那么其他线程以查出来的旧版本号就无法匹配得到数据行了，在这个过程中只有一个线程能够更新
6. 释放锁的过程就是将版本号改为当前时间戳

![乐观锁实现分布式锁](http://oss.eyescode.top/eyeshunt/content/6e5d4507-9632-270c-42c6-951072478204.png)

# 基于Redis实现

**单机模式**

加锁逻辑：
1. setnx 争抢 key 的锁，如果已有 key 存在，则不作操作，过段时间继续重试，保证只有一个客户端能持有锁
2. value 设置为 requestId（可以使用机器 ip 拼接当前线程名称），表示这把锁是哪个请求加的，在解锁的时候需要判断当前请求是否持有锁，防止误解锁。比如客户端 A 加锁，在执行解锁之前，锁过期了，此时客户端 B 尝试加锁成功，然后客户端 A 再执行`del()`方法，则将客户端 B 的锁给解除了
3. 再用 expire 给锁加一个过期时间，防止异常导致锁没有释放

解锁逻辑：首先获取锁对应的 value 值，检查是否与 requestId 相等，如果相等则删除锁。使用 lua 脚本实现原子操作，保证线程安全。

这个方案是基于 Redis 单机版的分布式锁讨论，还不是很完美。因为 Redis 一般都是集群部署的。

**集群模式**

集群模式下，加锁会变得复杂一点。如下图，如果线程一在 Redis 的 master 节点上拿到了锁，但是加锁的 key 还没同步到 slave 节点。恰好这时，master 节点发生故障，一个 slave 节点就会升级为 master 节点。线程二就可以顺理成章获取同个 key 的锁啦，但线程一也已经拿到锁了，锁的安全性就没了。

![集群模式](http://oss.eyescode.top/eyeshunt/content/cf3659a6-0696-8679-bb54-dc9d33f23971.png)

为了解决这个问题，Redis 作者 antirez 提出一种高级的分布式锁算法：Redlock。它的核心思想是这样的：部署多个 Redis master，以保证它们不会同时宕掉。并且这些 master 节点是完全相互独立的，相互之间不存在数据同步。同时，需要确保在这多个 master 实例上，是与在 Redis 单实例，使用相同方法来获取和释放锁。

我们假设当前有 5 个 Redis master 节点，在 5 台服务器上面运行这些 Redis 实例：
![集群示例](http://oss.eyescode.top/eyeshunt/content/9c482024-d9ef-1bdf-843e-4b065917d387.png)

RedLock 的实现步骤：
1. 获取当前时间，以毫秒为单位
2. 按顺序向 5 个 master 节点请求加锁。客户端设置网络连接和响应超时时间，并且超时时间要小于锁的失效时间。（假设锁自动失效时间为 10 秒，则超时时间一般在 5-50 毫秒之间,我们就假设超时时间是 50ms 吧）。如果超时，跳过该 master 节点，尽快去尝试下一个 master 节点。
3. 客户端使用当前时间减去开始获取锁时间（即步骤1记录的时间），得到获取锁使用的时间。当且仅当超过一半（`N/2+1`，这里是`5/2+1=3个`节点）的 Redis  master 节点都获得锁，并且使用的时间小于锁失效时间时，锁才算获取成功。（如上图，`10s> 30ms+40ms+50ms+4m0s+50ms`）
4. 如果取到了锁，key 的真正有效时间就变啦，需要减去获取锁所使用的时间
5. 如果获取锁失败（没有在至少`N/2+1`个 master 实例取到锁，有或者获取锁时间已经超过了有效时间），客户端要在所有的 master 节点上解锁（即便有些 master 节点根本就没有加锁成功，也需要解锁，以防止有些漏网之鱼）。

简化下步骤就是：
1. 按顺序向5个master节点请求加锁
2. 根据设置的超时时间来判断，是不是要跳过该master节点
3. 如果大于等于3个节点加锁成功，并且使用的时间小于锁的有效期，即可认定加锁成功
4. 如果获取锁失败，就解锁

**看门狗**

上面介绍的 redis 分布式锁是无法自动续期的，比如，一个锁设置了1分钟超时释放，如果拿到这个锁的线程在一分钟内没有执行完毕，那么这个锁就会被其他线程拿到，这可能会导致严重的线上问题。为了解决这个问题，就需要有一个自动续期功能。

Redisson 是一个开源的 Java 语言 Redis 客户端，提供了很多开箱即用的功能，不仅仅包括多种分布式锁的实现。并且，Redisson 还支持 Redis 单机、Redis Sentinel、Redis Cluster 等多种部署架构。Redisson 中的分布式锁自带自动续期机制，使用起来非常简单，原理也比较简单，其提供了一个专门用来监控和续期锁的 Watch Dog（看门狗），如果操作共享资源的线程还未执行完成的话，Watch Dog 会不断地延长锁的过期时间，进而保证锁不会因为超时而被释放。

![看门狗工作机制](http://oss.eyescode.top/eyeshunt/content/052094b1-92a7-de97-10fe-637707c65dee.jpg)

# 基于Zookeeper实现

ZooKeeper 分布式锁是基于 临时顺序节点 和 Watcher（事件监听器） 实现的。

获取锁：
1. 首先我们要有一个持久节点`/locks`，客户端获取锁就是在 locks 下创建临时顺序节点
2. 假设客户端 1 创建了`/locks/lock1`节点，创建成功之后，会判断 lock1 是否是`/locks`下最小的子节点
3. 如果 lock1 是最小的子节点，则获取锁成功。否则，获取锁失败
4. 如果获取锁失败，则说明有其他的客户端已经成功获取锁。客户端 1 并不会不停地循环去尝试加锁，而是在前一个节点比如`/locks/lock0`上注册一个事件监听器。这个监听器的作用是当前一个节点释放锁之后通知客户端 1（避免无效自旋），这样客户端 1 就加锁成功了

释放锁：
1. 成功获取锁的客户端在执行完业务流程之后，会将对应的子节点删除
2. 成功获取锁的客户端在出现故障之后，对应的子节点由于是临时顺序节点，也会被自动删除，避免了锁无法被释放
3. 我们前面说的事件监听器其实监听的就是这个子节点删除事件，子节点删除就意味着锁被释放

![zookeeper分布式锁](http://oss.eyescode.top/eyeshunt/content/5a2b5ce3-fb00-c4a0-e466-bc6165a1e98f.png)

实际项目中，推荐使用 Curator 来实现 ZooKeeper 分布式锁。Curator 是 Netflix 公司开源的一套 ZooKeeper Java 客户端框架，相比于 ZooKeeper 自带的客户端 zookeeper 来说，Curator 的封装更加完善，各种 API 都可以比较方便地使用。

**为什么要用临时顺序节点？**

每个数据节点在 ZooKeeper 中被称为 znode，它是 ZooKeeper 中数据的最小单元。我们通常是将 znode 分为 4 大类：
+ 持久（PERSISTENT）节点：一旦创建，即使 ZooKeeper 集群宕机它也一直存在，直到将其删除
+ 临时（EPHEMERAL）节点：临时节点的生命周期是与客户端会话（session）绑定的，会话消失则节点消失。并且临时节点只能做叶子节点，不能创建子节点
+ 持久顺序（PERSISTENT_SEQUENTIAL）节点：除了具有持久节点的特性之外，子节点的名称还具有顺序性。比如`/node1/app0000000001`、`/node1/app0000000002`
+ 临时顺序（EPHEMERAL_SEQUENTIAL）节点：除了具备临时节点的特性之外，子节点的名称还具有顺序性

可以看出，临时节点相比持久节点，最主要的是对会话失效的情况处理不一样，临时节点会话消失则对应的节点消失。这样的话，如果客户端发生异常导致没来得及释放锁也没关系，会话失效节点自动被删除，不会发生死锁的问题。

使用 Redis 实现分布式锁的时候，我们是通过过期时间来避免锁无法被释放导致死锁问题的，而 ZooKeeper 直接利用临时节点的特性即可。

假设不使用顺序节点的话，所有尝试获取锁的客户端都会对持有锁的子节点加监听器。当该锁被释放之后，势必会造成所有尝试获取锁的客户端来争夺锁，这样对性能不友好。使用顺序节点之后，只需要监听前一个节点就好了，对性能更友好。

**为什么要设置对前一个节点的监听？**

> Watcher（事件监听器），是 ZooKeeper 中的一个很重要的特性。ZooKeeper 允许用户在指定节点上注册一些 Watcher，并且在一些特定事件触发的时候，ZooKeeper 服务端会将事件通知到感兴趣的客户端上去，该机制是 ZooKeeper 实现分布式协调服务的重要特性。

同一时间段内，可能会有很多客户端同时获取锁，但只有一个可以获取成功。如果获取锁失败，则说明有其他的客户端已经成功获取锁。获取锁失败的客户端并不会不停地循环去尝试加锁，而是在前一个节点注册一个事件监听器。

这个事件监听器的作用是：当前一个节点对应的客户端释放锁之后（也就是前一个节点被删除之后，监听的是删除事件），通知获取锁失败的客户端（唤醒等待的线程，Java 中的 wait/notifyAll ），让它尝试去获取锁，然后就成功获取锁了。

# 方案对比

数据库分布式锁实现：
+ 优点：
  + 简单，使用方便，不需要引入 Redis、zookeeper 等中间件
+ 缺点：
  + 不适合高并发的场景
  + db操作性能较差
  
Redis分布式锁实现：
+ 优点：
  + 性能好，适合高并发场景
  + 较轻量级
  + 有较好的框架支持，如 Redisson
+ 缺点：
  + 过期时间不好控制
  + 需要考虑锁被别的线程误删场景

Zookeeper 分布式锁实现
+ 优点：
  + 有较好的性能和可靠性
  + 有封装较好的框架，如 Curator
+ 缺点：
  + 性能不如 redis 实现的分布式锁
  + 比较重

对比汇总：
+ 从性能角度（从高到低）：Redis > Zookeeper >= 数据库
+ 从理解的难易程度角度（从低到高）：数据库 > Redis > Zookeeper
+ 从实现的复杂性角度（从低到高）：Zookeeper > Redis > 数据库
+ 从可靠性角度（从高到低）：Zookeeper > Redis > 数据库

------
摘自：
+ [分布式锁](https://www.topjavaer.cn/advance/distributed/2-distributed-lock.html)
+ [分布式锁常见实现方案总结](https://javaguide.cn/distributed-system/distributed-lock-implementations.html)
+ [java分布式锁 - 基于数据库表实现乐观锁（三）](https://blog.csdn.net/weixin_45973130/article/details/120555676)
+ [redisson中的看门狗机制总结](https://www.cnblogs.com/jelly12345/p/14699492.html)

站长略有修改
