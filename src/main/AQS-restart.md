以下内容是时隔1年半，再次阅读AQS源码的笔记

1. 非公平锁很不老实啊，在acquire(1)之前就直接插队用cas更改state了，而且就算失败了，在acquire(1)里（nonfairTryAcquire）也不判断前面是否有人排队，又插一次队。单纯的拿锁就插了2次队。

2. 公平也好不公平也好，尝试拿锁前都是判断state的状态。state=0了，代表锁被解开了，此时就是判断是否有人排队（公平），然后和其他线程共同去cas上锁，返回上锁结果。
   如果state不等于0，说明锁已经被占用了，但是总得看看是不是自己占用吧，是的话就再充入一遍，state+1，不是就说明别人还占用着呢，告诉上层方法自己拿锁失败，要去排队了。

3. 在tryAcquire拿锁失败后，公平也好不公平也好都一视同仁了，全都包装成Node往队尾里插。但是插入队列前要想想，有没有可能队列还没生成？因为只有1个线程持锁的情况下是不会生成队列的。所以当队列没生成时，入队代码里会通过enq()创建队列。
   即使队列已经成圣了，可是很多个节点同时插入会不会有问题呢？毕竟tail指针只有1个，但是待插入的节点有很多个。AQS的做法是：
   1.直接将节点的prev指向tail
   2.用CAS(prev,自己)将tail改为自己
   但是那么多线程，只会有1个线程在2.成功，也就是说其他失败的线程都在1.将prev指向成功线程的prev了：

   ​            node-failed2
   ​               ↓
   head ↔ node1（原tail） ↔ node（成功）
   ​               ↑             
   ​             node-failed1

   这就是AQS里的“尾分叉”现象，别急，对于第1次入队失败的线程，会走到enq()里，也就是说enq()有2种作用：创建队列，或整理尾分叉。

4. enq这里，循环利用得很巧妙，先看一下enq前空队列的情况：

   1. 进入t == null代码块，通过cas给head设置一个哑节点，如果成功了，将tail也指向哑节点。走了这一步后，队列必定会被创建好。此时第一次循环结束。
   2. 开始第二次循环，此时只会走到else分支，else分支代表已经有队列的情况。这里的做法和addWaiter插入节点一样，也会出现尾分叉的情况。
   3. 最巧妙的来了，出现尾分叉代表当前线程的节点插入队列失败了，此时是不会走到return t;这段代码，for循环仍未结束，这时候就会开启第3次循环。
   4. 结合代码可以看到，第3次循环还是会走2.的步骤，再次CAS插入队尾，失败了再次循环，一直反复下去，直到成功插入到队尾，结束循环。当所有线程结束了enq方法后，AQS就不会再有尾分叉的情况。
   
5. 拿锁失败了，这时候该乖乖挂起等待唤醒了吧？但这一步之前还要做一个操作：自己现在还能不能挂起。首先需要明确一件事：只有前驱节点的waitStatus = SIGNAL，才能挂载它后面，因为自己睡着后是不会自然醒的，需要prev来唤醒自己，而SIGNAL则表示prev可以唤醒自己。在shouldParkAfterFailedAcquire里是怎么做的呢？第一步直接判断prev是否为SIGNAL，是的话直接返回true给上层，告诉上层我可以被挂起了。如果不是的话有两种情况：prev已经取消(1)，prev还在等锁(0)。如果是后者，直接CAS将prev的ws设为SIGNAL。如果prev已取消了，那可不能跟在它后面，毕竟它逻辑上已经不属于队列的一份子了，此时通过while循环挂载到离自己最近的、ws<=0的节点上，到了下一次acquireQueued循环的时候，就会直接将prev的ws设为SIGNAL。此时自己就能够挂起了，调用parkAndCheckInterrupt()，最终线程会停在LockSupport.park(this)这里。

6. head节点的状态：
   0：拿到锁后，没有后继节点
   -1：拿到锁后，有后继节点
   0：释放锁后，将自己的ws改为0
   head是由队列中上一次拿到锁的节点转换而来的（被非公平抢夺后不算），当队列节点拿到锁后，除了将exclusiveOwnerThread设为自己，还要将head里的thread设为null，充当哑节点。

   有以下队列：
   head(-1) ↔ node1(-1) ↔ node2(-1) ↔ node3(<=0)
   此时node1和node2都是挂起状态，node1等待head唤醒

   过了一段时间后，node1取消拿锁，状态变为cancel
   head(-1) ↔ node1(1) ↔ node2(-1) ↔ node3(<=0)

   此时持锁线程释放锁，将head的ws改为0，发现head的next是cancel状态的，于是从tail(node3)开始向前遍历，找到距离最近的、ws<=0的节点，即node2，然后唤醒它：

   head(0) ↔ node1(1) ↔ node2(我醒了！！) ↔ node3(<=0)

   此时node2就从parkAndCheckInterrupt()方法里醒来了，继续参与acquireQueued()的循环。
   但是！！！注意看此时的链表，node2仍挂载在node1后面，node1可不是head，所以node2就算醒来了，在acquireQueued里还是不会命中p == head这个条件，所以还是会走进shouldParkAfterFailedAcquire()

   可是node2明明被唤醒了啊，按道理该去拿锁，别急，都是因为node1取消了，node2才要这么麻烦多走一步，AQS会再判断一下node2是否应该被挂起。在shouldParkAfterFailedAcquire里，会根据node1的ws判断走进ws>0的代码块，这一步就是让node2不停地向前遍历，找到一个ws<=0的head挂载，此时就能找到head了：

     					node1(1) 
      	↙				↓

   head(0)  ↔ node2(我醒了！！) ↔ node3(<=0)

   这样整个链表在逻辑上就没有了node1，而node1也会被GC回收掉。

   好，此时shouldParkAfterFailedAcquire返回，回到acquireQueued的循环语句，这时候node2的prev就必定是head了（毕竟node2是被head选出来，离自己最近的，现在又挂载到自己后面了），这时候就去拿锁，公平锁的情况下必定是成功的，接着就是将自己设为head，设exclusiveOwnerThread，弹出旧head等常规操作。

   此时非公平拿锁呢？队外线程在入队前会拿2次锁，如果队外线程得逞了，head节点依旧不变，只是将exclusiveOwnerThread设为自己而已，那node2发现自己失败了，会又灰溜溜地走进shouldParkAfterFailedAcquire里2次，第1次将head的ws设为-1，第2次直接返回true，然后挂载自己，等待这个队外线程唤醒自己，唤醒后的node2流程也是一样的。