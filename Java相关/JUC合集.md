# 0.Java线程创建流程

1. 创建Java线程实例，堆中分配内存。
2. JVM为线程创建线程私有资源: 虚拟机栈、程序计数器
3. start()方法执行线程，操作系统为Java线程分配对应内核线程，线程处于就绪状态。
   - 内核线程 - 操作系统资源
4. 线程运行过程被CPU切换运行 （线程多时CPU切换频率可能加大，上下文切换频率加大）
5. 线程运行完毕/异常，Java线程被垃圾收集器回收。



# 1. synchronized理解

使用synchronized同步锁线程释放锁的情况

1. 线程执行完同步方法/同步代码块，释放锁
2. 线程执行时发生异常，JVM让线程自动释放锁
3. 同步代码块/方法中，锁对象执行了wait方法，线程释放锁



- Thread类的方法，线程状态转换对应的方法，Object类有关的方法 interrupt系列方法
- Volatile与Synchronized，Java锁升级



Entry Set: 锁池； Wait Set: 等待池

- Entry Set: 当前线程想要获取已被其他线程持有的对象锁，进入Entry Set，处于BLOCKED阻塞状态
  - 对象锁被释放时，JVM唤醒Entry Set中某个线程
- Wait Set: 线程调用wait()方法，释放对象锁，进入对象的 Wait Set 处于WAITING状态
  - notify()方法唤醒处于Wait Set中的一个线程于Entry Set中，notifyAll()则全部转移至Entry Set
- notify()死锁的场景
  - **基于生产者消费者模型，设容量为1。 首先两个消费者调用wait方法处于等待池；一个生产者线程占用锁生产另一个生产者线程在锁池；生产者线程执行完后执行唤醒操作将一个消费者从等待池拉回锁池子；生产者线程抢到锁wait进入等待池；消费者抢到锁，消费，唤醒一个消费者线程；死锁。**
- notifyAll()注意: 线程run方法执行完毕时会自动执行notifyAll()方法

## 1.1 Thread

1. start() 启动线程，系统开启新线程执行用户定义子任务，为线程分配所需资源，就绪

2. run() 方法，线程获取CPU后执行run方法中的任务

3. sleep() 方法，线程时间等待，交出CPU(处于阻塞/时间等待/等待)但不交出内部锁，结束后就绪

4. yield() 方法，线程交出CPU进入就绪态但不交出内部锁，让同/高优先级就绪线程得到执行

5. join()方法, 让父线程等待子线程执行完毕/一定时间再调度 - 调用的wait()

   - ```java
     while (thread1.isAlive() || thread2.isAlive()) {
         //只要两个线程中有任何一个线程还在活动，主线程就不会往下执行
     }
     ```

6. interrupt()方法，中断处于阻塞状态的线程，使其抛出异常而结束。默认不会中断运行状态线程
   - isInterrupted() : 判断线程是否处于中断状态 (默认不会改变线程中断状态)
   - isInterrupted(boolean) : true : 查看某线程中断状态，若boolean为true则改变线程中断状态
   - interrupted() 返回当前线程中断状态并清除中断状态

7. setDaemon() 设置守护线程

   - 守护线程依赖于创建它的线程，创建起的线程消亡则守护线程也消亡，用户线程不依赖。

# 2. JUC锁

相较于synchronized的优势

1. 读多写少情况下保持高效率 : ReentrantReadWriteLock读写锁
2. 占有锁的线程不会长时间阻塞
   - tryLock(time)方式限制获取时间；
   -  lockInterruptibly()方式，若等待时其他线程调用该线程的interrupt方法中断，线程可响应中断放弃锁



## 2.1 ReenTrantLock独占锁

### 2.1.1 Lock接口方法概述

lock() : 等待获取锁直至成功

**lockInterruptibly() : 获取锁，可其响应他线程的interrupt方法中断**

tryLock() : 尝试获取锁，成功/失败

tryLock(Time) : Time时间内尝试获取锁，可响应中断

unlock() : 释放锁

**注: 实现类lock系列方法锁的是实现对象(new ReenTrantLock())**

## 2.2.2 ReenTrantLock类结构

继承自Lock接口；内部类Sync继承自AbstractQueuedSynchronizer(AQS)抽象类，有FairSync与NonfairSync两种实现。

**可重入锁，允许一个线程多次获取锁，线程释放锁释放获取的次数使得state=0时其他线程才能获取锁**

### 方法实现

lock()方法 (基于FairSync)

1. acquire(1) : arg-计算重入次数stare+=arg

   - tryAcquire()尝试抢占锁，抢占成功直接返回；否则将当前线程封装为Node节点添加到AQS队列中

   1. tryAcquire(1)尝试抢占锁 true/false
   2. 若false，执行addWaiter(Node.EXCLUSIVE)将线程封装到EXELUSIVE型Node中添加到AQS
   3. 调用acquireQueued将线程阻塞
   4. 等待其他线程释放锁唤醒，唤醒后重新竞争锁的使用

2. tryAcquire(1) : 获取当前锁的状态，无锁-CAS(state)获取；重入锁-增加重入次数

   1. getState()获取state值，若为0通过CAS更新为1，CAS更新成功直接返回true，CAS失败返回
   2. state不为0，getExclusiveOwnerThread()方法判断当前锁持有线程，若为当前线程增加重入次数获取成功
   3. state不为0且锁线程非当前线程，获取锁失败返回false

3. addWaiter(Node.EXCLUSIVE)：线程抢占锁失败后，执行addWaiter(Node.EXCLUSIVE)方法将线程封装成EXCLUSIVE型Node节点追加到AQS

   1. 将当前线程封装为Node节点
   2. AQS同步队列的tail节点不为null，将当前线程节点追加到链表中 **(CAS tail为now)，CAS失败后呢？**
   3. 若同步队列为空 or 2步骤CAS失败，enq(node) 自旋(初始化链表)将当前线程追加到链表中

4. acquireQueued(newNode，1) 阻塞线程, newNode为新加入等待队列的线程节点

   - 自旋执行 

   1. 若newNode前驱节点为head则尝试获取锁，若获取成功则于AQS中移除newNode

   2. 否则通过shouldParkAfterFailedAcquire() && parkAndCheckInterrupt() 根据前驱节点类型判断是否阻塞并阻塞线程

      1. SIGNAL - 可以放心阻塞
      2. 若为CANCELLED则向前把CANCELLED状态节点移除返回false
      3. 前驱为默认0 or propagate， 修改为signal返回false

      - 前驱节点为SIGNAL时才可以放心阻塞，**否则需要设置前驱节点然后自旋判断** <u>(可能对应当前节点为头节点时不直接阻塞而是加一重循环判断避免直接阻塞造成效率缓慢？)</u>
      - parkAndCheckInterrupt方法会返回当前线程的中断状态，最终在node获取锁之后返回给acquire()方法时会根据true/false判断是否中断当前获取锁的线程。**可相应lockInterruptibly()方法抛出中断异常。**


unlock()方法， sync.release(1) -> tryRelease(1) -> unparkSuccessor()

1. sync.release(1),
2. tryRelease(1), 释放锁，若释放后无所则需唤醒后继线程
   1. 判断当前线程是否为锁持有者，若不是直接抛出异常
   2. 重入次数减少，判断当前线程是否完全释放锁。 
   3. 0-true， 锁拥有线程设置为null，锁状态设置为无锁state=0, unparkkSuccessor(Node)唤醒前驱界定啊
   4. false，当前线程仍只有锁，重入次数减少
3. unparkSuccessor()
   - 获取首个非CANCELLED状态Node，调用unpark(Thread)方法将对应线程唤醒

### 公平锁与非公平锁

unlock() 方法无差异

1. lock方法差异： 新线程优先通过cas操作尝试获取锁
2. tryAcquire(arg):  无论AQS是否有线程等待，若当前锁为无锁状态会先尝试通过CAS抢占锁

## 2.2.3 AQS同步队列

<u>JUC中绝大部分同步工具的核心组件</u>，FIFO的双向队列，线程获取锁失败时线程封装成Node节点来记录阻塞线程。

**CountDownLatch, CyclicBarrier**

state: int 表示当前线程获取锁的可重入次数

head,tail : Node 维护双端队列

exelusiveOwnerThread : Thread 继承自AbstractOwnableSynchronizer ：当前共享资源的持有线程



### AQS插入/删除步骤

插入: 初始化head，tail指向为null； 当有线程需添加时初始化一个Thread为null的Node null，head,tail均指向该null，然后再向null后方插入Thread对象为阻塞线程的new Node

删除: FIFO, head指向temp = null.next, temp.thread=null 



### Node内部结构

prev,next : 前驱后驱节点

thread : 存放进去AQS里面的线程

waitStatus : 记录当前线程等待状态

- CANCELLED:取消线程 1
- SIGNAL:需被唤醒线程 -1
- CONDITION:线程在条件队列中等待 -2
- PROPAGATE:线程释放共享资源时通知其他节点 -3
- 0 : 默认状态

Node节点类型 (对应支持AQS的共享模式/独占模式)

- SHARED:该线程获取共享资源时被阻塞挂起 *eg.读锁*
- EXELUSIVE:该线程获取独占资源被阻塞挂起 *eg.排他锁 写锁*

# 3. ThreadLocal 线程私有内存

ThreadLocal - **对于一个变量，每个线程对其访问都是访问线程自身的变量 -> 避免共享变量访问的线程不安全问题。**

结构： Thread类内置两个变量threadLocals与inheritableThreadLocals都是ThreadLocal内部类ThreadLocalMap(类似HashMap)的变量，线程第一次调用set/get方法时才会创建实例。

- threadLocals: ThreadLocal.ThtreadLocalMap (key-当前定义ThreadLocal变量this引用，value-set方法设置的值)
- 每个线程的本地变量存放在自己的本地内存变量threadLocals中
- **每个线程可以对应多个ThreadLocal**, 通过不同的ThreadLocal-key获取所需的不同value

- **InheritableThreadLocal类可以实现子线程访问父线程本地变量，继承自ThreadLocal 仍待解决？



- ThreadLocal内存泄漏的原因： TheadLocalMap使用ThreadLocal的弱引用作为key，当ThreadLocal不存在外部强引用时key在下一次GC时会被回收 -> ThreadLocalMap中的key为null，但value还存在强引用Thread Ref - Thread - ThreadLocalMap - Entry - Value  (线程不shutdown的话key为null的Entry的value一直存在)

  - 为什么key弱引用：key强引用时对应的ThreadLocal不会被回收，弱引用时ThreadLocal会被回收，*其对应的value在下一次调用get/set/remove方法时会被清楚*
    - 在ThreadLocal的get()、set()、remove()方法调用的时候会清除掉线程ThreadLocalMap中所有Entry中Key为null的Value，并将整个Entry设置为null，利于下次内存回收。

  1. 每次使用ThreadLocal都调用其remove方法清除数据
  2. 将ThreadLocal变量定义为static，一直存在ThreadLocal的强引用，保证任何时候都能通过ThreadLocal弱引用访问到Entry的value进而清除。



# 4. 并发控制类

## 3.1 CyclicBarrier栅栏

参数：

- lock -> ReentrantLock();  调用wait方法获取锁
- trip -> Condition(); 等待队列
- parties; 每次拦截的线程数
- barrierCommand -> Runnable(); 每次换代时执行的线程
- generation -> static Generation(); 当前代
- count; 计数器 0时换代



**构造器指定parties数以及换代执行线程Runnable**



核心方法await( (time, unit) ) -> dowait(time, nanos)

- 核心操作：**打翻栅栏，抛出异常**

1. lock获取锁
2. 若栅栏被打翻则抛出打翻异常
3. 若线程被中断则打翻栅栏，唤醒线程，抛出中断异常
4. 计数-1
5. 若计数减为0则执行换代线程，重新初始化并唤醒所有线程
6. 若计数不为0，等待
7. 若当前线程等待期间被中断则打翻栅栏唤醒其他线程



*Question：Thread.isInterrupted()的能量？  dowait方法中的各种打翻栅栏？*

1. while (!Thread.isInterrupted()) 没被打断就继续运行
2. 

Eg. 赛马，每秒(每轮)统计每匹马的已跑路程。

## 3.2 CountDownLatch

**目标线程阻塞直至CountDown方法将AQS中state减为1.**

1. countDown() 调用tryReleaseShared()方法降低state

   - tryReleaseShared()自旋重试减state

2. await() 阻塞直至state减为0

   - acquireSharedInterruptibly()响应中断，中断抛异常

   - 根据tryAcquireShared()方法判断state占用情况（1-0；-1-有）, <0则调用doAcquireInterruptibly()方法自旋尝试获取锁。 
   - doAcquireInterruptibly():  先addWaiter添加到AQS，自旋重试获取锁，队首节点直接判断，非队首阻塞**shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt()**阻塞组合拳，成功则获取锁。PACI方法返回中断标志true时抛出异常，执行cancelAcquire()方法取消获取锁。
     - 类似于acquireQueued()方法



区别于CyclicBarrier

- CountDownLatch中的屏障只可使用一次；而CyclicBarrier中的屏障可复用直至运行结束。



# 5. 线程池

## 5.1 阻塞队列参数

SynchronousQueue: 零容量无缓冲阻塞队列，put方法没有对应的take方法时会被阻塞，排除系统关闭时阻塞队列消息丢失。 （线程数可能很多）

LinkedBlockingQueue: 链表结构阻塞队列，可不指定容量(MAX)

ArrayBlockingQueue: 数组结构阻塞队列，需指定容量

PriorityBlockingQueue: 优先级无限阻塞队列，先执行优先级高 （指定Comparator比较规则）

## 5.2 Executors工厂

newCachedThreadPool(): 缓存线程池，线程池长超过处理需要时可灵活回收空闲线程；若无可回收则新建线程。

- SynchronousQueue

newFixedThreadPool(int nThread):

- core = maxi = nThread; time=0L(无线); LinkedBlockingQueue()无限阻塞队列。
- 创建可容纳固定数量线程的池子，每个线程存活时间无限。

newSingleThreadExecutor(): 

- core = maxi = 1, LinkedBlockingQueue() 同一时间只会执行一个任务

newScheduledThreadPool(int nThread):

- core = nThread; DelayedWorkQueue() 创建一个固定大小线程池



## 5.3 线程池使用说明

启发式设计线程池：根据访问需求动态调整线程池线程数量

- CPU负载不高 (<75%) 且线程池线程很忙
- CPU负载较高 (>90%) 线程不再放回线程池 **(return ??)**



初始参数设置：I/O密集型进行判断



## Question:

线程池自我设置， 判别run/start运行状态。



- ThreadPool调用execute()方法执行时
  1. 初始化Thread集合参数，将Thread中的run方法设置为wait()阻塞，等待被执行调用。
  2. 调用start方法时先将对应Thread中的wait()调用nofity()唤醒，执行后续业务逻辑。（设置其他方法、传参等）
  3. 阻塞队列的选择与判断？





# 补充问题

## 1. this引用逃逸

发生场景：类尚未完全初始化时将this抛出供其他线程使用。

- 产生原因: **指令重排序，多线程环境下若多个处理器同时访问同一块内存，操作系统允许多个处理器不按规定顺序将指令送往多个处理器处理单元以加快程序的执行，但保证方法执行过程中依赖赋值结果的地方**

- 场景：构造器抛出this引用供其他线程使用；构造器内部类使用外部类情况，外部类尚未初始化完全内部类就获取
- 避免：构造器内滥用this引用。

