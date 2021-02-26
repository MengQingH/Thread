Java主流锁的分类：
<br><img src=img/锁分类.png><br>

## 乐观锁 悲观锁
乐观锁和悲观锁时一种广义上的概念，体现了**看待线程同步的不同角度**。在Java和数据库中都有此概念对应的实际应用。

概念：对于同一个数据的并发操作，悲观锁认为自己在使用数据的时候一定有别的线程来修改数据，因此在获取数据的时候会先加锁，确保数据不会被别的线程修改。Java中，**synchronized关键字和Lock的实现类都是悲观锁**。
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

非公平锁是指多个线程加锁时直接尝试获取锁，获取不到时才会到等待队列的结尾排队。如果此时锁刚好可用，那么这个线程可以无需阻塞直接获取到锁，所以非公平锁有可能出现后申请锁的线程先获取锁的场景。非公平锁的优点是可以减少唤起线程的开销，整体的吞吐效率高，因为线程有几率不阻塞直接获得锁，CPU不必唤醒所有线程。缺点是处于等待队列中的线程可能会饿死，或者等很久才会获得锁。图示：
<br><img src=img/公平锁.png><img src=img/非公平锁.png><br>

ReenTrantLock里面有一个内部类Sync，Sync继承AQS，添加锁和释放锁的大部分操作实际上都是在Sync中实现的。它有公平锁FairSync和非公平锁NonfairSync两个子类。ReenTrantLock默认使用非公平锁，也可以通过构造器来显示的指定使用公平锁。
```java
    //非公平锁
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }

    //公平锁
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
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
通过源码对比，可以看出公平锁和非公平锁的lock()方法唯一的区别就在于公平锁在获取同步状态时多了一个限制条件：hasQueuedPredecessors()，该方法主要是判断当前线程是否位于同步队列中的第一个，如果是则返回true，否返回false。

综上，公平锁就是通过同步队列来实现多个线程按照申请锁的顺序来获取锁，从而实现公平的特性。非公平锁加锁时不考虑排队等待问题，直接尝试获取锁，所以存在后申请却先获取锁的情况。

## 可重入锁 非可重入锁
可重入锁又名递归锁，是指同一个线程在外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁（前提锁对象得是同一个对象或者class），不会因为之前已经获取过还没释放而阻塞。Java中ReentrantLock和synchronized都是可重入锁，可重入锁的优点是可以在一定程度上避免死锁。
```java
public class Widget {
    public synchronized void doSomething() {
        System.out.println("方法1执行...");
        doOthers();
    }

    public synchronized void doOthers() {
        System.out.println("方法2执行...");
    }
}
```
在上面的代码中，类中的两个方法都是synchronized修饰的，doSomething()方法中调用doOthers()方法。因为内置锁是可重入的，所以同一个线程在调用doOthers()时可以直接获取当前对象的锁，进入doOthers()中进行操作。

如果是一个不可重入锁，那么当前线程在调用doOthers()之前需要将执行doSomething()时获取当前对象的锁释放掉，实际上该对象锁已经被当前线程所持有，且无法释放，所以会出现死锁。

源码分析：可重入锁ReentrantLock和NonReentrantLock都继承父类AQS，父类AQS中维护了一个同步状态status来计数重入次数，status初始值为0。
当前线程获取锁时，可重入锁尝试获取并更新status值，如果status=0表示没有其他线程在执行同步代码，则把status值为1，当前线程开始执行。如果status!=0，则判断当前线程是否获取到这个锁的线程，如果是的话执行status+1，且当前线程可以再次获取锁。而非可重入锁时直接去获取并尝试更新当前status的值，如果status!=0会导致其获取锁失败，当前线程阻塞。
<br><img src=img/重入锁源码.png><br>

## 独享锁 共享锁
独享锁也叫排他锁，是指该锁一次只能被一个线程持有。如果线程T对数据A加上排他锁后，则其他线程不能再对A加任何类型的锁。获得排他锁的线程既能读取数据，又能修改数据。JDK的synchronized和JUC中的Lock的实现类就是互斥锁。

共享锁指的是该锁可被多个线程所持有。如果线程T对数据A加上共享锁后，则其他线程只能对A加共享锁，不能加排他锁。获得共享锁的线程只能读数据，不能修改数据。

独享锁和共享锁也是通过AQS来实现的，通过实现不同的方法，来实现独享或者共享。下面是ReentrantReadWriteLock的部分源码：
<br><img src=img/读写锁.png><br>

ReentrantReadWriteLock中有两个锁，ReadLock和WriteLock，一个读锁一个写锁，合成读写锁，这两个锁都是靠内部类Sync实现的锁，Sync是AQS的一个子类。

在ReentrantReadWriteLock里面，读锁和写锁的锁主体都是Sync，但读锁和写锁的加锁方式不一样。读锁是共享锁，写锁是独享锁。读锁的共享锁可保证并发读非常高效，而读写、写读、写写的过程互斥，因为读锁和写锁是分离的，所以ReentrantReadWriteLock的并发性相比一般的互斥锁有了很大提升。

AQS中有一个state字段(int类型，32位)，该字段用来描述有多少线程持有锁。在独享锁中这个值通常是0或1(如果是重入锁的话state就是重入的次数)，在共享锁中state就是持有锁的数量。但是在ReentrantReadWriteLock中有读写两把锁，所以需要在一个整型变量state上分别描述读锁的写锁的数量，于是将state变量切割为两部分，高16位表示读锁个数，低16位表示写锁个数。如下图所示：
<br><img src=img/state.png><br>

写锁的加锁源码：
```java
protected final boolean tryAcquire(int acquires) {
	Thread current = Thread.currentThread();
	int c = getState(); // 取到当前锁的个数
	int w = exclusiveCount(c); // 取写锁的个数w
	if (c != 0) { // 如果已经有线程持有了锁(c!=0)
    // (Note: if c != 0 and w == 0 then shared count != 0)
		if (w == 0 || current != getExclusiveOwnerThread()) // 如果写线程数（w）为0（换言之存在读锁） 或者持有锁的线程不是当前线程就返回失败
			return false;
		if (w + exclusiveCount(acquires) > MAX_COUNT)    // 如果写入锁的数量大于最大数（65535，2的16次方-1）就抛出一个Error。
            throw new Error("Maximum lock count exceeded");
		// Reentrant acquire
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() || !compareAndSetState(c, c + acquires)) // 如果当且写线程数为0，并且当前线程需要阻塞那么就返回失败；或者如果通过CAS增加写线程数失败也返回失败。
		return false;
	setExclusiveOwnerThread(current); // 如果c=0，w=0或者c>0，w>0（重入），则设置当前线程或锁的拥有者
	return true;
}
```
* 这段代码先获取到当前锁的个数c，然后通过c来获取写锁的个数w。因为写锁是低16位，所以取低16位的最大值与当前的c作与运算，高16位和0与运算后是0，剩下的就是低运算位的值，同时也是持有写锁的线程数目。
* 在取到写锁线程的数目后，首先判断是否已经有线程持有了锁。如果已经有线程持有了锁(c!=0)，则查看当前写锁线程的数目，如果写线程数为0（即此时存在读锁）或者持有锁的线程不是当前线程就返回失败（涉及到公平锁和非公平锁的实现）。
* 如果写入锁的数量大于最大值，就抛出异常。
* 如果当写线程数为0，那么读线程也应该为0(那么读线程也应该为0，因为上面已经处理了c!=0的情况)，并且当前线程需要阻塞那么就返回失败；如果通过CAS增加写线程数失败也返回失败。
* 如果c=0，w=0或者c>0,w>0（重入），则设置当前线程为锁的拥有者，返回成功。

tryAcquire()除了重入条件（当前线程为获取了写锁的线程）之外，增加了一个读锁是否存在的判断。如果存在读锁，则写锁不能被获取，原因在于：必须确保写锁的操作对读锁可见，如果允许读锁在已被获取的情况下对写锁的获取，那么正在运行的其他读线程就无法感知到当前写线程的操作。因此，只有等待其他读线程都释放了读锁，写锁才能被当前线程获取，而写锁一旦被获取，则其他读写线程的后续访问均被阻塞。写锁的释放与ReentrantLock的释放过程基本类似，每次释放均减少写状态，当写状态为0时表示写锁已被释放，然后等待的读写线程才能够继续访问读写锁，同时前次写线程的修改对后续的读写线程可见。

读锁的源码：
```java
protected final int tryAcquireShared(int unused) {
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;                 // 如果其他线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态
    int r = sharedCount(c);
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}
```
可以看到在tryAcquireShared(int unused)方法中，如果其他线程已经获取了写锁，则当前线程获取读锁失败，进入等待状态。如果当前线程获取了写锁或者写锁未被获取，则当前线程（线程安全，依靠CAS保证）增加读状态，成功获取读锁。读锁的每次释放（线程安全的，可能有多个读线程同时释放读锁）均减少读状态，减少的值是“1<<16”。所以读写锁才能实现读读的过程共享，而读写、写读、写写的过程互斥。