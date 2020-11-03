## J.U.C并发工具类: Exchange

`Exchange`是JUC中并发工具类最简单也是最复杂的, 简单在于API非常简单, 就一个构造方法和两个`exchange()`方法, 最复杂在于它的实现是最复杂的。所以这里只介绍怎么用。(因为原理我也晕了)

### Exchange 实现分析

先看下面的一段代码:

```java
public class ExchangeExample1 {

    public static void main(String[] args) {
        final Exchanger<String> exchanger = new Exchanger<>();

        new  Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName() + " start.");
                try {
                    String result = exchanger.exchange("I am come from T-A");
                    System.out.println(Thread.currentThread().getName() + " Get value [" + result + "]");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    System.out.println("Time out");
                }

                System.out.println(Thread.currentThread().getName() + " end.");
            }
        }, "==A==").start();

        new  Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println(Thread.currentThread().getName() + " start.");
                try {
                    String result = exchanger.exchange("I am come from T-B");
                    System.out.println(Thread.currentThread().getName() + " Get value [" + result + "]");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                System.out.println(Thread.currentThread().getName() + " end.");
            }
        }, "==B==").start();
    }
}
```

打印的结果:

```txt
==A== start.
==B== start.
==B== Get value [I am come from T-A]
==B== end.
==A== Get value [I am come from T-B]
==A== end.
```

上面的代码中, 我们先new Exchange(), 然后创建两个线程, 分别在两个线程中调用`exchange()`方法, 返回的结果是对方线程传入的参数。而且`exchange()`方法是阻塞方法, 只有获取到数据才会继续下面的逻辑。

对于exchange方法:

```java
V r = exchange(V v);
V r = exchange(V v, long timeout, TimeUnit unit);
```
参数v表示当前线程传入的值
返回值r表示对象线程返回的值

> 要注意的是: <br/>
> 1. 其中一个线程在等待对方线程数据时Time out, 那么另一个线程可能就会一直处于等待状态 <br/>
> 2. exchange一定是成对出现的, 也就是说只能有两个线程进行数据交换, 这也是Exchange的局限

