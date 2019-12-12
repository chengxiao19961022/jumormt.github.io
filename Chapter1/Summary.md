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



