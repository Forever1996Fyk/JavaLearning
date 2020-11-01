## 读写分离锁

我们知道通过`synchronized`关键字的修饰, 相当于给代码加了一把锁, 始终只能让拿到钥匙的线程才能通过, 其他线程只能阻塞。但是这种加锁方式直接让程序串行化。每次进行读操作时, 也只能有一个线程读取数据,这样效率很低。

所以我们可以设计一种模式, 当只有线程读的情况下, 可以让程序并行化执行。这就是读写分离锁。

> 对于读写分离锁的情况, 要注意的是只有当线程都是读操作的时候才能并行化, 其他情况都不能并行化。(后面我们还会说到`StampedLock`, 这个锁对读操作是乐观的, 即使在读操作, 依然可以进行写操作)

| | read | write |
| -- | -- | -- |
| read | 并行 | 串行 |
| write | 串行 | 串行 |

### 自定义读写分离锁

根据上面分析的读写分离的规则, 要判断是否有线程在读操作和写操作:

- 拿到读锁的时候需要判断是否有线程正在进行写操作, 如果有则不能读取, 使用`wait()`
- 拿到写锁的时候判断有线程正在进行读操作, 如果有则不能写入,使用`wait()`
- 如果存在偏向写操作的需求, 那么还需要判断是否存在无法写入的线程
- 无论是否拿到读锁和写锁都需要释放锁, 否则会造成线程的阻塞, 使用`notifyAll()`

所以编写读写分离锁代码如下:

##### 1. 编写读写锁ReadWriteLock

```java
public class ReadWriteLock {

    /**
     * 当前有多少线程进行读操作
     */
    private int readingReaders = 0;

    /**
     * 线程想读操作, 但是无法读取的线程个数
     */
    private int waitingReaders = 0;

    /**
     * 当前有多少线程在进行写操作, 肯定只有一个线程
     */
    private int writingWriters = 0;

    /**
     * 线程想写操作, 但是无法写入的线程个数
     */
    private int waitingWriters = 0;

    /**
     * 是否更偏向写操作
     */
    public boolean preferWriter = true;

    public ReadWriteLock() {
        this(true);
    }

    public ReadWriteLock(boolean preferWriter) {
        this.preferWriter = preferWriter;
    }

    public synchronized void readLock() throws InterruptedException {
        this.waitingReaders++;
        try {
            // 如果当前有线程在进行写操作, 就不能读取
            while (writingWriters > 0 || (preferWriter && waitingWriters > 0)) {
                this.wait();
            }
            //
            this.readingReaders++;
        } finally {
            this.waitingReaders--;
        }
    }

    public synchronized void readUnlock() {
        this.readingReaders--;
        this.notifyAll();
    }

    public synchronized void writeLock() throws InterruptedException {
        this.waitingWriters++;
        try {
            while (readingReaders > 0 || writingWriters > 0) {
                this.wait();
            }
            this.writingWriters++;
        } finally {
            this.waitingWriters--;
        }
    }

    public synchronized void writeUnlock() {
        this.writingWriters--;
        this.notifyAll();
    }
}
```

##### 2. 编写业务读写数据代码SharedData

```java
public class SharedData {
    private final char[] buffer;

    private final ReadWriteLock lock = new ReadWriteLock();

    public SharedData(int size) {
        this.buffer = new char[size];
        for (int i = 0; i < size; i++) {
            this.buffer[i] = '*';
        }
    }

    public char[] read() throws InterruptedException {
        try {
            lock.readLock();
            return doRead();
        } finally {
            lock.readUnlock();
        }
    }

    public void write(char c) throws InterruptedException {
        try {
            lock.writeLock();
            this.doWrite(c);
        } finally {
            lock.writeUnlock();
        }
    }

    private void doWrite(char c) {
        for (int i = 0; i < buffer.length; i++) {
            buffer[i] = c;
            slowly(10);
        }
    }

    private char[] doRead() {

        char[] newBuf = new char[buffer.length];
        for (int i = 0; i < buffer.length; i++) {
            newBuf[i] = buffer[i];
        }
        slowly(50);
        return newBuf;
    }

    private void slowly(int ms) {
        try {
            Thread.sleep(ms);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

##### 3. 编写读线程ReadWorker

```java
public class ReaderWorker extends Thread {
    private final SharedData data;

    public ReaderWorker(SharedData data) {
        this.data = data;
    }

    @Override
    public void run() {
        try {
            while (true) {
                char[] readBuff = data.read();
                System.out.println(Thread.currentThread().getName() + " reads " + String.valueOf(readBuff));
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

##### 4. 编写写线程WriteWorker

```java
public class WriterWorker extends Thread {

    private static final Random RANDOM = new Random(System.currentTimeMillis());

    private final SharedData data;

    private final String filler;

    private int index = 0;


    public WriterWorker(SharedData data, String filler) {
        this.data = data;
        this.filler = filler;
    }

    @Override
    public void run() {
        try {
            while (true) {
                char c = nextChar();
                data.write(c);
                Thread.sleep(RANDOM.nextInt(1000));
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private char nextChar() {
        char c = filler.charAt(index);
        index++;
        if (index >= filler.length()) {
            index = 0;
        }
        return c;
    }
}
```

##### 5. 测试

```java
public class ReadWriteLockClient {
    public static void main(String[] args) {
        final SharedData data = new SharedData(10);
        new ReaderWorker(data).start();
        new ReaderWorker(data).start();
        new ReaderWorker(data).start();
        new ReaderWorker(data).start();
        new ReaderWorker(data).start();

        new WriterWorker(data, "qwertyuipasdnsdn").start();
        new WriterWorker(data, "QWERTYUIPASDNSDN").start();
    }
}
```

测试结果

```txt
Thread-1 reads **********
Thread-3 reads **********
Thread-2 reads **********
Thread-0 reads **********
Thread-4 reads **********
Thread-2 reads qqqqqqqqqq
Thread-0 reads qqqqqqqqqq
Thread-3 reads qqqqqqqqqq
Thread-1 reads qqqqqqqqqq
Thread-4 reads qqqqqqqqqq
Thread-3 reads wwwwwwwwww
Thread-0 reads wwwwwwwwww
Thread-4 reads wwwwwwwwww

...

```
