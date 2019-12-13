# 知识点总结

## 并发编程的优缺点

### 优点

- 并发编程的形式可以将多核CPU的计算能力发挥到极致，性能得到提升；
- 面对复杂业务模型，并行程序会比串行程序更适应业务需求，而并发编程更能吻合这种业务拆分。

### 缺点

#### 频繁的上下文切换

 (特别的，cpu由于核心有限，往往是通过时分复用的方式实现并行多线程，会涉及大量的内存拷贝和上下文切换开销)

> 解决思路

- 无锁并发编程：可以参照concurrentHashMap锁分段的思想，不同的线程处理不同段的数据，这样在多线程竞争的条件下，可以减少上下文切换的时间。
- CAS算法，利用Atomic下使用CAS算法来更新数据，使用了乐观锁，可以有效的减少一部分不必要的锁竞争带来的上下文切换
- 使用最少线程：避免创建不需要的线程，比如任务很少，但是创建了很多的线程，这样会造成大量的线程都处于等待状态
- 协程：在单线程里实现多任务的调度，并在单线程里维持多个任务间的切换

可以使用Lmbench3测量上下文切换的时长 vmstat测量上下文切换次数

#### 线程安全问题

产生本质在于：

1. 主内存和java线程工作内存的不一致性；（为什么需要cpu缓存？因为cpu的频率太快了，主存跟不上，cache是为了缓解两者速度不匹配这一问题）
2. 重排序。（处理器为提高运算速度而做出违背代码原有顺序的优化）

__上述内容后面会仔细展开__

如死锁问题

```java
public class DeadLockDemo {
    private static String resource_a = "A";
    private static String resource_b = "B";

    public static void main(String[] args) {
        deadLock();
    }

    public static void deadLock() {
        Thread threadA = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (resource_a) {
                    System.out.println("get resource a");
                    try {
                        Thread.sleep(3000);
                        synchronized (resource_b) {
                            System.out.println("get resource b");
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
        Thread threadB = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (resource_b) {
                    System.out.println("get resource b");
                    synchronized (resource_a) {
                        System.out.println("get resource a");
                    }
                }
            }
        });
        threadA.start();
        threadB.start();

    }
}
```

避免死锁的方法：

1. 避免一个线程同时获得多个锁；
2. 避免一个线程在锁内部占有多个资源，尽量保证每个锁只占用一个资源；
3. 尝试使用定时锁，使用lock.tryLock(timeOut)，当超时等待时当前线程不会阻塞；
4. 对于数据库锁，加锁和解锁必须在一个数据库连接里，否则会出现解锁失败的情况

## 一些基本概念对比

### 同步VS异步

同步方法调用一开始，调用者必须等待被调用的方法结束后，调用者后面的代码才能执行。而异步调用，指的是，调用者不用管被调用方法是否完成，都会继续执行后面的代码。

### 并发与并行

并发指的是多个任务交替进行，而并行则是指真正意义上的“同时进行”。

### 临界区

临界区用来表示一种公共资源或者说是共享数据，可以被多个线程使用。但是每个线程使用时，一旦临界区资源被一个线程占有，那么其他线程必须等待。

### 阻塞和非阻塞

阻塞和非阻塞通常用来形容多线程间的相互影响，比如一个线程占有了临界区资源，那么其他线程需要这个资源就必须进行等待该资源的释放，会导致等待的线程挂起，这种情况就是阻塞，而非阻塞就恰好相反，它强调没有一个线程可以阻塞其他线程，所有的线程都会尝试地往前运行。


## 线程的状态转换以及基本操作

### 新建线程

实际上java程序天生就是一个多线程程序，包含了：（1）分发处理发送给给JVM信号的线程；（2）调用对象的finalize方法的线程；（3）清除Reference的线程；（4）main线程

- 实现多线程的方式：

  - 继承Thread类，使用run方法进行同步启动
  - 实现Runnable接口，使用start方法进行异步启动
  - 使用ExecutorService的exec(runnable)方法运行runnable类
  - 使用ExecutorService的submit(runnable/callable)，启动返回结果的线程，返回值为Future，再调用get()来获得结果。

- Thread和Runable的区别和联系（看看就好）：
  - 联系：
    Thread类实现了Runable接口。
    都需要重写里面Run方法。
  - 不同：
    实现Runnable的类更具有健壮性，避免了单继承的局限。
    Runnable更容易实现资源共享，能多个线程同时处理一个资源。
- 为什么我们调用start()方法时会执行run()方法，为什么我们不能直接调用run()方法？

    用start()来启动线程 ---> 异步执行
    而如果使用run()来启动线程 ---> 同步执行

    多线程就是为了并发执行，因此需要使用start

### 线程状态

![](https://i.loli.net/2019/12/12/4fjvQ5N2xZlT1Ig.png)

- `wait(),join(),LockSupport.lock()`方法线程会进入到**WAITING**
- `wait(long timeout)，sleep(long),join(long),LockSupport.parkNanos(),LockSupport.parkUtil()`方法线程会进入到**TIMED_WAITING**,当超时等待时间到达后，线程会切换到**Runable**的状态
- **WAITING**和**TIMED _WAITING**状态时可以通过`Object.notify(),Object.notifyAll()`方法使线程转换到**Runable**状态
- 当线程出现资源竞争时，即等待获取锁的时候，线程会进入到**BLOCKED**阻塞状态(位于AQS同步队列),当线程获取锁时，线程进入到**Runable**状态
- 线程运行结束后，线程进入到**TERMINATED**状态

当线程进入到`synchronized`方法或者`synchronized`代码块时，线程切换到的是**BLOCKED**状态，而使用`java.util.concurrent.locks`下`lock`进行加锁的时候线程切换的是**WAITING或者TIMED_WAITING**状态，因为`lock`会调用`LockSupport`的方法。

### 线程状态的基本操作

#### interrupted

其他线程可以调用该线程的`interrupt()`方法对其进行中断操作，同时该线程可以调用 `isInterrupted()`来感知其他线程对其自身的中断操作，从而做出响应。另外，同样可以调用Thread的静态方法 `interrupted()`对当前线程进行中断操作，该方法会清除中断标志位。需要注意的是，当抛出`InterruptedException`时候，会清除中断标志位，也就是说在调用`isInterrupted`会返回`false`。

一般在结束线程时通过中断标志位或者标志位的方式可以有机会去清理资源，相对于武断而直接的结束线程，这种方式要优雅和安全。

#### join

如果一个线程实例A执行了`threadB.join()`,其含义是：当前线程A会等待threadB线程终止后threadA才会继续执行。另外还提供了超时等待。

当threadB退出时会调用`notifyAll()`方法通知所有的等待线程(join让进程进入waiting状态)。

#### sleep

`sleep()` VS `wait()`:

1. `sleep()`方法是**Thread的静态方法**，而`wait()`是**Object实例方法**。
2. `wait()`方法必须要**在同步方法或者同步块中调用**（`wait/notify`方法依赖于`moniterenter`和`moniterexit`）也就是必须已经获得对象锁。而`sleep()`方法没有这个限制可以在任何地方种使用。另外，`wait()`方法**会释放占有的对象锁**，使得该线程进入等待池中，等待下一次获取资源。而`sleep()`方法只是会让出CPU并**不会释放掉对象锁**；
3. `sleep()`方法在休眠时间达到后如果再次获得CPU时间片就会继续执行，而`wait()`方法必须**等待`Object.notift/Object.notifyAll`通知**后，才会离开等待池，并且再次获得CPU时间片才会继续执行。

#### yield

这是一个静态方法，一旦执行，它会是当前线程让出CPU。
让出的CPU并不是代表当前线程不再运行了，如果在下一次竞争中，又获得了CPU时间片当前线程依然会继续运行,让出的时间片只会分配给当前线程相同优先级的线程。

在Java程序中，通过一个整型成员变量`Priority`来控制优先级，优先级的范围从`1~10`.在构建线程的时候可以通过`setPriority(int)`方法进行设置，默认优先级为5，优先级高的线程相较于优先级低的线程优先获得处理器时间片。

### 守护线程Daemon

守护线程是一种特殊的线程，就和它的名字一样，它是系统的守护者，在后台默默地守护一些系统服务，比如垃圾回收线程，JIT线程就可以理解守护线程。与之对应的就是用户线程，用户线程就可以认为是系统的工作线程，它会完成整个系统的业务操作。用户线程完全结束后就意味着整个系统的业务任务全部结束了，因此系统就没有对象需要守护的了，守护线程自然而然就会退。当一个Java应用，只有守护线程的时候，虚拟机就会自然退出。

```java
public class DaemonDemo {
    public static void main(String[] args) {
        Thread daemonThread = new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    try {
                        System.out.println("i am alive");
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        System.out.println("finally block");
                    }
                }
            }
        });
        daemonThread.setDaemon(true);
        daemonThread.start();
        //确保main线程结束前能给daemonThread能够分到时间片
        try {
            Thread.sleep(800);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

这里需要注意的是守护线程在退出的时候并不会执行finnaly块中的代码，所以将释放资源等操作不要放在finnaly块中执行，这种操作是不安全的

## Java内存模型以及happens-before规则

前面说到，出现线程安全的问题一般是因为主内存和工作内存数据不一致性和重排序导致的。

### 内存模型抽象结构

在并发编程中主要需要解决两个问题：1. 线程之间如何通信；2.线程之间如何完成同步（这里的线程指的是并发执行的活动实体）。通信是指线程之间以何种机制来交换信息，主要有两种：共享内存和消息传递。

java内存模型是**共享内存的并发模型**，线程之间主要通过**读-写共享变量来完成隐式通信**。

- 哪些是共享变量
    在java程序中所有**实例域，静态域和数组元素**都是放在堆内存中（所有线程均可访问到，是可以共享的），而局部变量，方法定义参数和异常处理器参数不会在线程间共享。共享数据会出现线程安全的问题，而非共享数据不会出现线程安全的问题。关于JVM运行时内存区域在后面的文章会讲到。
- JMM抽象结构模型
    CPU的处理速度和主存的读写速度不是一个量级的，为了平衡这种巨大的差距，每个CPU都会有缓存。共享变量会先放在主存中，每个线程都有属于自己的工作内存，并且会把位于主存中的共享变量拷贝到自己的工作内存，之后的读写操作均使用位于工作内存的变量副本
    ![](https://i.loli.net/2019/12/12/QCJ9USKVcdFrT5z.png)

    线程A和线程B之间要完成通信的话，要经历如下两步：
	1. 线程A从主内存中将共享变量读入线程A的工作内存后并进行操作，之后将数据重新写回到主内存中；
	2. 线程B从主存中读取最新的共享变量

### 重排序

为了提高性能，编译器和处理器常常会对指令进行重排序。
一般重排序可以分为如下三种：

1. **编译器优化的重排序**。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序；
2. **指令级并行的重排序**。现代处理器采用了指令级并行技术来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序；
3. **内存系统的重排序**。由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行的。

```
double pi = 3.14 //A
double r = 1.0 //B
double area = pi * r * r //C
```

这是一个计算圆面积的代码，由于A,B之间没有任何关系，对最终结果也不会存在关系，它们之间执行顺序可以重排序。因此可以执行顺序可以是A->B->C或者B->A->C执行最终结果都是3.14，即A和B之间没有数据依赖性。

如果两个操作访问同一个变量，且这两个操作有一个为写操作，此时这两个操作就存在数据依赖性这里就存在三种情况：**1. 读后写；2.写后写；3. 写后读**，这三种操作都是存在数据依赖性的，如果重排序会对最终执行结果会存在影响。**编译器和处理器在重排序时，会遵守数据依赖性，编译器和处理器不会改变存在数据依赖性关系的两个操作的执行顺序**。

- as-if-serial

不管怎么重排序（编译器和处理器为了提供并行度），（**单线程**）程序的执行结果不能被改变。编译器，`runtime`和处理器都必须遵守`as-if-serial`语义。`as-if-serial`语义把单线程程序保护了起来，遵守`as-if-serial`语义的编译器，`runtime`和处理器共同为编写单线程程序的程序员创建了一个幻觉：**单线程程序是按程序的顺序来执行的**。

### happens-before规则

JMM可以通过`happens-before`关系向程序员提供**跨线程的内存可见性保证**。

`as-if-serial` VS `happens-before`：

- 不同点：
  1. `as-if-serial`语义保证**单线程**内程序的执行结果不被改变，`happens-before`关系保证正确同步的**多线程**程序的执行结果不被改变。
  2. `as-if-serial`语义给编写**单线程**程序的程序员创造了一个幻境：单线程程序是按程序的顺序来执行的。`happens-before`关系给编写正确同步的**多线程**程序的程序员创造了一个幻境：正确同步的多线程程序是按`happens-before`指定的顺序来执行的。
- 相同点：`as-if-serial`语义和`happens-before`这么做的目的，都是为了在**不改变程序执行结果的前提下，尽可能地提高程序执行的并行度**。

具体的一共有8项规则：

1. 程序顺序规则：一个线程中的每个操作，`happens-before`于该线程中的任意后续操作。
2. 监视器锁规则：对一个锁的解锁，`happens-before`于随后对这个锁的加锁。（`syncronized`）
3. `volatile`变量规则：对一个`volatile`域的写，happens-before于任意后续对这个`volatile`域的读。
4. 传递性：如果`A happens-before B`，且`B happens-before C`，那么`A happens-before C`。
5. `start()`规则：如果线程A执行操作`ThreadB.start()`（启动线程B），那么A线程的`ThreadB.start()`操作`happens-before`于线程B中的任意操作。
6. `join()`规则：如果线程A执行操作`ThreadB.join()`并成功返回，那么线程B中的任意操作happens-before于线程A从`ThreadB.join()`操作成功返回。
7. 程序中断规则：对线程`interrupted()`方法的调用先行于被中断线程的代码检测到中断时间的发生。
8. 对象`finalize`规则：一个对象的初始化完成（构造函数执行结束）先行于发生它的`finalize(`)方法的开始。

### JMM的设计

![](https://i.loli.net/2019/12/12/hS6JVjEbBZnkwai.png)

在设计JMM时需要考虑两个关键因素:

1. 程序员对内存模型的使用 程序员希望内存模型**易于理解、易于编程**。程序员希望基于一个**强内存模型**来编写代码。
2. 编译器和处理器对内存模型的实现 编译器和处理器希望内存模型对它们的**束缚越少越好**，这样它们就可以做尽可能多的优化来提高性能。编译器和处理器希望实现一个**弱内存模型**。

实际上：

1. JMM向程序员提供的`happens-before`规则能满足程序员的需求。JMM的`happens-before`规则不但简单易懂，而且也向程序员提供了足够强的内存可见性保证（有些内存可见性保证其实并不一定真实存在，比如上面的`A happens-before B`）。
2. JMM对编译器和处理器的束缚已经尽可能少。从上面的分析可以看出，JMM其实是在遵循一个基本原则：**只要不改变程序的执行结果**（指的是单线程程序和正确同步的多线程程序），编译器和处理器怎么优化都行。例如，如果编译器经过细致的分析后，认定一个锁只会被单个线程访问，那么这个锁可以被消除。再如，如果编译器经过细致的分析后，认定一个`volatile`变量只会被单个线程访问，那么编译器可以把这个`volatile`变量当作一个普通变量来对待。这些优化既不会改变程序的执行结果，又能提高程序的执行效率。

## 彻底理解synchronized

### 实现原理

![](https://i.loli.net/2019/12/12/cV3hlvCUgKxEtyn.png)

`synchronized`可以用在方法上也可以使用在代码块中，其中方法是**实例方法和静态方法**分别锁的是该类的**实例对象和该类的对象**。而使用在代码块中也可以分为三种，具体的可以看上面的表格。这里的需要注意的是：如果锁的是**类对象**的话，尽管`new`多个实例对象，但他们仍然是属于同一个类依然会被锁住，即线程之间保证同步关系。

执行同步代码块后首先要先执行`monitorenter`指令，退出的时候`monitorexit`指令.当线程获取monitor后才能继续往下执行，否则就只能等待。而这个获取的过程是互斥的，即同一时刻只有一个线程能够获取到monitor。

**`monitorenter`** ：

每个对象都有一个`monitor`锁，包含线程持有者和计数器。

    1.如果计数器为0，则该线程进入monitor，然后将计数器设置为1，该线程即为monitor的所有者。
    2.如果线程已经占有该monitor，重新进入，则计数器加1.
    3.如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到计数器为0。

**`monitorexit`**：

    1. 指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor。
    2. 其他被这个monitor阻塞的线程可以尝试去获取这个 monitor 的所有权。

通过这两段描述，我们应该能很清楚的看出Synchronized的实现原理，Synchronized的语义底层是通过一个monitor的对象来完成，其实`wait/notify`等方法也依赖于`monitor`对象，这就是为什么只有在同步的块或者方法中才能调用wait/notify等方法，否则会抛出`java.lang.IllegalMonitorStateException`的异常的原因。


锁的**重入性**，即在同一锁程中，线程不需要再次获取同一把锁。`Synchronized`先天具有重入性。每个对象拥有一个计数器，当线程获取该对象锁后，计数器就会加一，释放锁后就会将计数器减一。(再次进入计数器加1，计数器为0时释放)

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ub3RlLnlvdWRhby5jb20veXdzL3B1YmxpYy9yZXNvdXJjZS9jYzI1NTk4YmJhZmYxOGE1NTU2OWMyNTZkZTRjZjFjNi94bWxub3RlLzEwQTk1QUI4MjM4NDQyNTY5MTA4RkM5Mzc1NUM5MUZGLzE3MDE)

### synchronized的happens-before关系

**对同一个监视器的解锁，happens-before于对该监视器的加锁**
![](https://i.loli.net/2019/12/12/jRrYKv61uMXHfIG.png)

在图中每一个箭头连接的两个节点就代表之间的happens-before关系，黑色的是通过程序顺序规则推导出来，红色的为监视器锁规则推导而出：线程A释放锁happens-before线程B加锁，蓝色的则是通过程序顺序规则和监视器锁规则推测出来happens-befor关系，通过传递性规则进一步推导的happens-before关系。现在我们来重点关注2 happens-before 5，通过这个关系我们可以得出什么？
根据happens-before的定义中的一条:如果A happens-before B，则A的执行结果对B可见，并且A的执行顺序先于B。线程A先对共享变量A进行加一，由2 happens-before 5关系可知线程A的执行结果对线程B可见即线程B所读取到的a的值为1。

### 锁获取和锁释放的内存语义

![](https://i.loli.net/2019/12/12/v3bOhW57N4AHuUz.png)

从整体上来看，线程A的执行结果（a=1）对线程B是可见的，实现原理为：释放锁的时候会将值刷新到主内存中，其他线程获取锁时会强制从主内存中获取最新的值。另外也验证了2 happens-before 5，2的执行结果对5是可见的。
从横向来看，这就像线程A通过主内存中的共享变量和线程B进行通信，A 告诉 B 我们俩的共享数据现在为1啦，这种线程间的通信机制正好吻合java的内存模型正好是共享内存的并发模型结构。

### synchronized优化

#### CAS操作

`CAS`比较交换的过程可以通俗的理解为`CAS(V,O,N)`，包含三个值分别为：`V` 内存地址存放的实际值；`O` 预期的值（旧值）；`N` 更新的新值。当多个线程使用CAS操作一个变量是，**只有一个线程会成功**，并成功更新，其余会失败。失败的线程会重新尝试，当然也可以选择挂起线程。

在JDK1.5后虚拟机才可以使用处理器提供的CMPXCHG指令实现.

Synchronized VS CAS

元老级的Synchronized(未优化前)最主要的问题是：在存在线程竞争的情况下会出现线程阻塞和唤醒锁带来的性能问题(**进入了内核态**)，因为这是一种互斥同步（阻塞同步）。而CAS并不是武断的间线程挂起，当CAS操作失败后会进行一定的尝试，而非进行耗时的挂起唤醒的操作，因此也叫做非阻塞同步。这是两者主要的区别。

#### CAS的问题

1. ABA问题 因为CAS会检查旧值有没有变化，这里存在这样一个有意思的问题。比如一个旧值A变为了成B，然后再变成A，刚好在做CAS时检查发现旧值并没有变化依然为A，但是实际上的确发生了变化。解决方案可以沿袭数据库中常用的乐观锁方式，添加一个版本号可以解决。原来的变化路径A->B->A就变成了1A->2B->3C。java这么优秀的语言，当然在java 1.5后的atomic包中提供了AtomicStampedReference来解决ABA问题，解决思路就是这样的。
2. 自旋时间过长
使用CAS时非阻塞同步，也就是说不会将线程挂起，会自旋（无非就是一个死循环）进行下一次尝试，如果这里自旋时间过长对性能是很大的消耗。如果JVM能支持处理器提供的pause指令，那么在效率上会有一定的提升。
3. 只能保证一个共享变量的原子操作
当对一个共享变量执行操作时CAS能保证其原子性，如果对多个共享变量进行操作,CAS就不能保证其原子性。有一个解决方案是利用对象整合多个共享变量，即一个类中的成员变量就是这几个共享变量。然后将这个对象做CAS操作就可以保证其原子性。atomic中提供了AtomicReference来保证引用对象之间的原子性。

## 彻底理解volatile

被volatile修饰的变量能够保证每个线程能够获取该变量的最新值，从而避免出现数据脏读的现象。

### 实现原理

在生成汇编代码时会在volatile修饰的共享变量进行写操作的时候会多出`Lock`前缀的指令。

主要有这两个方面的影响：

1. 将当前处理器缓存行的数据写回系统内存；
2. 这个写回内存的操作会使得其他CPU里缓存了该内存地址的数据无效

在多处理器下，为了保证各个处理器的缓存是一致的，就会实现**缓存一致性协议**，每个处理器通过**嗅探在总线上传播的数据来检查自己缓存的值是不是过期**了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。

1. Lock前缀的指令会引起处理器缓存写回内存；
2. 一个处理器的缓存回写到内存会导致其他处理器的缓存失效；
3. 当处理器发现本地缓存失效后，就会从内存中重读该变量数据，即可以获取当前最新值。

### volatile的happens-before关系

![](https://i.loli.net/2019/12/13/1f8M3By4DQSX9aU.png)

黑色的代表根据程序顺序规则推导出来，红色的是根据volatile变量的写happens-before 于任意后续对volatile变量的读，而蓝色的就是根据传递性规则推导出来的。

### volatile的内存语义

![](https://i.loli.net/2019/12/13/tJc1nE9KGOprXof.png)

![](https://i.loli.net/2019/12/13/MDIfRoNkmEnygYp.png)

从横向来看，线程A和线程B之间进行了一次通信，线程A在写volatile变量时，实际上就像是给B发送了一个消息告诉线程B你现在的值都是旧的了，然后线程B读这个volatile变量时就像是接收了线程A刚刚发送的消息。

## 原子性、可见性以及有序性

### 原子性

原子性是指一个操作是不可中断的，要么全部执行成功要么全部执行失败，有着“同生共死”的感觉。及时在多个线程一起执行的时候，一个操作一旦开始，就不会被其他线程所干扰。我们先来看看哪些是原子操作，哪些不是原子操作，有一个直观的印象：

```cpp
int a = 10; //1
a++; //2
int b=a; //3
a = a+1; //4
```

上面这四个语句中只有第1个语句是原子操作.

java内存模型中定义了8中操作都是原子的，不可再分的。

1. lock(锁定)：作用于主内存中的变量，它把一个变量标识为一个线程独占的状态；
2. unlock(解锁):作用于主内存中的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定
3. read（读取）：作用于主内存的变量，它把一个变量的值从主内存传输到线程的工作内存中，以便后面的load动作使用；
4. load（载入）：作用于工作内存中的变量，它把read操作从主内存中得到的变量值放入工作内存中的变量副本
5. use（使用）：作用于工作内存中的变量，它把工作内存中一个变量的值传递给执行引擎，每当虚拟机遇到一个需要使用到变量的值的字节码指令时将会执行这个操作；
6. assign（赋值）：作用于工作内存中的变量，它把一个从执行引擎接收到的值赋给工作内存的变量，每当虚拟机遇到一个给变量赋值的字节码指令时执行这个操作；
7. store（存储）：作用于工作内存的变量，它把工作内存中一个变量的值传送给主内存中以便随后的write操作使用；
8. write（操作）：作用于主内存的变量，它把store操作从工作内存中得到的变量的值放入主内存的变量中。


（重点）`AtomicX`类型之所以能实现原子性，是因为其源码中实现了`unsafe`类，而`unsafe`类的方法里会使用`CAS`方法（拿当前值与底层的值进行对比，如果相同则进行`swap`操作）

`AtomicInteger`类：

```java
public final int getAndIncrement() {
   return unsafe.getAndAddInt(this, valueOffset, 1);
}
```

`Unsafe`类：

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
   int var5;
   do {
       var5 = this.getIntVolatile(var1, var2);
   } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

   return var5;
}
```

附：AtomicLong和LongAdder类：

补充知识：对于64位的long及double类型，jvm允许将64位的读/写操作 拆分成 两次32位的读/写操作。

LongAdder：经过一系列方法确保效率高，高并发时优先使用，但精度可能会下降。

AtomicLong：序列号生成这一类需要准确的数值时，使用AtomicLong

Atomic的ABA问题：

线程 1 从内存位置V中取出A。
线程 2 从位置V中取出A。
线程 2 进行了一些操作，将B写入位置V。
线程 2 将A再次写入位置V。
线程 1 进行CAS操作，发现位置V中仍然是A，操作成功。
尽管线程 1 的CAS操作成功，但不代表这个过程没有问题——对于线程 1 ，线程 2 的修改已经丢失。

解决方法：用AtomicStampedReference/AtomicMarkableReference （维护了一个“状态戳”）

### 可见性

当一个线程修改了共享变量时，另一个线程可以读取到这个修改后的值。

a）`synchronize`：解锁前，把共享变量写入主内存。加锁时，清空工作内存中共享变量，确保从主内存中读到最新的值。（写对应释放锁，读对应加锁）

b）`volatile`：保证可见性和禁止重排序

保证可见性：volatile写操作会把共享变量写入主内存。volatile读操作会从主内存中读到最新的值，然后放到工作内存中。

禁止重排序：不允许代码段内出现重排序。

！！！！（注意：对于volatile int count，在多线程环境下count++ 并不能保证线程安全）即volatile不具备原子性！！！！

不具备的原因：https://blog.csdn.net/xdzhouxin/article/details/81236356

实际上，volatile不适合计数，但很适合作为状态量的标识

### 有序性

- synchronized
synchronized语义表示锁在同一时刻只能由一个线程进行获取，当锁被占用后，其他线程只能等待。因此，synchronized语义就要求线程在访问读写共享变量时只能“串行”执行，因此synchronized具有有序性。

- volatile
在java内存模型中说过，为了性能优化，编译器和处理器会进行指令重排序；也就是说java程序天然的有序性可以总结为：如果在本线程内观察，所有的操作都是有序的；如果在一个线程观察另一个线程，所有的操作都是无序的。在单例模式的实现上有一种双重检验锁定的方式（Double-checked Locking）。代码如下：

```java
public class Singleton {
    private Singleton() { }
    private volatile static Singleton instance;
    public Singleton getInstance(){
        if(instance==null){
            synchronized (Singleton.class){
                if(instance==null){
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

这里为什么要加volatile了？我们先来分析一下不加volatile的情况，有问题的语句是这条：
`instance = new Singleton();`
这条语句实际上包含了三个操作：1.分配对象的内存空间；2.初始化对象；3.设置instance指向刚分配的内存地址。但由于存在重排序的问题，可能有以下的执行顺序：

如果2和3进行了重排序的话，线程B进行判断if(instance==null)时就会为true，而实际上这个instance并没有初始化成功，显而易见对线程B来说之后的操作就会是错得。而用volatile修饰的话就可以禁止2和3操作重排序，从而避免这种情况。volatile包含禁止指令重排序的语义，其具有有序性。


## J.U.C

### J.U.C之AQS

(重要！背）`AQS`依赖一条**同步队列**（`FIFO`，由一个个`Node`组成，双向链表），且维护一个`volatile int`的共享资源`state`。

`state=0`表示同步状态可用（如果用于锁，则表示锁可用），`state=1`表示同步状态已被占用（锁被占用）。

`private volatile int state`

2种同步方式：**独占式，共享式**。独占式如`ReentrantLock`，共享式如`Semaphore和CountDownLatch`

模板方法模式:
　`tryAcquire(int arg)` : 独占式获取同步状态
　`tryRelease(int arg)` ：独占式释放同步状态
　`tryAcquireShared(int arg)` ：共享式获取同步状态
　`tryReleaseShared(int arg)` ：共享式释放同步状态

### CountDownLatch

![](https://img-blog.csdnimg.cn/20190120125742275.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)

（多线程运行，确保latch中的线程都执行完后其他线程才变成resume状态）

①调用countDown() 让计数减1

②调用await() 等待计数变成0后，其他线程（本例中为主线程）变成resume。

③await()可以设置时间，达到时间就接着往下进行。

注意：以下代码中的test方法中都会等待1秒，以便实现线程的堵塞。

![](https://img-blog.csdnimg.cn/20190120211335208.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)

### CyclicBarrier

（多线程计算数据，最后合并结果）

![](https://img-blog.csdnimg.cn/20190120214640816.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)

当test方法里使用await()的个数达到5个（即有五个线程ready），这五个同时变成resume。

注：这里的await()方法实际上用的就是Condition.await()；

![](https://img-blog.csdnimg.cn/20190120221301550.png)

常用场景：达到个数时回滚，以及多线程计算数据，最后合并结果：

![](https://img-blog.csdnimg.cn/20190120221950110.png)

**考题**：Java中CyclicBarrier 和 CountDownLatch有什么不同？

a）CountDownLatch 对外，CyclicBarrier对内。前者是一组线程都countDown后其他线程才能接着进行。而后者是内部多个线程相互等待，都ready后再一起执行。

b）与 CyclicBarrier 不同的是，CountdownLatch 不能重新使用。

### Semaphore

调用acquire() 和 release() 方法。先在构造函数中设置好资源数，当资源不够某个线程获取时，便进入堵塞状态。

![](https://img-blog.csdnimg.cn/20190120170506948.png)

`tryAcquire()`：立即尝试获取资源 ，获取不到便丢弃线程并返回false。（下图中只会有3个线程执行，其余的线程因为获取不到资源被丢弃）

![](https://img-blog.csdnimg.cn/20190120163145433.png)

tryAcquire()还可以带时间参数，表示多久

## 锁

### 锁的一些概念：

#### 公平锁，非公平锁：

- 公平锁：先请求锁的一定先被满足，即FIFO
- 非公平锁则反之，容易造成一个线程连续获得锁的情况，导致线程“饥饿”。

#### 悲观锁、乐观锁：

- 悲观锁：一般用于写操作，认为自己取数据的时候总是会有人在修改。再比如Java里面的同步原语synchronized关键字的实现也是悲观锁。
- 共享锁：一般用于读操作，是一种乐观锁。读的时候不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据（例如CAS），可以使用版本号等机制。

#### 排他锁、共享锁：（适用场景：1. 数据库的锁 2. ReentrantReadWriteLock）

- 共享锁（读锁）：读可以共享，但是读写之间互斥。
- 排他锁（写锁）：只有自己能操作，其他人都不能访问。
 
#### 自旋锁

对于互斥锁，如果资源已经被占用，资源申请者只能进入睡眠状态。但是自旋锁不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环在那里看是否该自旋锁的保持者已经释放了锁，“自旋”一词就是因此而得名。

#### 偏向锁，轻量级锁，重量级锁：（非重点）

- 偏向锁（无锁）：顾名思义，它会偏向于第一个访问锁的线程，如果只有一个线程运行，无竞争，那么会给这个线程一个偏向锁，消除这个线程锁重入（CAS）的开销。
- 轻量级锁（CAS）：轻量级锁是由偏向所升级来的，当第二个线程加入锁争用的时候，偏向锁就会升级为轻量级锁；轻量级锁的意图是在没有多线程竞争的情况下，通过CAS尝试将MarkWord更新为指向LockRecord的指针，减少了使用重量级锁的系统互斥量产生的性能消耗。
- 重量级锁（Synchronized）：当更新失败时，会检查MarkWord是否指向当前线程的栈帧。如果不是，说明这个锁被其他线程抢占，此时膨胀为重量级锁。
 

#### ReentrantLock （重入锁）

- 重入锁的原理

    a）和synchronize的原理一样都是使用计数器

    b）ReentrantLock中的Sync实现了AQS，所以实际上是有一条同步队列来进行控制的。

 

#### Synchronize和ReentrantLock的区别（初级程序员优先使用synchronize）

- 性能上：

    synchronized在资源竞争不是很激烈的情况下是很合适的。原因在于，编译程序通常会尽可能的优化synchronized，另外可读性非常好。 ReentrantLock: 当同步非常激烈的时候，ReentrantLock还能维持常态。

- 功能上：Synchronize有的ReentrantLock都有。但ReentrantLock有一些独有的功能：

    a）可指定是公平锁还是非公平锁 （默认是非公平锁）

    b）提供了Condition类，可分组唤醒线程

    c）lock.lockInterruptibly()，允许在等待时由其它线程调用等待线程的Thread.interrupt方法来中断等待线程的等待而直接返回，这时不用获取锁，而会抛出一个InterruptedException。

    注：ReentrantLock的锁释放一定要在finally中处理，否则可能会产生严重的后果。

 

#### Condition的使用：

Condition 实现等待的时候内部是一个**等待队列**，包含着head节点和tail节点，等待队列中的每一个节点是一个 AbstractQueuedSynchronizer.Node 实例。

Condition 的本质就是**等待队列和同步队列**的交互：

当一个持有锁的线程调用 `Condition.await()` 方法，那么该线程会**释放锁，然后构造成一个Node节点加入到等待队列的队尾**。

![](https://img-blog.csdnimg.cn/20190121184258246.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)

当一个持有锁的线程调用 `Condition.signal()` 时，它会执行以下操作：

将等待队列队首的节点移到**同步队列**，然后对其进行**唤醒操作**。

![](https://img-blog.csdnimg.cn/20190121184444400.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)

#### 读写锁 ReentrantReadWriterLock （没有读写的情况才能写）

分别有一个readLock和一个writeLock

readLock是共享锁，writeLock是排它锁，也就意味着：

1、读和读之间不互斥 （与非读写锁的区别就在这）

2、写和写之间互斥

3、读和写之间互斥（这个超级重要）

> 读写锁的锁降级概念：（获得读锁的特例）

- 锁降级：t0 上写锁 --> 修改值 --> 上读锁 --> 释放写锁 --> 使用数据 --> 释放读锁
也就是说，锁降级就是在释放写锁的前一步上读锁，因为是同一个线程，所以这两个锁不会冲突。
- 如果不使用锁降级，可能会出现以下原因：（重点看1和2）
t0 上写锁 --> 修改值 -->  释放写锁 --> 使用数据，即使用写锁后直接使用刚刚修改数据 ，与此同时，t1在t0释放写锁后获得写锁。这样t0使用的是t1修改前的旧数据。
t0 上写锁 --> 修改值 -->  释放写锁 -->上读锁 --> 使用数据，也就是用完写锁再去获取读锁，其缺点在于，如果有别的线程在等待获取写锁，那么上读锁时会进入等待队列进行等待，就无法立即使用刚修改的值。
t0 上写锁 --> 修改值 --> 使用数据 -->  释放写锁，这样做虽然不会有问题，但这样是以排它锁的方式使用读写锁，就没有了使用读写锁的优势。
- 读锁不互斥，写锁之间的互斥的，但读写之间是互斥的。

#### LockSupport：

LockSupport定义了一组以park开头的方法来阻塞当前线程，以及unpark方法来唤醒一个被阻塞的线程。

## 线程池

### 为什么用线程池?

1.创建/销毁线程伴随着系统开销，过于频繁的创建/销毁线程，会很大程度上影响处理效率。（线程复用）
2.线程并发数量过多，抢占系统资源从而导致阻塞。（控制并发数量）
3.对线程进行一些简单的管理。（管理线程）

### 线程池的原理

（1）线程复用：实现线程复用的原理应该就是要保持线程处于存活状态（就绪，运行或阻塞）

（2）控制并发数量：（核心线程和最大线程数控制）

（3）管理线程（设置线程的状态）

### 线程池的参数：

`corePoolSize`：核心线程数
`maximumPoolSize`：最大线程数(一般设置为INTMAX)
`keepAliveSeconds`:空闲存活时间 （在corePoreSize<maxPoolSize情况下有用,线程的空闲时间超过了keepAliveTime就会销毁）
`workQueue`：阻塞队列，用来保存等待被执行的任务

### 当一个任务被添加进线程池时，执行策略：

- 线程数量未达到`corePoolSize`，则新建一个线程(核心线程)执行任务
- 线程数量闲达到了`corePoolSize`，则将任务移入队列，等待空线程将其取出去执行  （通过`getTask()`方法从阻塞队列中获取等待的任务，如果队列中没有任务，`getTask`方法会被阻塞并挂起，不会占用cpu资源，整个getTask操作在**自旋**下完成）
- 队列已满，新建线程(非核心线程)执行任务
- 队列已满，总线程数又达到了maximumPoolSize，就会执行任务拒绝策略。

### 线程池的任务拒绝策略

`ThreadPoolExecutor.AbortPolicy`:丢弃任务并抛出RejectedExecutionException异常。
`ThreadPoolExecutor.DiscardPolicy`：也是丢弃任务，但是不抛出异常。
`ThreadPoolExecutor.DiscardOldestPolicy`：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
`ThreadPoolExecutor.CallerRunsPolicy`：由调用线程处理该任务

### 常见四种线程池：

#### 可缓存线程池`CachedThreadPool()`

```java
 public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

根据源码可以看出：

这种线程池内部没有核心线程，**线程的数量是有没限制的**。

在创建任务时，若有空闲的线程时则复用空闲的线程，若没有则新建线程。

没有工作的线程（闲置状态）在超过了60S还不做事，就会销毁。

**适用**：执行很多短期异步的小程序或者负载较轻的服务器。

#### FixedThreadPool 定长线程池

```java
  public static ExecutorService newFixedThreadPool(int nThreads) {
    	return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
    }
```
根据源码可以看出：

该线程池的最大线程数等于核心线程数，所以在默认情况下，该线程池的线程不会因为闲置状态超时而被销毁。

如果当前线程数小于核心线程数，并且也有闲置线程的时候提交了任务，这时也不会去复用之前的闲置线程，会创建新的线程去执行任务。如果当前执行任务数大于了核心线程数，大于的部分就会进入队列等待。等着有闲置的线程来执行这个任务。

**适用**：执行长期的任务，性能好很多。

#### SingleThreadPool

```java
    public static ExecutorService newSingleThreadExecutor() {
      return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
    }
```

根据源码可以看出：

有且仅有一个工作线程执行任务，所有任务按照指定顺序执行，即遵循FIFO规则。

**适用**：一个任务一个任务执行的场景。

#### ScheduledThreadPool

```java
    public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    	return new ScheduledThreadPoolExecutor(corePoolSize);
    }

    //ScheduledThreadPoolExecutor():
    public ScheduledThreadPoolExecutor(int corePoolSize) {
       super(corePoolSize, Integer.MAX_VALUE,
          DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
          new DelayedWorkQueue());
    }
```

根据源码可以看出：

`DEFAULT_KEEPALIVE_MILLIS`就是默认`10L`，这里就是10秒。这个线程池有点像是CachedThreadPool和FixedThreadPool 结合了一下。

不仅设置了核心线程数，最大线程数也是`Integer.MAX_VALUE`。

这个线程池是上述4个中**唯一一个有延迟执行和周期执行任务**的线程池。

**适用**：周期性执行任务的场景（定期的同步数据）

**总结**：除了new ScheduledThreadPool 的内部实现特殊一点之外，其它线程池内部都是基于ThreadPoolExecutor类实现的。

 

### 在ThreadPoolExecutor类中有几个非常重要的方法：

`execute()`方法实际上是Executor中声明的方法，在ThreadPoolExecutor进行了具体的实现，这个方法是ThreadPoolExecutor的核心方法，通过这个方法可以向线程池提交一个任务，交由线程池去执行。

submit()，这个方法也是用来向线程池提交任务的，实际上它还是调用的execute()方法，只不过它利用了**Future来获取任务执行结果**。

shutdown()**不会立即终止线程池**，而是要等所有任务缓存队列中的任务都执行完后才终止，但再也不会接受新的任务。

shutdownNow(）立即终止线程池，并尝试打断正在执行的任务，并且清空任务缓存队列，返回尚未执行的任务。

### 线程池中的最大线程数

一般说来，线程池的大小经验值应该这样设置：（其中N为CPU的个数）

如果是CPU密集型应用（CPU使用频率高，不适合频繁切换），则线程池大小设置为N+1
如果是IO密集型应用，则线程池大小设置为2N+1

## ThreadLocal

![](https://img-blog.csdnimg.cn/20190301233250729.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)

一句话：每个线程内部都有一个`ThreadLocalMap`，map中key为ThreadLocal自己（this），value为任意对象。对于不同的线程，threadLocal都是一样的，但value都是不同的。因此可以适合用来做**线程数据隔离**。

### 原理

ThreadLocal类提供的几个方法：

`get()` 方法

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null)
            return (T)e.value;
    }
    return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

set()方法

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}


void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
remove()方法 (略）

初始容量16，负载因子2/3，解决冲突的方法是再hash法，也就是：在当前hash的基础上再自增一个常量进行哈希。

###应用场景

最常见的ThreadLocal使用场景为 用来解决数据库连接、Session管理等。

```java
private static final ThreadLocal threadLocal = new ThreadLocal();

public static Session getSession() throws InfrastructureException {
    Session s = (Session) threadLocal.get();
    try {
        if (s == null) {
            s = getSessionFactory().openSession();
            threadLocal.set(s);
        }
    } catch (HibernateException ex) {
        throw new InfrastructureException(ex);
    }
    return s;
```

## 并发容器

**注意**：以下队列的共性：

1.**put()和take()是会阻塞的，而offer()和poll()是不会阻塞的**。

2.并发都是通过**ReentrantLock**实现的。 

3.**阻塞是通过两个condition实现的**（notEmpty和notFull）。

### 阻塞队列：BlockingQueue

这里blocking的具体含义：

当队列中**没有数据**的情况下，消费者端会被自动阻塞（挂起），直到有数据放入队列。

当队列中**填满数据**的情况下，生产者端的所有线程都会被自动阻塞（挂起），直到队列中有空的位置，线程被自动唤醒。

### ArrayBlockingQueue

读和写共用同一个ReentrantLock，也就意味着（读读、写写、读写都互斥）。

并发是由ReentrantLock来实现，而阻塞是有lock的两个Condition来实现。

```java
//仍然是有数组实现
final Object[] items;

//并发是由ReentrantLock来控制
final ReentrantLock lock;
//阻塞是有两个Condition来实现
private final Condition notEmpty;
private final Condition notFull;
//put方法会先加锁，操作完再解锁，如果阻塞进入await()
public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            //enqueue里有signal操作
            enqueue(e);
        } finally {
            lock.unlock();
        }
}

public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            //dequeue里有signal操作
            return dequeue();
        } finally {
            lock.unlock();
        }
    }

```

错误:在线程1读空队列阻塞，然后牢牢占据着锁不放 -> 线程2尝试写到该队列会发生死锁。
原因在于如果进入阻塞（await）会先把锁释放！并不会占据着锁不放！

### LinkedBlockingQueue

**生产者端和消费者端**分别采用了**独立的锁**来控制数据同步，也就意味着**读和写之间是不互斥的**！（注意这里和读写锁ReentrantReadWriteLock的区别）。

同样，阻塞是通过两个Condition实现的。

```java
//内部使用带next的Node来实现linked结构
    static class Node<E> {
        E item;

        Node<E> next;

        Node(E x) { item = x; }
    }


//读锁
private final ReentrantLock takeLock = new ReentrantLock();
//读阻塞
private final Condition notEmpty = takeLock.newCondition();

// 写锁
private final ReentrantLock putLock = new ReentrantLock();
//写阻塞
private final Condition notFull = putLock.newCondition();
```

### PriorityBlockingQueue

只有一个锁，内部控制线程同步的锁采用的是公平锁。`queue`中存储的类需要实现`compare`方法。

### LinkedBlockingDeque

双向队列，只有一个锁，两个`condition`。

### DelayQueue:延时获取

除此之外还有三种队列。

### ConcurrentHashMap

#### CocurrentHashMap（JDK 1.7）：

- CocurrentHashMap 是由 Segment 数组和 HashEntry 数组和链表组成。
- Segment：一个ReentrantLock,也就是说segment扮演者锁的角色。
核心数据如 value，以及链表都是 volatile 修饰的，保证了获取时的可见性
虽然 HashEntry 中的 value 是用 volatile 关键词修饰的，但是并不能保证并发的原子性，所以 put 操作时仍然需要加锁处理
- ConcurrentHashMap会对数据hash后 对摘要值进行二次hash，其目的是减少hash冲突，使元素均匀分布。

#### ConcurrentHashMap（JDK 1.8） （读不加锁、写加锁）

- JDK1.8 的实现已经摒弃了Segment的概念，恢复成HashMap的结构（bucket数组+链表+红黑树），并发控制使用 Synchronized 和 CAS 来操作，整个看起来就像是优化过且线程安全的HashMap.
- JDK1.7版本锁的粒度是基于Segment的，包含多个HashEntry，而JDK1.8锁的粒度就是HashEntry。
- JDK1.8使用内置锁synchronized来代替重入锁ReentrantLock
因为JDK1.8中，hashMap引入了红黑树，所以cocurrentHashMap也同样使用了红黑树。
- CocurrentHashMap（1.8）中get操作为什么不加锁？
get操作可以无锁是由于Node的元素val和指针next是用volatile修饰的，这样可保证多线程对node的可见性。
- CocurrentHashMap（1.8）中put操作为什么加锁？
正因为node中元素是用volatile修饰，并不能保证原子性，所以写操作需要加锁来保证原子性。
- 为什么使用ConcurrentHashMap而不使用hashtable？
它们都可以用于多线程的环境，但是当Hashtable的大小增加到一定的时候，性能会急剧下降，因为迭代时需要被锁定很长的时间。而ConcurrentHashMap的锁粒度是HashEntry，即便数组长度很长，也能保持比较好的性能。

源码解析：
- table初始化：
    table初始化操作会延缓到第一次put行为。但是put是可以并发执行的，所以初始化还是要考虑并发。
    sizeCtl默认为0，非默认sizeCtl会是一个2的幂次方的值（详见hashMap）。
    具体并发控制操作请看代码注释：

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
		//如果一个线程发现sizeCtl<0，意味着另外的线程执行CAS操作成功，当前线程只需要让出cpu时间片
        if ((sc = sizeCtl) < 0)
            Thread.yield();
        //尝试用CAS将sizeCtl设置成-1，成功的话说明由该线程来进行初始化
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
        //设置成功，进行初始化
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

- put操作：
    put操作采用CAS+synchronize来控制
    其中table中某一格为空，也就是第一次插入时采用CAS，而之后链表或者红黑树的插入就采用synchronize

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
	//关键在这一块！CAS操作的条件！
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        //下面使用synchronize来插入链表或红黑树（代码略）
    }
    addCount(1L, binCount);
    return null;
}
```

#### ConcurrentLinkedQueue：（了解）

是一个单向队列，队列由Node组成

![](https://img-blog.csdnimg.cn/20190311140849657.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2JpbnRvWXU=,size_16,color_FFFFFF,t_70)

并发的确保：head和tail设置为volatile，Node中的item和next也设置为volatile。

```cpp
private transient volatile Node<E> head;
private transient volatile Node<E> tail;

private static class Node<E> {
    volatile E item;
    volatile Node<E> next;
    ...
```

Node中操作节点数据的API，都是通过Unsafe机制的CAS函数实现的；例如casNext()是通过CAS函数“比较并设置节点的下一个节点”

## 补充题目

### 你需要实现一个高效的缓存，它允许多个用户读，但只允许一个用户写。

关键：使用ReentrantReadWriteLock

```java
class MyData{
	//数据
	private static String data = "0";
	//读写锁
	private static ReadWriteLock rw = new ReentrantReadWriteLock();
	//读数据
	public static void read(){
		rw.readLock().lock();
		System.out.println(Thread.currentThread()+"读取一次数据："+data+"时间："+new Date());
		try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			rw.readLock().unlock();
		}
	}
	//写数据
	public static void write(String data){
		rw.writeLock().lock();
		System.out.println(Thread.currentThread()+"对数据进行修改一次："+data+"时间："+new Date());
		try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		} finally {
			rw.writeLock().unlock();
		}
	}
}
```
### 你将如何使用thread dump？你将如何分析Thread dump？

kill -3 <pid>
说明： pid： Java 应用的进程 id ,也就是需要抓取 dump 文件的应用进程 id 。

当使用 kill -3 生成 dump 文件时，dump 文件会被输出到标准错误流。假如你的应用运行在 tomcat 上，dump 内容将被发送到/logs/catalina.out 文件里。

dump文件示例：
```
"pool-1-thread-13" prio=6 tid=0x000000000729a000 nid=0x2fb4 runnable [0x0000000007f0f000]
java.lang.Thread.State: RUNNABLE
at java.net.SocketInputStream.socketRead0(Native Method)
at java.net.SocketInputStream.read(SocketInputStream.java:129)
at sun.nio.cs.StreamDecoder.readBytes(StreamDecoder.java:264)
at sun.nio.cs.StreamDecoder.implRead(StreamDecoder.java:306)
at sun.nio.cs.StreamDecoder.read(StreamDecoder.java:158)
- locked <0x0000000780b7e688> (a java.io.InputStreamReader)
at java.io.InputStreamReader.read(InputStreamReader.java:167)
at java.io.BufferedReader.fill(BufferedReader.java:136)
at java.io.BufferedReader.readLine(BufferedReader.java:299)
- locked <0x0000000780b7e688> (a java.io.InputStreamReader)
at java.io.BufferedReader.readLine(BufferedReader.java:362)
 * 线程名称：pool-1-thread-13 

* jvm线程id：tid=0x000000000729a000

* 线程状态：runnable

* 起始栈地址：[0x0000000007f0f000]

```

### 用java写一个死锁

```java
public static void main(String[] args)
{
    Object lock1 = new Object();
    Object lock2 = new Object();
    ExecutorService exec = Executors.newCachedThreadPool();
    exec.execute(() ->{
        synchronized(lock1)
        {
            System.out.println("get lock1,want lock2...");

            Thread.sleep(1000);	//try-catch忽略

            synchronized (lock2)
            {
                System.out.println("get lock2");
            }
        }
    }
    );

    exec.execute(()->{
        synchronized (lock2)
        {
            System.out.println("get lock2,want lock1....");

            Thread.sleep(1000);	//try-catch忽略

            synchronized (lock1)
            {
                System.out.println("get lock1");
            }
        }
    });
}
```

### 使用notFull和notEmpty来实现生产者/消费者模型

```java
public class Depot
{
	ReentrantLock lock = new ReentrantLock();
	Condition notFull = lock.newCondition();
	Condition notEmpty = lock.newCondition();

	LinkedList<Integer> queue;
	int limit;

	public Depot(int limit)
	{
		queue = new LinkedList<>();
		this.limit = limit;
	}

	public void produce(int i)
	{
		lock.lock();
		try
		{
//			System.out.println("生产" + i);

			//满了阻塞
			if(queue.size() == limit)
//			{
//				System.out.println("队列已满，进入阻塞");
				notFull.await();
//				System.out.println(i+ "已被唤醒");

//			}

			queue.offer(i);
			notEmpty.signal();
		}catch(Exception e)
		{
			e.printStackTrace();
		}
		finally
		{
			lock.unlock();
		}
	}

	public int consume()
	{
		int num = -1;
		lock.lock();
		try
		{
			//空了阻塞
			if(queue.size() == 0)
//			{
//				System.out.println("队列已空，进入阻塞");
				notEmpty.await();
//				System.out.println("已被唤醒");
//			}

			num = queue.poll();
//			System.out.println("消费："+num);
			notFull.signal();
		}catch(Exception e)
		{
			e.printStackTrace();
		}
		finally
		{
			lock.unlock();
		}
		return num;
	}
}
public static void main(String[] args)
	{
		ExecutorService exec = Executors.newCachedThreadPool();
		Depot depot = new Depot(5);
		for(int i = 0; i < 3; i++)
		{
			final int num = i;
			exec.execute(()->
			{

				depot.produce(num);
			});
		}

		try
		{
			Thread.sleep(200);
		}
		catch (InterruptedException e)
		{
			// TODO Auto-generated catch block
			e.printStackTrace();
		}

		for(int i = 0; i < 7; i++)
		{
			final int num = i;
			exec.execute(()->
			{
				depot.consume();
			});
		}

		try
		{
			Thread.sleep(200);
		}catch (InterruptedException e)
		{
			// TODO Auto-generated catch block
			e.printStackTrace();
		}

		for(int i = 0; i < 3; i++)
		{
			final int num = i;
			exec.execute(()->
			{

				depot.produce(num);
			});
		}
	}
```

### 3个任务，返回结果，如果超时200ms，返回空：

```java
//调用三个任务，要求200ms内返回结果或者空
public class 两百秒内返回结果或者空
{
	public static void main(String[] args)
	{
		ExecutorService exec = Executors.newCachedThreadPool();

		Future<Integer> future1 = exec.submit(new MyTask());

		Integer i1 = future1.get(200,TimeUnit.MILLISECONDS);

		if(i1 == null)
			System.out.println("null");
		else
			System.out.println(i1);
	}

}


class MyTask implements Callable<Integer>
{
	@Override
	public Integer call() throws Exception
	{
		int sum = 0;
		for(int i = 0;i < 100; i++)
		{
			sum+=i;
		}
		Thread.sleep(300);
		return sum;
	}
}
```