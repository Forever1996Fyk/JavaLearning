## JVM 类加载器

类加载器可以说是JVM的基础, 也是重点。它也是类加载子系统中的加载(loading)环节重要部分。这一块, 我们先简单的介绍一下, 后面还会详细介绍。

### 1. 类加载器分类

JVM支持两种类型的加载器, 分别是

- **引导类加载器C/C++实现的: `BootStrap ClassLoader`**

- **自定义类加载器由Java实现**

> 要注意, 只要是所有派生与抽象类`ClassLoader`的类加载器都属于自定义类加载器。

![jvm_classloader1](/image/jvm_classloader1.png)

上图中加载器的关系是包含关系, 而不是继承关系。

一般情况下, 我们常见的类加载器是: 引导类加载器BootStrapClassLoader, 自定义类加载器(ExtensionClassLoader, SystemClassLoader, User-Defined ClassLoader)

#### 1.1 自定义类与核心类库的加载器

1. 对于用户自定义类来说: 使用系统类加载器Sysytem Class Loader中的`AppClassLoader`进行加载;

2. Java核心类库都是使用引导类加载器`BootStrapClassLoader`加载的。

一般自定义类是获取不到引导类加载器的。如下代码:

```java
/**
 * ClassLoader加载
 */
public class ClassLoaderTest {
    public static void main(String[] args) {
        //获取系统类加载器
        ClassLoader systemClassLoader = ClassLoader.getSystemClassLoader();
        System.out.println(systemClassLoader);//sun.misc.Launcher$AppClassLoader@18b4aac2

        //获取其上层  扩展类加载器
        ClassLoader extClassLoader = systemClassLoader.getParent();
        System.out.println(extClassLoader);//sun.misc.Launcher$ExtClassLoader@610455d6

        //获取其上层 获取不到引导类加载器
        ClassLoader bootStrapClassLoader = extClassLoader.getParent();
        System.out.println(bootStrapClassLoader);//null

        //对于用户自定义类来说：使用系统类加载器进行加载
        ClassLoader classLoader = ClassLoaderTest.class.getClassLoader();
        System.out.println(classLoader);//sun.misc.Launcher$AppClassLoader@18b4aac2

        //String 类使用引导类加载器进行加载的  -->java核心类库都是使用引导类加载器加载的
        ClassLoader classLoader1 = String.class.getClassLoader();
        System.out.println(classLoader1);//null获取不到间接证明了String 类使用引导类加载器进行加载的

    }
}
```

#### 1.2 虚拟机自带的加载器

**引导类加载器, 或者 启动类加载器, `BootStrapClassLoader`**

1. 这个类加载使用C/C++语言实现的, 嵌套在JVM内部;

2. 它用来加载Java核心类库(`JAVA_HOME/jre/lib/rt.jar、resources.jar`或者`sun.boot.class.path`路径下的内容), 用于提供JVM自身需要的类;

3. 不继承`java.lang.ClassLoader`, 没有父加载器;

4. **记载扩展类和应用程序类加载器, 并指定为它们的父加载器, 即ClassLoader。(注意, 这里的父加载器并不是指继承的父类)**

5. `BootStrap`直接在包名为java, javax, sun等开头的类。

**扩展类加载器(`Extension ClassLoader`)**

1. Java语言编写, 由`sun.misc.Launcher$ExtClassLoader`实现;

2. 派生于ClassLoader类;

3. 从`java.ext.dirs`系统属性所指定的目录中加载类库, 或从JDK安装目录的`jre/lib/ext`子目录(扩展目录)下加载类库。**如果用户创建的jar放在此目录下, 也会用扩展类加载器自动加载。**

**应用类加载器(`AppClassLoader`)**

1. Java语言编写, 由`sun.misc.Launcher$AppClassLoader`实现;

2. 派生于ClassLoader类;

3. 负责记载环境变量classpath或系统属性java.class.path指定路径下的类库;

4. **<font color='red'>`AppClassLoader`是程序中默认的类加载器</font>**, 一般来说,Java应用的类都是由它完成加载;

5. 通过`ClassLoader#getSystemClassLoader()`方法可以获取到该类加载器。

```java
/**
 * 虚拟机自带加载器
 */
public class ClassLoaderTest1 {
    public static void main(String[] args) {
        System.out.println("********启动类加载器*********");
        URL[] urls = sun.misc.Launcher.getBootstrapClassPath().getURLs();
        //获取BootStrapClassLoader能够加载的api路径
        for (URL e:urls){
            System.out.println(e.toExternalForm());
        }

        //从上面的路径中随意选择一个类 看看他的类加载器是什么
        //Provider位于 /jdk1.8.0_171.jdk/Contents/Home/jre/lib/jsse.jar 下，引导类加载器加载它
        ClassLoader classLoader = Provider.class.getClassLoader();
        System.out.println(classLoader);//null

        System.out.println("********拓展类加载器********");
        String extDirs = System.getProperty("java.ext.dirs");
        for (String path : extDirs.split(";")){
            System.out.println(path);
        }
        //从上面的路径中随意选择一个类 看看他的类加载器是什么:拓展类加载器
        ClassLoader classLoader1 = CurveDB.class.getClassLoader();
        System.out.println(classLoader1);//sun.misc.Launcher$ExtClassLoader@4dc63996
    }
}
```

> 我们一般使用的核心类库有, `java.time`, `java.util`, `java.nio`, `java.lang`, `java.text`, `java.sql`, `java.math`等等都是在`rt.jar`包下, 也就是说这些都是引导类加载器`BootStrapClassLoader`加载。

### 2. 双亲委派机制

Java虚拟机对class文件采用的时按需加载的方式, 也就是说当需要使用该类是才会将他的class文件加载到内存生成class对象。而加载某个类的class文件时, java虚拟机采用的是双亲委派模式, 即把加载请求交给父加载器处理。

#### 2.1 双亲委派机制工作原理

1. **如果一个类加载器收到了类加载的请求, 它并不会先去加载, 而是把这个请求委托给父加载器执行;**

2. **如果父加载器还存在父加载器, 则进一步向上委托, 依次递归, 请求最终到达引导类加载器;**

3. **如果父加载器可以完成类加载任务, 就成功返回, 否则再由子加载器尝试加载。**

![jvm_classloader2](/image/jvm_classloader2.png)

我们利用一个代码示例, 来说明:

我们建一个java.lang的包, 并且创建了一个自定义String类。

```java
package java.lang;

public class String {
    static {
        System.out.println("自定义String类的静态代码块");
    }

    public static void main(String[] args) {
        System.out.println("Hello String");
    }
}
```

当我们执行自定义String类时, 就会报错。

这是因为虽然我们自定义了一个java.lang包下的String尝试覆盖核心类库中的String, 但是由于双亲委派机制, `AppClassLoader`会委托到`BootStrapClassLoader`引导类加载器会加载Java核心类库中的String类(BootStrap引导类加载器只加载包名为java, javax, sun等开头的类), 而核心类库中的String类并没有main方法, 所以报错。 


#### 2.2 双亲委派机制的优势

1. 避免类的重复加载, 如上面的自定义String的代码; 又称为**沙箱安全机制**

2. 保护程序安全, 防止核心API被修改;

    > * 启动类加载器可以抢在扩展类装载器之前去装载类，而扩展类装载器可以抢在类路径加载器之前去装载那个类，类路径加载器又可以抢在自定义类加载器之前去加载它。<br/>
    > * 所以Java虚拟机先从最可信的Java核心API查找类型，这是为了防止不可靠的类扮演被信任的类。<br/>
    > * 试想一下，网络上有个名叫java.lang.Integer的类，它是某个黑客为了想混进java.lang包所起的名字，实际上里面含有恶意代码，但是这种 伎俩在双亲模式加载体系结构下是行不通的，因为网络类加载器在加载它的时候，它首先调用双亲类加载器，这样一直向上委托，直到启动类加载器，而启动类加载 器在核心Java API里发现了这个名字的类，所以它就直接加载Java核心API的java.lang.Integer类，然后将这个类返回，所以自始自终网络上的 java.lang.Integer的类是不会被加载的。

3. 保证核心API包的访问权限, 也就是说核心类库的下的包访问需要权限, 而且也是阻止我们用相同的包名。

    > * 我创建一个`java.lang.Virus`类, 也是不会被加载的。因为要取得访问和修改java.lang包中的权限, java.lang.Virus和java.lang下其他类必须是属于同一个运行时包。
    > * 运行时包指的是同一个类加载器加载的, 属于同一个包的类的集合。`java.lang.Virus`是自定义类加载器加载的, 而其他`java.lang`包下的类是引导类加载器加载的, 它们根本不是同一个类加载器, 所以没有权限加载。

#### 2.3 双亲委派机制在SPI中的应用

如果某个应用程序有双亲委派机制找到引导类加载器, 首先调用rt.jar包中的SPI核心, 但由于SPI核心当中有各种各种的接口需要被实现, 那么此时我们需要反向委托, 利用系统加载器会加载具体实现的jar。

![jvm_classloader3](/image/jvm_classloader3.png)

#### 2.4 JVM中两个Class对象是否为同一个类

在JVM中表示两个class对象是否为同一个类存在两个必要条件:

1. 类的完成类名必须一致, 包括所属包名;

2. 加载这个类的ClassLoader(指ClassLoader实例对象)必须相同, 是引导类加载器, 还是自定义类加载器。

