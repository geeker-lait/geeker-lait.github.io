#Java并发之原子变量和原子引用与volatile
        我们知道在并发编程中，多个线程共享某个变量或者对象时，必须要进行同步。同步的包含两层作用：1）互斥访问(原子性)；2）可见性；也就是多个线程对共享的变量互斥地访问，同时线程对共享变量的修改必须对其他线程可见，也就是所有线程访问到的都是最新的值。

###1. volatile变量和volatile引用

        volatile的作用是：保证可见性，但是没有互斥访问语义(原子性语义)。volatile能够保证它修饰的引用以及引用的对象的可见性，volatile不仅保证变量或者引用对所有访问它的线程的可见性，同时能够保证它所引用的对象对所有访问它的线程的可见性。volatile的使用要求满足下面的两个条件：

        1）对变量或者引用的写操作不依赖于变量或者引用的当前值(如果只有特定的单个线程修改共享变量，那么修改操作也是可以依赖于当前值)；

        2）该变量或者引用没有包含在其它的不变式条件中；

        volatile最常见的错误使用场景是使用volatile来实现并发 i++; 错误的原因是，该操作依赖于 i 变量的当前值，他是在 i 变量的当前值的基础上加一，所以说他依赖于 i 的当前值。多个线程执行 i++; 会丢失更新。比如两个线程同时读到 i 的当前值8，都进行加一，然后写回，最终 i 的结果是 9，而不是我们期待的10，丢失了更新。那么原子变量的引入就是针对volatile的这个缺陷的！！！原子变量的修改操作允许它依赖于当前值，所以说”原子变量“是比volatile的语义稍微强化一点！他不仅具有volatile的可见性，同时对原子变量的修改可以依赖于当前值。

###2. 原子变量和原子引用

    从Java 1.5开始引入了原子变量和原子引用：
    
    java.util.concurrent.atomic.AtomicBoolean
    
    java.util.concurrent.atomic.AtomicInteger
    
    java.util.concurrent.atomic.AtomicLong
    
    java.util.concurrent.atomic.AtomicReference
    
    以及他们对应的数组：
    
    java.util.concurrent.atomic.AtomicIntegerArray
    
    java.util.concurrent.atomic.AtomicLongArray
    
    java.util.concurrent.atomic.AtomicReferenceArray
    
    原子变量和引用都是使用compareAndSwap（CAS指令）来实现：依赖当前值的原子修改的。
    
    而且他们的实现都是使用volatile和Unsafe：volatile保证可见性，而Unsafe保证原子性。
    
    我们可以稍微分析下AtomicReference的源码：
```java
public class AtomicReference<V> implements java.io.Serializable {
    private static final long serialVersionUID = -1848883965231344442L;

    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicReference.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile V value;

    /**
     * Creates a new AtomicReference with the given initial value.
     *
     * @param initialValue the initial value
     */
    public AtomicReference(V initialValue) {
        value = initialValue;
    }

    /**
     * Creates a new AtomicReference with null initial value.
     */
    public AtomicReference() {
    }

    /**
     * Gets the current value.
     *
     * @return the current value
     */
    public final V get() {
        return value;
    }

    /**
     * Sets to the given value.
     *
     * @param newValue the new value
     */
    public final void set(V newValue) {
        value = newValue;
    }

    /**
     * Eventually sets to the given value.
     *
     * @param newValue the new value
     * @since 1.6
     */
    public final void lazySet(V newValue) {
        unsafe.putOrderedObject(this, valueOffset, newValue);
    }

    /**
     * Atomically sets the value to the given updated value
     * if the current value {@code ==} the expected value.
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that
     * the actual value was not equal to the expected value.
     */
    public final boolean compareAndSet(V expect, V update) {
        return unsafe.compareAndSwapObject(this, valueOffset, expect, update);
    }

    /**
     * Atomically sets the value to the given updated value
     * if the current value {@code ==} the expected value.
     *
     * <p><a href="package-summary.html#weakCompareAndSet">May fail
     * spuriously and does not provide ordering guarantees</a>, so is
     * only rarely an appropriate alternative to {@code compareAndSet}.
     *
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful
     */
    public final boolean weakCompareAndSet(V expect, V update) {
        return unsafe.compareAndSwapObject(this, valueOffset, expect, update);
    }

    /**
     * Atomically sets to the given value and returns the old value.
     *
     * @param newValue the new value
     * @return the previous value
     */
    @SuppressWarnings("unchecked")
    public final V getAndSet(V newValue) {
        return (V)unsafe.getAndSetObject(this, valueOffset, newValue);
    }

    /**
     * Atomically updates the current value with the results of
     * applying the given function, returning the previous value. The
     * function should be side-effect-free, since it may be re-applied
     * when attempted updates fail due to contention among threads.
     *
     * @param updateFunction a side-effect-free function
     * @return the previous value
     * @since 1.8
     */
    public final V getAndUpdate(UnaryOperator<V> updateFunction) {
        V prev, next;
        do {
            prev = get();
            next = updateFunction.apply(prev);
        } while (!compareAndSet(prev, next));
        return prev;
    }
......
```
我们可以看到使用了： private volatile V value; 来保证 value 的可见性；

同时：
```java
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicReference.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
```
    这段代码的意思是：获得AtomicReference<V>实例化对象中的 value 属性的在该对象在堆内存的偏移 valueOffset 位置，而：unsafe.compareAndSwapObject(this, valueOffset, expect, update);
    的作用就是通过比较valueOffset处的内存的值是否为expect，是的话就更新替换成新值update，这个操作是原子性的。所以volatile保证了可见性，而unsafe保证了原子性。源码中的UnaryOperator，BinaryOperator等等明显是模仿C++的，因为Java中没有函数指针，所以只能使用一元、二元操作对象来实现相应的功能。

#3. Unsafe
    
    Unsafe的源码可以参见：http://www.docjar.com/html/api/sun/misc/Unsafe.java.html
    
    他的实现主要是通过编译器，利用CPU的一些原子指令来实现的。
    
    Most methods in this class are very low-level, and correspond to a
    small number of hardware instructions (on typical machines).  Compilers
    are encouraged to optimize these methods accordingly.

#4. LongAdder（加法器）
        在jdk 1.8中又引入了进过充分优化的原子变量“加法器”：java.util.concurrent.atomic.LongAdder，它的性能在和其他原子变量以及volatile变量相比都是最好的，所以在能使用LongAdder的地方就不要使用其它原子变量了。但是LongAdder中并没有提供：依赖于当前变量的值来修改的操作。一般用于实现并发计数器是最好的。

LongAdder实现下列接口：

```java
public void add(long x);    // 加 x   
public void increment();   // 加1
public void decrement();  // 减1
public long sum();            // 求和
public void reset();          // 重置0
public String toString() {
        return Long.toString(sum());
    }

    /**
     * Equivalent to {@link #sum}.
     *
     * @return the sum
     */
    public long longValue() {
        return sum();
    }

    /**
     * Returns the {@link #sum} as an {@code int} after a narrowing
     * primitive conversion.
     */
    public int intValue() {
        return (int)sum();
    }

    /**
     * Returns the {@link #sum} as a {@code float}
     * after a widening primitive conversion.
     */
    public float floatValue() {
        return (float)sum();
    }

    /**
     * Returns the {@link #sum} as a {@code double} after a widening
     * primitive conversion.
     */
    public double doubleValue() {
        return (double)sum();
    }
```
总结：

就同步语义而言，volatile < 原子变量/原子引用 < lock/synchronized. volatile只保证可见性；原子变量和原子引用保证可见性的同时，利用CAS指令实现了原子修改——“修改操作允许它依赖于当前值”；而lock/synchronized则同时保证可见性和互斥性(原子性)。