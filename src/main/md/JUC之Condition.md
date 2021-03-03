# 回顾一下wait和notify的弊端

​	回想一下，wait有一个很严重的问题："假唤醒"（具体看JUC之回归多线程本质-wait,notify,notifyAll），虽然可以通过while()循环避免假唤醒引发的业务问题，但从性能方向考虑，假唤醒会使得原本没资格唤醒的线程被重新调度，然后再次进入阻塞状态。这个“假唤醒重新调度” → “重新阻塞”的过程导致了**很多无意义的时间与CPU资源的浪费**。问题是出在哪？出在notify本质上是唤醒waitset里其中一个线程，notifyAll唤醒waitset里所有线程，**根本没有判断哪些线程应该要唤醒，哪些线程应该要继续保持阻塞。**

​	有一个解决这个弊端的方式：在阻塞线程时，应设置一个类似标识的东西，当唤醒线程时，应通过该标识精准唤醒。而Condition实现了这个思路。**但是！！！Condition是基于Lock接口使用的，而非synchronized。**



# Condition机制

## await

​	功能类似Object::wait()，也是挂起一个线程并释放锁，

| Object                             | Condition                               | 区别                                                         |
| ---------------------------------- | --------------------------------------- | :----------------------------------------------------------- |
| void wait()                        | void await()                            | 一个是基于synchonized，一个是基于Lock                        |
| void wait(long timeout)            | long awaitNanos(long nanosTimeout)      | 时间单位，返回值                                             |
| void wait(long timeout, int nanos) | boolean await(long time, TimeUnit unit) | 时间单位，参数类型，返回值                                   |
| -                                  | void awaitUninterruptibly()             | await后发生中断不会响应，只会标记中断位                      |
| -                                  | boolean awaitUntil(Date deadline)       | 和boolean await(long time, TimeUnit unit)相比，用的是绝对时间 |

​	与Object::wait()相似的是，wait()的调用是基于线程获取锁对象，而Condition::await()的调用是基于线程调用了Lock::lock()。调用wait()后会使线程释放对象锁，并加入线程到该对象锁的wait_set里，而await()会释放该线程持有的lock锁（相当于Lock::unlock()），并将线程加入到**Condition对象对应的等待队列**里。

## signal&&signalAll

​	功能类似Object::notify()与Object::notifyAll()，唤醒一个/所有线程去竞争锁

| Object           | Condition        | 区别                                  |
| ---------------- | ---------------- | ------------------------------------- |
| void notify()    | void signal()    | 一个是基于synchonized，一个是基于Lock |
| void notifyAll() | void signalAll() | 一个是基于synchonized，一个是基于Lock |

​	与Object::notify()相似的是，signal的调用会唤醒对应Condtion的等待队列的head节点里的线程，并让他重新参与锁竞争。



# Condition Demo

```java
public class random {
    private Lock lock = new ReentrantLock();
    private Condition con = lock.newCondition();
    public static void main(String[] args) {
        random r= new random();
        new Thread(()->{
            r.put();
        }).start();

        new Thread(()->{
            r.get();
        }).start();
    }

    public void put(){
        lock.lock();
        try{
            System.out.println("睡眠5s后，准备调用await");
            Thread.sleep(5000);
            con.await();
            System.out.println("调用await的线程被唤醒了");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void get(){
        System.out.println("准备调用get()方法");
        lock.lock();
        try{
            System.out.println("get()方法调用成功了");
            con.signal();
        }finally {
            lock.unlock();
        }
    }
}
```



# 源码解析

## 前提

​	回忆一下AQS（具体看JUC之AQS），在AQS队列里节点之间通过next与prev指针连接下一个与上一个节点，当AQS被使用在共享锁里时，节点的nextWaiter也只不过是一个标记，用来指向一个空节点。但是在Condition等待队列里**（以下称为CQ）**却是通过nextWaiter来指向下一个等待被唤醒的节点，并不会使用到prev与next指针，也就是说cq和aqs一样，都是基于同一个node节点构成的队列，**不同的是cq是单链表**。

​	每一个Condition都有其独立的CQ，不同CQ间互不影响，**且CQ与AQS之间并没有直接的关联**，Condition::await()，Condition::signal()，Condition::signalAll()只会对本身的CQ进行操作。在CQ的节点中，实际用到的属性有三个：thread，waitStatus，nextWaiter。这里不得不提下waitStatus，类似AQS会关注节点的ws是否为SIGNAL，cq则是关注节点的ws是否为CONDITION，即-2。

​	当调用某个CQ的signal()时，会将CQ里**某个**节点弹出来，此时节点是先抢锁还是乖乖入队就得看是否公平了。需要注意的是，**即使调用了signalAll()也是一个一个弹出节点，而非整个CQ接在AQS后面，毕竟虽然节点是相同的，但连接方式大有区别**。

## 探究CQ源码

​	物理上的CQ实现类为CondtionObject，他是AQS的内部类，有两个核心属性，本质是头节点指针与尾节点指针。不过注意的是，默认构造方法很普通，说明CQ只有在使用时才会初始化队列。

```java
public class ConditionObject implements Condition, java.io.Serializable {
    private static final long serialVersionUID = 1173984872572414699L;
    /** First node of condition queue. */
    private transient Node firstWaiter;
    /** Last node of condition queue. */
    private transient Node lastWaiter;

    /**
     * Creates a new {@code ConditionObject} instance.
     */
    public ConditionObject() { }
}
```

### await

#### 封装

​	await()实际上还是比较复杂的，这里先对每个方法做个介绍。**不过要注意的是！！！与抢锁不同，await()与signal()都是被拿到锁的线程调用的，因此不用考虑并发安全问题。**

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 将当前线程包装为Node
    Node node = addConditionWaiter();
    // 释放该线程的锁，即从AQS里剔除
    int savedState = fullyRelease(node);
    
    int interruptMode = 0;
    // isOnSyncQueue()是用来判断该节点是否在AQS里，如果节点不在AQS里，则说明节点还未被调用signal()
    while (!isOnSyncQueue(node)) {
        // 既然没被singal()，那就乖乖挂起吧
        LockSupport.park(this);
        // 走到这里说明被唤醒了，此时看看唤醒的原因，无非有两个：1.被中断，2.被signal()
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            // 如果被中断，则直接跳出循环，走下面的代码（虽然被signal也会走下面的代码）
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

​	抽丝剥茧，先看下封装节点的过程

```java
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // 对于CQ节点来说，ws不为CONDITION只能是CANCEL，如果发现尾节点是CANCEL，则遍历整个CQ，清除掉所有CANCEL节点
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    
    // 包装线程为节点
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    
    // 常规插入
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

​	可以看到CQ的插入比AQS简单多了，只是一个单纯的单链表插入，且CQ的设计也比AQS简单，没有哑节点。不过插入时会像AQS一样判断节点的ws，只不过CQ判断尾节点的ws，只要尾节点ws为CANCEL则遍历整个CQ，清掉所有CANCEL节点。

```java
// 普通的剔除单链表节点操作而已
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    Node trail = null;
    while (t != null) {
        Node next = t.nextWaiter;
        if (t.waitStatus != Node.CONDITION) {
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
}
```

#### 解锁

​	线程被封装成节点后，得将该线程在AQS的节点里剔除掉，由于调用await的必定是持锁线程，所以剔除的做法是唤醒下一个AQS的节点。

```java
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        // 又是AQS的release()，不过这次是通过savedState释放整个锁，不是只释放1个重入单位
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

