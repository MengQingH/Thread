## Java对象头
在JVM中，对象在内存中的布局分为三块区域：对象头、实例数据和内存填充，如下图所示：
<br><img src=img/对象.png><br>
<br><img src=img/对象1.png><br>
* **实例变量**：存放对象的属性信息，包括父类的属性信息，如果是数组的实例部分还包括数组的长度，这部分按四字节对齐。
* **填充数据**：由于虚拟机要求对象起始地址必须是8字节的整数倍。填充数据不是必须存在的，仅仅是为了字节对齐。

* **对象头**：顶部的对象头是实现synchronized的基础，一般而言synchronized对象锁是存放在对象头中的。Java对象头一般**占有两个机器码**（在32位虚拟机中，1个机器码等于4字节，也就是32bit），（如果对象是数组类型，则需要三个机器码，因为JVM虚拟机可以通过Java对象的元数据信息确定Java对象的大小，但是无法从数组的元数据来确认数组的大小，所以用一块来记录数组长度。）。主要结构是由Mark word和Class Metadata Address组成，主要结构如下：
    * **Mark word**：存储对象的hashCode、锁信息或分代年龄或GC标志等信息。
    * **Class Metadata Address**：类型指针指向对象的类元数据，JVM通过这个指针确定该对象是哪个类的实例

### **Mark Word**
Mark Word用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等等。注意这个Mark Word结构并不是固定的，它会随着锁状态标志的变化而变化，而且里面的数据也会随着锁状态标志的变化而变化，这样做的目的是为了节省空间，32位JVM下，可能以下几种结构：
<br><img src=img/MarkWord2.png><br>
**重量级锁也就是通常说synchronized的对象锁**，锁标识位为10，其中**指针指向的是monitor对象**（也称为管程或监视器锁）的起始地址。

## Monitor（管程）
每个对象都存在着一个 monitor 与之关联，对象与其 monitor 之间的关系有存在多种实现方式，如monitor可以与对象一起创建销毁或当线程试图获取对象锁时自动生成，但当一个 monitor 被某个线程持有后，它便处于锁定状态。在Java虚拟机(HotSpot)中，monitor是由**ObjectMonitor**实现的，其主要数据结构如下（位于HotSpot虚拟机源码ObjectMonitor.hpp文件，C++实现的）：
```c
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录个数
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```
ObjectMonitor中有两个队列，**_WaitSet 和 _EntryList**，用来**保存ObjectWaiter对象列表**（每个等待锁的线程都会被封装成ObjectWaiter对象）

_owner指向**持有ObjectMonitor对象的线程**，当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，当线程获取到对象的monitor 后进入 _owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1，若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 _WaitSet 集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)。

monitor对象存在于每个Java对象的对象头中(存储的指针的指向)，synchronized锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因，同时也是notify/notifyAll/wait等方法存在于顶级对象Object中的原因。


## synchronized代码块的原理
反编译一个含有synchronized代码块的代码，得到的字节码如下：
```java
//syncTask即为含有synchronized代码块的方法
public void syncTask(){
    //同步代码块
    synchronized (this){
        i++;
    }
}

//得到的字节码
public void syncTask();
    descriptor: ()
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter  //注意此处，进入同步方法
         4: aload_0
         5: dup
         6: getfield      #2             // Field i:I
         9: iconst_1
        10: iadd
        11: putfield      #2            // Field i:I
        14: aload_1
        15: monitorexit   //注意此处，退出同步方法
        16: goto          24
        19: astore_2
        20: aload_1
        21: monitorexit //注意此处，退出同步方法
        22: aload_2
        23: athrow
        24: return
      Exception table:
      //省略其他字节码.......
}
```
从字节码中可知实现同步块使用的是**monitorenter和monitorexit**指令，其中**monitorenter指令指向同步代码块的开始位置**，**monitorexit指令则指明同步代码块的结束位置**。

当执行monitorenter指令时，当前线程将试图获取 objectref(即对象锁) 所对应的 monitor 的持有权，当 objectref 的 monitor 的进入计数器为 0，那线程可以成功取得 monitor，并将计数器值设置为 1，取锁成功。如果当前线程已经拥有 objectref 的 monitor 的持有权，那它可以重入这个 monitor (关于重入性稍后会分析)，重入时计数器的值也会加 1。倘若其他线程已经拥有 objectref 的 monitor 的所有权，那当前线程将被阻塞，直到正在执行线程执行完毕，即monitorexit指令被执行，执行线程将释放 monitor(锁)并设置计数器值为0 ，其他线程将有机会持有 monitor 。

值得注意的是编译器将会确保无论方法通过何种方式完成，方法中调用过的每条 monitorenter 指令都有执行其对应 monitorexit 指令，而无论这个方法是正常结束还是异常结束。为了保证在方法异常完成时 monitorenter 和 monitorexit 指令依然可以正确配对执行，编译器会自动产生一个异常处理器，这个异常处理器声明可处理所有的异常，它的目的就是用来执行 monitorexit 指令。从字节码中也可以看出**多了一个monitorexit指令，它就是异常结束时被执行的释放monitor 的指令**。

## synchronized方法的原理
方法级的同步是隐式，即无需通过字节码指令来控制的，它实现在方法调用和返回操作之中。JVM可以从方法常量池中的方法表结构(method_info Structure) 中的 **ACC_SYNCHRONIZED 访问标志区分一个方法是否同步方法**。

当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先持有monitor（虚拟机规范中用的是管程一词），然后再执行方法，最后在方法完成（无论是正常完成还是非正常完成）时释放monitor。在方法执行期间，执行线程持有了monitor，其他任何线程都无法再获得同一个monitor。如果一个同步方法执行期间抛出了异常，并且在方法内部无法处理此异常，那这个同步方法所持有的monitor将在异常抛到同步方法之外时自动释放。字节码层面的实现：
```java
//synchronized方法
public synchronized void syncTask(){
        i++;
}

//反编译获取的字节码
public synchronized void syncTask();
    descriptor: ()V
    //方法标识ACC_PUBLIC代表public修饰，ACC_SYNCHRONIZED指明该方法为同步方法
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #2                  // Field i:I
         5: iconst_1
         6: iadd
         7: putfield      #2                  // Field i:I
        10: return
      LineNumberTable:
        line 12: 0
        line 13: 10
}
```
从字节码中可以看出，synchronized修饰的方法并没有monitorenter指令和monitorexit指令，取得代之的确实是**ACC_SYNCHRONIZED标识，该标识指明了该方法是一个同步方法**，JVM通过该ACC_SYNCHRONIZED访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。
