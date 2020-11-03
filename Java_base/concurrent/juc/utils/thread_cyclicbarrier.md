## J.U.C并发工具类: CyclicBarrier

`CyclicBarrier`是一个同步辅助类, 让一组线程到达一个屏障时被阻塞, 直到最后一个线程到达屏障时, 所有被屏障拦截的线程才会继续执行后面的任务。

### CyclicBarrier实现分析

`CyclicBarrier`的内部是使用重入锁`ReentrantLock`和`Condition`。它有两个构造函数

- `CyclicBarrier(int parties)`
- `CyclicBarrier(int parties, Runnable barrierAction)`

`parties`表示拦截线程的数量, `barrierAction`是当所有线程都到达屏障时, 会优先执行barrierAction, 用于回调处理更加复杂的业务场景。

在CyclicBarrier中最重要的方法就是`await()`方法, 只有`parties`数量的线程调用到await()方法时, 线程才会执行后面的逻辑。

CyclicBarrier提供两个await()方法:

- `await()`
- `await(long timeout, TimeUnit unit)`: 设置超时时间

> `await()`的处理逻辑比较简单, 如果该线程不是到达最后一个线程, 则它会一直处于等待状态, 除非:<br/>
> 1. 最后一个线程到达<br/>
> 2. 超出指定时间<br/>
> 3. 其他某个线程中断当前线程<br/>
> 4. 其他某个线程中断另一个等待的线程<br/>
> 5. 其他某个线程在等待barrier超时<br/>
> 6. 其他某个线程在此barrier调用reset()方法。reset()方法用于将屏障重置为初始状态<br/>



### 应用

`CyclicBarrier`适用于多线程结果合并操作, 用于多线程计算数据, 最后合并计算结果的应用场景, 比如我们需要统计多个Excel中的数据, 最后需要统计一个总结果。
我们可以通过多线程处理么一个Excel, 执行完成后得到相应的结果, 最后用barrierAction来计算这些线程的计算结果, 得到所有的Excel的总和。

代码示例:

```java
public class CyclicBarrierExample1 {

    public static void main(String[] args) throws InterruptedException {
//        CyclicBarrier cyclicBarrier = new CyclicBarrier(2);

        // 这个runnable接口时当所有线程执行完成后, 进行回调
        CyclicBarrier cyclicBarrier = new CyclicBarrier(2, new Runnable() {
            @Override
            public void run() {
                System.out.println("all of finished");
            }
        });

        new Thread() {
            @Override
            public void run() {
                try {
                    TimeUnit.SECONDS.sleep(15);
                    System.out.println("T1 finished.");
                    cyclicBarrier.await();
                    System.out.println("T1 the other thread finished to.");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }
        }.start();

        new Thread() {
            @Override
            public void run() {
                try {
                    TimeUnit.SECONDS.sleep(10);
                    System.out.println("T2 finished.");
                    cyclicBarrier.await();
                    System.out.println("T2 the other thread finished to.");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }
        }.start();
    }
}
```

结果:

```txt
T2 finished.
T1 finished.
all of finished
T1 the other thread finished to.
T2 the other thread finished to.
```

根据结果发现: 只有当两个都调用到await()方法时, 就会立即执行barrierAction线程, 然后这两个线程也会继续执行后面的逻辑。
