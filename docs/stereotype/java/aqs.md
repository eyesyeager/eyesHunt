AQS 的全称为 AbstractQueuedSynchronizer，翻译过来的意思就是抽象队列同步器。这个抽象类在 `java.util.concurrent.locks` 包下面。AQS 为构建锁和同步器提供了一些通用功能的实现，因此，使用 AQS 能简单且高效地构造出应用广泛的大量的同步器，比如我们提到的 ReentrantLock，Semaphore，其他的诸如 ReentrantReadWriteLock，SynchronousQueue 等等皆是基于 AQS 的。

# AQS原理

AQS 核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制 AQS 是基于 CLH 锁 （Craig, Landin, and Hagersten locks） 实现的。

CLH 锁是对自旋锁的一种改进，是一个虚拟的双向队列（虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系），暂时获取不到锁的线程将被加入到该队列中。AQS 将每条请求共享资源的线程封装成一个 CLH 队列锁的一个结点（Node）来实现锁的分配。

在 CLH 队列锁中，一个 Node 表示一个线程，它保存着如下数据：
+ thread：引用线程，头节点不包含线程
+ waitStatus：当前节点在队列中的状态
  + `0`：节点初始化时的状态
  + `CANCELLED = 1`：节点引用线程由于等待超时或被打断时的状态
  + `SIGNAL = -1`：后继节点线程需要被唤醒时的当前节点状态。当队列中加入后继节点被挂起（block）时，其前驱节点会被设置为 SIGNAL 状态，表示该节点需要被唤醒
  + `CONDITION = -2`：当节点线程进入 condition 队列时的状态
  + `PROPAGATE = -3`：仅在释放共享锁 releaseShared 时对头节点使用
+ prev：前驱节点
+ next：后继节点

![CLH](http://oss.eyescode.top/eyeshunt/content/50661d8f-e68e-8586-e9c4-1b471ea779c4.png)

AQS（AbstractQueuedSynchronizer）的核心原理图：

![AQS 核心原理](http://oss.eyescode.top/eyeshunt/content/718b71fa-cdc5-cb26-fe0a-634f0e0bbc3e.png)

AQS 使用 int 成员变量 state 表示同步状态，通过内置的 FIFO 线程等待/等待队列 来完成获取资源线程的排队工作。state 变量由 volatile 修饰，用于展示当前临界资源的获锁情况：

```java
// 共享变量，使用 volatile 修饰保证线程可见性
// 对于不同的实现类，state 的值有不同的意义，所以这里不多说明
private volatile int state;
```

另外，状态信息 state 可以通过 protected 类型的`getState()`、`setState()`和`compareAndSetState()`进行操作。并且，这几个方法都是 final 修饰的，在子类中无法被重写。

```java
// 返回同步状态的当前值
protected final int getState() {
    return state;
}

// 设置同步状态的值
protected final void setState(int newState) {
    state = newState;
}

// 原子地（CAS操作）将同步状态值设置为给定值。如果当前同步状态的值等于expect（期望值）则更新
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

# AQS 资源共享方式

AQS 定义两种资源共享方式：
+ Exclusive（独占）：只有一个线程能执行，如 ReentrantLock
+ Share（共享）：多个线程可同时执行，如 Semaphore、CountDownLatch

一般来说，自定义同步器的共享方式要么是独占，要么是共享，他们也只需实现`tryAcquire-tryRelease`、`tryAcquireShared-tryReleaseShared`中的一种即可。但 AQS 也支持自定义同步器同时实现独占和共享两种方式，如 ReentrantReadWriteLock。

# 自定义同步器

同步器的设计是基于模板方法模式的，如果需要自定义同步器，一般的方式是这样（模板方法模式很经典的一个应用）：
1. 使用者继承 `AbstractQueuedSynchronizer` 并重写指定的方法
2. 将 AQS 组合在自定义同步组件的实现中，并调用其模板方法，而这些模板方法会调用使用者重写的方法。

这和我们以往通过实现接口的方式有很大区别，这是模板方法模式很经典的一个运用。AQS 使用了模板方法模式，自定义同步器时需要重写下面几个 AQS 提供的钩子方法：

```java
// 独占方式。尝试获取资源，成功则返回 true，失败则返回 false
protected boolean tryAcquire(int)

// 独占方式。尝试释放资源，成功则返回 true，失败则返回 false
protected boolean tryRelease(int)

// 共享方式。尝试获取资源。负数表示失败；0 表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源
protected int tryAcquireShared(int)

// 共享方式。尝试释放资源，成功则返回 true，失败则返回 false
protected boolean tryReleaseShared(int)

// 该线程是否正在独占资源。只有用到 condition 才需要去实现它
protected boolean isHeldExclusively()
```

什么是钩子方法呢？ 钩子方法是一种被声明在抽象类中的方法，一般使用 protected 关键字修饰，它可以是空方法（由子类实现），也可以是默认实现的方法。模板设计模式通过钩子方法控制固定步骤的实现。除了上面提到的钩子方法之外，AQS 类中的其他方法都是 final ，所以无法被其他类重写。

# 为什么AQS是双向链表而不是单向的？

双向链表有两个指针，一个指针指向前置节点，一个指针指向后继节点。所以，双向链表可以支持常量 `O(1)` 时间复杂度的情况下找到前驱节点。因此，双向链表在插入和删除操作的时候，要比单向链表简单、高效。

从双向链表的特性来看，AQS 使用双向链表有两个方面的原因：
+ 没有竞争到锁的线程加入到阻塞队列，并且阻塞等待的前提是，当前线程所在节点的前置节点是正常状态，这样设计是为了避免链表中存在异常线程导致无法唤醒后续线程的问题。所以，线程阻塞之前需要判断前置节点的状态，如果没有指针指向前置节点，就需要从 Head 节点开始遍历，性能非常低。
+ 在 Lock 接口里面有一个`lockInterruptibly()`方法，这个方法表示处于锁阻塞的线程允许被中断。也就是说，没有竞争到锁的线程加入到同步队列等待以后，是允许外部线程通过`interrupt()`方法触发唤醒并中断的。这个时候，被中断的线程的状态会修改成 CANCELLED。而被标记为 CANCELLED 状态的线程，是不需要去竞争锁的，但是它仍然存在于双向链表里面。这就意味着在后续的锁竞争中，需要把这个节点从链表里面移除，否则会导致锁阻塞的线程无法被正常唤醒。在这种情况下，如果是单向链表，就需要从 Head 节点开始往下逐个遍历，找到并移除异常状态的节点。同样效率也比较低，还会导致锁唤醒的操作和遍历操作之间的竞争。

------
摘自：
+ [AQS 详解](https://javaguide.cn/java/concurrent/aqs.html)
+ [Java并发之AQS详解](https://juejin.cn/post/7006895386103119908)
+ [为什么AQS是双向链表而不是单向的？](https://topjavaer.cn/java/java-concurrent.html#%E4%B8%BA%E4%BB%80%E4%B9%88aqs%E6%98%AF%E5%8F%8C%E5%90%91%E9%93%BE%E8%A1%A8%E8%80%8C%E4%B8%8D%E6%98%AF%E5%8D%95%E5%90%91%E7%9A%84)

站长略有修改

------
待补充：
+ AQS工作流程
+ AQS唤醒节点时，为什么是从后往前找