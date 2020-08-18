## <center>Dubbo 负载均衡</center>

Dubbo需要对服务消费者的调用请求进行分配, 避免少数服务提供者负载过大。服务提供者负载过大会导致部分请求超时。

Dubbo提供了4中负载均衡的实现, 分别是

- 基于权重随机算法的`RandomLoadBalance`
- 基于最少活跃调用书算法的`LeastActiveLoadBalacne`
- 基于hash一致性的`ConsistentHashLoadBalance`
- 基于加权轮询算法的`RoundRobinLoadBalance`

### 1. RandomLoadBalance

`RandomLoadBalance`是加权随机算法的具体实现。例如: 有一组服务器`servers=[A, B, C]`, 他们对应的权重为`weights=[5,3,2]`, 权重总和为10。那么[0, 5)区间属于服务器A, [5, 8)区间属于服务器B, [8, 10)区间属于服务器C。然后通过生成[0, 10)的随机数, 计算这个随机数会落到哪个区间上。那么那个服务器就会被调用。权重越大的机器, 越容易被调用。

`RandomLoadBalance`是一个比较简单的负载均衡算法, 它的缺点就是如果调用次数比较少, Random产生的随机树可能会比较集中, 此时多数请求会落到同一服务器上。但是`RandomLoadBalance`是一个简单高效的负载均衡实现, 所以Dubbo选择它作为缺省实现。

```java
public class RandomLoadBalance extends AbstractLoadBalance {

    public static final String NAME = "random";

    private final Random random = new Random();

    @Override
    protected <T> Invoker<T> doSelect(List<Invoker<T>> invokers, URL url, Invocation invocation) {
        int length = invokers.size();
        int totalWeight = 0;
        boolean sameWeight = true;
        // 下面这个循环有两个作用，第一是计算总权重 totalWeight，
        // 第二是检测每个服务提供者的权重是否相同
        for (int i = 0; i < length; i++) {
            int weight = getWeight(invokers.get(i), invocation);
            // 累加权重
            totalWeight += weight;
            // 检测当前服务提供者的权重与上一个服务提供者的权重是否相同，
            // 不相同的话，则将 sameWeight 置为 false。
            if (sameWeight && i > 0
                    && weight != getWeight(invokers.get(i - 1), invocation)) {
                sameWeight = false;
            }
        }
        
        // 下面的 if 分支主要用于获取随机数，并计算随机数落在哪个区间上
        if (totalWeight > 0 && !sameWeight) {
            // 随机获取一个 [0, totalWeight) 区间内的数字
            int offset = random.nextInt(totalWeight);
            // 循环让 offset 数减去服务提供者权重值，当 offset 小于0时，返回相应的 Invoker。
            // 举例说明一下，我们有 servers = [A, B, C]，weights = [5, 3, 2]，offset = 7。
            // 第一次循环，offset - 5 = 2 > 0，即 offset > 5，
            // 表明其不会落在服务器 A 对应的区间上。
            // 第二次循环，offset - 3 = -1 < 0，即 5 < offset < 8，
            // 表明其会落在服务器 B 对应的区间上
            for (int i = 0; i < length; i++) {
                // 让随机值 offset 减去权重值
                offset -= getWeight(invokers.get(i), invocation);
                if (offset < 0) {
                    // 返回相应的 Invoker
                    return invokers.get(i);
                }
            }
        }
        
        // 如果所有服务提供者权重值相同，此时直接随机返回一个即可
        return invokers.get(random.nextInt(length));
    }
}
```

### 2 LeastActiveLoadBalance

最小活跃数负载均衡, 即活跃调用数越小, 表明该服务提供者效率越高, 单位时间内可处理更多请求, 此时应优先将请求分配给该服务提供者。 

最小活跃数负载均衡的基本思想: **初始, 所有服务提供者活跃数均为0, 每收到一个请求, 活跃数加1, 完成请求后则活跃数减1.在服务运行一段时间后, 性能好的服务提供者处理请求速度更快, 因此活跃数下降的越快, 这样的服务提供者能够优先获取新的服务请求。**

`LeastActiveLoadBalance`还引入的权重值 即加权最小活跃数算法。也就数说, 某一时刻服务提供者集群的活跃数相同, 就会根据权重去分配, 权重越大, 获取新请求的概率越大。如果权重也相同, 就随机选一个服务。

### 3. ConsistentHashLoadBalance

有机会在记录, 有点复杂。

### 4. RoundRobinLoadBalance

Dubbo加权轮询负载均衡的实现————`RoundRobinLoadBalance`。

轮询是指将请求轮流分配给每台服务器, 是一种无状态负载均衡算法, 适用于每台服务器相近的场景下。但实际情况下, 我们并不能保证每台服务器性能相近, 如果将等量的请求分配给性能较差的服务器, 那是不合理的。所以在轮询的过程中进行加权, 这样每台服务器能够得到的请求数比例, 接近或等于它们的权重比。

例如: 服务器A, B, C权重比为5:2:1。那么在8次请求中, 服务器A将收到5此请求, B会受到2次请求, C会受到1次请求。