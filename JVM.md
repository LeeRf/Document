

## 1、类加载机制深度解析



### 1、类加载机制详解



#### 1、类加载机制的流程



类加载机制外层大体流程图如下（类加载机制很多代码是 C++ 代码实现的）

![image-20210616101923365](JVM.assets/image-20210616101923365.png)

>   其中的 loadClass 的类加载过程有如下几步：
>
>   加载 >> 验证 >> 准备 >> 解析 >> 初始化 >> 使用 >> 卸载



##### 1、加载：



>   **在磁盘上查找并通过 IO 读入 Java 的字节码文件、使用到类时才会被加载、例如调用的 main() 方法、new 对象等、在加载阶段会在内存中成一个代表这个类的 java.lang.Class 对象、作为方法区这个类的各种数据的访问入口**



一般来说加载也分为以下几步：

-   通过一个类的全限定名获取此类的二进制字节流
-   将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构
-   在内存中生成一个代表这个类的java.lang.Class对象，作为方法区这个类的各种数据的访问入口

创建名字为 C 的类，如果 C 不是数组类型，那么它就可以通过类加载器加载 C 的二进制表示（即Class文件）。如果是数组，则是通过 Java 虚拟机创建，虚拟机递归地采用上面提到的加载过程不断加载数组的组件

加载阶段与连接阶段的部分内容是交叉进行的，如：一部分字节码文件格式验证动作。加载阶段尚未完成，连接阶段可能已经开始了



##### 2、验证：



>   **校验字节码文件的正确性**



验证作为链接的第一步，用于确保类或接口的二进制表示结构上是正确的，从而确保字节流包含的信息对虚拟机来说是安全的。Java虚拟机规范中关于验证阶段的规则也是在不断增加的，但大体上会完成下面4个验证动作



###### 1、文件格式校验



>   主要验证字节流是否符合Class文件格式规范，并且能被当前版本的虚拟机处理

-   是否以魔数`0xCAFEBABE`开头
-   主次版本号是否在当前虚拟机处理范围之内
-   常量池的常量是否有不被支持的类型 (检查常量 tag 标志)
-   指向常量的各种索引值中是否有指向不存在的常量或不符合类型的常量
-   CONSTANT_Utf8_info 型的常量中是否有不符合 UTF8 编码的数据
-   Class 文件中各个部分及文件本身是否有被删除的或者附加的其他信息

这个阶段是基于二进制字节流进行的，只有通过了这个阶段的验证，字节流才会流入方法区中进行存储，后面3个阶段全是基于方法区的存储结构进行的，不会再直接操作字节流



###### 2、元数据校验



>   主要对字节码描述的信息进行语义分析，以保证其提供的信息符合 Java 语言规范的要求

主要验证点：

-   该类是否有父类（只有 Object 对象没有父类，其余都有）
-   该类是否继承了不允许被继承的类（被 final 修饰的类）
-   如果这个类不是抽象类，是否实现了其父类或接口之中要求实现的所有方法
-   类中的字段、方法是否与父类产生矛盾（例如覆盖了父类的 final 字段，出现不符合规则的方法重载，例如方法参数都一致，但是返回值类型却不同）



###### 3、字节码校验



>   这一阶段目的主要目的是确定程序语义是合法的、符合逻辑的。这个阶段主要对类的字节码进行校验分析，保证该类的方法不会在运行时做出危害虚拟机安全的事

-   保证任意时刻操作数栈的数据类型与指令代码序列都能配合工作，例如不会出险操作数栈上 int 类型的数据使用时按 long 类型加载进本地变量表中
-   保证跳转指令不会跳转到方法体以外的字节码指令上
-   保证方法体内的类型转化是有效的，可以把一个子类对象赋值给父类数据结构，这是安全的，而不能把父类赋值给子类甚至与它无关系的数据类型，这是危险和不合法的



###### 4、符号引用校验



>   这一阶段用来将符号引用转换为直接引用的时候，这个**转化将在解析阶段中发生**，符号引用验证可以看做是类对自身以外（常量池中各种符号引用）的信息进行匹配性校验，通常需要校验以下内容

-   符号引用中通过字符串描述的全限定名是否找到对应的类
-   在指定类中是否存在符合方法的字段描述符以及简单名称所描述的方法和字段
-   符号引用中的类、方法、字段的访问性（private、public、protected、default）是否可被当前类访问



符号引用验证的目的是确保解析动作能够正常执行，如果无法通过符号引用验证，那么将会抛出一个java.lang.IncompatibleClassChangeError 异常的子类，如 java.lang.IllegalAccessError、java.lang.NoSuchFieldError、java.lang.NoSuchMethodError 等

验证阶段非常重要，但不一定必要，如果所有代码极影被反复使用和验证过，那么可以通过虚拟机参数`-Xverify: none`来关闭验证，加速类加载时间



##### 3、准备：



>   **准备阶段的任务是为类或者接口的静态字段分配空间，并且默认初始化这些字段（final修饰的常量直接赋值）**

这些变量所使用的内存都将在方法区分配、实例变量会在对象实例化的时候跟对象一起在 java 堆中分配、这里的初始值指的是通常情况下的零值。假设一个类变量定义为

```java
public static int a = 123;
```

数据类型 零值   int - 0、long - 0L、short - (short)0、char - '\u0000'、byte - (byte)0、boolean - false、float - 0.0f、double - 0.0d、reference - null

那么变量 a 初始化的值是 0 而不是 123。如果变量同时是 final 类型，那么准备阶段就会被赋值为123，不必等到初始化阶段再赋值、**这个过程也解释了Java为什么定义一个变量不赋值，就有默认值，就是在这个阶段分配初始值的**



##### 4、解析：



>   **解析阶段是将虚拟机常量池内的 符号引用 替换为 直接引用 的过程、解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符7类符号进行**
>
>   符号引用就是 Class 文件中的 CONSTANT_Class_info**、 **CONSTANT_Fieldref_info**、**CONSTANT_Methodref_info 等类型的常量

下面我们看 符号引用 和 直接引用 的定义

**符号引用（Symbolic References）**：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要可以唯一定位到目标即可。符号引用于内存布局无关，所以所引用的对象不一定需要已经加载到内存中。各种虚拟机实现的内存布局可以不同，但是接受的符号引用必须是一致的，因为符号引用的字面量形式已经明确定义在 Class 文件格式中

**直接引用（Direct References）**：直接引用时直接指向目标的指针、相对偏移量或是一个能间接定位到目标的句柄。直接引用和虚拟机实现的内存布局相关，同一个符号引用在不同虚拟机上翻译出来的直接引用一般不会相同。如果有了直接引用，那么它一定已经存在于内存中了

以下Java虚拟机指令会将符号引用指向运行时常量池，执行任意一条指令都需要对它的符号引用进行解析

![image-20210616110733514](JVM.assets/image-20210616110733514.png)

对同一个符号进行多次解析请求是很常见的，除了 invokedynamic 指令以外，虚拟机基本都会对第一次解析的结果进行缓存，后面再遇到时，直接引用，从而避免解析动作重复。

对于 invokedynamic 指令，上面规则不成立。当遇到前面已经由 invokedynamic 指令触发过解析的符号引用时，并不意味着这个解析结果对于其他 invokedynamic 指令同样生效。这是由 invokedynamic 指令的语义决定的，它本来就是用于动态语言支持的，也就是必须等到程序实际运行这条指令的时候，解析动作才会执行。其它的命令都是“静态”的，可以再刚刚完成记载阶段，还没有开始执行代码时就解析



###### 1、类或接口的解析



>   假设Java虚拟机在类D的方法体中引用了类N或者接口C，那么会执行下面步骤：

1.  如果 C 不是数组类型，D 的定义类加载器被用来创建类 N 或者接口 C、加载过程中出现任何异常，可以被认为是类和接口解析失败
2.  如果 C 是数组类型，并且它的元素类型是引用类型。那么表示元素类型的类或接口的符号引用会通过递归调用来解析
3.  检查 C 的访问权限，如果 D 对 C 没有访问权限，则会抛出`java.lang.IllegalAccessError`异常



###### 2、字段解析



对字段表内 class_index 项中索引的 CONSTANT_Class_info 符号引用进行解析，也就是字段所属的类或接口的符号引用，如果解析这个类或符号引用的过程中出现任何异常，都会导致字段符号引用解析的失败。如果解析成功，这个字段对应的类或接口用 C 表示，接下来沿着 A 和它的父类/父接口寻找是否有这个字段，如果有会进行权限验证，如果不具备权限则抛出异常。如果这个过程不出错，则会在找到符合字段的时候返回这个字段的直接饮用，查找结束



###### 3、类 (静态) 方法解析



>   类方法解析首先也要首先解析出类方法表 class_index 项中索引的方法所属的类或接口的符号引用，解析成功用 C 表示

1.  类方法和接口方法符号引用的常量类型定义是分开的，如果在类方法表中索引类是个接口，直接抛出异常
2.  如果通过了第一步，在类 C 中查找是否有简单名称和描述符都与目标匹配的方法，有则返回这个方法的直接引用，查找结束
3.  否则在类的父类递归查找是否有这个方法，有则返回直接引用，查找结束
4.  否则在类的接口列表和父接口递归查找，如果存在匹配的方法，说明类 C 是一个抽象类，查找结束，抛出异常
5.  否则宣告查找失败，抛出异常

最后如果查找成功返回了直接引用，还要对这个方法进行权限验证，如果不具备权限，则会抛出异常



###### 4、接口方法解析



>   接口方法需要先解析出接口方法表的 class_index 项中索引的方法所属的类或接口的符号引用

1.  如果发现 class_index 中的索引 C 是个类而不是接口，直接抛出异常
2.  否则在接口 C 中查找是否有描述符和名称都匹配的方法，有则返回方法的直接引用，查找结束
3.  否则在其父接口中递归查找，匹配就返回方法的直接引用，查找结束
4.  否则宣告方法查找失败



##### 5、初始化：



>   **对类的静态变量初始化为指定的值、执行静态代码块**

类初始化是类加载过程的最后一步。前面的类加载过程中，除了加载阶段可以自定义类加载器干预之外，其余动作完全由虚拟机主导。到了初始化阶段，才真正开始执行 java 代码

我们知道，在前面的准备阶段，已经对类变量分配过内存并设置初始值。在初始化阶段，则是为类变量或其它资源设置程序中声明的值。注意这里仍然是**类变量**，不包括实例变量。**或者明确的说，这一阶段，是执行 static 关键字修饰的变量或代码块**。本质上，初始化是执行类构造器 <client>方法的过程

>   `<clinit>()`方法是由编译器 自动收集类中的所有类变量的赋值动作 和 静态语句块 ( static 语句块) 中的语句合并生成的，**编译器收集的顺序是由语句在源文件中出现的顺序决定的，静态语句块中只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块中可以赋值，但是不能访问**
>
>   ```java
>   class Test {
>       static {
>           num = 10;  //可以赋值
>           //编译器会提示“非法向前引用”
>           System.out.print(num);
>       }
>       static int num = 5;
>   }
>   ```

此外平时可能会遇到这种问题：如下代码

```java
public class Loader {

    private static Loader loader = new Loader();

    public static int a;
    public static int b = 0;

    private Loader() {
        a++;
        b++;
    }

    public static Loader getInstance() {
        return loader;
    }

    public static void main(String[] args) {
        Loader instance = Loader.getInstance();

        System.out.println(" a = " + Loader.a);
        System.out.println(" b = " + Loader.b);
    }
}
```

输出结果是：

```java
a = 1
b = 0
```

可能有人问为什么，其实把类加载的过程逻辑理清楚，也不是问题。我们知道在类加载的准备阶段会给类变量分配内存和赋初始值。在外部调用 Client.getInstance() 时，因为之前类没有被加载过，会引发类加载，到了准备阶段就会给类变量赋初始值。赋值顺序同一个类中是按声明的顺序，也就是

```java
loader = null；
a = 0;
b = 0;
```

然后解析完开始初始化，按程序声明的值给类变量赋值。首先执行 clinet = new Client() 其实关键就是这里 new 的过程会调用构造函数，调用完后

```java
a = 1;
b = 1;
```

接着继续初始化，a 只是声明没有赋值，所以没有任何操作，b 声明且赋值为0，所以初始化完成后

```java
a = 1;
b = 0;
```



![image-20210616102331562](JVM.assets/image-20210616102331562.png)

##### 6、小结：



-   **类加载到方法区后主要包含**： 
    -   运行时常量池、类型信息、字段信息、方法信息、类加载器的引用、对应 class 实例的引用
-   **类加载器的引用**：
    -   这个类到类加载器实例的引用
-   **对应Class实例的引用**：
    -   类加载器在加载类信息放到方法区中、会创建一个对应的 Class 类型的对象实例放到堆（Heap）中、开发人员访问方法区中类定义的入口和切入点



#### 2、类加载的时机



>   其中情况 1 中的 4 条字节码指令在 Java 里最常见的场景是：
>
>   1.  new 一个对象时
>   2.  set 或者 get 一个类的静态字段（除去那种被 final 修饰放入常量池的静态字段）
>   3.  调用一个类的静态方法

![image-20210616102942267](JVM.assets/image-20210616102942267.png)



#### 3、类加载机制的懒加载



>   类加载机制很多都是懒加载、jar 包和 war 包的类也不是一次性加载的、只有当程序用到调到的时候才会加载、走类加载的五个步骤

以下程序可以很好的演示 JVM 的懒加载机制

```java
/**
 * 演示类加载器的懒加载过程
 *   用到时才去执行类加载过程
 */
public class LazyClassLoader {

    static{
        System.out.println(".............Static LazyClassLoader.............");
    }

    public static void main(String[] args) {
        new A();
        System.out.println(".............Test main.............");
        B b = null;
    }
}

class A{
    static{
        System.out.println(".............Static A.............");
    }
    public A(){
        System.out.println(".............Init A.............");
    }
}

class B{
    static{
        System.out.println(".............Static B.............");
    }
    public B(){
        System.out.println(".............Init B.............");
    }
}
```

-   输出结果如下：

```java
.............Static LazyClassLoader.............
.............Static A.............
.............Init A.............
.............Test main.............
```



### 2、类加载器



#### 1、类加载器的种类



上面的类加载过程主要通过类加载器来实现的、Java里有如下几种类加载器

```java
public class ClassLoaderType {
    public static void main(String[] args) {
        //引导类加载器
        System.out.println(String.class.getClassLoader());
        //扩展类加载器
        System.out.println(com.sun.crypto.provider.DESKeyFactory.class.getClassLoader());
        //应用恒旭类加载器
        System.out.println(ClassLoaderType.class.getClassLoader());
    }
}
```

输出如下：

-   如上输出可以看到为什么引导类加载器的输出为什么是 null 呢？因为引导类加载器是C++端代码实现的、所以Java这边获取不到

```java
null
sun.misc.Launcher$ExtClassLoader@7291c18f
sun.misc.Launcher$AppClassLoader@18b4aac2
```



##### 1、引导类加载器(BootstrapClassLoader)：



负责加载支撑 JVM 运行的位于 JRE 的 lib 目录下 （<Java_Home>\lib）的核心类库、比如 rt.jar、charsets.jar 等、我们常用的 String 类也是核心类库的



##### 2、扩展类加载器(extClassLoader)：



负责加载支撑 JVM 运行的位于 JRE 的 lib 目录下的 ext 扩展目录（<Java_Home>\lib\ext）中的 JAR 包



##### 3、应用程序类加载器(appClassLoader)：



负责加载 ClassPath路径下的类包、主要就是加载你自己写的那些类



##### 4、自定义加载器(CustomClassLoader)：



负责加载用户自定义路径下的类包



#### 2、类加载器的源码分析



分析 Launcher 类的源码：参见类运行加载全过程图（图一）可知其中会创建 JVM 启动器实例 sum.misc.Launcher、初始化使用了单例模式设计、保证一个 JVM 虚拟机内只有一个 sun.misc.Launcher实例、如下源码图

![image-20210616140429559](JVM.assets/image-20210616140429559.png)



在 Launcher 构造方法内部、其创建了两个类加载器、分别是 sun.misc.Launcher.ExtClassLoader（扩展类加载器）和 sun.misc.Launcher.AppClassLoader（应用类加载器）、JVM 默认使用 Launcher 的 getClassLoader() 方法返回的类加载器 AppClassLoader 的实例加载我们的应用程序。如下图 Launcher 类的构造方法、如下源码图



![image-20210616140504535](JVM.assets/image-20210616140504535.png)



#### 3、双亲委派机制介绍



##### 1、什么是双亲委派机制：



当某个类加载器需要加载某个 .class 文件时.它首先把这个任务委托给他的上级类加载器. 递归这个操作. 如果上级的类加载器没有加载. 自己才会去加载这个类



-   通过以下代码我们可以得知各个类加载器需要加载的 jar 包

```java
public static void main(String[] args) {
    /**
     * 双亲委派机制
     */
    System.out.println("\n bootstrapLoader 加载以下文件：");
    URL[] urls = Launcher.getBootstrapClassPath().getURLs();
    for (int i = 0; i < urls.length; i++) {
        System.out.println(urls[i]);
    }

    System.out.println("\n extClassLoader 加载以下文件：");
    System.out.println(System.getProperty("java.ext.dirs"));

    System.out.println("\n appClassLoader加载以下文件：");
    System.out.println(System.getProperty("java.class.path"));
}
```

控制台输出如下

```java
 bootstrapLoader 加载以下文件：
file:/C:/Program%20Files/Java/jdk1.8.0_271/jre/lib/resources.jar
file:/C:/Program%20Files/Java/jdk1.8.0_271/jre/lib/rt.jar
file:/C:/Program%20Files/Java/jdk1.8.0_271/jre/lib/sunrsasign.jar
file:/C:/Program%20Files/Java/jdk1.8.0_271/jre/lib/jsse.jar
file:/C:/Program%20Files/Java/jdk1.8.0_271/jre/lib/jce.jar
file:/C:/Program%20Files/Java/jdk1.8.0_271/jre/lib/charsets.jar
file:/C:/Program%20Files/Java/jdk1.8.0_271/jre/lib/jfr.jar
file:/C:/Program%20Files/Java/jdk1.8.0_271/jre/classes

 extClassLoader 加载以下文件：
C:\Program Files\Java\jdk1.8.0_271\jre\lib\ext;C:\WINDOWS\Sun\Java\lib\ext

 appClassLoader加载以下文件：
C:\Program Files\Java\jdk1.8.0_271\jre\lib\charsets.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\deploy.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\ext\access-bridge-64.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\ext\cldrdata.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\ext\dnsns.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\ext\jaccess.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\ext\jfxrt.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\ext\localedata.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\ext\nashorn.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\ext\sunec.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\ext\sunjce_provider.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\ext\sunmscapi.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\ext\sunpkcs11.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\ext\zipfs.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\javaws.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\jce.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\jfr.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\jfxswt.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\jsse.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\management-agent.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\plugin.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\resources.jar;
C:\Program Files\Java\jdk1.8.0_271\jre\lib\rt.jar;
D:\Java\LeeLearning1\target\classes;D:\Apache\repository\org\springframework\boot\spring-boot-starter-web\2.4.4\spring-boot-starter-web-2.4.4.jar;
D:\Apache\repository\org\springframework\boot\spring-boot-starter\2.4.4\spring-boot-starter-2.4.4.jar;D:\Apache\repository\org\springframework\boot\spring-boot\2.4.4\spring-boot-2.4.4.jar;D:\Apache\repository\org\springframework\boot\spring-boot-autoconfigure\2.4.4\spring-boot-autoconfigure-2.4.4.jar;D:\Apache\repository\org\springframework\boot\spring-boot-starter-logging\2.4.4\spring-boot-starter-logging-2.4.4.jar;D:\Apache\repository\ch\qos\logback\logback-classic\1.2.3\logback-classic-1.2.3.jar;D:\Apache\repository\ch\qos\logback\logback-core\1.2.3\logback-core-1.2.3.jar;D:\Apache\repository\org\apache\logging\log4j\log4j-to-slf4j\2.13.3\log4j-to-slf4j-2.13.3.jar;D:\Apache\repository\org\apache\logging\log4j\log4j-api\2.13.3\log4j-api-2.13.3.jar;D:\Apache\repository\org\slf4j\jul-to-slf4j\1.7.30\jul-to-slf4j-1.7.30.jar;D:\Apache\repository\jakarta\annotation\jakarta.annotation-api\1.3.5\jakarta.annotation-api-1.3.5.jar;D:\Apache\repository\org\yaml\snakeyaml\1.27\snakeyaml-1.27.jar;D:\Apache\repository\org\springframework\boot\spring-boot-starter-json\2.4.4\spring-boot-starter-json-2.4.4.jar;D:\Apache\repository\com\fasterxml\jackson\core\jackson-databind\2.11.4\jackson-databind-2.11.4.jar;D:\Apache\repository\com\fasterxml\jackson\core\jackson-annotations\2.11.4\jackson-annotations-2.11.4.jar;D:\Apache\repository\com\fasterxml\jackson\core\jackson-core\2.11.4\jackson-core-2.11.4.jar;D:\Apache\repository\com\fasterxml\jackson\datatype\jackson-datatype-jdk8\2.11.4\jackson-datatype-jdk8-2.11.4.jar;D:\Apache\repository\com\fasterxml\jackson\datatype\jackson-datatype-jsr310\2.11.4\jackson-datatype-jsr310-2.11.4.jar;D:\Apache\repository\com\fasterxml\jackson\module\jackson-module-parameter-names\2.11.4\jackson-module-parameter-names-2.11.4.jar;D:\Apache\repository\org\springframework\boot\spring-boot-starter-tomcat\2.4.4\spring-boot-starter-tomcat-2.4.4.jar;D:\Apache\repository\org\apache\tomcat\embed\tomcat-embed-core\9.0.44\tomcat-embed-core-9.0.44.jar;D:\Apache\repository\org\glassfish\jakarta.el\3.0.3\jakarta.el-3.0.3.jar;D:\Apache\repository\org\apache\tomcat\embed\tomcat-embed-websocket\9.0.44\tomcat-embed-websocket-9.0.44.jar;D:\Apache\repository\org\springframework\spring-web\5.3.5\spring-web-5.3.5.jar;D:\Apache\repository\org\springframework\spring-beans\5.3.5\spring-beans-5.3.5.jar;D:\Apache\repository\org\springframework\spring-webmvc\5.3.5\spring-webmvc-5.3.5.jar;D:\Apache\repository\org\springframework\spring-aop\5.3.5\spring-aop-5.3.5.jar;D:\Apache\repository\org\springframework\spring-context\5.3.5\spring-context-5.3.5.jar;D:\Apache\repository\org\springframework\spring-expression\5.3.5\spring-expression-5.3.5.jar;D:\Apache\repository\org\slf4j\slf4j-api\1.7.30\slf4j-api-1.7.30.jar;D:\Apache\repository\org\springframework\spring-core\5.3.5\spring-core-5.3.5.jar;D:\Apache\repository\org\springframework\spring-jcl\5.3.5\spring-jcl-5.3.5.jar;D:\MySoft\Intellij IDEA\IntelliJ IDEA 2020.1.2\lib\idea_rt.jar
```



问题分析：我们看到了为什么 appClassLoader 加载器也会加载 引导类加载器加载的类路径呢？其实这个下面会做出解释、虽然这里会输出在 appClassLoader 里、并不是说 appClassLoader 就加载了这些类、



##### 2、双亲委派机制流程图



>   双亲委派机制

我们先忽略自定义类加载器来讲、一般是从底向上进行加载、先从应用程序类加载器开始加载、如果当前加载器没有找到当前类、那么会往上进行委托、这时就到了扩展类加载器、如果还没有找到要加载的当前类的话、继续往上委托引导类加载器、这时引导类加载器会检查自己的 JRE 目录下有没有当前类。相反引导类加载器没找到也可以往下进行委托加载

![image-20210616141250305](JVM.assets/image-20210616141250305.png)



>   注意：这几个类加载器之间不是继承的关系、只是应用程序类加载器的一个 parent 属性的值是 上一级加载器. 如下
>
>   ![image-20210616141416397](JVM.assets/image-20210616141416397.png)



##### 3、为什么JVM要先从应用程序加载类加载？



我们来思考这样一个问题，我么平时开发的代码而言，平均 90% 以上都是我们自己写的类、所以虽然第一次加载最坏的情况循环了两圈、但是一旦引导类加载器加载了就会缓存到了内存中的集合里维护，下次再来不用再次加载、直接内存使用。如果设计成 从 引导类加载器开始、那么基本上每次加载都要跑一圈相互加载委托、性能上不行



##### 4、双亲委派机制核心源码分析



>   代码分析

所有类加载器的父类都是 ClassLoader 、都会执行到 loadClass() 方法。第一次进来肯定是 appClassLoader 开始.

可以看到这个类会先调用 findLoadedClass(name) [C++实现] 方法、查询这个类是否已经被加载过、然后才会获取当前加载器的 parent 属性、在调用 parent 类的加载器，如果为空、就说明已经到了顶层引导加载器了

现在顶层开始往下查找调用 findBootstrapClassOrNull(name) [C++实现的本地方法] 方法、C++ 端也会先判断该类是否已经被加载过、如果没有加载在调用引导类加载器去加载并返回、最后的 findClass(name) 方法。

进行类路径定位的定位并通过一系列的本地方法调用、简单来说就是在当前类加载器的内存中去尝试加载指定的类、如果返回为空就继续用别的类加载器的 findClass(name) 方法去加载

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```



##### 5、为什么要设计双亲委派机制?



-   **沙箱安全机制**：自己写的 java.lang.String.class 类不会被加载、这样可以防止核心库 API 被随意的篡改
-   **避免类的重复加载**：当父类已经加载了该类、就没有必要子 ClassLoader 在加载一次、保证被加载类的唯一性

>   小插曲

如果自己写的 java.lang.String 类运行会被加载吗？不行、看以下报错

![image-20210616142659282](JVM.assets/image-20210616142659282.png)

**以上报错分析**

一开始会经过 应用程序加载器 来加载、会先看看缓存集合有没有加载过该类，如果没有就向上委托、一直到顶层、引导类加载器看到自己 lib 目录下确实有 java.lang.String 类、就将自己的 String 类加载到内存中去、这样在自己写的 String 中定义了 main方法运行，可是系统核心库的 String 类中就没有定义该方法、所以抛出异常、这样也防止了黑客篡改系统核心类库植入后门程序



##### 6、全盘负责委托机制



全盘负责 是指一个 ClassLoader 装在一个类时、除非显式的使用另一个 ClassLoader、该类所依赖及引用的类也由这个 ClassLoader 载入



##### 7、自定义类加载器



自定义类加载器只需要继承 java.lang.ClassLoader 类、该类有两个核心方法、一个时 loadClass(String, boolean)、实现了双亲委派机制，还有一个方法就是 findClass()、默认实现是空方法、所以我们自定义类加载器主要是重写 findClass() 方法

自定义类加载器默认的加载器是 应用程序类加载器、因为初始化的时候先初始化父类的 ClassLoader、然后将 parent 属性直接赋值为 AppClassLoader



##### 8、手动打破双亲委派机制



>   再来一个沙箱安全机制实例、尝试打破双亲委派机制、用自定义类加载器加载我们自己实现的 java.lang.String.class 类



比如用自定义的类加载器继承了 ClassLoader 类、之后在覆盖父类的 loadClass(String, boolean) 方法、将里面的双亲委派机制代码删除、然后我们指定只加载我们自定义路径下的类文件、此时就会遇到一个问题、因为所有类的父类都是 Object 类、那么自定义路径下肯定没有该类的，那么就会导致加载失败、如果我们把 Object 类的 字节码文件拷贝过来、运行会触发 JDK 的沙箱安全机制：如下

![image-20210616143103139](JVM.assets/image-20210616143103139.png)

此时如何解决？JDK 不允许核心库的字节码文件在自定义类加载器中加载的、而此时双亲委派已经打破、已经不会向上去引导类加载器去加载类文件、那么请看

如下解决方案、在自己重写的  findClass() 方法中添加以下逻辑：

![image-20210616143137597](JVM.assets/image-20210616143137597.png)

此时如果是我们自己的类路径下的字节码文件我们就走自己的类加载器、否则就还是走双亲委派机制、此时双亲委派机制被打破



##### 9、loadClass、findClass、defineClass 的区别



ClassLoader中和类加载有关的方法有很多，前面提到了loadClass，除此之外，还有findClass和defineClass等，那么这几个方法有什么区别呢？

-   loadClass()
    -   就是主要进行类加载的方法，默认的双亲委派机制就实现在这个方法中。
-   findClass()
    -   根据名称或位置加载.class字节码
-   definclass()
    -   把字节码转化为Class

这里面需要展开讲一下 loadClass 和 findClass ，我们前面说过，当我们想要自定义一个类加载器的时候，并且像破坏双亲委派原则时，我们会重写loadClass方法。

那么，如果我们想定义一个类加载器，但是不想破坏双亲委派模型的时候呢？

这时候，就可以继承 ClassLoader，并且只重写 findClass 方法。findClass() 方法是JDK1.2之后的 ClassLoader 新添加的一个方法



#### 4、历史上被打破的双亲委派机制



引导语：

其实、在Java 模块化出现之前，双亲委派模型历史上主要出现过 3 次较大规模的 “被破坏” 的情况



##### 1、历史上第一次被打破



>   双亲委派模型的第一次 “被破坏” 其实发生在双亲委派模型出现之前 —— 即 JDK 1.2 出现以前的 “远古“ 时代

由于双亲委派模型在 JDK1.2 之后才被引入，但是类加载器的概念和抽象类 java.lang.ClassLoader 则在 Java 的第一个版本中就已经存在，面对已经存在的用户自定义类加载器的代码，Java 设计者们引人双亲委派模型时不得不做出一些妥协

为了兼容这些已有代码，无法再以技术手段避免 loadClass() 被子类覆盖的可能性，只能在JDK1.2之后的java.lang.ClassLoader 中添加一个新的 protected 方法 findClass()，并引导用户编写的类加载逻辑时尽可能去重写这个方法，而不是在 loadClass() 中编写代码。上节我们已经分析过 loadClass() 方法，双亲委派的具体逻辑就实现在这里面、按照 loadClass() 方法的逻辑，如果父类加载失败，会自动调用自己的 findClass() 方法来完成加载，这样既不影响用户按照自己的意愿去加载类，又可以保证新写出来的类加载器是符合双亲委派规则的



##### 2、历史上第二次被打破



双亲委派模型的第二次 “被破坏” 是由这个模型自身的缺陷导致的、双亲委派很好地解决了各个类加载器协作时基础类型的一致性问题（越基础的类由越上层的加载器进行加载）、基础类型之所以被称为 “基础”、是因为它们总是作为被用户代码继承、调用的 API 存在，但程序设计往往没有绝对不变的完美规则，如果有基础类型又要调用回用户的代码，那该怎么办呢?

>   这并非是不可能出现的事情，一个典型的例子便是 JNDI 服务、JNDI 现在已经是 Java 的标准服务，它的代码由启动类加载器来完成加载 (在JDK 1.3时加入到 rt.jar 的)、肯定属于Java中很基础的类型了

但 JNDI 存在的目的就是对资源进行查找和集中管理、比如 JDBC 的链接池信息，它需要调用由其他厂商实现并部署在应用程序的 ClassPath 下的 JNDI 服务提供者接口（Service Provider Interface，SPI)的代码，现在问题来了，启动类加载器是绝不可能认识、加载这些代码的，那该怎么办?

为了解决这个困境，Java 的设计团队只好引人了一个不太优雅的设计、线程上下文类加载器 (Thread Context ClassLoader)、这个类加载器可以通过 java.lang.Thread 类的 setContext-ClassLoader() 方法进行设置、如果创建线程时还未设置，它将会从父线程中继承一个，如果在应用程序的全局范围内都没有设置过的话，那这个类加载器默认就是应用程序类加载器。

有了线程上下文加载器、JNDI 服务使用这个线程上下文加载器去加载所需要的 SPI 服务代码、这其实是一种父类加载器去请求子类加载器完成类加载的行为、实际上是打破了双亲委派机制模型的层次顺序来逆向使用类加载器

>   Java 中涉及 SPI 的加载基本上都采用了这种方式来完成，例如 JNDI、JDBC、JCE、JAXB、和 JBI 等，为了消除这种极不优雅的的实现方式

不过在 JDK 6 时，提供了 java.uitl.ServiceLoader 类，以 META-INF/services 中的配置信息、并以责任链模式，才算是给 SPI 的加载提供了一种相对合理的解决方案



##### 3、历史上第三次被打破



>   双亲委派模型的第三次 ”被打破“，是由于用户对程序动态性的追求而导致的，比如：代码热替换、模块的热部署等，说白了就是希望 Java 应用程序不用重启服务就可以使用到最新修改的配置或者代码

OSGi 是如何通过类加载器实现热部署的？

>   OSGI(Open Service gateway initactive) 是 java 动态化模块系统的一系列规范、OSGI 的主要工作是让组件尽可能的解耦（通过定义的规范实现），并且能让组件动态的发现其他组件

OSGi 实现模块化部署的关键时它自定义的类加载器机制的实现、每一个程序模块 (OSGi 中称为 Bundle) 都有一个自己的类加载器，当需要更换一个 Bundle 时，就把 Bundle 连同类加载器一起换掉以实现代码的热部署。在OSGi 环境下，类加载器不再双亲委派模型推荐的树状结构，而是进一步发展为更加复杂的网络结构，当收到类加载请求时，OSGi 将按照下面的顺序进行类搜索：

![image-20210616214317411](JVM.assets/image-20210616214317411.png)

-   将以 java.* 开头的类，委托给父类加载器加载
-   否则，将委托列表名单内的类，委托给父类加载器加载
-   否则，将 Import 列表中的类，委托给 Export 这个类的 Bundle 的类加载器加载
-   否则，查找当前 Bundle 的 ClassPath，使用自己的类加载器加载
-   否则，查找类是否在自己的 Fragment Bundle 中，如果在，则委派给 Fragment Bundle 的类加载器加载
-   否则，查找 Dynamic Import 列表的 Bundle，委托给对应的 Bundle 的类加载器加载
-   否则，类查找失败

上面查找的顺序中只有开头两点仍然符合双亲委派模型的原则，其余的类查找都是在平级的类加载器中进行的



##### 4、历史上第四次被打破



在 JDK 9 中对 Java 进行了一次重要的升级，就是引入了 Java 模块化系统，不过为了保证兼容性，JDK 9 并没有从根本上动摇从 JDK1.2 以来运行了二十年之久的三层类加载器架构以及双亲委派机制模型，但是也为了模块化系统的顺利运行，模块化下的类加载器仍然发生了一些变化：

-   扩展类加载器（Extension Class Loader）被平台类加载器（Platform Class Loader）取代
-   其次应用程序类加载器都不再派生子 java.net.URLClassLoader，如果有程序直接依赖了这种继承关系，则很可能会在 JDK 1.9 及更高版本中崩溃
-   当平台或应用程序类加载前，先判断该类是否能够归属到某一个系统模块中，如果可以找到归属关系，那么就优先派给负责那个模块的加载器完成

至此的第三条，也可以算是对双亲委派机制的第四次打破



## 2、JVM内存模型深度解析



### 1、JDK 完整体系结构如下



![image-20210616153046713](JVM.assets/image-20210616153046713.png)



Java语言跨平台特性（一次编写、到处运行）

![image-20210616153200368](JVM.assets/image-20210616153200368.png)



>   java 命令行编译命令

javap 指令：可以通过 javap -c 将具体字节码文件编译成可读性更强的文件、具体用法、需要进入具体 target 目录的路径下使用命令

```java
javap -c Math.class > Math.txt
```



### 2、JVM 内存模型



#### 1、类装载子系统



就是类加载机制的流程（见上个章节的内容）



#### 2、运行时数据区 (内存模型)



##### 1、堆：

存放 new 出来的对象、JVM调优主要是在堆中进行的

![image-20210616154751694](JVM.assets/image-20210616154751694.png)

##### 2、虚拟机栈 (线程栈)：



>   **栈帧**：一旦虚拟机开始运行一个方法，比如 main() 方法、就会在栈里划分一块小区域来存放这个main() 方法的里面独有的内存区域、包括以下内容

-   **局部变量表**：就是存放 main 方法里面定义的局部变量信息
-   **操作数栈**：看以下方法、就是将 1 、2、 a + b 的值 压入操作数栈、再将结果乘以 10 压入操作数栈

```java
public int compute(){
    int a = 1;
    int b = 2;
    int c = (a + b) * 10;
    return c;
}
```

-   **动态链接**：难理解、抽象
-   **方法出口**：当一个 main() 方法中调用了 compute 方法并执行完毕后、那么该返回继续执行 main 方法、那么这时方法出口就记录了 compute 执行完毕后出来该执行哪行指令等信息



##### 3、本地方法栈：



>   C++ 代码运行需要的栈空间



##### 4、方法区：



存放、常量、静态变量、类信息、八大数据类型和字符串等 ( 方法区在 JDK1.8 之前叫做永久代、之后改了一个名称叫做元空间、直接内存)



##### 5、程序计数器：



为了保证程序（进程）能够连续地执行下去，处理器必须具有某些手段来确定下一条指令的地址。而程序计数器正是起到这种作用、通俗的来讲就是记录代码当前运行到哪条指令了、比如 main() 方法运行到一半、突然 CPU 时间片被另一个线程抢走了、另一个线程执行完了，回到 main() 方法、利用程序计数器就可以知道接下来该执行的指令、通常来说每个线程都有自己的程序计数器



#### 3、字节码执行引擎



字节码执行引擎后台会开一个线程去做GC回收



#### 4、整体内存模型图如下



![image-20210616154228660](JVM.assets/image-20210616154228660.png)



## 3、JVM 内存参数设置



Spring Boot程序的JVM参数设置格式（Tomcat 启动直接加到 bin 目录下的 catalina.sh 文件中）：

```java
java ‐Xms2048M ‐Xmx2048M ‐Xmn1024M ‐Xss512K ‐XX:MetaspaceSize=256M ‐XX:MaxMetaspaceSize=256M ‐jar microservice‐eurek a‐server.jar
```



### 1、JVM 参数详解



#### 1、设置线程栈大小



>   -Xss512K (默认是1M)

JVM中的线程栈大小（虚拟机栈）、左边的 512K 是给一个线程来用的、一个线程会从线程栈中分配一小块大小、这个大小就是 512K

```java
public void stackOverFlow(){
    stackOverFlow();
}
```

分析：看以上代码、如果运行以上代码、最终都会触发 StackOverflowError 异常、因为每次调用都要对 stackOverFlow 方法在线程栈上分配一块空间、这属于死循环嵌套调用、发生异常的时间取决于 -Xss 线程栈设置的大小

结论：-Xss 设置越小 count 值越小，说明一个线程栈里能分配的栈帧就越少，但是对 JVM 整体来说能开启的线程数会更多



#### 2、元空间参数设置：



**注意：元空间的默认初始大小是21MB、默认的元空间的最大值是无限**

```java
-XX:MetaspaceSize
```

设置元空间的初始空间大小、以字节为单位、默认为 21M 、达到该值就会触发 Full GC 进行类型卸载、同时收集器会对该值进行自动扩容机制：如果释放了大量的空间、就适当降低该值、如果释放了很少的空间、那么在不超过 -XX:MaxMetaspaceSize（如果设置了的话）的情况下、适当提高该值、这个跟早期 JDK 版本的 -XX:PermSize 参数意思不一样、它原来代表永久代的初始容量 

```
-XX:MaxMetaspaceSize
```

设置元空间的最大值、默认是 -1、即不限制、或者说之受限于本地内存大小

>   分析：由于调整元空间的大小需要 Full GC、触发 STW（Step the World）、这是非常昂贵的操作，如果应用在启动的时候发生大量 Full GC、通常都是由于永久代或者元空间发生了大小调整，基于这种情况，不建议不设置元空间得大小、一般建议 JVM 参数中将 MetaspaceSize 和 MaxMetaspaceSize 设置成一样的值、并设置得比初始值要大、对于8G 物理内存得机器来说、一般这两个值都设置为 256M



## 4、JVM 内存分配机制详解



### 1、对象创建和内存分配流程



**类加载检查 => 是否已加载类 => 分配内存 => 初始化 => 设置对象头 => 执行 <init> 方法**



#### 1、类加载检查



虚拟机遇到一条 new 指令时、首先去检查这个指令得参数是否能在常量池中定位到一个类得符号引用、并检查这个符号引用代表的类是否已加载、解析和初始化过、如果没有、那必须先执行相应的类加载过程



#### 2、分配内存



在类加载检查后、接下来虚拟机将为新生对象分配内存、对象所需内存的大小在类加载完后便可以完全确定、为对象分配空间的任务等同于把 一块确定大小的内存从 Java 堆中划分出来

![image-20210616182804957](JVM.assets/image-20210616182804957.png)



##### 1、对象栈上分配



我们通过 JVM 内存分配得知：Java 的对象一般都是在堆上进行分配的、当对象没有被引用的时候，需要依靠 GC 进行回收内存，如果对象数量较多的时候、会给 GC 带来较大的压力，也间接的影响了应用的性能，为了减少临时对象在堆内存分配的数量，JVM 通过 逃逸分析 确定该对象不会被外部访问，不会逃逸的话就可以将对象在栈上分配内存，这样该对象所占用的内存空间就可以随着栈而销毁，也就减轻了垃圾回收的压力



##### 2、对象逃逸分析（栈上分配机制）



就是分析对象动态作用域，当一个对象在方法中被定义后、它可能被外部方法所引用、例如作为方法返回值、或者调用参数传递到其他地方中，如下面两个例子演示

-   此方法对象将堆上分配、因为对象被作为参数返回、对象的作用域范围不确定

```java
public Lee getLee(){
    Lee lee = new Lee("对象逃逸");
    lee.setAddress("test");
    return lee;
}

```

-   该方法的 lee 对象仅在 createLee 方法作用域内、未发生外部引用、未逃逸、此时对象分配到栈上、即用即毁

```java
public void createLee(){
    Lee lee = new Lee("对象未逃逸");
    lee.setAddress("test2");
}
```



###### 1、对象逃逸分析参数



JVM 对于以上情况可以通过开启以下逃逸分析参数来优化对象内存分配的位置

```java
-XX:+DoEscapeAnalysis // JDK 7 之后默认开启逃逸分析
```

使其通过标量替换优先分配在栈上（栈上分配），如果想要关闭逃逸分析请使用如下参数

```java
XX:-DoEscapeAnalysis  // 关闭对象逃逸分析
```



###### 2、标量替换 (配合逃逸分析使用)



通过逃逸分析分析确定该对象不会被外部访问、并且对象可以进一步分解时、JVM 不会创建该对象，而是将该对象成员变量分解成若干个被这个方法使用的成员变量所替代，这些替代的成员变量在栈帧或者寄存器上分配空间，这样就不会因为没有一块连续的空间导致对象内存不够分配

```java
-XX:+EliminateAllocations // 开启标量替换参数：JDK 7 之后默认开启  
```



###### 3、标量与聚合量



标量即不可被进一步分解的量、而 Java 的基本数据类型就是标量（如：int、long 等基本数据类型以及 reference 类型等、），标量的对立就是可以被进一步分解的量、而这种量称之为聚合量，而在 Java 中对象就是可以被进一步分解的聚合量



##### 3、对象堆上的 Eden 区分配



大多情况下、对象在新生代中 Eden 区分配、当 Eden 区没有足够的空间时，虚拟机将发起一次 Minor GC

-   Minor GC / Young GC：指的是新生代的垃圾收集动作、Minor GC 非常频繁，回收速度一般比较快
-   Major GC / Full GC：一般会回收老年代、年轻代、方法区的垃圾对象、速度一般会比 Minor GC 慢 10 以上

```java
-XX:+PrintGC 或 -XX:+PrintGCDetails // 开启打印GC日志：（详细日志）
//在实际问题排查中，收集器日志常会打印到文件后通过工具进行分析
```

>   Eden 与 Survivor 区默认比例为 8 : 1 : 1

大量的对象被分配在 eden 区，eden 区满了后会出发 Minor GC、可能会有 99% 以上的对象成为垃圾被回收掉，剩下的存活的对象会被挪到为空的那块 survivor 区

下一次 eden 区满了后又会出发 Minor GC、把 Eden 区和 Survivor 区垃圾对象回收，把剩余存活的对象一次性挪动到另一块为空的 Survivor 区，因为新生代的对象都是朝生夕死的，存活时间很短，**所以 JVM 默认的 8 : 1 : 1 的比例是很合适的，让 eden 区尽量很大，survivor 区够用即可**



JVM默认有这个参数如下、会导致这个比例会自动变化

```java
-XX:+UseAdaptiveSizePolicy //（默认开启）
```

如果不想这个比例有变化就关闭这个参数： -XX:-UseAdaptiveSizePolicy



#### 3、初始化



内存分配完毕后、虚拟机不要将分配的内存空间都初始化为零值（不包括对象头）、如果使用 TLAB （本地线程分配缓冲）、这一工作过程也可以提前至 TLAB 分配时进行、**这一步操作保证了对象的实例在 Java 代码中可以不赋初始值就直接使用，程序能访问到这些字段的数据类型所对应的默认值**

```java
private int num = 10;         //初始值 0
private double age = 24.5;    //初始值 0.0
private Object parent = new Test(); //初始值 null
```



#### 4、设置对象头 (重要)



##### 1、对象头的组成



初始化默认值之后、虚拟机要对对象进行必要的设置、例如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的GC分代年龄等信息? 其实这些信息存放在对象的对象头 Object Header 中.

>   在HotSpot虚拟机中、对象在内存中存储的布局可以分为3块区域

-   对象头（Header）
-   实例数据（Instance Data）
-   对齐填充（Padding）

HotSpot 虚拟机的对象头包括两部分信息、第一部分用于存储对象自身运行时数据、如哈希码（HashCode）、GC的分代年龄、锁状态标志、线程持有的锁、偏向线程 ID 偏向时间戳等、对象头的另一部分是类型指针，即对象指向它的类元数据的指针、虚拟机通过这个指针来确定这个对象是哪个类的实例



##### 2、32 位对象头结构图



![image-20210616173656370](JVM.assets/image-20210616173656370.png)

其中Object Header 对象头有几个重要的点：

**Mark Word 标记字段**：（32位占4字节、64位占8字节），自身运行时数据哈希码、GC分代年龄、锁状态标志，线程持有锁、偏向线程ID、偏向时间戳

**Klass Pointer 类型指针**：（开启指针压缩压缩占4字节、否则占8字节）对象指向它的类元数据（方法区）的指针

**数组长度**：只有数组类型才有、占4字节



>   我们可以引入 jol-core 包查看打印对象头信息

```xml
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
    <version>0.16</version>
    <scope>compile</scope>
</dependency>
```

>   如下 Java 代码获取对象头信息

```java
package lee.learning.video.jvm.other;

import org.openjdk.jol.info.ClassLayout;

/**
 * 计算对象大小
 */
public class JQLSample {

    public static void main(String[] args) {
        //打印 Object 的对象头信息
        ClassLayout layout = ClassLayout.parseInstance(new Object());
        System.out.println(layout.toPrintable());

	    //打印 int[] 的对象头信息
        ClassLayout layout1 = ClassLayout.parseInstance(new int[]{});
        System.out.println(layout1.toPrintable());

        //打印 A 的对象头信息
        ClassLayout layout2 = ClassLayout.parseInstance(new A());
        System.out.println(layout2.toPrintable());
    }

    /**
     * ‐XX:+UseCompressedOops 默认开启的压缩所有指针
     * ‐XX:+UseCompressedClassPointers 默认开启的压缩对象头里的类型指针Klass Pointer
     * Oops : Ordinary Object Pointers
     */
    public static class A {
        /**
         * 8B mark word
         * 4B Klass Pointer 
         * 如果关闭压缩 ‐XX:‐UseCompressedClassPointers 或 ‐XX:‐UseCompressedOops 则占用 8B
         */
        int id;        // 4B
        String name;   // 4B 如果关闭压缩 ‐XX:‐UseCompressedOops、则占用 8B
        byte b;        // 1B
        Object o;      // 4B 如果关闭压缩 ‐XX:‐UseCompressedOops、则占用 8B
    }
}
```

-   输出如下

```java
// Object 的对象头信息
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0xf80001e5
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

// int[] 的对象头信息
[I object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0xf800016d
 12   4        (array length)            0
 12   4        (alignment/padding gap)   
 16   0    int [I.<elements>             N/A
Instance size: 16 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total

// A 的对象头信息
com.lee.test.jvm.ClassLayoutTest$A object internals:
OFF  SZ               TYPE DESCRIPTION               VALUE
  0   8                    (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4                    (object header: class)    0xf800cd26
 12   4                int A.id                      0
 16   1               byte A.b                       0
 17   3                    (alignment/padding gap)   
 20   4   java.lang.String A.name                    null
 24   4   java.lang.Object A.o                       null
 28   4                    (object alignment gap)    
Instance size: 32 bytes
Space losses: 3 bytes internal + 4 bytes external = 7 bytes total
```



##### 3、指针压缩参数



jdk 1.6 update14 开始、在 64 bit 操作系统中，JVM 支持指针压缩

jvm 配置参数：UseCompressedOops、compressed--压缩、opp(ordinary object pointer)--对象指针

```java
-XX:+UseCompressedOops  // 默认开启, 启用所有指针压缩,包括 klass Pointers 和 成员变量
-XX:-UseCompressedOops  // 禁止指针压缩
-XX:+UseCompressedClassPointers  //只压缩 class Pointers 的类型指针
```



##### 4、为什么要进行指针压缩？



-   在 64 位平台的 HotSpot 中使用 32 位指针、内存使用会多出 1.5 倍左右，使用较大指针在主内存和缓存之间移动数据、占用较大的宽带同时 GC 也会承受较大压力
-   为了较少 64 位平台内存的消耗、启用指针压缩功能
-   在 JVM 中、32 位地址最大支持 4G 内存 ( 2 的 32 次方 ) 、可以通过对象指针的压缩编码、解码方式进行优化、使得JVM 只用 32 位的地址就可以支持更大的内存配置
-   堆内存小于 4G 时、不需要启用指针压缩、JVM 会直接去除高32位地址、即使用低虚拟地址空间
-   堆内存大于 32G 时、压缩指针会失效、会强制使用 64 位（即 8 字节）来对 java 对象寻址、这就会出现 1 的问题、所以堆内存不要大于 32G 为好



#### 5、执行 init 方法



执行<init>方法、即对象按照程序员的意愿进行初始化、对应到语言层面上讲、就是为属性赋值（注意：这与上面的赋默认值不同、这是初始化程序员设定的值）和执行构造方法



### 2、对象内存分配的问题



#### 1、如何划分内存



##### 1、指针碰撞 (Bump The Pointer)



如果 Java 堆中内存是绝对规整的、所有用过的内存都放在一边排列，空闲的内存存放在另一边，中间有一个指针作为分界点指示器，到所分配的内存就仅仅是把那个指针向空闲的那边移动一段与对象大小相等的距离



##### 2、空闲列表 ( Free List )



如果 Java 堆中的内存不是规整的，已使用的内存和空闲的内存相互交错，那就没有办法进行简单的指针碰撞法了，虚拟机就必须维护一个列表，记录哪些内存是可用的、在分配的时候从列表中找到一块足够大的空间划分给对象实例、并更新列表上的记录



#### 2、并发情况下的问题



在并发情况下、可能出现正在给对象A分配内存、指针还没来得及修改、对象B又同时使用了原来的指针来分配内存的情况、如何解决？



##### 1、CAS ( Compare And Swap )



虚拟机采用 CAS 配上失败重试的方式保证更新操作的原子性来对分配内存空间的动作进行同步处理、通俗的讲就是几个线程同时去抢这个指针位置、如果抢到了就放入、没抢到就继续抢（失败重试）



##### 2、本地线程分配缓冲 ( Thread Local Allocation Buffer, TLAB )



把内存分配的动作按照线程划分在不同空间之中进行、即每个线程在Java堆中预先分配一小块内存

```java
-XX:+/-UseTLAB   // 设定虚拟机是否使用 TLAB ( JDK1.8 版本JVM默认开启) 
-XX:TLABSize     // 指定 TLAB 的大小、默认是伊甸园区的百分之一
```

通俗的将就是 JVM 会在堆（伊甸园区）划出一小块专供这个线程的内存空间、不同的线程划分出不同块的独立内存区域、这样就避免了并发争抢的问题



### 3、对象在堆内存的动态分配



#### 1、大对象直接进入老年代



大对象就是需要大量连续内存空间的对象（比如：字符串、数组）

```java
-XX:PretenureSizeThreshold // JVM参数：可以设置大对象的大小
```

如果对象超过设置大小会直接进入老年代，不会进入年轻代，**这个参数仅在 Serial 和 ParNew 两个垃圾收集器下有效**

比如设置JVM参数： 在执行下上面的第一个程序会发现大量对象进了老年代

```java
//PretenureSizeThreshold 单位字节、并设置垃圾收集器为 Serial
-XX:PretenureSizeThreshold = 1000000,-XX:+UseSerialGC
```

>   为什么要这样？因为避免了为大对象分配内存时的复制操作而降低效率



#### 2、长期存活的对象进入老年代



既然虚拟机采用了分代收集的思想来管理内存、那么内存回收时就必须标识哪些对象应该放在新生代，哪些对象应该放在老年代中、为了做到这一点，虚拟机给每个对象一个对象年龄 (age) 计数器，如果对象在 Eden 出生并经过一次 Minor GC 后仍然能够存活，并且能够被 Survivor 容纳的话，将被移动到 Survivor 空间中、并将对象年龄设定为 1 ，对象在 Survivor 中每熬过一次 Minor GC、年龄就增加 1 岁，当它的年龄增加到一定的程度

>   默认为 15 岁， CMS 收集器默认为 6 岁、不同的垃圾收集器略微不同

就会被晋升到老年代中、对象晋升到老年代的年龄阈值，可以通过如下参数设置

```java
-XX:MaxTenuringThreshold
```



#### 3、对象动态年龄判断



前放对象的 Survivor 区域里（其中一块区域，放对象的那块 S 区），一批对象的总大小大于这块 Survivor 区域内存大小的50%

```java
-XX:TargetSurvivorRatio  //可以指定
```

那么此时大于等于这批对象年龄最大值的对象，就可以直接进入老年代了，例如 Survivor 区域里现有一批对象，年龄1 + 年龄2 + 年龄n 的多个年龄对象总和超过了 Survivor 区域 的 50%，此时就会把 年龄n(含) 以上的对象都放入老年代，**这个规则其实是希望那些长期存活的对象**，今早进入老年代，对象动态年龄判断机制一般是在 Minor GC 之后触发的



#### 4、老年代空间分配担保机制



年轻代每次 Minor GC 之前 JVM 都会计算老年代剩余可用空间、如果这个可用空间小于年轻代里现有的所有对象大小之和（包括垃圾对象）就会判断如下这个参数是否设置

```java
-XX:HandlePromotionFailure //JDK 1.8 默认开启
```

如果有这个参数，就会判断老年代的可用内存大小，是否大于之前每一次 Minor GC 后进入老年代的对象的 平均大小，

 如果上一步结果是小于或者之前说的参数没有设置，那么就会触发一次 Full GC、对于老年代和年轻代一起回收一次垃圾，如果回收完还是没有足够的空间存放新的对象就会发生 OutOfMemoryError

当然，如果 Minorr GC 之后剩余存活的需要挪动到老年代的对象大小还是大于老年代可用空间，那么也会触发 Full GC 、后如果还是没有空间放 Minor GC 之后的存活对象、则也会发生 OutOfMemoryError

![image-20210616191751685](JVM.assets/image-20210616191751685.png)





















## 5、JVM 垃圾回收算法







