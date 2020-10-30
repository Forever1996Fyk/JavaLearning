## 多线程的观察者模式

如果我们要关心一些事物发生的状态, 那么我们可能就需要不断的对这个事物的状态进行查询, 而这个很明显是低效的方式。最好的方式应该是当这个事物发生变化时, 它会主动通知我们, 这种方式就是观察者模式。

> 当某个对象发生状态改变需要通知第三方的时候, 就非常适合使用观察者模式。

### 1. 观察者模式

**观察者模式**其实是一种对象间一对多的依赖关系, 当每一个对象改变状态, 则依赖于它的对象都会得到通知并自动更新。

> 观察者模式还有其他别名: 发布-订阅(publish/subscribe)模式, 模型-视图(model/view)模式, 源-监听(source/listener)模式

![观察者模式UML](/image/observer_model.png)

**要注意的是: 上面的Subject中存入观察者的集合用的是Vector, 而不是List, 因为在多线程环境中Vector时线程安全的, 而List是不安全的**

举一个简单的例子: 转换int类型的数字为二进制和八进制。每转换一次就打印一次

抽象观察者`Observer`:
```java
public abstract class Observer {

    /**
     * 订阅者, 事件源
     */
    protected Subject subject;

    public Observer(Subject subject) {
        this.subject = subject;
        subject.attach(this);
    }

    public abstract void update();
}
```

具体的观察者:

```java
public class BinaryObserver extends Observer {

    public BinaryObserver(Subject subject) {
        super(subject);
    }
    @Override
    public void update() {
        System.out.println("Binary String:" + Integer.toBinaryString(subject.getState()));
    }
}

public class OctalObserver extends Observer {

    public OctalObserver(Subject subject) {
        super(subject);
    }

    @Override
    public void update() {
        System.out.println("Octal String:" + Integer.toOctalString(subject.getState()));
    }
}
```

被观察者`Subject`(这里简化了, 按照UML类图, 应该有一个抽象的被观察者):

```java
public class Subject {

    private List<Observer> observers  = new ArrayList<>();

    /**
     * 每次改变的状态
     */
    private int state;

    public int getState() {
        return state;
    }

    /**
     * 状态发生变化
     * @param state
     */
    public void setState(int state) {
        // 注意这里, 第一次赋值state时, 是不会通知的, 也就是第一次进行setState(xx)时, 此时是state == this.state是true
        if (state == this.state) {
            return;
        }
        this.state = state;
        notifyAllObserver();
    }

    /**
     * 附加每一个观察者observer
     * @param observer
     */
    public void attach(Observer observer) {
        observers.add(observer);
    }

    /**
     * 通知所有的观察者observer
     */
    private void notifyAllObserver() {
        observers.stream().forEach(Observer::update);
    }
 }
```

测试`ObserverClient`:

```java
public class ObserverClient {
    public static void main(String[] args) {
        final Subject subject = new Subject();
        new BinaryObserver(subject);
        new OctalObserver(subject);

        System.out.println("=====================");
        subject.setState(10);
        System.out.println("=====================");
        subject.setState(10);
        System.out.println("=====================");
        subject.setState(15);
        System.out.println("=====================");
    }
}
```

结果:

```txt
=====================
Binary String:1010
Octal String:12
=====================
=====================
Binary String:1111
Octal String:17
=====================
```

### 2. 多线程下观察者模式的使用

利用观察者模式我们可以监听线程任务的生命周期。当任务状态发生变化时, 任务就会通知调用者, 让调用者执行相关逻辑。

1. 定义任务的生命周期状态(我这里简化了)

```java
    public enum RunnableState {
        RUNNING, ERROR, DONE;
    }
```

2. 定义一个任务执行的监听者接口, 当任务发生变化时发送通知处理相关逻辑。相当于Subject

```java
public interface LifeCycleListener {
    void onEvent(ObserverRunnable.RunnableEvent event);
}
```

3. 定义观察者接口(我这里写的抽象类)

```java
public abstract class ObserverRunnable implements Runnable {
    final protected LifeCycleListener listener;

    public ObserverRunnable(LifeCycleListener listener) {
        this.listener = listener;
    }

    protected void notifyChange(final RunnableEvent event) {
        listener.onEvent(event);
    }

    public enum RunnableState {
        RUNNING, ERROR, DONE;
    }

    public static class RunnableEvent {
        private final RunnableState state;
        private final Thread thread;
        private final Throwable throwable;

        public RunnableEvent(RunnableState state, Thread thread, Throwable throwable) {
            this.state = state;
            this.thread = thread;
            this.throwable = throwable;
        }

        public RunnableState getState() {
            return state;
        }

        public Thread getThread() {
            return thread;
        }

        public Throwable getThrowable() {
            return throwable;
        }
    }
}
```

4. 实现任务执行的监听者接口

```java
public class ThreadLifeCycleObserver implements LifeCycleListener {
    private final Object LOCK = new Object();

    public void concurrentQuery(List<String> ids) {
        if (ids == null || ids.isEmpty()) {
            return;
        }

        ids.stream().forEach(id -> new Thread(new ObserverRunnable(this) {
            @Override
            public void run() {
                try {
                    // 每个线程执行方法前, 通知Observer监听器, 打印状态
                    notifyChange(new RunnableEvent(RunnableState.RUNNING, Thread.currentThread(), null));
                    System.out.println("query for the id " + id);
                    Thread.sleep(1000L);
                    int x = 1/0;
                    // 每个线程执行方法完之后, 通知Observer监听器, 打印状态
                    notifyChange(new RunnableEvent(RunnableState.DONE, Thread.currentThread(), null));
                } catch (Exception e) {
                    // 每个线程执行方法异常时, 通知Observer监听器, 打印状态
                    notifyChange(new RunnableEvent(RunnableState.ERROR, Thread.currentThread(), e));
                }
            }
        }, id).start());
    }

    @Override
    public void onEvent(ObserverRunnable.RunnableEvent event) {
        synchronized (LOCK) {
            System.out.println("The runnable [" + event.getThread().getName() + "] data changed and state is [" + event.getState() + "]");
            if (event.getThrowable() != null) {
                System.out.println("The runnable [" + event.getThread().getName() + "] process failed.");
                event.getThrowable().printStackTrace();
            }
        }
    }
}
```

测试类:

```java
public class ThreadLifeCycleClient {

    public static void main(String[] args) {
        new ThreadLifeCycleObserver().concurrentQuery(Arrays.asList("1", "2"));
    }
}
```