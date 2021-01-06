## JVM字符串常量池StringTable

### 1. String的基本特性

下面是jdk8的String源码定义:

```java
public final class String implements Serializable, Comparable<String>, CharSequence {
    private final char[] value;

    ...
}
```

- 首先`String`类被声明为final, 也就意味着不可继承;

- `String`实现了`Serializable`接口: 表示字符串是支持序列化的;

- `String`实现了`Comparable`接口, 表示String可以比较大小;

- `String`在JDK8以及以前内部定义为`private final char value[]`字符数组的形式存储, 但是在JDK9之后改为`byte[]`。

    > JDk9之后, `String`再也不用char[] 来存储了，改成了byte[] 加上字符编码集的标识, 节约空间。**基于字符串的内存, 决定用何种编码去存储。**

jdk9的String源码:

```java
public final class String implements Serializable, Comparable<String>, CharSequence {

    @Stable
    private final byte[] value;

    ...
}
```

### 2. String的不可变性

**对于`String`的任何操作, 原来的字符串是不会变化的。**

- 当对字符串重新赋值时, 实际上是重新指定内存区域赋值, 不会覆盖原来的值;

    ```java
    String str = "Hello";

    str = "Hello1";// 这里是重新指向"Hello1"的地址, "Hello"仍然存在
    ```

- 对现有的字符串进行连接操作时, 也是重新指定内存区域赋值, 不会覆盖原来的值;

    ```java
    String str = "Hello";

    str += "1";// 这里是重新指向"Hello1"的地址, "Hello"仍然存在
    ```

- 当调用String的replace()方法时修改指定字符或字符串, 同样是重新指定内存区域赋值;

    ```java
    String str = "Hello";

    String str2 = str.replace("e", "a");// 这里是重新指向"Hallo"的地址, "Hello"仍然存在
    ```

- **通过字面量的方式给一个字符串赋值, 此时的字符串值声明在常量池中。而且字符串常量池中的字符串不允许重复。**

字符串常量池的底层是一个固定大小的`HashTable`, 类似于HashSet(底层是利用HashMap的key, key值是唯一不重复的)。所以字符串常量池中的值都是不可重复的。

如果放进常量池的字符串非常多, 就会造成Hash冲突, 从而导致链表过长, 而链表过长就会导致在插入新的字符串时性能大大下降。

可以通过`-XX:StringTableSize`设置StringTable的长度。

> 在JDK6中StringTable是固定的, 长度为1009; JDK7中, StringTable的默认值是60013; JDK8开始, StringTable可设置的最小值是1009。


### 3. String的内存分配

Java中的8中基本数据类型和String, 在运行过程中操作速度更快, 更节省内存, 都提供了一种常量池的概念。

**常量池就类似一个Java系统级别提供的缓存, 8中基本数据类型的常量池都是系统协调的, String类型的常量池比较特殊。它的主要使用方法有两种:**

- 直接使用双引号声明出来的String对象会直接存储到常量池中。例如: `String info = "abc";`

- 如果不是用双引号声明的String对象, 可以使用String提供的`intern()`方法。

Java6之前, 字符串常量池`StringTable`存放在永久代; 但是JAVA7之后, StringTable的位置移到了Java堆内。

> 为什么要调整StringTable的位置?

1. 永久代默认存储空间较小;

2. 永久代的垃圾回收频率低;

3. 所有字符串都保存在堆中, 就和其他普通对象一样, 可以在进行调优时仅需要调整堆大小即可;

```java
class Memory {
    public static void main(String[] args) {//line 1
        int i = 1;//line 2
        Object obj = new Object();//line 3
        Memory mem = new Memory();//line 4
        mem.foo(obj);//line 5
    }//line 9

    private void foo(Object param) {//line 6
        String str = param.toString();//line 7
        System.out.println(str);
    }//line 8
}
```

显示调用toString()方法在字符串常量池中生成调用者对应的字符串, 并且返回常量池地址。

![jvm_stringtable1](/image/jvm_stringtable1.png)

### 4. 字符串拼接操作

