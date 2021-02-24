# AbstractQueuedSynchronizer

​	是JUC中锁（如Lock、ReentrantLock）的核心，本质上属于双向链表，含有头与尾指针，用来存放Node节点，Node内包含了参与竞争锁的线程。AQS的实现主要依赖这三要素：1.双向链表、2.CAS、3.state

​	1.双向链表：代表着AQS底层数据结构，每一个节点由Node构成，Node内包含了prev与next指针，用于指向上一个/下一个Node节点，每一个Node包含thread指针，用于指向对AQS进行操作的线程。双线链表包含了头指针与尾指针，用来指向头节点与尾节点。值得注意的是！头节点不包含任何有效数据，属于一个哑节点。

​	2.CAS：AQS里有多个volatile变量，用来记录头、尾、state、next等信息，由于AQS本身用来实现同步锁机制，因此不会用synchronized来对这些变量进行同步。

​	3.state：AQS里最核心的变量，如果为0代表没有线程持有锁，如果大于1（可重入）则代表已有线程持有锁。



# AQS核心变量

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    
    // （这个变量在父类AbstractOwnableSynchronizer里）代表持有锁的线程
    private transient Thread exclusiveOwnerThread
    
    // 头指针，指向头节点，属于哑节点
    private transient volatile Node head;

    // 尾指针，指向尾节点
    private transient volatile Node tail;

    // 核心state，用来代表是否被占有锁
    private volatile long state;
    
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    // state偏移量
    private static final long stateOffset;
    // 头节点偏移量
    private static final long headOffset;
    // 尾节点偏移量
    private static final long tailOffset;
    
    private static final long waitStatusOffset;
    // 下一个节点的偏移量
    private static final long nextOffset;

    static {
        try {
            // 静态代码块里会直接规定不同对象里这些属性的偏移量是多少（偏移量是个常量值）
            stateOffset = unsafe.objectFieldOffset
                (AbstractQueuedLongSynchronizer.class.getDeclaredField("state"));
            headOffset = unsafe.objectFieldOffset
                (AbstractQueuedLongSynchronizer.class.getDeclaredField("head"));
            tailOffset = unsafe.objectFieldOffset
                (AbstractQueuedLongSynchronizer.class.getDeclaredField("tail"));
            waitStatusOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("waitStatus"));
            nextOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("next"));

        } catch (Exception ex) { throw new Error(ex); }
    }
}
```

## Node核心变量

```java
static final class Node {
    // 是否为互斥锁
    static final Node EXCLUSIVE = null;
	
    // waitStatus的值
    static final int CANCELLED =  1;

    static final int SIGNAL    = -1;

    static final int CONDITION = -2;

    static final int PROPAGATE = -3;
    
	// Node状态，直接影响了下一个节点
    volatile int waitStatus;
	// 上一个节点
    volatile Node prev;
	// 下一个节点
    volatile Node next;
	
    // 当前线程
    volatile Thread thread;
	
    // 在独占模式下为null
    Node nextWaiter;
}
```

# AQS获取锁过程之公平锁

## 拿锁

​	AQS对外暴露了一个tryAcquire()方法，子类通过重写该方法的逻辑用来获取锁，这里只看ReentrantLock的逻辑。ReentrantLock有分公平锁实现与非公平锁实现。

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = 7373984872572414699L;
   	
    abstract static class Sync extends AbstractQueuedSynchronizer {
    	// .......
    }
    
    private final Sync sync;
    
    public void lock() {
        sync.lock();
    }
    
    public ReentrantLock() {
        sync = new NonfairSync();
    }
    
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
}
```

​	可以看到ReentrantLock默认使用了非公平锁NonfairSync，而Sync类又继承了AQS，当ReentrantLock.lock()后，实际

是调用了对应Sync的lock()。看一下默认两种Sync的lock()实现。

```java
static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
}
```

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        acquire(1);
    }
}
```

​	可以看到，公平锁是老老实实用acquire()排队，非公平锁是先插队拿锁，拿不到再排队。在acquire(1)里其实调用了上面说的tryAcquire()，这是整个AQS里最核心的方法。

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

​	tryAcquire(arg)：尝试获取锁。

​	addWaiter(Node.EXCLUSIVE)：声明一个互斥节点，并加入到AQS里并返回。在tryAcquire(arg)获取锁失败（返回false后），会走到这一步。

​	acquireQueued(node,arg)：对addWaiter()里插入的节点做进一步操作，如判断是否可以立刻拿锁（前驱节点是head）以及实际挂起线程。

​	要明白，如果tryAcquire(arg)获取锁失败了， 返回了false，才会继续执行addWaiter()与acquireQueued()方法，所以先看看实际获取锁的代码内容

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // hasQueuedPredecessors()是公平锁的特有判断，如果前面有人排队，则返回true，否则返回false
        if (!hasQueuedPredecessors() &&
            // CAS将0设为1，即拿锁
            compareAndSetState(0, acquires)) {
            // 当条件判断式里两个方法都返回true后，代表当前线程已拿到锁了，此时将独占线程设为自己
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 如果状态不等于0，此时有两种情况：1.当前拿锁的线程不是自己、2.拿锁的线程是自己，自己重入拿锁了
    
    // 如果拿锁的线程是自己
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        // 此时不用CAS，直接赋值即可，因为这种情况下已拿到了锁，不会有其他线程来竞争数据
        setState(nextc);
        return true;
    }
    return false;
}
```

​	hasQueuedPredecessors()是整个公平锁的核心，公平锁不能插队，因此拿锁前先看下头节点后面是否有其他节点排队，且这个节点不是自己？如果有，则返回true，回到上层方法后不参与锁竞争，乖乖排队。如果没有，则返回false，回到上层方法后参与锁竞争（CAS），竞争到锁后将AQS的持有线程设为自己（当老大哥），最后逐层返回，继续执行拿锁后的业务代码。

```java
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    /*
    	这是公平锁最核心的一行代码
    	如果队列不为空 && (头节点下一个节点为空 || 头节点下一个节点不为空的话，持有线程不等于自己)
    	当这个表达式为true时，代表队列里有一个节点应当去拿锁，自己不该去插队，上面整个tryAcquire都会返回false
    */
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

## 入队

​	回到acquire(int arg)，如果拿锁失败了，tryAcquire(int acquires)返回false，此时会继续执行acquireQueued(addWaiter(Node.EXCLUSIVE), arg)，先看下addWaiter(Node.EXCLUSIVE)是怎么添加一个节点到AQS里的？

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    // 如果尾节点不为空
    if (pred != null) {
        // 先将新节点的前驱设为尾节点（这里会有个“尾分叉”现象，待会儿说）
        node.prev = pred;
        // CAS操作将尾节点设为当前节点
        if (compareAndSetTail(pred, node)) {
            // 如果CAS成功了，将之前的尾节点的后继节点设为新节点
            pred.next = node;
            return node;
        }
    }
    /**
    	走到这里有两种可能
    	1.尾节点为空（往空队列插入新节点）
    	2.CAS前被其他线程先插入了节点
    **/
    enq(node);
    // 最后将新增入队的节点返回给上层方法
    return node;
}
```

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { 
            /**
            	对应上面的情况1.尾节点为空，同时代表整个队列为空，头节点也为空
            	此时用CAS循环操作新建一个头节点，这个节点是个哑节点
            	然后进入下一次循环
            */
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // 进入到这里，代表队列已初始化完毕，要处理情况2.CAS前被其他线程先插入了节点
            
            // 这里和addWaiter()里“尾节点不为空”的处理一样，区别是如果失败了，外层会有个循环反复执行
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                // 最后将新增入队的节点返回给上层方法
                return t;
            }
        }
    }
}
```

## 尾分叉

​	总得来说，addWaiter()只会尝试一次添加新节点，如果失败了，会交给enq()做循环处理，直到添加成功。这里有个小细节，addWaiter和enq()里**实际添加节点**的代码部分都是

```java
node.prev = t;
if (compareAndSetTail(t, node)) {
    t.next = node;
    return t;
}
```

​	会有一个尾分叉的情况发生，即在node.prev = t;部分，因为有多个线程并发操作的可能，**会有多个新节点同时指向旧尾节点，但经过CAS操作后只会有一个新节点被旧尾节点指向。**其他失败的新节点会进入下一次循环，**重新进行尾分叉操作，重新CAS抢夺尾节点占有权**。

## 入队后是直接拿？还是睡眠

​	addWaiter()再AQS里插入了一个新节点后，回到上层方法acquireQueued()，这个方法是对新入队的节点做下一步处理，先看下具体源码

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            // 看一下当前节点的前驱节点是不是head，如果是的话重新尝试去拿锁
            if (p == head && tryAcquire(arg)) {
                /*
                	当拿到锁后，做了以下三步
                	1.将自己设为头节点，将自己的前驱设为null，节点线程设为null（符合哑节点特性）
                	2.将原头节点的next设为null，此时原头节点能被GC清掉
                	3.返回interrupted为false，即当前节点没有被阻断（挂起）
                */
                setHead(node);
                p.next = null; 
                failed = false;
                return interrupted;
            }
            /*
            	能走到这里的判断，说明当前节点的前驱节点并不是头节点，此时会做两步操作
            	1.调用shouldParkAfterFailedAcquire()根据前驱节点判断当前节点是否应该挂起
            	2.调用parkAndCheckInterrupt()实际挂起，此时线程就在这个方法体内被挂起了
            */
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 如果前驱节点的waitStatus是SIGNAL，直接返回true，说明应该被挂起
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        // 如果不是SIGNAL且＞0呢？直接向前遍历，直到找到一个为SIGNAL的前驱节点，并挂载他后面（可以理解为插队了）
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 既不是SIGNAL,又不＞0，则用CAS操作将前驱节点的waitStatus设为SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

​	可以看到，在shouldParkAfterFailedAcquire()里，如果前驱节点的waitStatus不为SIGNAL（比如有的节点取消了等锁操作，此时waitStatus为CANCEL），会做进一步操作，如＞0的话就向前遍历直到找到了waitStatus为SIGNAL的，然后直接挂在它后面；＜0的话直接用CAS操作将前驱节点的waitStatus改为SIGNAL。

​	但不管是哪种做法，只要前驱的waitStatus不为SIGNAL，shouldParkAfterFailedAcquire()都会返回false，回到上层方法acquireQueued()里继续循环一遍，**这是又会做前驱是不是头节点啊，自己是不是有资格被挂起啊之类的判断**。

​	值得注意的是，节点的唤醒是依靠前驱节点的，这个很好理解，假设队列为

​	head（原先持有锁的线程节点，由来变为头节点了） → A → B（tail） 

​	此时A和B都是在挂起状态，若head节点释放了锁，由于A在挂起，是无法感知锁被施放的，因此需要head节点给A一个通知，使A节点的线程唤醒，接着去拿锁（公平锁排第一不用竞争，直接拿就完事了）。而节点拥有唤醒后继节点的依据是waitStatus = SIGNAL，这也是为什么即将挂起的节点必须挂在SIGNAL节点后面，**就是为了能让SIGNAL节点唤醒自己。**

### 入队前拿锁和入队后拿锁

​	源码里还有一个细节，acquireQueued()里拿到锁后，需要将自身变为head，而tryAcquire()却不用。这是因为tryAcquire()属于入队前拿锁，此时队列还未初始化，自己也没作为节点插入到AQS里。

​	而acquireQueued()是入队后拿锁，当前线程已作为一个节点插入到AQS里，自己拿到锁后，本应从AQS里剔除，但AQS源码的做法是把自己设为head，并将旧head剔除，**这本质属于一个变相的出队操作**。

### 最终挂起

​	当发现自己已挂载SINGAL节点后面，有资格被挂起后，调用parkAndCheckInterrupt()将自己挂起，**等待前驱节点将自己唤醒**。

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);//执行到这里就被挂起了
    return Thread.interrupted();
}
```

```java
public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(false, 0L);
    setBlocker(t, null);//执行到这里就被挂起了
}
```

## 唤醒（释放锁）

### 找到目标节点

​	锁的释放是基于AQS的release()函数执行，由于锁的释放不涉及公平与不公平的概念，所以在ReentrantLock里unlock()都是调用release()函数。

```java
public final boolean release(int arg) {
    // 尝试释放锁
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            // 唤醒后继节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

​	由于tryRealse()本质是一个待重写方法，所以这里就看看ReentrantLock的实现

```java
protected final boolean tryRelease(int releases) {
    // 先算出解锁后的state
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        // 自己不是持锁的线程，却要解锁，直接抛异常（很好奇这种场景怎么发生？）
        throw new IllegalMonitorStateException();
    boolean free = false;
    // 如果解锁后直接为0了，代表可重入的锁全都解了
    if (c == 0) {
        free = true;
        // 将独占线程设为null
        setExclusiveOwnerThread(null);
    }
    // 更改state
    setState(c);
    // 返回解锁结果
    return free;
}
```

​	解锁结果有两种可能：1.true，可重入的锁已全部解锁完成，这代表整个锁已完全解锁，可以交给下一个线程获取，2.false，可重入的锁还没解锁完成，本线程还是持有着锁，需要再次解锁直到重入部分全部解锁完，整个锁才能完全解锁。当返回true时，回到上层方法，执行unparkSuccessor()唤醒后继节点

​	值得注意的是！！！tryRelease()如果返回true，则代表锁被释放了，如果是非公平锁的话，此时入队前线程可以和队内节点共同竞争锁（插队），但公平锁的话会先判断hasQueuedPredecessors()，为true则乖乖去队尾排队。

​	回到这一段代码，代表锁释放后唤醒后继节点（为什么他们那么反人类，喜欢不加{}呢）

```java
if (h != null && h.waitStatus != 0)
    unparkSuccessor(h);
```

​	当锁释放后，需要判断头节点 != null && 头节点的waitStatus != 0，这里再回顾下waitStatus的值    

```java
static final int CANCELLED =  1;

static final int SIGNAL    = -1;

static final int CONDITION = -2;

static final int PROPAGATE = -3;
```

​	回忆一下，有关设置waitStauts值的情况

​	1.是在shouldPark方法那，当前驱节点既不是SIGNAL,又不是CANCELLED，则用CAS操作将前驱节点的waitStatus设为SIGNAL。

​	2.发现获取不到锁，需要入队时，addWaiter()将线程包装成一个节点Node，Node的waitStatus默认值本身就为0。

​	也就是说，当一个节点的waitStatus为0，则代表这个节点并没有后继节点挂载，这个节点也没必要去唤醒后继节点，接下来继续看唤醒代码。

```java
private void unparkSuccessor(Node node) {
    
    int ws = node.waitStatus;
    if (ws < 0)
        // ＜0的话，先把ws改为0
        compareAndSetWaitStatus(node, ws, 0);

    // 获取挂载的后继节点
    Node s = node.next;
    // todo 实在想不懂会有后继节点为null的情况
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 如果后继节点的ws是CANCEL,代表自己不参与拿锁了，此时从队尾开始向前遍历，直到找到一个ws≤0的节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        // 找到后解锁
        LockSupport.unpark(s.thread);
}
```

​	这里有一个点值得思考，首先会先判断后继节点是否CANCEL了（ws > 0），这里要注意！！！虽然上面shouldPark()方法保证节点在排队时优先排到SIGNAL节点后面，但有可能在排队后，这个SIGNAL节点取消了排队变为CANCEL，如

​	head(原拿到锁的节点) → A(0) 		我要插入B(0)

​	**其实一开始A是0，B(0)将A的ws设为SIGNAL，并挂载到后面**

​	head(原拿到锁的节点) → A(SIGNAL) 		我要插入B(0)

​	head(原拿到锁的节点) → A(SIGNAL) → B(0)

​	**B挂载上去后，A有可能放弃拿锁，变为CANCEL**

​	head(原拿到锁的节点) → A(CANCEL) → B(0)

​	此时head发现自己下一个节点A是CANCEL状态，那不应该唤醒它，而是唤醒它后面的节点，但是！！！这里寻找下一个节点是从尾遍历到头的，这又是为什么？回到上面尾分叉的代码。

```java
node.prev = t;	
if (compareAndSetTail(t, node)) {
    t.next = node;
    return t;
}
```

​	因为AQS的尾分叉特性，这部分代码是不能保证原子性的，有可能节点A刚把tail设为自己，同时前驱设为原尾节点，此时原尾节点的后继还是null，这时候持锁线程从head往后遍历ws≤0节点时，会遍历不到节点A。

### 被唤醒后该做什么？

​	head ↔ 原tail ← A(当前tail)	此时还没执行到t.next = node;从左到右无法遍历到节点A。因此才会从尾节点开始遍历，**找到离head最近的ws<0节点**，获取节点线程，调用LockSupport.unpark(s.thread)唤醒。被唤醒的线程最终回到这里

```java
private final boolean parkAndCheckInterrupt() {
	LockSupport.park(this);//我在这里醒来了
    // 调用Thread.interrupted()将线程中断位设为false
	return Thread.interrupted();
}
```

​	然后返回到上层方法

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            // 2.再进行下一次循环，被唤醒后自己的前驱就是head了，由于是公平锁，自己是能拿到锁的（非公平锁的话还有可能与插队的线程竞争锁）
            final Node p = node.predecessor();
            // 3.最后将自己变为头节点，返回true（代表自己被唤醒了）
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 4.补充，如果是非公平锁，万一抢不到锁的话，还得回去继续挂起
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 1.返回后，给interrupted设为true
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

```java
if (!tryAcquire(arg) &&
    acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    /**
    	当acquireQueued()返回true后，代表自己一开始拿不到锁、入队睡眠后被唤醒、唤醒后才拿到锁
    	这种情况下获取到锁的线程，需要调用selfInterrupt()，里面实际调用interrupt()方法
    **/
    selfInterrupt();
```

## Thread.interrupted()与thread.interrupt()

​	为什么线程被唤醒后，需要调用一次Thread.interrupted()，获取完锁后，还要调用一次thread.interrupt()？先从这两个方法的区别入手，首先要明确一点，想要挂起线程LockSurport.park()，线程的中断标志位必须为false。

​	Thread.interrupted()：返回当前线程的中断状态，被唤醒后中断状态为true，同时将线程中断状态改为false

​	thread.interrupt()：对于执行中的线程，interrupt()只是将线程中断状态设为true

​	TODO 这里先留个坑，等把多线程重新复习一遍在做笔记