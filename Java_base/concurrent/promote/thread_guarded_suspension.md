## Guarded Suspension模式

`Guarded Suspension`翻译过来叫做 **警戒等待**。这是一种中比较简单的设计模式, 当一个线程在处理一个不可放弃的任务时, 此时又来一个新的任务, 就让新的任务放入队列, 当处理完当前任务时, 再去处理队列中的任务。

### 模式案例

我们利用一种简单的消息队列的模型, 来实现`Guarded Suspension`模式。客户端不断的发送消息请求, 并将请求缓存在队列中, 然后发送给服务端线程进行处理。

##### 1. 定义请求模型

```java
public class Request {
    private final String value;

    public Request(String value) {
        this.value = value;
    }

    public String getValue() {
        return value;
    }
}
```

##### 2. 定义消息请求队列

```java
public class RequestQueue {
    private final LinkedList<Request> queue = new LinkedList<>();

    public Request getRequest() {
        synchronized (queue) {
            // 当队列小于等于0时, 也就是没有任务, 那就处于等待状态
            while (queue.size() <= 0) {
                try {
                    queue.wait();
                } catch (InterruptedException e) {
                    return null;
                }
            }
            return queue.removeFirst();
        }
    }

    public void putRequest(Request request) {
        synchronized (queue) {
            queue.addLast(request);
            queue.notifyAll();
        }
    }
}
```

##### 3. 定义客户端线程, 不断的去发送request, 加入队列

```java
public class ThreadClient extends Thread {

    private final RequestQueue queue;

    private final Random random;

    private final String value;

    public ThreadClient(RequestQueue queue, String value) {
        this.queue = queue;
        this.value = value;
        random = new Random(System.currentTimeMillis());
    }

    @Override
    public void run() {
        for (int i = 0; i < 10; i++) {
            System.out.println("Client -> request " + value);
            queue.putRequest(new Request(value));
            try {
                Thread.sleep(random.nextInt(1000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

##### 4. 定义服务端线程, 不断的处理request, 每次处理都移除队列第一个request

```java
public class ThreadServer extends Thread {

    private final RequestQueue queue;

    private final Random random;

    private volatile boolean flag = true;


    public ThreadServer(RequestQueue queue) {
        this.queue = queue;
        random = new Random(System.currentTimeMillis());
    }

    @Override
    public void run() {
        while (flag) {
            Request request = queue.getRequest();
            if (request == null) {
                System.out.println("Received th empty request.");
                continue;
            }
            System.out.println("Server -> " + request.getValue());
            try {
                Thread.sleep(random.nextInt(1000));
            } catch (InterruptedException e) {
                return;
            }
        }
    }

    public void close() {
        this.flag = false;
        this.interrupt();
    }
}
```

##### 5. 测试

```java
public class SuspensionClient {
    public static void main(String[] args) throws InterruptedException {
        final RequestQueue queue = new RequestQueue();
        new ThreadClient(queue, "YK").start();
        ThreadServer threadServer = new ThreadServer(queue);
        threadServer.start();

        Thread.sleep(100000);
        threadServer.close();
    }
}
```

结果:

```txt
Client -> request YK
Server -> YK
Client -> request YK
Server -> YK
Client -> request YK
Server -> YK
Client -> request YK
Server -> YK
Client -> request YK
Server -> YK
Client -> request YK
Client -> request YK
Client -> request YK
Server -> YK
Client -> request YK
Client -> request YK
Server -> YK
Server -> YK
Server -> YK
Server -> YK
```