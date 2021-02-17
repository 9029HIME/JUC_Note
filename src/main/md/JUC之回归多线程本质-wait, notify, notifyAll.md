# 方法简介

​	wait()、notify()、notifyAll()的调度要基于监视器锁，即这三个方法必须在synchronized代码块内使用（The current thread must own this object's monitor）。

​	wait()方法的实际作用是：阻塞当前线程，让出监视器锁，此时当前线程不会再次参与监视器锁的竞争。wait()方法还能设置超时时间，超时后接触阻塞，重新争夺监视器锁。

​	notify()、notifyAll()方法的实际作用是：唤醒**其中一个**/**所有**调用了wait()方法阻塞的线程，让他们重新争夺锁。

# 源码分析

## wait()

```java
public final void wait(long timeout, int nanos) throws InterruptedException {
    if (timeout < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (nanos < 0 || nanos > 999999) {
        throw new IllegalArgumentException(
                            "nanosecond timeout value out of range");
    }

    if (nanos > 0) {
        timeout++;
    }

    wait(timeout);
}
```

```java
public final void wait() throws InterruptedException {
    wait(0);
}
```

```java
public final native void wait(long timeout) throws InterruptedException;
```

​	可以看最后又回到了JNI，这里直接看注释了解过程

```java
/**
 * Causes the current thread to wait until either another thread invokes the
 * {@link java.lang.Object#notify()} method or the
 * {@link java.lang.Object#notifyAll()} method for this object, or a
 * specified amount of time has elapsed.
 * <p>
 * The current thread must own this object's monitor.
 * <p>
 * This method causes the current thread (call it <var>T</var>) to
 * place itself in the wait set for this object and then to relinquish
 * any and all synchronization claims on this object. Thread <var>T</var>
 * becomes disabled for thread scheduling purposes and lies dormant
 * until one of four things happens:
 * <ul>
 * <li>Some other thread invokes the {@code notify} method for this
 * object and thread <var>T</var> happens to be arbitrarily chosen as
 * the thread to be awakened.
 * <li>Some other thread invokes the {@code notifyAll} method for this
 * object.
 * <li>Some other thread {@linkplain Thread#interrupt() interrupts}
 * thread <var>T</var>.
 * <li>The specified amount of real time has elapsed, more or less.  If
 * {@code timeout} is zero, however, then real time is not taken into
 * consideration and the thread simply waits until notified.
 * </ul>
 * The thread <var>T</var> is then removed from the wait set for this
 * object and re-enabled for thread scheduling. It then competes in the
 * usual manner with other threads for the right to synchronize on the
 * object; once it has gained control of the object, all its
 * synchronization claims on the object are restored to the status quo
 * ante - that is, to the situation as of the time that the {@code wait}
 * method was invoked. Thread <var>T</var> then returns from the
 * invocation of the {@code wait} method. Thus, on return from the
 * {@code wait} method, the synchronization state of the object and of
 * thread {@code T} is exactly as it was when the {@code wait} method
 * was invoked.
 * <p>
 * A thread can also wake up without being notified, interrupted, or
 * timing out, a so-called <i>spurious wakeup</i>.  While this will rarely
 * occur in practice, applications must guard against it by testing for
 * the condition that should have caused the thread to be awakened, and
 * continuing to wait if the condition is not satisfied.  In other words,
 * waits should always occur in loops, like this one:
 * <pre>
 *     synchronized (obj) {
 *         while (&lt;condition does not hold&gt;)
 *             obj.wait(timeout);
 *         ... // Perform action appropriate to condition
 *     }
 * </pre>
 * (For more information on this topic, see Section 3.2.3 in Doug Lea's
 * "Concurrent Programming in Java (Second Edition)" (Addison-Wesley,
 * 2000), or Item 50 in Joshua Bloch's "Effective Java Programming
 * Language Guide" (Addison-Wesley, 2001).
 *
 * <p>If the current thread is {@linkplain java.lang.Thread#interrupt()
 * interrupted} by any thread before or while it is waiting, then an
 * {@code InterruptedException} is thrown.  This exception is not
 * thrown until the lock status of this object has been restored as
 * described above.
 *
 * <p>
 * Note that the {@code wait} method, as it places the current thread
 * into the wait set for this object, unlocks only this object; any
 * other objects on which the current thread may be synchronized remain
 * locked while the thread waits.
 * <p>
 * This method should only be called by a thread that is the owner
 * of this object's monitor. See the {@code notify} method for a
 * description of the ways in which a thread can become the owner of
 * a monitor.
 *
 * @param      timeout   the maximum time to wait in milliseconds.
 * @throws  IllegalArgumentException      if the value of timeout is
 *               negative.
 * @throws  IllegalMonitorStateException  if the current thread is not
 *               the owner of the object's monitor.
 * @throws  InterruptedException if any thread interrupted the
 *             current thread before or while the current thread
 *             was waiting for a notification.  The <i>interrupted
 *             status</i> of the current thread is cleared when
 *             this exception is thrown.
 * @see        java.lang.Object#notify()
 * @see        java.lang.Object#notifyAll()
 */
```

​	大致意思是，当线程A在监视器锁L的同步代码块里调用了wait()方法后

​	1.线程A先进入L的**等待队列wait set**里

​	2.让出L的所有权，让其他线程去争夺

​	3.挂起自己

​	当以下事件触发后，线程A解除挂起，**从L的等待队列wait set里移除**，A重新去争夺L

​	1.其他线程调用了notify()，**且notify()唤醒的线程刚好是自己**

​	2.其他线程调用了notifyAll()

​	3.其他线程interrupt()了我

​	4.wait()的超时时间到了，如果超时时间为0，则永久挂起

​	**如果线程再调用了wait()后被其他线程interrupt()了，会抛出InterruptedException异常**，同时线程调用wait()后只会释放当前同步代码块的监视器锁，并不会释放其他同步资源。

### 假唤醒

​	一般出现在if代码块里调用wait()，假设一个场景：有2个生产者与2个消费者，生产者用于生成资料，消费者用于消费资料，当资料总数<=0时，消费者调用wait()挂起自己，把锁让给生产者，等生产者生产完资料后，调用notifyAll()将所有消费者唤醒，继续消费。以下是demo

```java
public class random {
    /**
     * 资料个数，不用volatile，毕竟synchronized自动刷值
     */
    private int count  = 0;


    public synchronized void set() throws InterruptedException {
        if (count >= 5){
            wait();
        }
        count+=1;
        System.out.println("当前生产者已生产1个，剩余个数"+count);
        notifyAll();
    }

    public synchronized void get() throws InterruptedException {
        if (count <= 0){
            wait();
        }
        count-=1;
        System.out.println("当前消费者已消费1个，剩余个数"+count);
        notifyAll();
    }


    public static void main(String[] args) throws InterruptedException {
        random r = new random();
        /*
         *  这里是不合规范,不应该用new Thrad()，不过demo就不在意那么多了 
         */
        new Thread(()->{		// t1
            try {
                r.get();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();


        new Thread(()->{		// t2
            try {
                r.get();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();


        new Thread(()->{		//t3
            try {
                r.set();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();


        new Thread(()->{		//t4
            try {
                r.set();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```

​	此时输出的结果是：

```
当前生产者已生产1个，剩余个数1
当前消费者已消费1个，剩余个数0
当前消费者已消费1个，剩余个数-1
当前生产者已生产1个，剩余个数0

Process finished with exit code 0
```

​	为什么会有-1的情况？现在解释一下消费者两个线程的调度过程

1. t1准备消费，进入if判断，发现初始count为0，于是都调用wait()，让出对象锁

2. t2准备消费，进入if判断，发现初始count为0，于是都调用wait()，让出对象锁

3. t3准备生产，进入if判断，发现初始count<5，于是生产，生产完毕后调用notifyAll()唤醒了t1,t2

4. 此时t1与t2被唤醒了，假设t1抢到了锁，消费了一个资料，此时资料数为0

5. 重点来了！这时候t2还有竞争锁资源的权力，当t2抢到锁后在wait()方法处唤醒，**由于是if判断，t2还是会继续消费，此时资料数变为-1了**

```java
public synchronized void get() throws InterruptedException {
    if (count <= 0){
        wait();	// 1.此时资料数已为0，t2在这里唤醒
    }
    // 2.t2继续消费资料，将0改为-1
    count-=1;
    System.out.println("当前消费者已消费1个，剩余个数"+count);
    notifyAll();
}
```

​	可以看到，假唤醒的前提是，消费者被同时唤醒后依次拿到锁，中途没有生产者拿到锁，如果用if仅是有发生假唤醒的可能。为了避免假唤醒，需要将wait()放到while代码块了，而不是if。

```java
public synchronized void get() throws InterruptedException {
    // 2.再次进入循环，发现我醒了，但资料数还是为0（我总觉得这才是真正的假唤醒）
    while (count <= 0){
        // 3.于是再次进入睡眠
        wait();	// 1.此时资料已为0，t2在这里唤醒
    }
    count-=1;
    System.out.println("当前消费者已消费1个，剩余个数"+count);
    notifyAll();
}
```

## notify()、notifyAll()

​	这两个也是JNI，所以直接看注释

```java
/**
 * Wakes up a single thread that is waiting on this object's
 * monitor. If any threads are waiting on this object, one of them
 * is chosen to be awakened. The choice is arbitrary and occurs at
 * the discretion of the implementation. A thread waits on an object's
 * monitor by calling one of the {@code wait} methods.
 * <p>
 * The awakened thread will not be able to proceed until the current
 * thread relinquishes the lock on this object. The awakened thread will
 * compete in the usual manner with any other threads that might be
 * actively competing to synchronize on this object; for example, the
 * awakened thread enjoys no reliable privilege or disadvantage in being
 * the next thread to lock this object.
 * <p>
 * This method should only be called by a thread that is the owner
 * of this object's monitor. A thread becomes the owner of the
 * object's monitor in one of three ways:
 * <ul>
 * <li>By executing a synchronized instance method of that objsect.
 * <li>By executing the body of a {@code synchronized} statement
 *     that synchronizes on the object.
 * <li>For objects of type {@code Class,} by executing a
 *     synchronized static method of that class.
 * </ul>
 * <p>
 * Only one thread at a time can own an object's monitor.
 *
 * @throws  IllegalMonitorStateException  if the current thread is not
 *               the owner of this object's monitor.
 * @see        java.lang.Object#notifyAll()
 * @see        java.lang.Object#wait()
 */
```

​	大致意思是，当线程A在监视器锁L的同步代码块里调用了notify()方法后

​	1.唤醒L的等待队列wait set里**其中一个线程B**

​	2.线程B被唤醒后，仅拥有竞争锁的权力，但并未获取到锁，还需等A释放锁后自己去竞争

​	3.线程B的竞争力和其他竞争线程一致

​	而notifyAll()与notify()的区别是：notifyAll()在1.唤醒的是等待队列里所有线程