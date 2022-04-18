<!--
 * @Author: QingHui Meng
 * @Date: 2021-12-03 15:31:00
-->
## 简介
AbstractQueuedSynchronizer，简称AQS，是一个用于构建锁和相关同步器的框架，依赖于FIFO的等待队列实现。

AQS解决了实现同步容器时设计的大量细节问题，concurrent包中许多类都是基于AQS构建，例如ReentrantLock，ReentrantReadWriteLock，Semaphore，CountDownLatch，FutureTask等。

AQS维护了一个FIFO队列，用于记录等待的线程，上锁和释放锁的过程可以认为是线程入队列和出队列的过程：获取不到锁，入队列等待，出队列时被唤醒。

AQS中有一个表示状态的字段state，ReenTrantLock用它来表示线程重入锁的次数，对state变量值的更新都采用CAS操作保证更新的原子性。

AbstractQueuedSynchronizer继承了AbstractOwnableSynchronizer，这个类只有一个变量：exclusiveOwnerThread，表示当前占用该锁的线程，并且提供了相应的get，set方法。AQS使用此字段记录当前占有锁的线程。

### 队列
AQS用一个FIFO的等待队列表示排队等待锁的线程，队列结构如图：
<img src=./img/AQS队列.png/>

队列的头节点成为哨兵节点或者哑节点，不与任何线程关联，其他的节点和一个等待线程关联。

每次一个线程释放锁以后，从Head向后开始寻找，找到一个waitStatus是SIGNAL的节点，然后通过LockSupport.unpark唤醒被挂起的线程，被挂起的线程继续执行尝试上锁逻辑；新的线程尝试获取锁时，如果获取不到锁，将会创建一个Node并加入到队尾，然后将自己挂起，等待挂起时间到或者被perv对应的节点唤醒。

Node的主要属性：
* waitStatus   int    等待状态
* prev         Node   队列中的前一个节点
* next         Node   队列中的后一个节点
* thread       Thread  Node持有的线程
* nextWaiter   Node       下一个等待condition的Node

waitStatus的取值
* 状态         状态值      描述
* Cancelled    1          取消状态，例如因为等待锁超时而取消的线程，处于这种状态的Node会被踢出队列，被GC回收
* SIGNAL        -1        表示这个Node的继任Node被阻塞了，到时需要通知它
* CONDITION     -2        表示这个Node在条件队列中，因为等待某个条件而被阻塞
* PROPAGATE     -3        使用在共享模式头Node可能会处于这种状态，表示锁的下一次可以无条件传播
* 其他          0         初始状态

### 队列示例
<img src=./img/AQSFIFO流程.png>

