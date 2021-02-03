Java主流锁的分类：
<br><img src=img/锁分类.png><br>

## 乐观锁 悲观锁
乐观锁和悲观锁时一种广义上的概念，体现了**看待线程同步的不同角度**。在Java和数据库中都有此概念对应的实际应用。

概念：对于同一个数据的并发操作，悲观锁认为自己在使用数据的时候一定有别的线程来修改数据，因此在获取数据的时候会先加锁，确保数据不会被别的线程修改。Java中，s**ynchronized关键字和Lock的实现类都是悲观锁**。
乐观锁认为自己在使用数据时不会有别的线程修改数据，所以不会添加锁，只是在更新数据的时候去判断有没有别的线程更新了这个数据。如果这个数据没有更新，当前线程将自己修改数据成功写入；如果数据已经被其他线程修改，则根据不同的实现方式执行不同的操作(例如报错或者自动重试)。乐观锁在Java中是通过**使用无锁编程实现的，最常用的是CAS算法**，**Java原子类中的递增操作就通过CAS自旋**实现。
<br><img src=img/乐观锁和悲观锁.png><br>

* 悲观锁适合写操作多的场景，先加锁可以保证写操作时数据正确。
* 乐观锁适合读操作多的场景，不加锁的特点能够使其读操作的性能大幅提升。

乐观锁和悲观锁的调用方式：
```java
// ------------------------- 悲观锁的调用方式 -------------------------
// synchronized
public synchronized void testMethod() {
	// 操作同步资源
}
// ReentrantLock
private ReentrantLock lock = new ReentrantLock(); // 需要保证多个线程使用的是同一个锁
public void modifyPublicResources() {
	lock.lock();
	// 操作同步资源
	lock.unlock();
}

// ------------------------- 乐观锁的调用方式 -------------------------
private AtomicInteger atomicInteger = new AtomicInteger();  // 需要保证多个线程使用的是同一个AtomicInteger
atomicInteger.incrementAndGet(); //执行自增1
```
通过上面的实例可以得出，悲观锁都是在显示的锁定之后再操作同步资源，而乐观锁则直接的去操作同步资源。乐观锁的主要实现方式CAS的技术原理：

CAS全称为Compare And Swap，是一种无锁算法。在不使用锁的情况下实现多线程之间的变量同步。java.util.concurrent包中的原子类就是通过CAS实现的乐观锁。CAS算法涉及三个操作数：
* 需要读写的内存值V。
* 要进行比较的值A。
* 要新写入的值B。
当且仅当V的值等于A时，CAS通过原子方式用新值B来更新V的值( 比较+更新 整体是一个原子操作)，否则不会执行任何操作。一般情况下，“更新”是一个不断重试的操作。AtomicInteger的定义以及自增函数incrementAndGet的源码：
```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    //获取并操作内存中的数据
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;//存储value在AtomicInteger中的偏移量

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;

    //自增方法，调用的是unsafe.getAndAddInt方法
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }


// Unsafe.java
public final int getAndAddInt(Object o, long offset, int delta) {
   int v;
   do {
       //获取对象o偏移量为offset的值v
       v = getIntVolatile(o, offset);
   } while (!compareAndSwapInt(o, offset, v, v + delta));//比较内存中值是否等于v，如果等于v，那么就没有更新，将内存值设置为v+delta，否则返回false，继续循环重试，直到设置成功才能退出循环，并且将旧值返回。
   return v;
}
```
getAndAddInt()方法循环获取给定对象o中的偏移量为offset的值v，然后判断内存值是否等于v。如果等于v，那么就没有更新，将内存值设置为v+delta，否则返回false，继续循环重试，直到设置成功才能退出循环，并且将旧值返回。
整个“比较+更新”操作封装在compareAndSwapInt()中，在JNI里是借助于一个CPU指令来完成的，属于原子操作，保证多个线程都能够看到同一个变量的修改值。后续JDK通过CPU的cmpxchg指令，去比较寄存器中的A和内存中的V。如果相等，就要把写入的新值B存入内存中。如果不相等，九江内存值V赋值给寄存器中的值A。然后通过Java代码中的while循环再次调用cmpxchg指令进行重试，知道设置成功为止。

CAS虽然高效，但是也存在三大问题：
1. ABA问题，CAS需要在操作值的时候检查内存值是否发生变化，没有发生变化才会更新内存值。但是如果内存值是A，后来变成了B，然后又变成了A，那么CAS进行检查时会发现值没有变化，但是实际上是有变化的。ABA问题的解决思路就是在变量前添加版本号，每次变量更新的时候都把版本号加1。
    * JDK从1.5开始提供了AtomicStampedReference类来解决ABA问题，具体操作封装在compareAndSet()中。compareAndSet()首先检查当前引用和当前标志与预期引用和预期标志是否相等，如果都相等，则以原子方式将引用值和标志的值设置为给定的更新值。
2. 循环时间开销大。CAS操作如果长时间不成功，会导致一直自旋，给CPU带来非常大的开销。
3. 只能保证一个共享变量的原子操作，对一个共享变量执行操作时，CAS能够保证原子操作，但是对多个共享变量进行操作时，CAS是无法保证操作的原子性的。

## 自旋锁 适应性自旋锁
阻塞或唤醒一个Java线程需要操作系统切换CPU状态来完成，这种状态切换需要耗费处理器时间。如果同步代码块过于简单，状态转换消耗的时间可能比用户执行代码的时间还要长。

在许多场景下，同步资源的锁定时间很短，为了这一小段时间去切换线程，线程挂起和恢复现场的花费可能会让系统得不偿失。如果物理机器有多个处理器，能够让两个或以上的线程同时并发执行，我们就可以让后面那个请求锁的线程不放弃CPU的执行时间，看看持有锁的线程是否很快就会释放锁。

而为了让当前线程等一下，我们需要让当前线程进行自旋，如果在自旋完成后前面锁定同步资源的线程已经释放了锁，那么当前线程就可以不必阻塞而是直接获取同步资源，从而避免切换线程的开销。这就是自旋锁。
<br><img src=img/自旋锁.png><br>

自旋锁本身是有缺点的，不能代替阻塞。自旋等待虽然避免了线程切换的开销，但是需要占用处理器时间。如果锁被占用的时间非常短，自旋等待的效果就会非常好。反之，如果锁被占用的时间很长，那么自旋的线程只会白白浪费处理器资源。所以，自旋锁等待的时间必须要有一定的限度，如果自旋超过了限定次数(默认是10次，可以使用-XX:PreBlockSpin更改)没有成功获得锁，就应当挂起线程。

自旋锁的实现原理也是CAS。

JDK1.4.2中引入了自旋锁，JDK1.6中默认开启，并且引入了自适应的自旋锁。自适应意味着自旋的次数不再固定，而是由前一次在同一个锁上的自旋时间及锁的拥有者来决定的。如果在同一个锁对象上，自旋等待刚刚成功获取到锁，并且持有锁的线程正在运行，那么虚拟机就会认为这次自旋也是很有可能再次成功，进而它将允许自旋持续相对更长的时间。如果对于某个锁，自旋很少成功获得过，那在以后尝试获取这个锁时将可能省略自旋过程，直接阻塞线程，避免浪费资源。

## 无锁 偏向锁 轻量级锁 重量级锁
这四种锁是指的是锁的状态，专门针对synchronized的。

## 公平锁 非公平锁
公平锁是指多个线程按照申请锁的顺序来获取锁，线程直接进入队列中排队，队列中的第一个线程才能获得锁。公平锁的优点是等待锁的线程不会饿死。缺点是整体的吞吐效率相对非公平锁要低，等待队列中除第一个以外所有的线程都会阻塞，CPU唤醒阻塞线程的开销比非公平锁大。

非公平锁是指多个线程加锁时直接尝试获取锁，获取不到时才会到等待队列的结尾排队。如果此时锁刚好可用，那么这个线程可以无需阻塞直接获取到锁，