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

在深入字符串拼接操作的原理之前, 我们要先了解定义字符串的原理。

一般定义字符串有两种方式:

1. 字面量方式`String s = "Hello";`

    > 使用字面量方式定义字符串, 根据字节码指令可以看到: JVM会直接在字符串常量池中创建对应的字符串值, 并返回该值在常量池中的内存地址;

2. 创建字符串对象`new String("Hello");`

    > 使用new方式创建字符串, 根据字节码指令可以看到: JVM会先在Java堆中创建String对象实例, 并设定初始值, 然后在字符串常量池中创建对应的字符串值;

    **也就是说new String(); 一般情况下会创建两个对象: 一个是在Java堆中, 一个是字符串常量池**

    ```java
    public class StringDemo {

        public static void main(String[] args) {
            String s = new String("Hello");
        }
    }
    ```
    字节码指令: 我们可以看到, 除了new指令在Java堆分配内存之外, 还是一个ldc指令, 即加载字符串常量池。

    ```java
    0 new #2 <java/lang/String> // 在Java堆中分配内存, 创建String对象
    3 dup
    4 ldc #3 <Hello> // 加载字符串常量池指令
    6 invokespecial #4 <java/lang/String.<init>>
    9 astore_1
    10 return
    ```
    
还有一种比较特殊的方式:

3. 利用StringBuilder/StringBuffer的toString方法: **要注意的是, 这个toString方法的底层也是new String(), 但是利用这个方法, 只会在Java堆中创建String对象实例, 不会再常量池中创建字符串。**

    >   这里的toString是隐式调用, 只是把两个变量进行拼接, 结果只会放到堆区。

    ```java
    @Override
    public String toString() {
        // Create a copy, don't share the array
        return new String(value, 0, count);
    }
    ```
        
    字节码指令: 并没有指令ldc。
    ```java
    0 new #80 <java/lang/String>
    3 dup
    4 aload_0
    5 getfield #234 <java/lang/StringBuilder.value>
    8 iconst_0
    9 aload_0
    10 getfield #233 <java/lang/StringBuilder.count>
    13 invokespecial #291 <java/lang/String.<init>>
    16 areturn
    ```

#### 4.1 字符串拼接: 编译器优化

我们先看下面代码:

`StringDemo2.java`:

```java
public class StringDemo2 {

    public static void main(String[] args) {
        String s1 = "a" + "b" + "c";
        String s2 = "abc";

        System.out.println(s1 == s2);
        System.out.println(s1.equals(s2));;
    }
}
```

这段代码的结果是 true, ture。

这是因为`String s1 = "a" + "b" + "c"`, 这种字符常量的直接拼接, 在编译期间就直接被优化成`String s1 = "abc"`; 而这种方式对直接在常量池中创建对应的字符串值而且只会有一个, 所以s1, s2不管是字面值, 还是内存地址都是一样的。(我们利用.class文件反编译, 就会看到s1其实被编译成了"abc")

#### 4.2 字符串拼接: 字符变量拼接

我们先看下面一段代码:

```java
public class StringDemo3 {
    public static void main(String[] args) {
        test();
    }

    public void test() {
        String s1 = "a";
        String s2 = "b";
        String s3 = "ab";

        String s4 = s1 + s2;
        System.out.println(s3 == s4);
    }
}
```

字节码指令:

```java
 0 ldc #2 <a>
 2 astore_1
 3 ldc #3 <b>
 5 astore_2
 6 ldc #4 <ab>
 8 astore_3
 9 new #5 <java/lang/StringBuilder>
12 dup
13 invokespecial #6 <java/lang/StringBuilder.<init>>
16 aload_1
17 invokevirtual #7 <java/lang/StringBuilder.append>
20 aload_2
21 invokevirtual #7 <java/lang/StringBuilder.append>
24 invokevirtual #8 <java/lang/StringBuilder.toString>
27 astore 4
29 getstatic #9 <java/lang/System.out>
32 aload_3
33 aload 4
35 if_acmpne 42 (+7)
38 iconst_1
39 goto 43 (+4)
42 iconst_0
43 invokevirtual #10 <java/io/PrintStream.println>
46 return
```

**从字节码指令可以看到, 当字符串变量进行拼接操作时, 会先创建StringBuilder对象, 利用StringBuilder的append方法通过字符数组的方式连接, 并且最终通过toString方法返回字符串。**

所以 `s1 + s2`的执行细节如下:

1. StringBuilder s = new StringBuilder(); **这个s变量是不存在, 这里为了演示临时定义的**

2. s.append("a");

3. s.append("b");

4. s.toString() --> 相当于new String("ab"); 只是字符串没有在常量池创建

由于StringBuilder的toString方法没有在常量池创建字符串值(但是由于s3先定义的, 所以"ab"已经在常量池存在了), 所以最终返回的地址是Java堆中的String对象地址, 而s3指向的是常量池中的地址, 这两个地址很明显是不同的, 所以 s3 != s4。

#### 4.1 字符串拼接: final常量拼接

由于被final修饰的常量, 会在类加载过程中就已经赋值完了。所以被final修饰的字符串, 在拼接过程中也会编译器优化, 并创建到常量池中。

```java
public void test(){
    final String s1 = "a";
    final String s2 = "b";
    String s3 = "ab";
    String s4 = s1 + s2;
    System.out.println(s3 == s4);//true
}
```

上面代码的结果是true。

s1, s2被final修饰, 所以值在类加载期间已经赋值结束了。所以s1 + s2会进行编译期优化, 相当于"a" + "b" ==> "ab"。

上面代码的class文件经过反编译的结果, s4 = s1 + s2已经被优化为 ==> s4 = "ab";

### 5. 拼接操作与append的效率

结论: **append效率要比字符串直接拼接高很多!**

我们先看下面的测试代码:

```java
public class StringDemo4 {

    public static void main(String[] args) {
        StringDemo4 stringDemo4 = new StringDemo4();
        stringDemo4.test();
    }

    public void test() {
        long start = System.currentTimeMillis();

//        method1(100000);//4933
        method2(100000);// 6
        long end = System.currentTimeMillis();

        System.out.println("花费时间为:" + (end - start));
    }

    public void method1(int highLevel) {
        String str = "";
        for (int i = 0; i < highLevel; i++) {
            str = str + "a";
        }
    }

    public void method2(int highLevel) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < highLevel; i++) {
            sb.append("a");
        }
    }
}
```

上面代码的运行结果很明显。append方法更加高效。

原因: 

1. 字符串直接拼接每次都会创建一个StringBuilder对象, 最后返回toString时还会创建一个String对象。这最终会导致每次创建对象都需要时间, 如果需要进行GC, 还需要花费额外的时间; 而且内存占用更多。**所以字符串直接拼接空间和时间上效率都很低。**

2. StringBuilder的append()方法, 只会创建一次StringBuilder对象。

**代码优化:**

- 对于上面的代码, 如果在需要大量字符串拼接操作的场景下, 必须要使用StringBuilder/StringBuffer来处理;

- 在实际开发中, 如果能基本确定这个字符串的长度不高于某个最大值的情况下, 在构造StringBuilder时, 需要指定初始容器大小(默认大小是16)。`StringBuilder s = new StringBuilder(highLevel);`

    > 因为String底层还是通过数组的结构在存储字符串的, 而在拼接过程中, 如果超过了数组的大小,那么就会进行数组的扩容, 并进行数组的拷贝(在内存找到一块新的连续的内存地址, 并且把原有的字符, 复制到新的容器中, 原有的内存空间变成垃圾需要回收) 这样是浪费时间的。

### 6. intern()的使用

`intern()`方法是String类中的本地方法:

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {

    ...

    public native String intern();

    ...
}
```

`intern()`方法是用途, 就是把字符串存放在常量池中, 如果该字符串已经存在了, 那么就直接返回常量池中的字符串; 否在就存在常量池中再返回。

但是这个方法的使用会根据jdk版本的不同, 处理的方式不同:

1. jdk1.6中:

    - 如果常量池中已存在, 就不会放入; 直接返回常量池中的字符串地址;

    - 如果不存在, 会直接把这个字符串存入常量池, 并返回常量池中的字符串地址。

2. 在jdk1.7开始:

    - 如果常量池中已存在, 就不会放入; 直接返回常量池中的字符串地址;

    - **如果常量池不存在该字符串, 而在Java堆区中存在这个字符串的实例, 那么就会把这个对象的实例引用地址复制一份, 存入常量池中, 并返回这个对象实例地址;**


根据上面的解释, 如果`String myInfo = new String("Hello).intern();` 那么返回的结果所指向的地址, 一定是常量池中的字符串地址。

所以 `new String("Hello).intern() == "Hello"`一定是true。

#### 6.1 String.intern()的面试题

下面是一道跟容易出错的面试题, 它的关键是在于`intern`方法以及String底层知识的理解。

```java
public class StringDemo5 {

    public static void main(String[] args) {
       method1();
       method2();
    }
    
    public static void method1() {
        String s = new String("1"); // 返回的是Java堆中对象实例地址, 但是常量池中也存入了字符串
        s.intern(); // 调用此方法之前, 常量池已经存在了"1", 所以这个方法其实没有用
        String s1 = "1";
        System.out.println(s == s1);
    }
    
    public static void method2() {
        String s3 = new String("1") + new String("1");
        s3.intern();
        
        String s4 = "11";
        System.out.println(s3 == s4);
    }
}
```

- method1: 结果肯定是false。 s是指向堆空间的"1"的内存地址, s1是指向常量池中已存在的"1"的内存地址;

- method2: 首先new String("1")时, 已经在常量池中存入"1", 但是进行字符串拼接操作时, 使用的是StringBuilder的append方法, 并且toString返回字符串时, 没有把"11"存入常量池中。
也就是常量池中并不存在"11"; 那么此时调用`intern`方法时要注意的是**jdk6和jdk7/8的处理方式不同**。所以jdk6返回false, 而jdk7之后返回true。

    > jdk7版本, 此时s4指向的虽然是常量池的地址, 但是常量池中存的实际上是指向堆区的内存地址, 所以实际上s3, s4指向的地址是一样的。

    ![jvm_stringtable2](/image/jvm_stringtable2.png)

    ![jvm_stringtable3](/image/jvm_stringtable3.png)

### 7. intern()效率测试

当需要大量存储字符串时, 调用intern()方法, 会明显降低内存的大小。

例如下面的代码:

```java
public class StringDemo6 {

    static final int MAX_COUNT = 1000 * 1000;
    static final String[] arr = new String[MAX_COUNT];

    public static void main(String[] args) {
        Integer[] data = new Integer[]{1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

        long start = System.currentTimeMillis();

        for (int i = 0; i < MAX_COUNT; i++) {
            arr[i] = new String(String.valueOf(data[i % data.length]));
        }

        long end = System.currentTimeMillis();
        System.out.println("花费的时间为：" + (end - start));

        try {
            Thread.sleep(1000000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.gc();
    }
}
```

当我们执行这段代码, 并启动jprofile时, 查看Live Memory时, 可以发现存在大量的String实例占用内存:

![jvm_stringtable4](/image/jvm_stringtable4.png)

如果我们使用intern()方法时, 内存情况如下:

```java
public class StringDemo6 {

    static final int MAX_COUNT = 1000 * 1000;
    static final String[] arr = new String[MAX_COUNT];

    public static void main(String[] args) {
        Integer[] data = new Integer[]{1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

        long start = System.currentTimeMillis();

        for (int i = 0; i < MAX_COUNT; i++) {
            arr[i] = new String(String.valueOf(data[i % data.length])).intern();
        }

        long end = System.currentTimeMillis();
        System.out.println("花费的时间为：" + (end - start));

        try {
            Thread.sleep(1000000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.gc();
    }
}
```

![jvm_stringtable5](/image/jvm_stringtable5.png)

比较这两者的数据, 可以看到使用intern()方法时内存很明显小很多。

> 结论：对于程序中大量存在存在的字符串，尤其其中存在很多重复字符串时，使用intern()可以节省内存空间。
