如果之前没有接触过 ReentrantLock 的话，建议先阅读这两篇博客：
+ [并发编程——ReentrantLock](http://space.eyescode.top/blog/details/236)
+ [AQS问题](http://hunt.eyescode.top/stereotype/java/aqs.html)

# ReentrantLock 结构

![ReentrantLock](http://oss.eyescode.top/eyeshunt/content/4194b5d4-fdee-4847-ed04-0ef7781377a9.png)

ReentrantLock 继承自父类 Lock，然后有 3 个内部类：
+ Sync 内部类继承自 AQS
+ FairSync 和 NonfairSync 内部类继承自 Sync，这两个类分别是用来实现公平锁和非公平锁的

通过 Sync 重写的方法`tryAcquire`、`tryRelease`可以知道，ReentrantLock 实现的是 AQS 的独占模式，也就是独占锁，这个锁是悲观锁。

ReentrantLock 有个重要的成员变量：

```java
private final Sync sync;
```

这个变量是用来指向 Sync 的子类的，也就是 FairSync 或者 NonfairSync，这个也就是多态的父类引用指向子类。至于具体 Sycn 指向哪个子类，看构造方法：

```java
// 无参构造，默认创建非公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}

// 有参构造，自行决定公平或非公平
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

# 非公平锁的实现原理

**获取锁**

ReentrantLock 是使用`lock`方法获取锁，该方法的执行逻辑如下：

```java
final void lock() {
    // CAS 操作设置 state 的值
    if (compareAndSetState(0, 1))
        // 设置成功，直接将锁的所有者设置为当前线程，流程结束
        setExclusiveOwnerThread(Thread.currentThread());
    else
        // 设置失败 则进行后续的加入同步队列准备
        acquire(1);
}


public final void acquire(int arg) {
    // 调用子类重写的 tryAcquire 方法，如果 tryAcquire 方法返回 false，
    // 那么线程就会进入同步队列
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

可以看到，当获取锁失败的时候，会调用重写的`tryAcquire`方法，之后的逻辑如下：

```java
// 子类重写的 tryAcquire 方法
protected final boolean tryAcquire(int acquires) {
    // 调用 nonfairTryAcquire 方法
    return nonfairTryAcquire(acquires);
}

final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    // 如果状态 state=0，即在这段时间内，锁的所有者把锁释放了 那么这里 state 就为 0
    if (c == 0) {
        // 使用 CAS 操作设置 state 的值
        if (compareAndSetState(0, acquires)) {
            // 操作成功则将锁的所有者设置成当前线程，且返回true，
            // 这就体现了非公平性，因为当前线程不会进入同步队列，而是 CAS 抢锁
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 如果状态 state!=0，也就是有线程正在占用锁，那么先检查一下这个线程是不是自己
    else if (current == getExclusiveOwnerThread()) {
        // 如果线程就是自己了，那么直接将 state + 1，返回 true，
        // 不需要再获取锁，因为锁就在自己身上了
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    // 如果 state 不等于0，且锁的所有者又不是自己，那么线程就会进入到同步队列
    return false;
}
```

![获取锁](http://oss.eyescode.top/eyeshunt/content/90b3204b-14b5-b7c1-6136-2c4acbc1988d.png)

**锁的释放**

ReentrantLock 是使用`unlock`方法释放锁，该方法的执行逻辑如下：

```java
public void unlock() {
    sync.release(1);
}

public final boolean release(int arg) {
    // 子类重写的 tryRelease 方法，需要等锁的 state=0，即 tryRelease 返回 true 的时候，
    // 才会去唤醒其它线程进行尝试获取锁
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
    
protected final boolean tryRelease(int releases) {
    // 状态的 state 减去 releases
    int c = getState() - releases;
    // 判断锁的所有者是不是该线程
    if (Thread.currentThread() != getExclusiveOwnerThread())
        // 如果所的所有者不是该线程，则抛出异常，也就是锁释放的前提是线程拥有这个锁，
        throw new IllegalMonitorStateException();
    boolean free = false;
    // 如果该线程释放锁之后，状态 state=0，即锁没有重入，那么直接将将锁的所有者设置成 null
    // 并且返回 true，即代表可以唤醒其他线程去获取锁了。如果该线程释放锁之后 state 不等于 0，
    // 那么代表锁重入了，返回 false，代表锁还未正在释放，不用去唤醒其他线程。
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

![释放锁](http://oss.eyescode.top/eyeshunt/content/9932972c-7985-3a62-5676-93c982d09b74.png)

# 公平锁的实现原理

**获取锁**

ReentrantLock 是使用`lock`方法获取锁，该方法的执行逻辑如下：

```java
final void lock() {
    acquire(1);
}

public final void acquire(int arg) {
    // 如果同步队列中有线程且锁的所有者不是当前线程，那么将线程加入到同步队列的尾部
    // 保证了公平性，也就是先来的线程先获得锁，后来的不能抢先获取
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    // 判断状态 state 是否等于0，等于 0 代表锁没有被占用，不等于 0 则代表锁被占用着
    if (c == 0) {
        // 调用 hasQueuedPredecessors 方法判断同步队列中是否有线程在等待，如果同步队列中没有
        // 线程在等待，则当前线程成为锁的所有者，如果同步队列中有线程在等待，则继续往下执行
        // 这个机制就是公平锁的机制，也就是先让先来的线程获取锁，后来的不能抢先获取
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 判断当前线程是否为锁的所有者，如果是，那么直接更新状态 state，然后返回 true
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    // 如果同步队列中有线程存在且锁的所有者不是当前线程，那么线程就会进入到同步队列
    return false;
}
```

![获取锁](http://oss.eyescode.top/eyeshunt/content/3fece9cf-c79f-b78a-e30e-d092b3740d5a.png)

**tryRelease锁的释放**

公平锁和非公平锁的公平性是在获取锁的时候体现出来的，释放的时候都是一样释放的，所以这里就不重复了。

# ReentrantLock 是如何实现可重入性的

这是一个常见的面试题。通过上文分析可以知道，ReentrantLock 内部自定义了同步器 sync，在加锁的时候通过 CAS 算法，将线程对象放到一个双向链表中
+ 每次获取锁的时候，检查当前维护的那个线程 ID 和当前请求的线程 ID 是否一致，如果一致，同步状态 state 加 1
+ 每次解锁的时候，检查当前维护的那个线程 ID 和当前请求的线程 ID 是否一致，如果一致，同步状态 state 减 1
+ 如果 state 等于 0，则表示锁已释放，可以通知同步队列唤醒其他线程
+ 如果 state 大于 0，则表示锁还未完全释放，不用唤醒其他线程

------
摘自
+ [深入理解ReentrantLock的实现原理](https://juejin.cn/post/6844903805683761165)
+ [ReentrantLock 是如何实现可重入性的？](https://www.topjavaer.cn/java/java-concurrent.html)


站长略有修改
