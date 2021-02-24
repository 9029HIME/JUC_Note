# synchronized简介

​	直奔主题，synchronized是一个悲观锁（这一点其实不太准确，毕竟锁会升级 on 2021-02-19），回忆一下乐观锁和悲观锁的优缺点。

​	乐观锁：优点是内存占用小，在竞争较少的场景下性能较好，缺点是竞争多的话，会有很多线程空跑无意义的自旋，导致CPU利用率低。

​	悲观锁：优点是CPU利用率高，当线程发现锁被占用后会立即释放自己的CPU时间片给其他线程，然后挂起自己。但是挂起与唤醒本质需要操作系统来完成用户态与内核态的转换（Java的用户态线程与内核态线程是1:1），是一种耗费性能的做法，缺点是竞争少的情况下悲观锁性能较低，只有在竞争多的情况下才能发挥优势。

# synchronized实现

​	synchronized代码块是搭配锁对象来使用的，这个锁对象本质是一个普通java对象，如果是synchronized对象方法，锁对象为this，如果是synchronized静态方法，锁对象为class。参考CAS，多个线程拿不到锁的依据是发现一个state属性不符合自己的期望。**那么锁对象是否也有类似state的属性或者标识，用来阻隔其他线程呢？**

​	有，这玩意就叫对象头，他存储了对象的元数据，包括但不限于锁信息、hashcode，其中对象头描述锁信息的部分被称为mark word（mark word还描述了hashCode、分代年龄、GC标志）。**mark word存储的锁信息**本质是一个指针，他指向一个monitor对象（被称为管程或监视器锁），大致关系如下，**注意！！！这里的关系是重量级锁！！锁的类型会再下面介绍**

​	thread → 锁对象对象头(里的mark word部分) → monitor对象

![thread-monitor](https://user-images.githubusercontent.com/48977889/108450790-b744b800-72a0-11eb-89cb-1a69d477be8b.png)

![64位虚拟机MarkWord](https://user-images.githubusercontent.com/48977889/108477497-41574580-72ce-11eb-986e-2751c1fc3c0d.png)

​	**一个或多个线程**的调用栈指向了**堆中的锁对象**，锁对象的mark word包含一个指针，这个指针指向了Monitor对象，即监视器锁。监视器锁本质是一个规范，需要JVM的实现，在HotSpot中他的实现源码是

```c++
ObjectMonitor() {
 _header = NULL; 
 _count = 0; // 该线程获取锁的次数
 _waiters = 0, 
 _recursions = 0; 
 _object = NULL; 
 _owner = NULL; // 当前拥有锁的线程
 _WaitSet = NULL; // 前面说的等待队列wait set，调用wait()后就会来这里
 _WaitSetLock = 0 ; 
 _Responsible = NULL ; 
 _succ = NULL ; 
 _cxq = NULL ; 
 FreeNext = NULL ; 
 _EntryList = NULL ; // 竞争该锁失败时，等待的线程队列
 _SpinFreq = 0 ; 
 _SpinClock = 0 ; 
 OwnerIsThread = 0 ; 
 _previous_owner_tid = 0; 
 } 
```

​				![monitor-set](https://user-images.githubusercontent.com/48977889/108450615-7a78c100-72a0-11eb-91f0-b20c77e39f36.jpg)

​	当线程A**竞争(注意已经变为竞争状态了)**synchronized代码块前，会被封装成ObjectWaiter对象，然后被放入该synchronized对应的monitor对象的 "_EntryList"里。在**竞争过程中**如果线程A获取到锁则会变成  _owner，其余未竞争到锁的线程则继续呆在 _EntryList里，如果 _owner再调用wait()方法则会释放锁，然后进入到 _WaitSet里等待被其他线程唤醒，**然后再次进入 _EntryList内**。

​	值得注意的是，竞争过程涉及到锁的转换（偏向锁→轻量级锁→重量级锁）和Monitor原理，TODO 之后再开这个坑



# 锁的转换

​	首先要明白，sychronized底层依赖Monitor，而Monitor底层又依赖OS的Mutex Lock，但是通过Mutex Lock来阻塞、唤醒一个线程涉及到用户态与内核态的转换（要记得Java是1:1），**这是性能较差的做法**。因此后期Java引入了无锁，偏向锁 → 轻量级锁 → 重量级锁的概念，他们是**单向**逐步增强的锁，JVM会根据不同的场景进行动态提升，而非一开始就使用重量级锁。

​	而MarkWord在不同锁的情景下也是记录这不同的东西，具体如下

| 锁状态   | 存储内容                                                    | 锁标志位 |
| :------- | :---------------------------------------------------------- | :------- |
| 无锁     | 对象的hashCode、对象分代年龄、是否是偏向锁（0）             | 01       |
| 偏向锁   | **偏向线程ID**、偏向时间戳、对象分代年龄、是否是偏向锁（1） | 01       |
| 轻量级锁 | 指向栈中锁记录的指针                                        | 00       |
| 重量级锁 | 指向互斥量（重量级锁）的指针                                | 10       |

## 无锁

​	最直接的体现为CAS，并没有一把锁用于线程间的互斥，每一个线程都有资格去操作共享变量A，但只能有一个线程操作成功，其他线程操作失败。这时候其他线程要么放弃，要么循环重新去操作。

## 偏向锁

​	所谓偏向，就是偏向第一个拿到锁的线程，**也就是说偏向锁就是为单独一个线程服务的**。当线程去拿偏向锁时，先判断MarkWord记录的线程id是否为自己，如果是直接获取锁，如果不是则用**一次CAS**将线程ID改为自己，成功则代表获取偏向锁成功，当线程再次获取锁时只需判断owner是否为自己，而不用多余的CAS，**这里就是偏向的体现了**。如果CAS失败了，则代表有其他线程开始竞争锁，此时就该膨胀为轻量级锁了。

​	以下是偏向锁的详细调度过程

![偏向锁转轻量锁](https://user-images.githubusercontent.com/48977889/108476861-7e6f0800-72cd-11eb-851c-3b395c6312c5.png)

## 轻量级锁

​	轻量级锁的特点是：当锁被其他线程占有时，竞争线程会自旋 + CAS重新拿锁，此时竞争线程不会阻塞，