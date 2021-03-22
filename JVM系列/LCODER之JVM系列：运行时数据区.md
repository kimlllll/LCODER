本文的目录大纲：
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a615f1ae79474bfba87172b0e0ee0fff~tplv-k3u1fbpfcp-watermark.image)
### 什么是JVM
JVM 全称 Java Virtual Machine，也就是我们耳熟能详的Java虚拟机。它能识别.class后缀的文件，并且能够解析它的指令，最终调用操作系统上的函数，完成我们想要的操作。

### JVM的运行过程：
HelloWorld.java通过javac的编译，编译成HelloWorld.class,class文件通过Java类加载器ClassLoader加载到运行时数据区（JVM管理的内存），通过执行引擎解释执行。
JVM的工作就是把Class文件解释成机器可以识别的机器码，解释执行就是，字节码和机器码，

## 运行时数据区
### 1.定义
Java虚拟机在执行Java程序的过程中，会把它所管理的内存划分成若干个不同的数据区域。
### 2.分类
这些不同的数据区域分为：虚拟机栈、本地方法栈、程序计数器、方法区、堆。其中前三个区域是线程私有的区域，后两个区域是由所有线程共享的数据区。

### 2.1 程序计数器：
#### 2.1.1 定义
指向当前线程正在执行的字节码的指令地址。
#### 2.1.2 为什么需要程序计数器？（时间片轮转机制）
由于Java虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的，在任何一个确定的时刻，一个处理器或者说多核处理器的一个内核都只会执行一条线程中的指令。因此，为了线程切换后能恢复到正确的执行位置，每条线程都需要一个独立的程序计数器，各条线程之间的计数器互不影响，独立存储，因此，程序计数器必须是线程私有的。
#### 2.1.3 为什么程序计数器是JVM中唯一不会OOM的区域？
程序计数器是一块很小的内存区域，它只需要记录程序运行的地址，一个int类型的长度就足够了。

### 2.2 虚拟机栈：
#### 2.2.1 定义
存储当前线程运行方法所需的数据、指令、返回地址。

对于栈这个数据模型，它的特征就是先进后出，后进先出。虚拟机栈也不例外。Java中每一个方法在执行的同时，都会创建一个栈帧，栈帧中存储了局部变量表、操作数栈、动态链接、方法出口灯信息。每一个方法调用到结束的过程，就对应一个栈帧在虚拟机栈中入栈到出栈的过程。
如果把虚拟机栈比喻成子弹夹，那么栈帧就是子弹。

#### 2.2.2 局部变量表
##### 2.2.2.1 定义
所谓局部变量表，就是在方法执行时，各种局部变量存放的地方。

局部变量表可以存放的数据类型只有8种基本类型、引用类型和returnAddress类型。引用类型它是指向对象起始地址的引用指针或者指向一个代表对象的句柄。returnAddress类型，是指向一条字节码指令的地址，当带有返回值的方法完成时，方法完成就要出栈，出栈的地址在哪，就是使用这个值记着的。一般来说，是在该方法的下一行。
局部变量表所需的空间在编译期间完成分配，当进入一个方法时，这个方法需要在帧中分配多大的局部变量空间是完全确定的，方法运行期间不会改变局部变量表的大小。
##### 2.2.2.2 异常处理
 Java虚拟机规范中，对这个区域规定了两种的异常状况：StackOverflowError和OutOfMemoryError。
如果线程请求的栈深度>虚拟机栈的深度，将抛出StackOverflowError异常。如果虚拟机栈可以动态扩展，如果在扩展时无法申请到足够的内存，将抛出OutOfMemoryError。

##### 2.2.2.3 方法执行时，栈帧是如何工作的？
```
public class Person{

    public int work(){
        int x = 1;
        int y = 2;
        int z = (x+y)*10;
        return z;
    }

    public static void main(String[] args){
          Person person = new Person();
          person.work();
    }
}
```
这样的一段代码，使用javac编译之后，得到Person.class文件，使用Javap -c进行反汇编，得到下面的代码：
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/54bccb7ba00b4a56922d1ba0417c864c~tplv-k3u1fbpfcp-watermark.image)
分析一下这段代码执行时，虚拟机栈中的内存变化是怎样的。
执行过程：
> 1. 执行main()，main()的栈帧入栈
> 2. 执行work(),work()栈帧将main()栈帧压入栈底

![第一、二步](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1f9acfffb50b481db4b2e142bd46061d~tplv-k3u1fbpfcp-watermark.image)



> 3. 执行work()中的int x = 1； 

![第三步](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80e9c0e160e1483992041b54b9b162c8~tplv-k3u1fbpfcp-watermark.image)

> 4. 执行work()中的int y = 2;

![第四步](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ae2871ca2a34411b3e8b57354813c1b~tplv-k3u1fbpfcp-watermark.image)

> 5.执行代码 int z = (x+y)*10;  执行完成之后，操作数栈清空

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8d4f23e99360437296064e41e1a8f2b6~tplv-k3u1fbpfcp-watermark.image)

> 6 最终，11: iload_3  12: ireturn 将局部变量表中下标为3的值压入操作数栈中，作为返回
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f61d2e1c7e3743638ae249a3acbdc8eb~tplv-k3u1fbpfcp-watermark.image)

以上就是work()执行时，在虚拟机栈中内存的变化过程

JVM指令集可以参照腾讯云社区的这篇[java虚拟机 JVM字节码 指令集 bytecode 操作码 指令分类用法 助记符](https://cloud.tencent.com/developer/article/1333540)

### 2.3 本地方法栈
本地方法栈和虚拟机栈发挥的作用是很类似的，只不过虚拟机栈用于管理Java方法的调用，而本地方法栈则用于管理Native方法的调用。本地方法栈和虚拟机栈十分类似，虚拟机规范对其中方法使用的语言、使用方式和数据结构并没有强制规定，因此各虚拟机可以自由的实现它。HotSpot虚拟机，直接把本地方法栈和虚拟机栈合二为一。

### 2.4 方法区
方法区用于存储已被虚拟机加载的类信息（ClassLoader加载类信息就加载在这里）、常量、静态变量、即时编译器编译后的代码等数据。是所有线程共享的区域。

JVM在执行某个类的时候，必须先加载，在加载类（加载、验证、准备、解析、初始化）的时候，JVM会先加载class文件，而在class文件中除了有类的版本、字段、方法和接口等描述信息外，还有一项信息是常量池，用于存放编译期间，生成的各种字面量和符号引用。

当类加载到内存中后，JVM就会将class文件常量池中的内容存放到运行时常量池中；在解析阶段，JVM会把符号引用替换成直接引用（对象的索引值）。运行时常量池是全局共享的，多个类共用一个运行时常量池。class 文件中常量池多个相同的字符串在运行时常量池只会存在一份。有关这部分的内容，会在下面的Class文件结构中详细的讲解。

在JDK1.7之前，在HotSpot虚拟机中，使用永久代来实现方法区。这样做的好处是，HotSpot的垃圾回收器可以像管理Java堆一样来管理这部分的内存，省去了专门为方法区编写内存管理代码的工作，但是这样做会导致别的问题发生：1. 永久代里面的数据，回收的效率很低，但堆中放的是对象和数组，是需要频繁回收的数据。如果跟堆中一样进行垃圾回收，无疑是一种资源的浪费；2. 永久代里的内存经常不使用容易发生内存溢出，永久代从Java堆中划分，它的大小仍然是受制于堆的大小，它长时间无法回收，这块区域就很容易发生内存溢出，因此，在HotSpot虚拟机，JDK1.7版本中，已经将永久代的静态变量和运行时常量池转移到了堆中。在JDK1.8之后，更是去掉了方法区中的永久代，改为元空间，元空间所的存储位置是在机器内存中，它的大小不再受制于Java堆。也就能解决永久代内存溢出的问题。

### 2.5 堆
Java堆，是JVM所管理的内存中最大的一块。是所有线程共享的一块区域，在虚拟机启动时创建，此内存区域唯一的目的就是存放对象实例和数组，几乎所有的对象实例都在这里分配内存。

另外Java堆也是垃圾回收器管理的主要区域，随着对象的不断创建，堆空间占用越来越多，就需要不定期的对不再使用的对象进行回收。从内存回收的角度看，由现在收集器基本都采用分代算法，所以Java堆中还可以细分为新生代和老年代，新生代中又分为Eden区、From Survivor区、To Survivor区。

根据Java虚拟机规范的规定，Java堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可。

堆的大小参数：
|  参数名称   | 含义 |
|  ----  | ----  |
|  -Xms | 堆的最小值 |
|  -Xmx  |堆的最大值 |
| -Xmn  |新生代的大小 |
|  -XX:NewSize  |新生代最小值 |
|  -XX:MaxNewSize |新生代最大值 |

### 2.6 使用HSDB工具从底层深入理解运行时数据区
HSDB是JDK自带的Hotspot Debuger工具，透过它能够让我们更直观的查看运行中的java对象在内存中的存在形式和状态，如对象的oops、类信息、线程栈信息、堆信息、方法字节码和JIT编译后的汇编代码等。

对下面这段代码进行内存的分析
```
public class Test {

    public final static String MAN_TYPE = "man";
    public static String WOMAN_TYPE = "woman";

    public static void main(String[] args)throws Exception{
        Person p1 = new Person();
        p1.name = "niuniu";
        p1.age = 18;
        p1.sex = MAN_TYPE;
        // 垃圾回收15次
        for(int i = 0;i<15;i++){
            System.gc();
        }
        Person p2 = new Person();
        p2.name = "yangyang";
        p2.age = 19;
        p2.sex = WOMAN_TYPE;
        Thread.sleep(Integer.MAX_VALUE);
    }
}

class Person{
    String name;
    String sex;
    int age;
}
```

#### 2.6.1 打开HSDB
要是用HSDB，第一步，必须把sawindbg.dll复制到对应目录的jre下，否则运行HSDB时，就会报这样的错误：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e69d70f81e4b452daac61584f678b3bc~tplv-k3u1fbpfcp-watermark.image)

复制到Java目录下的jre的bin目录下，在我的电脑中就是C:\Program Files\Java\jre1.8.0_271\bin

第二步：在cmd中，打开sa-jdi.jar所在的目录，使用命令，java -cp .\sa-jdi.jar sun.jvm.hotspot.HSDB 打开HSDB工具。
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1006265f4161451dbbc136cdb2435c30~tplv-k3u1fbpfcp-watermark.image)

#### 2.6.2 向HSDB中添加进程
使用jps找到所要添加的进程

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cdcca58b731745c2a9c29d527ce7539c~tplv-k3u1fbpfcp-watermark.image)

可以看到，2732就是我们所要找的进程号，把2732添加到HDSB中。   
第一步：选择File -> Attach to HotSpot Process
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3aef5c11f0d94e81b62d07b00c8dc4a5~tplv-k3u1fbpfcp-watermark.image)

第二步：在弹出的对话框中输入进程号2732
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bb98ce058c274971afda6123c0985101~tplv-k3u1fbpfcp-watermark.image)
然后就可以看到要找的线程main
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ab0266c14cbe41708a2df753ed455464~tplv-k3u1fbpfcp-watermark.image)

#### 2.6.3 分析栈内的内存分布情况
点击如图所示的stack memory按钮
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fe4ff3392ae44abeb40b9e6c13f69ed4~tplv-k3u1fbpfcp-watermark.image)
得到如下图所示的内容：上半部分是Sleep方法的栈帧，下半部分是main方法的栈帧，类似于0x000000008143dfe0的数据，是栈内存的地址。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/38dfc14f5f2348b2b09ca1ac3d89a92f~tplv-k3u1fbpfcp-watermark.image)
把Main方法的栈帧放大来看
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d17d2e7cb033462abbef4167cd4b7857~tplv-k3u1fbpfcp-watermark.image)

#### 2.6.4 分析堆内内存的分配情况 
首先，来看下堆区在内存中的分配情况 依次点击HSDB中的Tools-> Heap Parameters得到如下图所示的内容。
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f7eb469f232c4ad4a9c012301145bece~tplv-k3u1fbpfcp-watermark.image)
其中的PSYoungGen（年轻代）、PSOldGen（年老代），年轻代中又有三个区，分别是eden区，from区，to区。
根据上面的图，我们可以总结出内存的分布情况如下表所示：年轻代的三个区域是在一块连续的内存中。
|     | 内存地址 |
|  ----  | ----  |
|  年轻代eden区 | 0x00000000d5c00000 - 0x00000000d7c80000 |
|  年轻代from区  |0x00000000d7c80000 - 0x00000000d8180000 |
|  年轻代to区  |0x00000000d8180000 - 0x00000000d8680000 |
|  年老代  |0x0000000081400000 - 0x0000000086980000 |

然后我们来看上面那一段代码中的两个Person对象在堆中的哪里。
依次点击HSDB工具中的Tools -> Object Histogram，找到Person类
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/967a661792d54c2ea8f2922388833c58~tplv-k3u1fbpfcp-watermark.image)
双击Person类，获得两个Person的具体情况
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6424f6a5f02f4a2ab9b6da6cf623c63f~tplv-k3u1fbpfcp-watermark.image)
第一个name“yangyang”的Person对象，地址是0x00000000d5c00000，根据比对，这个Person对象位于堆区的年轻代的eden区，根据内存回收机制（下面的内容里面会详细讲解），新创建的对象，位于年轻代的eden区。
第二个name“niuniu”的Person对象，地址是0x00000000814199d0，根据比对，这个Person对象位于堆区的老年代，根据内存回收机制，被回收15次的对象，如果还持有强引用，位于堆区的老年代。








