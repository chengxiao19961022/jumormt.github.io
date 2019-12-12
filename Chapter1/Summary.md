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

Synchronized VS CAS

元老级的Synchronized(未优化前)最主要的问题是：在存在线程竞争的情况下会出现线程阻塞和唤醒锁带来的性能问题(**进入了内核态**)，因为这是一种互斥同步（阻塞同步）。而CAS并不是武断的间线程挂起，当CAS操作失败后会进行一定的尝试，而非进行耗时的挂起唤醒的操作，因此也叫做非阻塞同步。这是两者主要的区别。

#### CAS的问题

1. ABA问题 因为CAS会检查旧值有没有变化，这里存在这样一个有意思的问题。比如一个旧值A变为了成B，然后再变成A，刚好在做CAS时检查发现旧值并没有变化依然为A，但是实际上的确发生了变化。解决方案可以沿袭数据库中常用的乐观锁方式，添加一个版本号可以解决。原来的变化路径A->B->A就变成了1A->2B->3C。java这么优秀的语言，当然在java 1.5后的atomic包中提供了AtomicStampedReference来解决ABA问题，解决思路就是这样的。
2. 自旋时间过长
使用CAS时非阻塞同步，也就是说不会将线程挂起，会自旋（无非就是一个死循环）进行下一次尝试，如果这里自旋时间过长对性能是很大的消耗。如果JVM能支持处理器提供的pause指令，那么在效率上会有一定的提升。
3. 只能保证一个共享变量的原子操作
当对一个共享变量执行操作时CAS能保证其原子性，如果对多个共享变量进行操作,CAS就不能保证其原子性。有一个解决方案是利用对象整合多个共享变量，即一个类中的成员变量就是这几个共享变量。然后将这个对象做CAS操作就可以保证其原子性。atomic中提供了AtomicReference来保证引用对象之间的原子性。
