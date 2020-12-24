## Java虚拟机栈--栈帧

> 栈帧的内部结构

- **局部变量表(Local Variables)**

- **操作数栈(Operand Stack)**

- **动态链接(Dynamic Linking)**

- 方法返回地址

- 附加信息

![jvm_stackframe1](/image/jvm_stackframe1.png)


### 2. 局部变量表

**局部变量表**的本质其实就是一个整型数组, 主要用于存储方法参数和定义在方法内的局部变量。包括基本数据类型, 对象引用地址。

**局部变量表所需的容量大小是在编译期就确定下来的。**, 所以在方法执行期间是不会改变局部变量表的大小。(可以理解为数组是固定大小的, 无法扩容)

**局部变量表中的变量只能在当前方法调用中有效。**

我们对下面的代码进行编译:

```java
public class LocalVariables {

    public void test(byte b) {
        int num = 10;
        String str = "Hello";
        System.out.println(str);
    }
}
```

> - 通过 `javap -v LocalVariables.class`命令查看字节码文件, 可以看到对应栈帧的局部变量表; 

```java
 LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      14     0     b   B
            3      11     1   num   I
            6       8     2   str   Ljava/lang/String;
```

我们对上面的字节码进行解释:

- `Start`表示变量定义的字节码指令位置, 也就是说`byte b`变量是在字节码指令0的位置定义的; 换句话说, 表示这个变量的作用域生效位置

- `Length`表示这个变量到哪个字节码指令位置 作用域结束;

    > 针对`Start`, `Length`的意思, 其实很好理解。 例如, 变量num, 它是从字节码指令地址3开始定义的, 也就是从3往后, num的作用域才开始生效, 3前面是无法调用num变量; 而整个代码的字节码指令长度是14, 所以变量num的作用域范围就是从3开始直接到14, 总共有长度11的字节码偏移量。

- `Slot`表示该变量占用的局部变量表的槽位置, 可以理解为数组的下标。 例如, num变量所在局部变量表中的位置就是1。(**要注意的是, 如果不是静态方法, 那么第一个槽位置, 也就是0所在的位置, 存储的是`this`当前对象的引用变量**)

- `Name`表示变量名称;

- `Signature` 表示变量的类型;
    - B -> byte; 
    - I -> int, 
    - L -> 表示类描述符, 表示这是一个类; LLjava/lang/String就表示类型String;
    - [ -> 数组类型; [I就表示 int[];


> - IDEA上安装jclasslib byte viewcoder插件, 也可以看到字节码文件信息

![jvm_localvariables](/image/jvm_localvariables.png)

![jvm_localvariables2](/image/jvm_localvariables2.png)

#### 2.1 变量槽Slot

局部变量都是存放在局部变量数组中, 从index0开始, 而局部变量表最基本的存储单元是Slot(变量槽)。

**32位以内的类型只占用一个`slot`, 64位的类型(long, double)占用两个slot。**

> byte, short, char, float在存储前会被转换为int, boolean也被转换为int, 0表示false, 非0表示true;

Slot存储结构如下图:

![jvm_slot1](image/jvm_slot1.png)

示例代码:

```java
public static void test1() {
    long l = 10l;
    double d = 2.5;
}
```

局部变量表为:

![jvm_slot2](image/jvm_slot2.png)

我们可以看到, long和double类型都占用两个slot, 而且会取最前的序号作为当前变量的槽。long l -> 0 (0, 1); double d -> 2 (2, 3);

JVM会为局部变量表中的每一个slot分配一个访问索引, 通过这个索引就可以访问局部变量表中指定的变量值。

**我们上面的示例代码都是静态方法, 它的slot index0的位置就是第一个变量值所在的位置, 但是如果是实例方法, index0则是this。**

实例方法:

```java
public void test2() {
    int num = 10;
    System.out.println("Hello");
}
```

局部变量表为:

![jvm_localvariables3](/image/jvm_localvariables3.png)

> **这也是为什么静态方法中不能引用this, 因为静态方法所对应的栈帧中的局部变量表不存在this。**

从上图可以看到this表示的就是当前类的实例变量, 而静态方法在类加载阶段就已经加载到内存当中了, 此时实例还没有被创建, 所以也就不存在this。

栈帧中的局部变量表中的槽位是可以重复利用的, 如果一个局部变量过了作用域, 那么之后申明的变量就可能会复用过期的局部变量的槽位, 从而达到节省资源的目的。

```java
private void test3() {
    int a = 0;
    {
        int b = 0;
        b = a+1;
    }
    //变量c使用之前以及经销毁的变量b占据的slot位置
    int c = a + 1;
}
```

上面代码对应的栈帧中局部变量表只会有3个slot, 因为变量b的作用域是

```java
{
    int b = 0;
    b = a+1;
}
```

index0: this; index1: a; 变量c重复使用了b的索引号。


### 3. 操作数栈(Operand Stack)


