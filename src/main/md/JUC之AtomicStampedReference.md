# 前提

​	对于操作volatile对象变量里的属性时，对于其他线程也具有可见性，但AtomicStampedReference不应该用于操作对象内属性，而是操作对象本身，他应该用来关心这个对象的引用是否变了，而不是对象内的属性是否变了。一般是搭配每个对象代表独立意义的类来使用，如Integer。



# AtomicStampedReference核心成员

```java
public class AtomicStampedReference<V> {
	
    private static class Pair<T> {
        // 引用对象。
        final T reference;
        // 
        final int stamp;
        private Pair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }
        static <T> Pair<T> of(T reference, int stamp) {
            return new Pair<T>(reference, stamp);
        }
    }
	// Pair对象，是volatile的。
    private volatile Pair<V> pair;
    // AtomicStampedReference的创建需要导入一个实体类对象以及初始版本号。
    public AtomicStampedReference(V initialRef, int initialStamp) {
        pair = Pair.of(initialRef, initialStamp);
    }
}
```



# AtomicStampedReference核心方法

```java
// 要知道pair在AtomicStampedReference里是volatile变量对象，pair的成员的获取与写也是具有volatile特性的。
public V getReference() {
    return pair.reference;
}

public int getStamp() {
    return pair.stamp;
}

// 这里与上面的获取不同，这里是一次性获取版本号与实例。传入一个size>1的数组，get()会将版本号放在这个数组的第一个下标那。
public V get(int[] stampHolder) {
    Pair<V> pair = this.pair;
    stampHolder[0] = pair.stamp;
    return pair.reference;
}
// 不过值得注意，这个方法不保证原子性，有可能线程A执行到stampHolder[0] = pair.stamp，线程B修改了实体与版本号，线程A再return线程B修改的实体，就会出现版本号与实体不对应这种诡异的情况。
```

```java
public boolean compareAndSet(V   expectedReference,
                             V   newReference,
                             int expectedStamp,
                             int newStamp) {
    Pair<V> current = pair;
    return
        // 你手头上的原值是否和pair里的相等？
        expectedReference == current.reference &&
        // 你手头上的版本号是否合pair的相等？
        expectedStamp == current.stamp &&
        // 你给我的新值是否和pair里的相等？ 这个一般来说都是false。
        ((newReference == current.reference &&
          // 你给我的新版本号是否和pair里的相等？ 这个一般来说都是false。
          newStamp == current.stamp) ||
         // 因为上面为false，所以直接走到这一步，调用casPair()。
         casPair(current, Pair.of(newReference, newStamp)));
}

/*
	cmp:当前的pair对象。
	val:要更改为的新pair对象。
*/
private boolean casPair(Pair<V> cmp, Pair<V> val) {
    	/*
    	这里调的是native方法了，具有原子性，流程是：
    	1.先通过this指针与pairOffset偏移量找到内存中的pair对象。
    	2.比较内存中的pair对象与传入的"当前pair对象"是否一致？
    	3.如果2.判断一致，则将内存中的pair修改为新的pair。
    	*/
        return UNSAFE.compareAndSwapObject(this, pairOffset, cmp, val);
}

public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);
```

​	上面的源码解析可以看到，版本号的比较是在调用JNI方法前，先比较版本号与是否一致（因为pair是volatile的，所以获取的版本号也是volatile的），如果一致则调用JNI方法用CAS为pair赋新值。

​	值得注意的是，compareAndSet()这个CAS操作并没有循环，只会执行一次

