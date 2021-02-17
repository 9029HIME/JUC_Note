# synchronized简介

​	直奔主题，synchronized是一个悲观锁，回忆一下乐观锁和悲观锁的优缺点。

​	乐观锁：优点是内存占用小，在竞争较少的场景下性能较好，缺点是竞争多的话，会有很多线程空跑无意义的自旋，导致CPU利用率低。

​	悲观锁：优点是CPU利用率高，当线程发现锁被占用后会立即释放自己的CPU时间片给其他线程，然后挂起自己。但是挂起与唤醒本质需要操作系统来完成用户态与内核态的转换（Java的用户态线程与内核态线程是1:1），是一种耗费性能的做法，缺点是竞争少的情况下悲观锁性能较低，只有在竞争多的情况下才能发挥优势。

# synchronized实现

​	synchronized代码块是搭配锁对象来使用的，这个锁对象本质是一个普通java对象，如果是synchronized对象方法，锁对象为this，如果是synchronized静态方法，锁对象为class。参考CAS，多个线程拿不到锁的依据是发现一个state属性不符合自己的期望。**那么锁对象是否也有类似state的属性或者标识，用来阻隔其他线程呢？**

​	有，这玩意就叫对象头，他存储了对象的元数据，包括但不限于锁信息、hashcode，其中对象头描述锁信息的部分被称为mark word（mark word还描述了hashCode、分代年龄、GC标志）。**mark word存储的锁信息**本质是一个指针，他指向一个monitor对象（被称为管程或监视器锁），大致关系如下

​	thread → 锁对象对象头(里的mark word部分) → monitor对象

​	**一个或多个线程**的调用栈指向了**堆中的锁对象**，锁对象的mark word包含一个指针，这个指针指向了Monitor对象，即监视器锁。监视器锁本质是一个规范，需要JVM的实现，在HotSpot中他的实现源码是

```c++
ObjectMonitor() {
 _header = NULL; 
 _count = 0; 
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

