<!--
 * @Author: QingHui Meng
 * @Date: 2021-12-03 15:08:05
-->
## 简介
ReentrantLock是一个可重入的互斥锁。ReentrantLock类实现了Lock，拥有和synchronized相同的并发性和内存语义，但是添加了类似锁投票、定时锁等候和可中断锁等候的一些特性。此外，它还提供了在激烈争用情况下更佳的性能，即当许多资源都想访问共享资源时，JVM可以花更少的时间来调度线程，把更多的时间用在执行线程上。

互斥锁，在同一时间只能被一个线程持有；可重入锁，可以被单个线程多次获取。ReentrantLock分为公平锁和非公平锁，区别在于在获取锁的机制上是否公平。

“锁”是为了保护竞争资源，防止多个线程同时操作线程而出错，ReentrantLock在同一个时间点只能被一个线程获取(当某线程获取到“锁”时，其它线程就必须等待)；ReentraantLock是通过一个**FIFO的等待队列**来管理获取该锁所有线程的。在“公平锁”的机制下，线程依次排队获取锁；而“非公平锁”在锁是可获取状态时，不管自己是不是在队列的开头都会获取锁。

### 函数列表：
```java
// 创建一个 ReentrantLock ，默认是“非公平锁”。
ReentrantLock()
// 创建策略是fair的 ReentrantLock。fair为true表示是公平锁，fair为false表示是非公平锁。
ReentrantLock(boolean fair)

// 查询当前线程保持此锁的次数。
int getHoldCount()
// 返回目前拥有此锁的线程，如果此锁不被任何线程拥有，则返回 null。
protected Thread getOwner()
// 返回一个 collection，它包含可能正等待获取此锁的线程。
protected Collection<Thread> getQueuedThreads()
// 返回正等待获取此锁的线程估计数。
int getQueueLength()
// 返回一个 collection，它包含可能正在等待与此锁相关给定条件的那些线程。
protected Collection<Thread> getWaitingThreads(Condition condition)
// 返回等待与此锁相关的给定条件的线程估计数。
int getWaitQueueLength(Condition condition)
// 查询给定线程是否正在等待获取此锁。
boolean hasQueuedThread(Thread thread)
// 查询是否有些线程正在等待获取此锁。
boolean hasQueuedThreads()
// 查询是否有些线程正在等待与此锁有关的给定条件。
boolean hasWaiters(Condition condition)
// 如果是“公平锁”返回true，否则返回false。
boolean isFair()
// 查询当前线程是否保持此锁。
boolean isHeldByCurrentThread()
// 查询此锁是否由任意线程保持。
boolean isLocked()
// 获取锁。
void lock()
// 如果当前线程未被中断，则获取锁。
void lockInterruptibly()
// 返回用来与此 Lock 实例一起使用的 Condition 实例。
Condition newCondition()
// 仅在调用时锁未被另一个线程保持的情况下，才获取该锁。
boolean tryLock()
// 如果锁在给定等待时间内没有被另一个线程保持，且当前线程未被中断，则获取该锁。
boolean tryLock(long timeout, TimeUnit unit)
// 试图释放此锁。
void unlock()
```

### 通常的使用方法：
```java
ReentrantLock lock = new ReentrantLock(); // not a fair lock
lock.lock();

try {

    // synchronized do something

} finally {
    lock.unlock();
}
```

## 流程
### NonFairSync
非公平锁加锁流程：
<img src =../img/nonFairSync流程.png/>

<img src=../img/nonFairSync流程2.png/>

### FairSync
公平锁加锁流程：
<img src=../img/fairSync流程.png/>

公平和非公平的区别：
* NonfairSync执行lock的第一步就是尝试通过compareAndSetState(0, 1)获取锁；而FairSync不会进行此操作。
* NonfairSync通过nonfairTryAcquire获取锁；FairSync通过tryAcquire获取锁。这两个方法的主要逻辑相同，唯一的差别在于FairSync在获取锁前会先检查FIFO队列中是否有Node存在，如果没有，则当前线程执行compareAndSetState(0, 1)操作，尝试获取锁。

### 释放锁
公平锁和非公平锁释放锁的过程是相同的，释放锁的流程如下：
<img src=../img/unlock流程.png/>


## 源码分析
### 加锁
非公平锁lock第一步就尝试通过compareAndSetState(0,1)获取锁
```java
    // ReentrantLock lock
    public void lock() {
        sync.lock();
    }

    // NonFairLock lock
    final void lock() {
        // 首先通过CAS获取锁，如果获取成功，把占用该锁的线程设置为当前线程
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    // FairLock lock
    final void lock() {
        acquire(1);
    }
```
exclusiveOwnerThread代表持有的线程信息，所以一旦上锁成功，就要将exclusiveOwnerThread设置为当前线程。

compareAndSetState是通过cas操作把state从0改成1。ReentrantLock中state表示可重入的次数，state=0表示当前没有线程持有锁，state>0表示加锁次数。

### acquire
acquire是属于AQS中的方法，逻辑如下：
* 尝试获取锁，如果成功，直接返回。
* 如果获取锁失败，创建一个Node.EXCLUSIVE类型的Node，加入AQS的FIFO队列中。
* 已经入队列的线程尝试获取锁，如果获取成功，返回，获取失败，会被挂起。
```java
    public final void acquire(int arg) {
        // 尝试获取锁
        if (!tryAcquire(arg) &&
            // 获取失败，把节点加入到等待队列
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

### tryAcquire
tryAcquire方式是尝试获取锁，如果state=0，说明已经被释放了，所以可以重新尝试上锁操作，即进行compareAndSetState(0,1)操作；如果state!=0，并且当前线程持有锁，那么重入，更新state，state++。

仅当以下两种情况，此方法返回true
* 如果锁已经被释放了，并且当前线程compareAndSetState(0,1)成功。
* 当前线程持有锁，重入，更新state。
```java
    // NonFairSync tryAcquire 
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }

    // NonFairSync nonFairTryAcquire
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // state = 0表示没有线程占用锁
        if (c == 0) {
            // 使用CAS尝试获取锁
            if (compareAndSetState(0, acquires)) {
                // 获取成功把当前占用锁的线程设置为当前线程
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // state != 0表示有线程占用锁，如果是当前线程占用锁，那么相当于是重入操作，所以state++。
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

    // FairSync tryAquire
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 如果head的next节点对应的线程不是当前线程，那么当前线程不能尝试获取锁，这样才能保证按照获取所得顺序公平的获取锁。
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
```

### addWaiter  enq
addWaiter属于AQS中的方法，此方法的主要逻辑是：创建一个Node，与当前线程关联，然后尝试加入到FIFO队列的尾部。加入到FIFO队列时，如果发现队列没有初始化过，即Head为空，那么先创建一个空的不和任何线程关联的Node作为Head，然后将Node加入到队列尾部。

enq方法的作用是，如果FIFO队列尚未初始化，那么先初始化，如果已经初始化了，尝试将node加入到队尾，失败则再试一次。
```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        // tail为空，可以认为队列尚未初始化过；不为空，那么尝试将Node加入到队列尾部
        if (pred != null) {
            node.prev = pred;
            // 通过CAS操作将Node插入队尾
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        // tail为空，初始化队列
        enq(node);
        return node;
    }

    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                // 队列为空，使用CAS设置一个空节点作为head节点。
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

### acquireQueued
acquireQueued是AQS中的方法，主要逻辑为：
* 获取node的prev节点。
* 如果prev节点是head节点，说明当前node是第二个节点，那么可以尝试让当前线程获取锁。如果获取成功，重新设置head，并返回interrupted标志位。
* 如果prev节点不是head，那么认为不用尝试获取锁，此时判断是否应该挂起当前线程，如果应该挂起，通过LockSupport.park挂起线程。
```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 如果prev节点是head，那么说明自己是第二个node，此时尝试获取锁；如果获取成功，那么将node的thread、prev清除掉，然后作为head节点
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 判断是否应该挂起当前线程，如果应该挂起，那么通过LockSupport挂起当前线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### shouldParkAfterFailedAcquire
此方法是用来判断是否应该挂起当前节点对应的线程。允许挂起的条件是：prev节点的waitStatus=SIGNAL。这个状态的意思是：prev节点对应的线程在释放锁以后应该唤醒（unpack）node节点对应的线程。

如果prev节点waitStatus>0，即为CANCELLED状态时，需要将prev从队列中移除。重试此操作直到找到一个prve节点的状态不为CANCELLED。

如果prev节点waitStatus<=0（当然不包括SIGNAL状态），那么通过CAS操作设置waitStatus= SIGNAL
```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
            * This node has already set status asking a release
            * to signal it, so it can safely park.
            */
        return true;
    if (ws > 0) {
        /*
            * Predecessor was cancelled. Skip over predecessors and
            * indicate retry.
            */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
            * waitStatus must be 0 or PROPAGATE.  Indicate that we
            * need a signal, but don't park yet.  Caller will need to
            * retry to make sure it cannot acquire before parking.
            */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

## 等待超时放弃获取锁
在循环中会检测等待时间是否超过指定时间，如果超过了，那么将Node的waitStatus改为CANCELLED状态。
```java
    // AQS tryLock
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

    // AQS tryAcquireNanos
    public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }

    // AQS doAcquireNanos
    private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)
                    return false;
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

## 释放锁
### release
释放锁，主要逻辑：
* 尝试释放锁
* 如果此次释放完之后，state=0，即持有锁的线程已经将锁彻底释放了，那么尝试唤醒head后面Node对应的线程。（这里有一个检查Node节点对应的线程是否已经被CANCELLED的过程，对于这种节点会将他们从队列中移除）
```java
    // ReentrantLock unlock
    public void unlock() {
        sync.release(1);
    }

    // Sync release
    public final boolean release(int arg) {
        // 尝试释放锁，只有释放锁之后state = 0，才返回true
        if (tryRelease(arg)) {
            // 释放锁成功，唤醒head后面Node对应的线程
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

    // 
```

### tryRelease
* 如果当前线程不是占用锁的线程，不允许释放。
* 设置state = state-releases。
* 如果state = 0，那么将锁的占用线程清理掉。
* 释放后如果state = 0，那么返回true；即只有在Sync不被任何线程占用，才返回true。
```java
    // ReentrantLock tryRelease
    protected final boolean tryRelease(int releases) {
        // 释放锁之后的state值
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        // 如果释放锁之后state = 0，返回true
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
```

### unparkSuccessor
* 如果node的waitStatus != CANCELLED状态，那么将node的waitStatus改成0。这里是一个清理head节点状态的过程，清理指的是改成初始状态。
* 如果node的next节点处于CANCELLD状态，那么将次next节点移除队列；重试此过程直到找到一个不处于CANCELLED状态的节点或者队列中没有状态为止。
* 如果上一步没有找到这样的节点，那么通过LockSupport.unpark去唤醒这个节点对应的线程。
```java
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        // 如果head的waitStatus不等于0，通过cas把head的waitStatus改成0.
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

## Condition
线程获取锁以后，如果条件不满足，挂起线程释放资源；如果条件满足，唤醒线程继续处理。

ReentrantLock可以支持多条件等待，原理：每次调用newCondition方法，都会创建一个ConditionObject对象，每个ConditionObject对象都可以挂起一个等待队列；如果希望同时等待多个条件，只需要简单的多次调用newCondition创建多个条件就可以了。

同步等待队列和条件等待队列：
* 同步等待队列是AQS中的FIFO队列。条件等待队列是ConditionObject对象上的条件等待队列。