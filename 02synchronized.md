## synchronized
synchronized用来给实例对象、类对象或者代码块**加锁**，加锁之后线程要进入这段代码就需要**获取锁**，线程执行完成或者抛出异常的时候需要**释放锁**。当它锁定一个对象或者方法或者代码块的时候，同一时刻只有一个线程可以执行这段代码，其他线程处于等待状态，直到当前线程处理完成释放锁。

## 三种应用
Java中的每一个对象都可以作为锁，synchronized有一下三种应用方式：
* 修饰实例方法，作用于当前**实例对象（通过new创建的对象）**加锁，进入同步代码前要获取当前实例的锁。
* 修饰静态方法，作用于当前**类对象（类的.class对象）**加锁，进入同步代码前要获得当前类对象的锁
* 修饰代码块，指定加锁对象，给指定对象加锁，进入代码块之前要获取给定对象的锁。

## 作用于实例方法
sychronized修饰实例对象中的实例方法，不包括静态方法。加锁对象是这个实例对象，进入该对象前需要获取锁。

下面的代码中，开启两个线程操作同一个共享变量i，如果第二个线程在第一个线程读取旧值和写回新值期间读取i的域值，那么第二个线程就会与第一个线程一起看到同一个值，并执行相同值的加1操作，就会造成错误。所以要对increase()方法加synchronized关键字，如果没有使用synchronized关键字，那么值很可能小于20000。
```java
public class SynDemo1 implements Runnable {

    static int i;

    synchronized public void increase(){
        i++;
    }

    @Override
    public void run() {
        for (int j = 0; j < 10000; j++) {
            increase();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        SynDemo1 s = new SynDemo1();
        Thread t1 = new Thread(s);
        Thread t2 = new Thread(s);

//        SynDemo1 s1 = new SynDemo1();
//        t2 = new Thread(s1);

        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(i);

    }
}
```
在上面的代码中，两个线程都是在s**对象**上操作的线程，获取的是s对象上的锁，这里是s这**一个对象**。如果使用代码中注释的部分，那么就算加了synchronized关键字，也会出现错误，因为这时创建了两个对象，存在两个不同的实例锁，两个线程分别进入各自的对象锁，也就是说t1、t2线程使用的是不同的对象锁，因此线程安全是无法保证的。解决方法是将synchronized作用于静态的increase方法，这样的话对象锁就是当前类对象，无论创建多少个实例对象，锁都只有一个。

## 作用于静态方法
当synchronized作用于静态方法时，其锁就是**当前类的class对象锁**。由于静态成员不专属于任何一个实例对象，是类成员，因此通过class对象锁可以控制静态成员的并发操作。需要注意的是如果一个线程A调用一个实例对象的非static synchronized方法，而线程B需要调用这个实例对象所属类的静态synchronized方法，是允许的，不会发生互斥现象，因为访问静态synchronized方法占用的锁是当前类的class对象，而访问非静态synchronized方法占用的锁是当前实例对象锁，但是会产生安全问题。
```java
public class SynDemo2 implements Runnable{
    static int i;

    synchronized public static void increase(){
        i++;
    }

    @Override
    public void run() {
        for (int j = 0; j < 10000; j++) {
            increase();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(new SynDemo2());
        Thread t2 = new Thread(new SynDemo2());

        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(i);
    }
}
```

## 作用域代码块
除了使用关键字修饰实例方法和静态方法外，还可以使用同步代码块，代码块可以**自定义使用的锁**，可以使用次对象的对象锁，也可以使用类对象锁。在某些情况下，我们编写的方法体可能比较大，同时存在一些比较耗时的操作，而需要同步的代码又只有一小部分，如果直接对整个方法进行同步操作，可能会得不偿失，此时我们可以使用同步代码块的方式对需要同步的代码进行包裹，这样就无需对整个方法进行同步操作。
```java
public class SynDemo3 implements Runnable {

    static int i;

    @Override
    public void run() {
        //使用this给当前实例对象加锁或者使用.class
        synchronized (this) {
            for (int j = 0; j < 10000; j++) {
                i++;
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        SynDemo3 s = new SynDemo3();
        Thread t1 = new Thread(s);
        Thread t2 = new Thread(s);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(i);
    }
}
```