# AtomicInteger核心成员

```java
// 进行指针运算的后门
private static final Unsafe unsafe = Unsafe.getUnsafe();
// value的偏移量
private static final long valueOffset;

static {
    try {
        //获取value的偏移量，赋值到valueOffset里
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}
// 实际的value值
private volatile int value;
```

​	**AtomicInteger三大核心要素：volatile，循环，CAS**。

# AtomicInteger核心方法

```java
// 原子性地[获取旧值，并设置新值]
public final int getAndSet(int newValue) {
    return unsafe.getAndSetInt(this, valueOffset, newValue);
}	

/* 
这里主要做了三步
	1.直接通过内存偏移量，获取内存中的value值
	2.将获取到的value值（var5）与内存中的value值再进行比较，如果相等，则将内存中的value变为var4
	3.如果设值成功，返回var5
*/
public final int getAndSetInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var4));

        return var5;
    }


```

​	getAndSetInt()需要重点关注第一步与第二步，为什么你get到值后，在设值前还要比较一遍？首先getAndSetInt()是没有上锁的，我们假设以下场景发生：

​	A线程：var5 = this.getIntVolatile(var1, var2);直接从内存获取value，假设var5为0。

​	B线程：调用AtomicInteger.set将value设为1。

​	A线程：不进行CAS，直接设value的值为2，同时返回0。

​	此时A线程调用的结果是：我把value从0 → 2，但实际情况是我把value从1 → 2。

​	AtomicInteger.set()和AtomicInteger.get()是原子性，但两个合在一起就不是了，因此为了防止A线程在get后、赋值前被其他线程污染了数据，需要A线程再赋值前CAS一遍，如果不正确，则重新get值。

​	**CAS里最核心的代码**是compareAndSwapXXX(Object var1, long var2, XXX var3, XXX var4)，它是jni方法，看不到源码，但大体流程是：

​	1.先通过var2这个偏移量，从var1里找到其成员变量temp。

​	2.temp用来与var3进行比较，是否相等?将var1里var2偏移量的变量设为var4，返回true:返回false。

​	compareAndSwapXXX是原子操作，不用担心其安全性。