## 深入解析CAS

CAS, `Compare And Swap`, 即比较并交换。`Doug Lea`大神在同步组件中大量使用CAS技术非常巧妙的显示了无锁化的Java多线程的并发操作(这里的无锁化指的是Java代码级别的, 在汇编级别还是对数据进行加锁操作)。整个AQS同步组件, Atomic原子操作等等都是以CAS实现的, 甚至`ConcurrentHashMap`在1.8版本中也调整为CAS+Synchronized。所以 **CAS是整合JUC的基石。**

![cas_base](/image/cas_base.png)

看到上面这个图, 我们就知道为什么我们要先学习CAS, 再去学习JUC下的工具类。

### 案例引入

我们先看下面的一段代码:

```java
public class Test {
    private static volatile int i = 0;

    public static void main(String[] args) {
        new Thread(() -> {
            for (int j = 0; j < 10; j++) {
                i++;
                System.out.println(i);
            }
        }).start();

        new Thread(() -> {
            for (int j = 0; j < 10; j++) {
                i++;
                System.out.println(i);
            }
        }).start();
    }
}
```

上面的代码的输出结果, 会发现有很多重复的值, 原因我们之前已经分析过了, 因为volatile只保证可见性和有序性, 但是并不保证原子性。i++操作是一个非原子性操作, 所以会导致重复数据。也就是线程安全问题。

但是换成下面的代码, 问题就解决了:

```java
public class Test {
    private static AtomicInteger value = new AtomicInteger(0);

    public static void main(String[] args) {
         new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                int result = value.addAndGet(1);
                System.out.println(result);
            }
        }).start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                int result = value.addAndGet(1);
                System.out.println(result);
            }
        }).start();
    }
}
```

我们发现输入的结果, 一定不会重复。也就是说对于value是线程安全的。**而addAndGet()方法的原理就是CAS。**

### CAS分析

在CAS中有三个参数: 当前值E, 当前新值(预期值)N, 更新值V。 当且仅当 E == N时, 才会将当前值E改为V, 否则就一直处于循环中。伪代码如下:

```java
if (this.value == N) {
    this.value = V;
    return true;
} else {
    return false;
}
```

对于上面`addAndGet(int value)`的源码如下:

```java
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

static {
    try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}

private volatile int value;

...

public final int addAndGet(int delta) {
    return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
}

// unsafe的getAndAddInt()方法
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```

`Unsafe`是CAS的核心类, Java是无法直接访问底层操作系统, 而是通过本地(native)方法来访问。但是尽管如此, JVM还是开了一个后门: **`Unsafe`**, 它提供了硬件级别的原子操作。

- `valueOffset`为变量值在内存中的偏移地址, unsafe就是通过偏移地址来得到数据的原值
- `value`是当前值, 使用volatile修饰, 保证多线程环境下看见的是同一个。

所以我们看到`addAndGet`方法的核心在于`Unsafe`类下的natvie方法`compareAndSwapInt`:

```java
// unsafe的 native方法 compareAndSwapInt()
public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```

该方法为本地方法, 有四个参数:
- **Object var1**: 对象
- **long var2**: 对象的地址
- **int var4**: 预期值
- **int var5**: 修改值

这个方法是比较内存中的一个值(整型)和我们的期望值(`var4`)是否一样, 如果一样则将内存中的这个值更新为`var5`, 参数中的`var1`是值所在的对象, `var2`是值在对象中的内存偏移量, **参数`var1`和`var2`是为了定位出值所在内存的地址。**


所以了解CAS基本原理, 和addAndGet()方法的源码之后, 我们在对上面的Test代码进行分析:

> 对于Test中的代码, 其实就是做 i = 1; i = i + 1操作; <br/>
> 1. 结合CAS的流程, 当线程A进入`addAndGet(int delta)`方法时, 此时当前值E = 1, 更新值V = E + delta = 2, <br/>
> 2. 如果此时线程A放弃了CPU执行权 然后线程B进入`addAndGet()`方法时, 此时当前值E = 1, 更新值V = E + delta = 2, 而且当前新值(也就是预期值)N = value = 1, 所以 E == N, E = V = 2 所以线程B更新成功, 
> 3. 此时当前值E = 2, 而对于线程A来说当前值E = 1, 而当前新值N = 2, 所以 E != N, 所以线程A一直处于循环中, 一直重复上面的步骤直到E == N。<br/>

### CAS缺陷

我们之前分析了代码的步骤, CAS虽然高效的解决了原子操作, 但是还是存在一些缺陷, 主要表现为:
- **循环时间太长**
    有很大可能发生一种情况, 如果自旋CAS长时间地不成功, 则会给CPU带来非常大的开销。在JUC有些地方就会限制CAS自旋的次数, 例如:`BlockingQueue`中的`SynchronizedQueue`。

- **只能保证一个共享变量原子操作**
    对于CAS的实现, 就发现了这只能针对一个共享变量, 如果是多个共享变量就只能使用锁。

- **ABA问题**
    CAS需要检查操作值有没有发生改变, 如果，没有改变则更新。但是存在这样一种情况: **如果一个值原来是A, 变成了B, 然后又变成了A, 那么在CAS检查时会发现没有改变, 但实际上它已经发生了改变, 这就是ABA问题。**

    `ABA问题`带来的危害:

    ![cas_aba_1](/image/cas_aba_1.png)

    现有一个单向链表实现的堆栈, 栈顶为A, 这是线程1已经获取到A.next=B, 然后希望用CAS将A改为B, 
    ```java
    head.compareAndSet(A, B);
    ```
    但是在线程1执行上面代码之前, 线程2获取到CPU执行权, 将A, B出栈, 在push D, C, A; 此时堆栈结构如下:

    ![cas_aba_2](/image/cas_aba_2.png)

    此时线程1执行CAS操作, 经过compare发现栈顶仍为A, 所以CAS成功, 栈顶变为B, 但实际上B已经出栈了, 所以B.next=null

    ![cas_aba_3](/image/cas_aba_3.png)

    其中堆栈只有B一个元素, C和D组成的链表不再存在于堆栈中, 平白无故就把C, D丢掉了。

### 解决ABA问题

CAS整个的流程

![cas_process](/image/cas_process.png)

CAS的ABA隐患问题, 解决方案是版本号, 每次只要对原来的值进行更新, 就会加上一个版本号, 这样每次判断只要发现版本号变化了, 那么就更新失败。即A->B->A, 变成 A1->B2->A3。

在JUC中提供了`AtomicStampedReference`来解决这个问题, `AtomicStampedReference`通过包装[E, Integer]的元祖来对对象标记版本戳stamp, 从而避免ABA问题。

`AtomicStampedReference`的`compareAndSet()`方法源码如下:

```java
/**
* Atomically sets the value of both the reference and stamp
* to the given update values if the
* current reference is {@code ==} to the expected reference
* and the current stamp is equal to the expected stamp.
*
* @param expectedReference the expected value of the reference
* @param newReference the new value for the reference
* @param expectedStamp the expected value of the stamp
* @param newStamp the new value for the stamp
* @return {@code true} if successful
*/
public boolean compareAndSet(V   expectedReference,
                                V   newReference,
                                int expectedStamp,
                                int newStamp) {
    Pair<V> current = pair;
    return
        expectedReference == current.reference &&
        expectedStamp == current.stamp &&
        ((newReference == current.reference &&
            newStamp == current.stamp) ||
            casPair(current, Pair.of(newReference, newStamp)));
}
```

`compareAndSet`有四个参数:

- `expectedReference`: 预期引用
- `newReference`: 更新后的引用
- `expectedStamp`: 预期版本
- `newStamp`: 更新后的版本

源码的判断不难理解, 

`expectedReference == current.reference`(预期引用==当前引用), 

`expectedStamp == current.stamp`(预期的版本==当前版本), 

`(newReference == current.reference && newStamp == current.stamp)`(更新后的引用和版本与当前引用和版本相等) 

或者

`casPair(current, Pair.of(newReference, newStamp))`通过Pair的of方法生成一个新的Pair对象, 与当前pair进行CAS替换, 如果casPair返回ture, 也就是可以交换成功。

```java
private static class Pair<T> {
    final T reference;
    final int stamp;
    private Pair(T reference, int stamp) {
        this.reference = reference;
        this.stamp = stamp;
    }
    static <T> Pair<T> of(T reference, int stamp) {
        return new Pair<T>(reference, stamp);
    }
}

private volatile Pair<V> pair;
```

> Pair为`AtomicStampedReference`的内部类, 主要记录对象的引用和版本戳, 版本戳为int型, 保持自增。同时Pair是不可变对象, 其所有属性全部定义为final, 对外提供一个of方法, 该方法返回一个新的Pair对象。而且pair对象定义为`volatile`, 保证多线程环境下的可见性。

下面的代码案例诠释了`AtomicStampedReference`解决ABA问题的过程：

**要注意下面代码的执行顺序, 我们现在t2线程中获取了当前的stamp值,并且在睡眠2s后打印(主要是想让第三次在前两次更新后在打印), 此时的值还未在第一次更新前就获取了, 所以值为0。**

```java
public class AtomicStampedReferenceTest {

    /**
     * 设置初始值 和 版本号
     */
    private static AtomicStampedReference<Integer> atomicRef = new AtomicStampedReference<>(100, 0);

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            try {
                TimeUnit.SECONDS.sleep(1);
                System.out.println("Before first update stamp = " + atomicRef.getStamp());
                boolean success = atomicRef.compareAndSet(100, 101, atomicRef.getStamp(), atomicRef.getStamp() + 1);
                System.out.println(success);

                System.out.println("Before second update stamp = " + atomicRef.getStamp());

                // 这样操作最终值, 还是100, 但是版本号为1
                boolean stampSuccess = atomicRef.compareAndSet(101, 100, atomicRef.getStamp(), atomicRef.getStamp() + 1);
                System.out.println(stampSuccess);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        Thread t2 = new Thread(() -> {
            try {

                // t2先获取当前的stamp, 此时应该是0
                int stamp = atomicRef.getStamp();
                TimeUnit.SECONDS.sleep(2);
                System.out.println("Before third update stamp = " + stamp);
                // 在进行compareAndSet时, 由于t1已经改过两次了, 所以这个版本号必须是2才会修改成功。
                boolean success = atomicRef.compareAndSet(100, 101, stamp, stamp + 1);
                System.out.println(success);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        t1.start();
        t2.start();
        t2.join();
        t2.join();
    }
}
```

结果为:

```txt
Before first update stamp = 0
true
Before second update stamp = 1
true
Before third update stamp = 0
false
```