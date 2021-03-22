本文大纲：
![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/77ebaf6409704059b10571bf1d581b2c~tplv-k3u1fbpfcp-watermark.image)
  计算机只认识0和1，所以我们编写的程序，需要经过编译器编译成由0和1构成的二进制格式才能由计算机执行。然而现在的虚拟机已经不再是将我们编写的代码编译成二进制的本地机器码让计算机识别，而是编译成了与操作系统和平台无关的字节码，这个字节码就是Class文件。Java虚拟机的作用是将Java文件编译成为Class文件，让计算机识别并运行。Java虚拟机只与Class文件关联，Class文件是一种特定的二进制文件格式，它其中包含了Java虚拟机指令集和符号表以及若干其他的辅助信息。
  
举例来说，下面这一段代码，通过Java虚拟机编译之后，得到了一个二进制文件。

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
通过javac指令编译之后得到Class文件：使用NodePad++中的Hex转化成16进制查看得到：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/548e698494fd4cb1aab997d38bd69f68~tplv-k3u1fbpfcp-watermark.image)
## 识别Class文件结构
### 1. 概念

任何一个Class文件都对应着唯一一个类或接口的定义信息，但反过来说，类或接口并不一定都得定义在文件里（譬如类或接口也可以通过类加载器直接生成）。本章中，只是通俗地将任意一个有效的类或接口所应当满足的格式称为“Class文件格式”，实际上它并不一定以磁盘文件的形式存在。“Class文件”应当是一串二进制的字节流，无论以何种形式存在。
Class文件是一组以8位字节为基础单位的二进制流，各个数据项目严格按照顺序紧凑地排列在Class文件之中，当遇到需要占用8位字节以上空间的数据项时，则会按照高位在前（Big-Endian）的方式分割成若干个8位字节进行存储。无符号数据类型最大占8个字节。

要分析Class文件，首先要理解几个概念。根据Java虚拟机规范的规定，Class文件格式采用一种类似于C语言结构体的伪结构来存储数据，这种伪结构中只有两种数据类型：无符号数和表。   
**无符号数：数值，属于基本的数据类型，以u1、u2、u4、u8来代表1个字节、2个字节、4个字节和8个字节的无符号数**    
**表：是由多个无符号数或者其他表作为数据项构成的复合数据类型。所有的表都习惯以info结尾。**

### 2. 开始一个字节一个字节的认识Class文件
Class文件是一组以8字节为基础单位的二进制流，各个数据项目严格按照顺序紧凑的排列在Class文件中，中间没有添加任何分隔符。
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e0889b94a9354155be0a76d2f5bc5206~tplv-k3u1fbpfcp-watermark.image)
#### 2.1 魔数(magic)
上面的Class文件中前四个字节，也就是图中第一个u4,对应的值为："CA FE BA BE" 这4个字节称为魔数。   
它的作用是确定这个文件是否为一个能被虚拟机接受的Class文件。很多文件存储标准中都使用魔数来进行身份识别。

#### 2.2 Class文件的版本号(Minor Version、Major Version)
紧接着魔数的两个u2，是Class文件的版本号，也就是"CA FE BA BE"后面的"00 00 00 34",前两个“00 00”次版本号，后两个“00 34”
是主版本号，这个Class文件是由十六进制的Hex打开并查看的，因此，十六进制的0x0034转换成十进制结果是52，52对应的JDK版本号是1.8
下面这个表格是JDK各个版本号对应的Class版本号和16进制数值。
|  JDK版本号   | Class版本号  | 16进制数值  |
|  ----  | ----  | ----  |
| 1.1  | 45 |00 00 00 2D |
| 1.2  | 46 |00 00 00 2E |
| 1.3  | 47 |00 00 00 2F |
| 1.4  | 48 |00 00 00 30 |
| 1.5  | 49 |00 00 00 31 |
| 1.6  | 50 |00 00 00 32 |
| 1.7  | 51 |00 00 00 33 |
| 1.8  | 52 |00 00 00 34 |

#### 2.3 常量池
紧接着版本号后面的是常量池入口。常量池，可以理解为Class文件中的资源仓库。紧接这版本号后面的第一个u2,代表的是常量池容量的计数值(constant_pool_count)，在上面的Class文件中就是"00 14",这个容量计数是从1开始计数的，0x0014,转换成十进制就是20，这就代表常量池中有19个常量。   
常量池中主要存放两大类常量：字面量和符号引用。字面量通俗来讲就是 表达式中 “=” 右边的值，比如说int i = 3，3就是字面量。说官方一点：字面量比较接近于Java语言中的常量的概念，如字符串，声明为final的常量值等。符号引用包含了下面三部分的常量：
> 类和接口的全限定名（如 java/lang/Object）   
> 字段的名称和描述符（如private/public等）   
> 方法的名称和描述符（如private/public等）  

接下来，开始分析常量池中的内容。下图中紫色的选中的部分都是常量池中的内容。
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/abba4c63e1d14fd28dfbad89f704311a~tplv-k3u1fbpfcp-watermark.image)

分析常量池中的内容需要借助两个表，第一个表格是常量池的项目标志
|  编号  |  类型   | 标志  | 描述  |
|  :----:| :----:  | :----: | :----:  |
| 1 | CONSTANT_Utf8_info  | 1 |UTF-8编码字符串 |
| 2 | CONSTANT_Integer_info  | 3 |整型字面量 |
| 3 | CONSTANT_Float_info  | 4 |浮点型字面量 |
| 4 | CONSTANT_Long_info  | 5 |长整型字面量 |
| 5 | CONSTANT_Double_info  | 6 |双精度浮点型字面量 |
| 6 | CONSTANT_Class_info  | 7 |类或接口的符号引用 |
| 7 | CONSTANT_String_info  | 8 |字符串类型字面量 |
| 8 | CONSTANT_Fieldref_info  | 9 |字段的符号引用 |
| 9 | CONSTANT_Methodref_info  | 10 |类中方法的符号引用 |
| 10 | CONSTANT_InterfaceMethodref_info  | 11 |接口中方法的符号引用 |
| 11 | CONSTANT_NameAndType_info  | 12 |字段或方法的部分符号引用 |
| 12 | CONSTANT_MethodHandle_info  | 15 |标识方法句柄 |
| 13 | CONSTANT_MethodType_info  | 16 |标识方法类型 |
| 14 | CONSTANT_InvokeDynamic_info  | 18 |动态方法调用点 |

之所以说常量池是最繁琐的数据，是因为这14种常量类型各自均有自己的结构，这就要涉及到第二张表：常量池中常量项的结构表
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/19f1cb8d87c44d9f9e99df33606b1ac5~tplv-k3u1fbpfcp-watermark.image)

有了这两张表，就可以根据表中的内容，分析Class文件中的常量池内容。首先根据Class文件中的十六进制数据分析。

```
00 14 -> 常量池容量计数值，从1开始计数
0a  -> 转换成十进制的值10，查找表一中标志为10的是CONSTANT_Methodref_info，类中方法的符号引用。
       再根据表二中的内容，查找到CONSTANT_Methodref_info，
       它是由一个u1,两个u2组成，第一个u1,就是标志位，也就是0a，再在Class文件中往后找两个u2，分别是
       00 05和00 10 这两个值的类型都是index，解释分别是指向生命方法的类描述符CONSTANT_Class_info的        索引项和指向字段描述符CONSTANT_NameAndType_info的索引项。
     00 05 转换成十进制是5 记为#5
     00 10 转换成十进制是16 记为#16 
紧接着分析下一个：
07 -> 转换成十进制的值7，找到表一中标志为7的是CONSTANT_Class_info，类或接口的符号引用。
      再根据表二的内容找到CONSTANT_Class_info，发现它是由一个u1、一个u2构成，u1是标志位07，再往后找       一个u2，也就是00 11，这个值是一个index，它的意思是指向全限定名常量项的索引。
   00 11 转换成十进制是17，记为#17
   
....常量池中接下来的数据都按照上面这个方式分析。   
      
```
分析到这里，我们知道了怎么分析Class文件中常量池中的一个常量，但是问题也来了，分析出来的#5、#16、#17分别代表什么意思呢？ 这样分析很麻烦，其实，JDK早就为我们准备好了一个专门用于分析Class文件的字节码工具javap，这里要使用到javap -v这个指令 输出Class文件的字节码内容。
```
javap -v Person.class
```
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8e9e83e052aa4101ae9c369272a04147~tplv-k3u1fbpfcp-watermark.image)
上图中红色框框框起来的是方法体的内容，蓝色框框框起来的是常量池的内容。下面分别来看

首先分析第一个方法Person()，默认的构造方法Person()，分析在执行方法时，是怎么去使用常量池中的数据的。
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f10ece723b724e75926322eac9dd9d9e~tplv-k3u1fbpfcp-watermark.image)

然后分析一下main()方法中
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0f53400e7f204b33b2b61f92051d2e27~tplv-k3u1fbpfcp-watermark.image)

至于work()方法，其中都是一些基本类型的数据计算，涉及到的都是一些局部变量，局部变量是保存在虚拟机栈中局部变量表中，这部分内容在上面已经讲解过了，因此其中并没有与常量池相关的值，这里就不做分析了。

常量池中，主要就是存放了一些方法名，类名，返回值名称，返回值类型等，这些数据被虚拟机作为一种元数据（也就是描述类的数据）存放在常量池中。

##### 怎么判断常量池的结束位置在哪里呢？
看常量池中最后一个值，#19,19号常量是一个uft8类型，java/lang/Object,在我们的16进制Class文件中找一下，就可以找到是在这里：
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1282672c70014a39b54f3db7647e21db~tplv-k3u1fbpfcp-watermark.image) 
查看表二中utf8类型的常量的数据类型：
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8ef3899f76fa48d2a02f2b6bd4b40dff~tplv-k3u1fbpfcp-watermark.image)
在字符串的前面，是一个u2类型的描述字符串长度的无符号数，再往前是一个标志位。也就是前面的"01 00 10"."01"是标志位，代表接下来的常量是一个uft8编码的字符串，"00 10"是这串字符串的长度，单位是字节，这里转换成10进制是16字节，接下来是长度为16的utf8的字符串内容：java/lang/object。
这里就是常量池的结尾。接下来分析紧随常量池之后的访问标志。

#### 2.4 访问标志（类访问标志）
紧接着常量池后面的一个u2，代表的是访问标志（access_flags），这个标志用于识别一些类或接口层次的访问信息。包括：这个Class是类还是接口、是否为Public类型、是否定义为abstract类型等。具体的标志位含义见下表
|  标志名称  |  标志值   | 含义  | 
|  :----| :----:  | :---- | 
| ACC_PUBLIC | 0x0001	  | 是否为Public类型|
| ACC_FINAL | 0x0010  | 是否被声明为final，只有类可以设置 |
| ACC_SUPER | 0x0020 | 是否允许使用invokespecial字节码指令的新语义，在JDK1.0.2之后都为真|
|ACC_INTERFACE | 0x0200  | 标志这是一个接口 |
|ACC_ABSTRACT | 0x0400  | 是否为abstract类型，对于接口或者抽象类来说，次标志值为真，其他类型为假|
|ACC_SYNTHETIC | 0x1000  | 标志这个类并非由用户代码产生|
|ACC_ANNOTATION | 0x2000  | 标志这是一个注解 |
|ACC_ENUM  | 0x4000  | 标志这是一个枚举 |

这个值在上面16进制的Class文件中，是"00 21"，这个值并不在上面的表中，其实这个值应该是
```
0x0001 | 0x0020 = 0x0021
或运算，需要转换成二进制来运算：
0000 0001 | 0010 0000 = 0010 0001 
结果是：00100001转换成十六进制：21。
```
Person类，是public的，因此0x0001为真，使用JDK1.0.2之后的编译器进行编译，所以0x0020为真。计算出来的值就是0x0021。


附表： 二进制和十六进制的转换
|  二进制  |  十六进制   | 
|  :----| :----:  | 
| 0000 | 0	  | 
| 0001 | 1    | 
| 0010 | 2    | 
| 0011 | 3    | 
| 0100 | 4    |
| 0101 | 5    |
| 0110 | 6    |
| 0111 | 7    |
| 1000 | 8    |
| 1001 | 9    |
| 1010 | A    |
| 1011 | B    |
| 1100 | C    |
| 1101 | D    |
| 1110 | E    |
| 1111 | F    |

#### 2.5 类索引、父类索引、接口索引集合
类索引和父类索引是一个u2类型的数据，接口索引集合是一组u2类型的数据的集合，Class文件中由这三项数据来确定这个类的继承关系。类索引用于确定这个类的全限定名，父类索引用来确定这个类的父类的全限定名。在上面的16进制Class文件中，紧跟着访问标志之后的两个u2就是类索引和父类索引，分别是"00 02"和"00 05",对应常量池中的#2常量和#5常量，分别是：Person和java/lang/Object
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b092b2404e8443a98b1b4747444e08ea~tplv-k3u1fbpfcp-watermark.image)

Java不允许多继承，所以父类索引只有一个，除了java.lang.Object之外，所有的Java类都有父类，因此除了java.lang.Object外，所有的Java类的父类索引都不为0.

对于接口索引集合，入口的第一项u2,是接口计数器，表示索引表的容量。如果该类没有实现任何接口，则该索引计数器值为0，后面接口的索引表不再占用任何字节。上面的演示代码，没有继承任何接口，所以接口索引集合中的第一项u2，值就为"00 00"，如果把代码稍微改一下，重新编译之后，会得到下面的结果。
```
public abstract class Person implements Comparable{

  }
```
得到的16进制Class文件是：
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/732fe08b044d462db2952cf97dfd4ae5~tplv-k3u1fbpfcp-watermark.image)
上图中标紫色的位置，就是接口索引集合，其中第一个u2："00 01",是接口计数器，实现了1个接口；接下来的u2："00 04"，对应的是常量池中的#4常量，也就是下图中的红框框出来的部分。
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b9572024d6e849b4ae76f48d9ba5f514~tplv-k3u1fbpfcp-watermark.image)

#### 2.5 字段表集合
紧接着接口集合的是字段表，字段表用于描述接口或者是类中声明的变量。字段包括类级变量以及实例级变量，但不包括在方法内部声明的局部变量。字段表的结构如下图所示:
|  类型  |  名称   | 数量  | 含义 |
|  :----:| :----:  | :----: | :---- |
| u2 | access_flags     | 1 | 字段修饰符 |
| u2 | name_index  | 1 | 字段的简单名称索引 |
| u2 | descriptor_index | 1 | 字段和方法的描述符索引 |
|u2 | attributes_count  | 1 | 属性表长度 |
|attribute_info | attributes  | 1| 属性表 |

把上面的代码改一改，添加一个全局变量，再来看Class文件
```
public abstract class Person implements Comparable{
   private int i;
  }
```
编译上面那段代码，得到下图所示的结果
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/956cbbd9bfbd4b2c87487cbde7066881~tplv-k3u1fbpfcp-watermark.image)

第一个u2： field_count，容量计数器，记录这个类有多少个字段表数据。紧随其后的第二个u2，是字段访问标志，和类的访问标志含义一样，具体的意思在下表中描述的很清楚。
|  标志名称  |  标志值   | 含义  | 
|  :----| :----:  | :---- | 
| ACC_PUBLIC | 0x0001	  | 字段是否为Public|
| ACC_PROVATE | 0x0002	  | 字段是否为private|
| ACC_PROTECTED | 0x0004	  | 字段是否为protected|
| ACC_STATIC | 0x0008	  | 字段是否为static|
| ACC_FINAL | 0x0010  | 字段是否是final |
| ACC_VOLATILE | 0x0040  | 字段是否volatie |
| ACC_TRANSIENT | 0x0080  | 字段是否是transient |
|ACC_SYNTHETIC | 0x1000  | 字段是否由编译器自动产生的|
|ACC_ENUM  | 0x4000  | 字段是否enum |

上面的例子中，字段访问标志是"00 02"，说明该字段是private的，紧随其后的简单名称索引，指向了常量池中的#5常量，后面的是方法和字段描述符索引，指向了常量池中的#6常量。
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a75f6b178d3545298505f44213317053~tplv-k3u1fbpfcp-watermark.image)

其中的#5号常量是一个utf8的字符串i，是字段的简单名称，什么是简单名称呢？   
**简单名称**
简单名称指的是没有类型和参数修饰的方法或者字段名称，如i字段的简单名称就是i。   
#6号常量也是一个utf8类型的字符串I，这个I是什么意思呢？这个u2含义是方法和字段的描述符的索引，描述符的作用是用来描述字段的数据类型、方法参数列表（包括数量、类型和顺序）和返回值，根据描述符规则，基本数据类型（byte、char、double、float、int、long、short、boolean）以及代表无返回值的void类型都用一个大写字符来表示，而对象类型则用字符L加对象的全限定名来表示。如下表所示
|  标识字符  |  含义   | 
|  :----:| :----  | 
| B | 基本类型byte	 | 
| C | 基本类型char    | 
| D | 基本类型double    | 
| F | 基本类型float    | 
| I | 基本类型int    | 
| J | 基本类型long    | 
| S | 基本类型shory    | 
| Z | 基本类型boolean    | 
| V | 特殊类型Void    | 
| L | 对象类型，如Ljava/lang/Object  | 

对于数组类型，每一个维度将使用一个前置的 "[" 字符来描述，如一个定义为"java.lang.string[][]"类型的数组，将被记为"[[Ljava/lang/String;",一个整数数组"int[]"将被记为"[I"。

用描述符来描述方法时，按照先参数列表，后返回值的顺序描述，参数列表按照参数的严格顺序放在一组小括号中，如方法返回值是"Void",那么描述符就是"()V",方法java.lang.String toString()的描述符为“()Ljava/lang/String”,方法int indexOf(char[]source,int sourceOffset,int sourceCount,char[]target,int targetOffset,int targetCount,int fromIndex)的描述符为“([CII[CIII)I”。

清楚了描述符的具体含义，再结合前面的分析，就可以翻译Class文件中的内容，上面的“00 01 00 02 00 05 00 06”翻译过来就是 private int i;

字段表包含的固定数据项目，到这里为止就结束了。不过在方法和字段描述符索引之后跟随着一个属性表集合，用于存储一些额外的信息，这些信息，会在后面介绍属性表的时候详细讲解。

#### 2.6 方法表集合
紧随字段表之后的，是方法表集合，Class中对方法的描述喝对字段的描述几乎是一致的，方法表的结构和字段表一样。都是访问标志+名称索引+描述符索引构成。这三者中唯一的不同是访问标志，比如volatile关键字就不能修饰方法，方法访问标志在下表中进行了总结。

|  标志名称  |  标志值   | 含义  | 
|  :----| :----:  | :---- | 
| ACC_PUBLIC | 0x0001	  | 方法是否为Public|
| ACC_PROVATE | 0x0002	  | 方法是否为private|
| ACC_PROTECTED | 0x0004	  | 方法是否为protected|
| ACC_STATIC | 0x0008	  | 方法是否为static|
| ACC_FINAL | 0x0010  | 方法是否是final |
| ACC_SYNCHRONIZED | 0x0020  | 方法是否为synchronized |
| ACC_BRIDGE | 0x0040  | 方法是否是由编译器产生的桥接方法 |
| ACC_VARARGS | 0x0080  | 方法是否接受不定参数 |
|ACC_NATIVE | 0x0100  | 方法是否是native|
|ACC_ABSTRACT  | 0x0400  | 方法是否是abstract |
|ACC_STRICTFP | 0x0800  | 方法是否为strictfp|
|ACC_SYNTHETIC  | 0x1000  | 方法是否是由编译器自动产生的 |

方法中的Java代码,通过编译器编译成字节码指令后,存放在方法属性表集合中的Code属性里面,有关"code"的内容,会在下一节中详细讲解.

#### 2.7 属性表
Class文件，字段表，方法表都可以带有自己的属性表集合，以用于描述某些场景专有的信息。Java虚拟机规范中预定义了一些虚拟机能识别的属性，如下表所示。
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/76a527b69a454a25b036bf227ca673d4~tplv-k3u1fbpfcp-watermark.image)
**属性表的结构：**   
对于属性表中的每个属性，都有自己的结构，它的名称需要从常量池中引用一个utf8类型的常量来表示，而属性值的结构则完全自定义，只需要通过一个u4长度属性去说明属性值所占用的位数即可。属性表的结构如下表所示：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/68cfb1c5d32e43709e272a946aba80f5~tplv-k3u1fbpfcp-watermark.image)
具体分析：看下面一段代码
```
public class Test{

	final static long m = 1L;
	static int n = 2;
	static int i = 2;
    
	public void desc(){}
    
	public int inc(){
		int x;
		try{
			x = 1;
			return x;
		}catch(Exception e){
			x = 2;
			return x;
		}finally{
			x = 3;
			return x;
		}
	}

  }
```
编译成Class文件：
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/367cfe2c30354ed0acce1b9ff9ba28c5~tplv-k3u1fbpfcp-watermark.image)
##### 2.7.1 Code属性
Java程序方法体中的代码经过Javac编译器处理之后，最终变成字节码指令存储在Code属性内。Code属性出现在方法表中的集合中，但并非所有的方法表都必须存在这个属性，接口或者抽象类中的方法就不存在Code属性，如果方法表中有Code属性存在，那么它的结构如下所示：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4b4675e53ff4e82a5f4f11b67b4026d~tplv-k3u1fbpfcp-watermark.image)

1. attribute_name_index是一项指向CONSTANT_Utf8_info型常量的索引，常量值固定为Code。
2. attribute_length指示了属性的长度。
3. max_stack代表了操作数栈深度的最大值，在方法执行的任意时刻，操作数栈不会大于这个深度。
4. max_loca在ls代表了局部变量所需要的存储空间。max_locals单位是slot，slot是虚拟机为局部变量分配内存所使用的最小单位。对于byte,char,float,int,short,boolean,reference,return Address等长度不超过32位的数据类型，每个局部变量使用1个slot，而double和long这两种64位数据类型则使用2个slot。注意，slot可以重用，当代码执行超出一个局部变量的作用域时，这个局部变量所占用的slot就可以被其它局部变量使用。
5. code_length和code用于存储Java源程序编译后生成的字节码指令。code_length代表字节码长度，code用于存储字节码指令的一系列字节流。code由u1表示，虚拟机讲到一个字节码时就知道怎么理解，后续带什么参数等等。u1的取值是0到255，也就是说一共可以表达255条指令。
6. code有点类似cpu上的指令集，例如+号会被编译成 iadd 虚拟机字节码指令。
7. code_length由一个u4表示，理论上最大值是232-1，但虚拟机规范中限制方法不能超过65535。


这段代码中有三个方法：构造方法、desc()、inc()，通过分析代码最多的inc()方法来认识属性表。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c7669955da0d4b519066b997cf7c7e75~tplv-k3u1fbpfcp-watermark.image)
上图中，用紫色标出来的是inc()的方法表内容。需要结合Javap工具显示的常量池和方法体的内容进行分析，具体的解释在右侧使用红字都标注清楚了。   
使用Javap查看常量池的内容：上面的字节码需要和常量池里面的内容对照。
![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7eca55fbab3442d8ad4076a3a6f1eefb~tplv-k3u1fbpfcp-watermark.image)

使用Javap查看inc方法：上面图上的字节码分析，需要和方法中的字节码进行对照。（有关字节码的内容，会在后面的章节详细分析）

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8da7952aaef94a6a81007c7dd90a0413~tplv-k3u1fbpfcp-watermark.image)


这里有几个需要注意的点：   
max_statcks，最大栈深，在这个方法中，每调用一个外界方法就会产生一个栈帧，这些内容在前面已经讲过了，inc()没有调用其他方法，所以最大栈深是1。    
max_locals,方法的本地变量数。
args_size = 1, 方法的参数，非static方法，默认有一个参数 this。

在属性表结构的图中，可以清晰的看到，code之后的u2是异常表的长度，再往后是异常表的内容。认识下异常表中的结构：
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ab7d4439d8645029f40268ccc18d9c3~tplv-k3u1fbpfcp-watermark.image)
结合Class文件来分析异常表：
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/394492b51a1e4761899e5bc566546420~tplv-k3u1fbpfcp-watermark.image)
具体的解释在图中右边已经标注的很清楚了。

##### 2.7.2 Exceptions属性
Exceptions是在方法表中与Code属性平级的一项属性，与异常表不一样，异常表是Code的下级属性。Exceptions属性的作用是列举出方法中可能抛出的受查异常，也就是 throws 关键字后列表的异常，它的结构表如下：
![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/696e95f162204f0ab1c73e7fd00051d3~tplv-k3u1fbpfcp-watermark.image)

也就是紧跟着Code属性之后的属性。number_of_exceptions项表示方法有可能抛出多少种异常，每一种异常由一个exception_index_table项表示，exception_index_table是一个指向常量池中 utf8 类型的索引。

##### 2.7.3 LineNumberTable属性
LineNumberTable属性用于描述 Java 源码行号与字节码行号 之间的对应关系。它不是必须属性，如果不生成它，那么产生异常后，堆栈中将不会显示出错的行号，并且在调试时也无法在源码中设置断点。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/be98bbc453d9469b884a64a6ad2c3243~tplv-k3u1fbpfcp-watermark.image)
line_number_table是一个数量为line_number_table_length、类型为line_number_info的集合，line_number_info表包括了start_pc和line_number两个u2数据项，前者是字节码行号，后者 是Java源码行号。

##### 2.7.4 LocalVariableTable属性
LocalVariableTable用于描述栈桢中局部变量表中的变量与Java源码中定义的变量之间的关系。也不是必须的，它的结构如下：
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/756ed5972ea146dd9a0103ab2d27991b~tplv-k3u1fbpfcp-watermark.image)
其中local_variable_info项目代表了一个栈桢与源码中局部变量的关联，结构如下
![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7e7e365cbe0e4345a1e6132f449f5a76~tplv-k3u1fbpfcp-watermark.image)
start_pc和length属性分别代表这个局部变量的生命周期的字节码偏移量及其作用范围覆盖的长度，两者结合起来就是这个局部变量在字节码之中的作用域范围。
name_index和descriptor_index都是指向常量池中 utf8 类型的常量，分别代表局部变量的名称及局部变量的描述符。
index是这个局部变量在栈桢局部变量表中slot的位置，如果是64位类型，则它占用的slot为index 和 index+1的两个位置。

##### 2.7.5 SourceFile属性
SourceFile属性，用于记录Class文件的源码文件名称。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0fc42421feed40b19fef7be83d3981ef~tplv-k3u1fbpfcp-watermark.image)


##### 2.7.6 其它属性
属性表中还有其它的属性：ConstantValue属性、InnerClasses属性等，就不再一一讲解了，查找Class文件的方法就是按图索骥，根据规定的每个表的结构，一个一个的查找，对比即可。具体可以看JVM经典书：《深入理解Java虚拟机》，里面会有很详细的讲解。













