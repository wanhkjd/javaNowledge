# JVM 面试题

> 整理说明：本文仅清理标题、编号、空行、列表缩进和图片排版，文章内容尽量保持原文不变。

## 一、JVM 基础

### 1. JVM是什么

Java Virtual Machine Java程序的运行环境（java二进制字节码的运行环境）

好处：
- 一次编写，到处运行

- 自动内存管理，垃圾回收机制


### 2. JVM由哪些部分组成，运行流程是什么？

![[java八股文/资源/JVM组成.PNG|496]]

- ClassLoader（类加载器）

- Runtime Data Area（运行时数据区，内存分区）

- Execution Engine（执行引擎）

- Native Method Library（本地库接口）


运行流程：

（1）类加载器（ClassLoader）把Java代码转换为字节码

（2）运行时数据区（Runtime Data Area）把字节码加载到内存中，而字节码文件只是JVM的一套指令集规范，并不能直接交给底层系统去执行，而是有执行引擎运行

（3）执行引擎（Execution Engine）将字节码翻译为底层系统指令，再交由CPU执行去执行，此时需要调用其他语言的本地库接口（Native Method Library）来实现整个程序的功能。

### 3. 什么是程序计数器？

程序计数器是线程私有的(线程安全)，每个线程一份，内部存储的是字节码的行号(记录上一次该线程执行到哪个行号了)，用于记录正在运行的字节码指令的地址。程序计数器不存在OOM，不会被GC回收。

### 4. 你能给我详细的介绍Java堆吗?

- 线程共享的区域：主要用来保存对象实例，数组等，当堆中没有内存空间可分配给实例，也无法再扩展时，则抛出OutOfMemoryError异常。

- 年轻代被划分为三部分，Eden区和两个大小严格相同的Survivor区，根据JVM的策略，在经过几次垃圾收集后，任然存活于Survivor的对象将被移动到老年代区间。

- 老年代主要保存生命周期长的对象，一般是一些老的对象

- 元空间保存的类信息、静态变量、常量、编译后的代码

![[java八股文/资源/jvm内存结构.png|655]]

- 为了避免方法区出现OOM，所以在java8中将堆上的方法区【永久代】给移动到了本地内存上，重新开辟了一块空间，叫做**元空间**。那么现在就可以避免掉OOM的出现了。
-  元空间(MetaSpace)介绍
- 在 HotSpot JVM 中，永久代（ ≈ 方法区）中用于存放类和方法的元数据以及常量池，比如Class 和 Method。每当一个类初次被加载的时候，它的元数据都会放到永久代中。
- 永久代是有大小限制的，因此如果加载的类太多，很有可能导致永久代内存溢出，即OutOfMemoryError
- 元空间的本质和永久代类似，都是对 JVM 规范中方法区的实现。不过元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。因此，默认情况下，元空间的大小仅受本地内存限制。

### 5. 什么是虚拟机栈

- Java Virtual machine Stacks (java 虚拟机栈)

- 每个线程运行时所需要的内存，称为虚拟机栈，先进后出

- 每个栈由多个栈帧（frame）组成，对应着每次方法调用时所占用的内存

- 每个线程只能有一个活动栈帧，对应着当前正在执行的那个方法

#### 相关问题

1. 垃圾回收是否涉及栈内存？

2. 垃圾回收主要指就是堆内存，当栈帧弹栈以后，内存就会释放

3. 栈内存分配越大越好吗？

4. 未必，默认的栈内存通常为1024k

5. 栈帧过大会导致线程数变少，例如，机器总内存为512m，目前能活动的线程数则为512个，如果把栈内存改为2048k，那么能活动的栈帧就会减半

6. 方法内的局部变量是否线程安全？

   1. 如果方法内局部变量没有逃离方法的作用范围，它是线程安全的

   2. 如果是局部变量引用了对象，并逃离方法的作用范围，需要考虑线程安全


**栈内存溢出情况**

- 栈帧过多导致栈内存溢出，典型问题：递归调用

- 栈帧过大导致栈内存溢出

### 12. 说一下 JVM 运行时数据区


组成部分：堆、方法区、栈、本地方法栈、程序计数器

1、堆解决的是对象实例存储的问题，垃圾回收器管理的主要区域。

2、方法区可以认为是堆的一部分，用于存储已被虚拟机加载的信息，常量、静态变量、即时编译器编译后的代码。

3、栈解决的是程序运行的问题，栈里面存的是栈帧，栈帧里面存的是局部变量表、操作数栈、动态链接、方法出口等信息。

4、本地方法栈与栈功能相同，本地方法栈执行的是本地方法，一个Java调用非Java代码的接口。

5、程序计数器（PC寄存器）程序计数器中存放的是当前线程所执行的字节码的行数。JVM工作时就是通过改变这个计数器的值来选取下一个需要执行的字节码指令。

### 13. 能不能解释一下方法区？

- 方法区(Method Area)是各个线程共享的内存区域

- 主要存储类的信息、运行时常量池

- 虚拟机启动的时候创建，关闭虚拟机时释放

- 如果方法区域中的内存无法满足分配请求，则会抛出OutOfMemoryError: Metaspace

### 14. 常量池

- 可以看作是一张表，虚拟机指令根据这张常量表找到要执行的类名、方法名、参数类型、字面量等信息
- 查看字节码结构（类的基本信息、常量池、方法定义）`javap -v xx.class`
- 找到类对应的class文件存放目录，执行命令：`javap -v Application.class` 查看字节码结构
- main方法按照指令执行的时候，需要到常量池中查表翻译找到具体的类和方法地址去执行


### 15. 运行时常量池

常量池是 *.class 文件中的，当该类被加载，它的常量池信息就会放入运行时常量池，并把里面的符号地址变为真实地址

### 16. 你听过直接内存吗？


难易程度：☆☆☆

出现频率：☆☆☆

不受 JVM 内存回收管理，是虚拟机的系统内存，常见于 NIO 操作时，用于数据缓冲区，分配回收成本较高，但读写性能高，不受 JVM 内存回收管理

### 17. 传统的IO操作

- java代码发起读请求后，Java 虚拟机（JVM）将控制权交给操作系统，通过系统调用进入内核态，操作系统将数据从磁盘读入系统缓存区，再拷贝到Java堆内存中的缓冲区，操作系统完成数据拷贝后，将控制权交还给 JVM，最后切回用户态由CPU处理数据。

![[java八股文/资源/传统的IO操作.png]]

### 18. NIO操作

![[java八股文/资源/NIO.png]]

NIO传输数据的流程，在这个里面主要使用到了一个直接内存，不需要在堆中开辟空间进行数据的拷贝，jvm可以直接操作直接内存，从而使数据读写传输更快。

### 19. 堆栈的区别是什么？

1、栈内存一般会用来存储局部变量和方法调用，但堆内存是用来存储Java对象和数组的的。堆会GC垃圾回收，而栈不会。

2、栈内存是线程私有的，而堆内存是线程共有的。

3,、两者异常错误不同，但如果栈内存或者堆内存不足都会抛出异常。

栈空间不足：java.lang.StackOverFlowError。

堆空间不足：java.lang.OutOfMemoryError。

### 20. 什么是类加载器，类加载器有哪些?

![[java八股文/资源/类加载器.png]]

- 类加载器：用于装载字节码文件(.class文件)

- 运行时数据区：用于分配存储空间

- 执行引擎：执行字节码文件或本地方法

- 垃圾回收器：用于对JVM中的垃圾内容进行回收

**类加载器**

JVM只会运行二进制文件，而类加载器（ClassLoader）的主要作用就是将**字节码文件加载到JVM中**，从而让Java程序能够启动起来。现有的类加载器基本上都是java.lang.ClassLoader的子类，该类的只要职责就是用于将指定的类找到或生成对应的字节码文件，同时类加载器还会负责加载程序所需要的资源

**类加载器种类**

类加载器根据各自加载范围的不同，划分为四种类加载器：

- **启动类加载器(BootStrap ClassLoader)：**

- 该类并不继承ClassLoader类，其是由C++编写实现。用于加载**JAVA_HOME/jre/lib**目录下的类库。

- **扩展类加载器(ExtClassLoader)：**

- 该类是ClassLoader的子类，主要加载**JAVA_HOME/jre/lib/ext**目录中的类库。

- **应用类加载器(AppClassLoader)：**

- 该类是ClassLoader的子类，主要用于加载**classPath**下的类，也就是加载开发者自己编写的Java类。

- **自定义类加载器：**

- 开发者自定义类继承ClassLoader，实现自定义类加载规则。

![[java八股文/资源/类加载器委派体系.png]]

### 21. 什么是双亲委派模型？

如果一个类加载器在接到加载类的请求时，它首先不会自己尝试去加载这个类，而是把这个请求任务委托给父类加载器去完成，依次递归，如果父类加载器可以完成类加载任务，就返回成功；只有父类加载器无法完成此加载任务时，才由下一级去加载。

### 22. JVM为什么采用双亲委派机制

（1）通过双亲委派机制可以避免某一个类被重复加载，当父类已经加载后则无需重复加载，保证唯一性。

（2）为了安全，保证类库API不会被修改

在工程中新建java.lang包，接着在该包下新建String类，并定义main函数

```Java
public class String {

     public static void main(String[] args) {

         System.out.println("demo info");
     }
 }
```

此时执行main函数，会出现异常，在类 java.lang.String 中找不到 main 方法

```Java
错误：在类java.lang.string中找不到main方氵去，请将main方法定义为：PUbtiCstaticVOidmain(String[]args)否则JavaFX应用程序类必须扩展javafx.apptication.App1ication
```

出现该信息是因为由双亲委派的机制，java.lang.String的在启动类加载器(Bootstrap classLoader)得到加载，因为在核心jre库中有其相同名字的类文件，但该类中并没有main方法。这样就能防止恶意篡改核心API库。

![[java八股文/资源/类加载流程.png]]

### 23. 说一下类装载的执行过程？

类从加载到虚拟机中开始，直到卸载为止，它的整个生命周期包括了：加载、验证、准备、解析、初始化、使用和卸载这7个阶段。其中，验证、准备和解析这三个部分统称为连接（linking）。
- 1．加载，通过类的全限定名获取字节码文件，并将其转换为方法区内的运行时数据结构。
- 2．验证，对字节码进行校验，确保符合java虚拟机规范。
- 3．准备，为类的静态变量分配内存，并设置默认初始值。
- 4．解析，将符号引用转换为直接引用，即将类、方法、字段等解析为具体的内存地址。5．初始化，执行类的初始化代码，包括静态变量赋值和静态代码块的执行。
- 6.  使用,JVM开始从入口方法开始执行用户的程序代码
- 7.  卸载：当用户程序代码执行完毕后，JVM便开始销毁创建的Class对象

![[java八股文/资源/类加载流程.png]]

## 二、垃圾回收

### 1. 简述Java垃圾回收机制？（GC是什么？为什么要GC）

- GC是什么？
GC 就是 Java 的垃圾回收机制（Garbage Collection），由 JVM 自动管理内存，负责回收程序中不再使用的对象所占用的堆内存。

- 为什么需要 GC：

避免内存泄漏和内存溢出：如果对象不回收，内存会越占越多，最终导致 OOM。
解放开发者：不需要像 C/C++ 那样手动 free、delete，减少手动管理内存的错误。
提高开发效率与安全性：降低野指针、重复释放、内存泄漏等问题。

### 2. 对象什么时候可以被垃圾器回收

如果一个或多个对象没有任何的引用指向它了，那么这个对象现在就是垃圾，如果定位了垃圾，则有可能会被垃圾回收器回收。
有两种方式来确定垃圾，
第一个是引用计数法，
第二个是可达性分析算法

- 引用计数法：
  - 一个对象被引用了一次，在当前的对象头上递增一次引用次数，如果这个对象的引用次数为0，代表这个对象可回收
优点：
  - 实时性较高，无需等到内存不够的时候，才开始回收，运行时根据对象的计数器是否为0，就可以直接回收。
  - 在垃圾回收过程中，应用无需挂起。如果申请内存时，内存不足，则立刻报OOM错误。
  - 区域性，更新对象的计数器时，只是影响到该对象，不会扫描全部对象。
缺点：
  - 每次对象被引用时，都需要去更新计数器，有一点时间开销。
  - **浪费CPU资源**，即使内存够用，仍然在运行时进行计数器的统计。
  - **无法解决****循环引用****问题，会引发****内存泄露**。（最大的缺点）
- 可达性分析算法
  - 算法以 GC Roots 为起点，沿着引用链逐层向下遍历：从根节点找到直接引用的对象，再从这些对象出发寻找它们的引用对象，递归延伸，直到所有可达的对象都被标记完毕。
  可以充当 GC  ROOTS的对象有哪些
  1. 虚拟机栈种引用的对象
  2. 方法去种类静态属性引用的对象
  3. 方法区中常量引用的对象
  4. 本地方法栈中本地方法应用的对象


### 3. JVM 垃圾回收算法有哪些？

**标记清除算法**
- 分为标记和清除两个核心阶段
  - 阶段1:标记(Mark)-标记存活对象
1.起点:从所有GC Roots(虚拟机栈局部变量、静态变量、活跃线程等)出发，遍历整个对象引用链;
2.动作:给每一个可达的存活对象，打上标记(比如在对象头中设置一个标记位);
3.结果:内存中被标记的对象=存活对象;未被标记的对象=可回收对象(无任何GCRoots可达)
  - 阶段2:清除(Sweep)-清除未标记的可回收对象
1.动作:垃圾回收器遍历整个堆内存，释放所有未被标记的对象占用的内存空间;
2.结果:释放的内存会被记录为"空闲内存块”，等待后续分配新对象。
缺点：
- 效率较低，**标记和清除两个动作都需要遍历所有的对象**，并且在GC时，**需要停止应用程序**，对于交互性要求比较高的应用而言这个体验是非常差的。

  - （**重要**）通过标记清除算法清理出来的内存，碎片化较为严重，因为被回收的对象可能存在于内存的各个角落，所以清理出来的内存是不连贯的。

   **标记整理算法**
1）标记垃圾。

2）需要清除向右边走，不需要清除的向左边走。

   3）清除边界以外的垃圾。
- 优缺点同标记清除算法，解决了标记清除算法的碎片化的问题，同时，标记压缩算法多了一步，对象移动内存位置的步骤，其效率也有有一定的影响。
- 与复制算法对比：复制算法标记完就复制，但标记整理算法得等把所有存活对象都标记完毕，再进行整理
 **复制算法**
- 复制算法的核心就是，**将原有的内存空间一分为二，每次只用其中的一块**，在垃圾回收时，将正在使用的对象复制到另一个内存空间中，然后将该内存空间清空，交换两个内存的角色，完成垃圾的回收。
- 1）将内存区域分成两部分，每次操作其中一个。
2）当进行垃圾回收时，将正在使用的内存区域中的存活对象移动到未使用的内存区域。当移动完对这部分内存区域一次性清除。
3）周而复始。
优点：
在垃圾对象多的情况下，效率较高
清理后，内存无碎片
缺点：
分配的2块内存空间，在同一个时刻，只能使用一半，内存使用率较低

### 4. 分代收集算法

在java8时，堆被分为了两份：**新生代和老年代【1：2】**，在java7时，还存在一个永久代。

对于新生代，内部又被分为了三个区域。Eden区，S0区，S1区【8：1：1】

当对新生代产生GC：MinorGC【young GC】

当对老年代代产生GC：Major GC

当对新生代和老年代产生FullGC： 新生代 + 老年代完整垃圾回收，暂停时间长，**应尽力避免**

#### 工作机制


![](https://heuqqdmbyk.feishu.cn/space/api/box/stream/download/asynccode/?code=MTY4MGE0MjNjNTNmYzI4N2I2NDhhNzE0NTJkNzBkMDNfSTRJT01wZVpiY2d2ZlpaOWtQZzFCSFR1T084OTZHZlhfVG9rZW46RGo3TGI1M1hTb1FjU2Z4TTY2VmN2WkUzbkdmXzE3ODE3MDI4NTU6MTc4MTcwNjQ1NV9WNA&add_watermark=true&scene_type=CCM)

- 新创建的对象，都会先分配到eden区


![](https://heuqqdmbyk.feishu.cn/space/api/box/stream/download/asynccode/?code=NjU0NTg0NmM1YmY4MTg5ZDNmOGY4Y2NlNzcxMTNhYjhfMWdOVEVrYVdRSjNLOGdKWnlqTU80enByRHFJMjRpTGVfVG9rZW46SGVUTmJFRzlkb01MV094cWNwTGNRYTZibkljXzE3ODE3MDI4NTU6MTc4MTcwNjQ1NV9WNA&add_watermark=true&scene_type=CCM)

- 当伊甸园内存不足，标记伊甸园与 from（现阶段没有）的存活对象

- 将存活对象采用复制算法复制到 to 中，复制完毕后，伊甸园和 from 内存都得到释放


![](https://heuqqdmbyk.feishu.cn/space/api/box/stream/download/asynccode/?code=NDg5NTYyZDlmZGUwNzJlYjhhNzA4ZjA1N2I0ODQzNTZfRjNuc1hXMHpBYWJJVkpHeXg5MXVuTk13RzF0cDVPTGpfVG9rZW46UG9XTGJEZDVtb1RveTZ4ZzN6ZGM2SkZHbk52XzE3ODE3MDI4NTU6MTc4MTcwNjQ1NV9WNA&add_watermark=true&scene_type=CCM)

- 经过一段时间后伊甸园的内存又出现不足，标记eden区域to区存活的对象，将存活的对象复制到from区


![](https://heuqqdmbyk.feishu.cn/space/api/box/stream/download/asynccode/?code=MmUxZDNmYmNmNDQ4MDFlZjFmNDAyOGIyNjk3OGM5ZWNfc0dHUTJFMGpzUnBaU2ZBZEtwUm9kcjlhUThSMG5TbGhfVG9rZW46VDhNTmI1WmZ2b3pnQWl4d3VVOWNLVTlLbkVjXzE3ODE3MDI4NTU6MTc4MTcwNjQ1NV9WNA&add_watermark=true&scene_type=CCM)

![](https://heuqqdmbyk.feishu.cn/space/api/box/stream/download/asynccode/?code=MWUxM2EyY2EzMDQ3MTRmMzNiMzc5OGU0ZWQxYzUzZTVfTTZJZWdnR3RzTHlrY2gzSGZPYnQ5Y0NQbHQwbzA5cXdfVG9rZW46TnhpYmJvWWVTb1lERXV4QU56Z2NQRUtIbktkXzE3ODE3MDI4NTU6MTc4MTcwNjQ1NV9WNA&add_watermark=true&scene_type=CCM)

- 当幸存区对象熬过几次回收（最多15次），晋升到老年代（幸存区内存不足或大对象会导致提前晋升）


**MinorGC、 Mixed GC 、 FullGC的区别是什么**

![](https://heuqqdmbyk.feishu.cn/space/api/box/stream/download/asynccode/?code=YWNmZDcyZDY3MzFjOTU5ODQyZWJjMGRmNDliNjg2ZDBfblVQcFByeXBTeEx6UWVUWjR6VWx0VHZ0andpZFNnMHFfVG9rZW46S1JUV2J3NW81b05Xc1R4Skp5R2NXU2JWbldoXzE3ODE3MDI4NTU6MTc4MTcwNjQ1NV9WNA&add_watermark=true&scene_type=CCM)

- MinorGC【young GC】发生在新生代的垃圾回收，暂停时间短（STW）

- Mixed GC 新生代 + 老年代部分区域的垃圾回收，G1 收集器特有

- FullGC： 新生代 + 老年代完整垃圾回收，暂停时间长（STW），应尽力避免？


STW（Stop-The-World）：暂停所有应用程序线程，等待垃圾回收的完成

### 5. 说一下 JVM 有哪些垃圾回收器？

在jvm中，实现了多种垃圾收集器，包括：

- 串行垃圾收集器

- 并行垃圾收集器

- CMS（并发）垃圾收集器

- G1垃圾收集器

#### 串行垃圾收集器

-  Serial和Serial Old串行垃圾收集器，是指使用单线程进行垃圾回收，堆内存较小，适合个人电脑
- Serial 作用于新生代，采用复制算法

- Serial Old 作用于老年代，采用标记-整理算法

垃圾回收时，只有一个线程在工作，并且java应用中的所有线程都要暂停（STW），等待垃圾回收的完成。

#### 并行垃圾收集器

- Parallel New和Parallel Old是一个并行垃圾回收器，**JDK8默认使用此垃圾回收器**

- Parallel New作用于新生代，采用复制算法

- Parallel Old作用于老年代，采用标记-整理算法

垃圾回收时，多个线程在工作，并且java应用中的所有线程都要暂停（STW），等待垃圾回收的完成。

#### CMS（并发）垃圾收集器

- CMS全称 Concurrent Mark Sweep，是一款并发的、使用标记-清除算法的垃圾回收器，该回收器是针对老年代垃圾回收的，是一款以获取最短回收停顿时间为目标的收集器，停顿时间短，用户体验就好。其最大特点是在进行垃圾回收时，应用仍然能正常运行。

#### 详细聊一下G1垃圾回收器

- 应用于新生代和老年代，**在JDK9之后默认使用G1**

- 划分成多个区域，每个区域都可以充当 eden，survivor，old， humongous，其中 humongous 专为大对象准备

- 采用复制算法

- 响应时间与吞吐量兼顾

- 分成三个阶段：新生代回收、并发标记、混合收集

- 如果并发失败（即回收速度赶不上创建新对象速度），会触发 Full GC

#### Young Collection(年轻代垃圾回收)


- 初始时，所有区域都处于空闲状态

- 创建了一些对象，挑出一些空闲区域作为伊甸园区存储这些对象

- 当伊甸园需要垃圾回收时，挑出一个空闲区域作为幸存区，用复制算法复制存活对象，需要暂停用户线程

- 随着时间流逝，伊甸园的内存又有不足

- 将伊甸园以及之前幸存区中的存活对象，采用复制算法，复制到新的幸存区，其中较老对象晋升至老年代


#### Young Collection + Concurrent Mark (年轻代垃圾回收+并发标记)


当老年代占用内存超过阈值(默认是45%)后，触发并发标记，这时无需暂停用户线程

- 并发标记之后，会有重新标记阶段解决漏标问题，此时需要暂停用户线程。

- 这些都完成后就知道了老年代有哪些存活对象，随后进入混合收集阶段。此时不会对所有老年代区域进行回收，而是根据暂停时间目标优先回收价值高（存活对象少）的区域（这也是 Gabage First 名称的由来）。


#### Mixed Collection (混合垃圾回收)


复制完成，内存得到释放。进入下一轮的新生代回收、并发标记、混合收集

其中H叫做巨型对象，如果对象非常大，会开辟一块连续的空间存储巨型对象

### 6. 强引用、软引用、弱引用、虚引用的区别？

- 强引用


强引用：只有所有 GC Roots 对象都不通过【强引用】引用该对象，该对象才能被垃圾回收

```Java
User user = new User();
```

![](https://heuqqdmbyk.feishu.cn/space/api/box/stream/download/asynccode/?code=NDIwYjc0MDI0ZGZiOGFhMmJlZWU0NjNmOWQ5NzljYWRfekJDUHY2TDM2SG1IRldrb0ZTWURJS1RwODUxdUU2RFFfVG9rZW46U3NlY2JCRTd6b0VxWmJ4bEw1RGNadUJFbjRmXzE3ODE3MDMyNzk6MTc4MTcwNjg3OV9WNA&add_watermark=true&scene_type=CCM)

#### 软引用


软引用：仅有软引用引用该对象时，在垃圾回收后，内存仍不足时会再次出发垃圾回收

```Java
User user = new User();
SoftReference softReference = new SoftReference(user);
```

![](https://heuqqdmbyk.feishu.cn/space/api/box/stream/download/asynccode/?code=YzBhYmVkNzM4MjBkYWI2MmQ2OGZkMjU2YWU1ODkxNTRfUkxUVml5Y014bHY5VmxZVmhXWERwRVJOWnQ4RXY2WU9fVG9rZW46Wk94aWJ0ZkN1b0Y1N3N4WUlkZWM0WTUwbkxnXzE3ODE3MDMyNzk6MTc4MTcwNjg3OV9WNA&add_watermark=true&scene_type=CCM)

#### 弱引用


弱引用：仅有弱引用引用该对象时，在垃圾回收时，无论内存是否充足，都会回收弱引用对象

```Java
User user = new User();
WeakReference weakReference = new WeakReference(user);
```

延伸话题：ThreadLocal内存泄漏问题

ThreadLocal用的就是弱引用，看以下源码：

```Java
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
         super(k);
         value = v; //强引用，不会被回收
     }
}
```

`Entry`的key是当前ThreadLocal，value值是我们要设置的数据。

`WeakReference`表示的是弱引用，当JVM进行GC时，一旦发现了只具有弱引用的对象，不管当前内存空间是否足够，都会回收它的内存。但是`value`是强引用，它不会被回收掉。

> ThreadLocal使用建议：使用完毕后注意调用清理方法。

#### 虚引用


虚引用：必须配合引用队列使用，被引用对象回收时，会将虚引用入队，由 Reference Handler 线程调用虚引用相关方法释放直接内存
```Java
Object obj = new Object();

ReferenceQueue<Object> queue = new ReferenceQueue<>();

PhantomReference<Object> pr = new PhantomReference<>(obj, queue);

```

## 三、JVM 调优

### 1. JVM 调优的参数可以在哪里设置参数值？

#### tomcat的设置vm参数

修改TOMCAT_HOME/bin/catalina.sh文件，如下图

`JAVA_OPTS="``-Xms512m -Xmx1024m``"`

![](https://heuqqdmbyk.feishu.cn/space/api/box/stream/download/asynccode/?code=OWE1NTg4ZDRhMzI0NmQ5YjVjZmIyZDgwOTEzYzJlN2JfRW9PUHRJNmQ0R3R6S0V6b2hqQXdJU3hmY2lKUWxiWjlfVG9rZW46RGt1QmJjZkpNb0dxdm14T2F0eWNwTnBNblNlXzE3ODE3MDM2OTc6MTc4MTcwNzI5N19WNA&add_watermark=true&scene_type=CCM)

#### springboot项目jar文件启动


通常在linux系统下直接加参数启动springboot项目

`nohup java -Xms512m -Xmx1024m -jar xxxx.jar --spring.profiles.active=prod &`

> nohup : 用于在系统后台不挂断地运行命令，退出终端不会影响程序的运行
>
> 参数 **&** ：让命令在后台执行，终端退出后命令仍旧执行。

### 2. 用的 JVM 调优的参数都有哪些？


对于JVM调优，主要就是调整年轻代、年老大、元空间的内存空间大小及使用的垃圾回收器类型。

[https://www.oracle.com/java/technologies/javase/vmoptions-jsp.html](https://www.oracle.com/java/technologies/javase/vmoptions-jsp.html)

1）设置堆的初始大小和最大大小，为了防止垃圾收集器在初始大小、最大大小之间收缩堆而产生额外的时间，通常把最大、初始大小设置为相同的值。

> -Xms：设置堆的初始化大小
>
> -Xmx：设置堆的最大大小

2） 设置年轻代中Eden区和两个Survivor区的大小比例。该值如果不设置，则默认比例为8:1:1。Java官方通过增大Eden区的大小，来减少YGC发生的次数，但有时我们发现，虽然次数减少了，但Eden区满

的时候，由于占用的空间较大，导致释放缓慢，此时STW的时间较长，因此需要按照程序情况去调优。

> -XXSurvivorRatio=3，表示年轻代中的分配比率：survivor:eden = 2:3

3）年轻代和老年代默认比例为1：2。可以通过调整二者空间大小比率来设置两者的大小。

> -XX:newSize 设置年轻代的初始大小
>
> -XX:MaxNewSize 设置年轻代的最大大小， 初始大小和最大大小两个值通常相同

4）线程堆栈的设置：**每个线程默认会开启1M的堆栈**，用于存放栈帧、调用参数、局部变量等，但一般256K就够用。通常减少每个线程的堆栈，可以产生更多的线程，但这实际上还受限于操作系统。

> -Xss 对每个线程stack大小的调整,-Xss128k

5）一般来说，当survivor区不够大或者占用量达到50%，就会把一些对象放到老年区。通过设置合理的eden区，survivor区及使用率，可以将年轻对象保存在年轻代，从而避免full GC，使用-Xmn设置年轻代的大小

6）系统CPU持续飙高的话，首先先排查代码问题，如果代码没问题，则咨询运维或者云服务器供应商，通常服务器重启或者服务器迁移即可解决。

7）对于占用内存比较多的大对象，一般会选择在老年代分配内存。如果在年轻代给大对象分配内存，年轻代内存不够了，就要在eden区移动大量对象到老年代，然后这些移动的对象可能很快消亡，因此导致full GC。通过设置参数：-XX:PetenureSizeThreshold=1000000，单位为B，标明对象大小超过1M时，在老年代(tenured)分配内存空间。

8）一般情况下，年轻对象放在eden区，当第一次GC后，如果对象还存活，放到survivor区，此后，每GC一次，年龄增加1，当对象的年龄达到阈值，就被放到tenured老年区。这个阈值可以同构-XX:MaxTenuringThreshold设置。如果想让对象留在年轻代，可以设置比较大的阈值。

> （1）-XX:+UseParallelGC:年轻代使用并行垃圾回收收集器。这是一个关注吞吐量的收集器，可以尽可能的减少垃圾回收时间。
>
> （2）-XX:+UseParallelOldGC:设置老年代使用并行垃圾回收收集器。

9）尝试使用大的内存分页：使用大的内存分页增加CPU的内存寻址能力，从而系统的性能。

> -XX:+LargePageSizeInBytes 设置内存页的大小

10）使用非占用的垃圾收集器。

> -XX:+UseConcMarkSweepGC老年代使用CMS收集器降低停顿。

### 3. 说一下 JVM 调优的工具？


> 难易程度：☆☆☆☆
>
> 出现频率：☆☆☆☆

#### 命令工具


#### jps（Java Process Status）


输出JVM中运行的进程状态信息(现在一般使用jconsole)

![](https://heuqqdmbyk.feishu.cn/space/api/box/stream/download/asynccode/?code=MmMxMzkwYzQ4MzcyNzNkN2RlMTA1MzVkNDFmOTE4YTZfSXdnamJtcUlPZmRBOGtud1FmSloxNG13MU5yaGw4S1lfVG9rZW46VGFnZWJZTldDbzdNUDN4WEhhSmNQckpJbm1lXzE3ODE3MDM2OTc6MTc4MTcwNzI5N19WNA&add_watermark=true&scene_type=CCM)

#### jstack


查看java进程内**线程的堆栈**信息。

`jstack [option] <pid>`

java案例

```Java
package com.heima.jvm;

public class Application {

    public static void main(String[] args) throws InterruptedException {
        while (true){
            Thread.sleep(1000);
            System.out.println("哈哈哈");
        }
    }
}
```

使用jstack查看进行堆栈运行信息

![](https://heuqqdmbyk.feishu.cn/space/api/box/stream/download/asynccode/?code=NzBlNTY1NGViYmEyNThhNTRhMTZiOTQ0YWMzYjliNmZfOFF6V2JoS3VZaWNpNHV6MjJsQUpIV3VLb0RZMGI0anBfVG9rZW46SEVCWWI4NEFUb2FYeGd4TzdIUmNjWlpUbnFoXzE3ODE3MDM2OTc6MTc4MTcwNzI5N19WNA&add_watermark=true&scene_type=CCM)

#### jmap


用于生成堆转存快照

> jmap [options] pid 内存映像信息
>
> jmap -heap pid 显示Java堆的信息
>
> jmap -dump:format=b,file=heap.hprof pid
>
> format=b表示以hprof二进制格式转储Java堆的内存
>
> file=<filename>用于指定快照dump文件的文件名。

## 四、面试现场


### 1. JVM组成


> **面试官**：JVM由那些部分组成，运行流程是什么？
>
> **候选人:**
>
> 嗯，好的~~
>
> 在JVM中共有四大部分，分别是ClassLoader（类加载器）、Runtime Data Area（运行时数据区，内存分区）、Execution Engine（执行引擎）、Native Method Library（本地库接口）
>
> 它们的运行流程是：
>
> 第一，类加载器（ClassLoader）把Java代码转换为字节码
>
> 第二，运行时数据区（Runtime Data Area）把字节码加载到内存中，而字节码文件只是JVM的一套指令集规范，并不能直接交给底层系统去执行，而是有执行引擎运行
>
> 第三，执行引擎（Execution Engine）将字节码翻译为底层系统指令，再交由CPU执行去执行，此时需要调用其他语言的本地库接口（Native Method Library）来实现整个程序的功能。
>
> **面试官**：好的，你能详细说一下 JVM 运行时数据区吗？
>
> **候选人:**
>
> 嗯，好~
>
> 运行时数据区包含了堆、方法区、栈、本地方法栈、程序计数器这几部分，每个功能作用不一样。
>
> - 堆解决的是对象实例存储的问题，垃圾回收器管理的主要区域。
>
> - 方法区可以认为是堆的一部分，用于存储已被虚拟机加载的信息，常量、静态变量、即时编译器编译后的代码。
>
> - 栈解决的是程序运行的问题，栈里面存的是栈帧，栈帧里面存的是局部变量表、操作数栈、动态链接、方法出口等信息。
>
> - 本地方法栈与栈功能相同，本地方法栈执行的是本地方法，一个Java调用非Java代码的接口。
>
> - 程序计数器（PC寄存器）程序计数器中存放的是当前线程所执行的字节码的行数。JVM工作时就是通过改变这个计数器的值来选取下一个需要执行的字节码指令。
>
>
> **面试官**：好的，你再详细介绍一下程序计数器的作用？
>
> **候选人:**
>
> 嗯，是这样~~
>
> java虚拟机对于多线程是通过线程轮流切换并且分配线程执行时间。在任何的一个时间点上，一个处理器只会处理执行一个线程，如果当前被执行的这个线程它所分配的执行时间用完了【挂起】。处理器会切换到另外的一个线程上来进行执行。并且这个线程的执行时间用完了，接着处理器就会又来执行被挂起的这个线程。这时候程序计数器就起到了关键作用，程序计数器在来回切换的线程中记录他上一次执行的行号，然后接着继续向下执行。
>
> **面试官**：你能给我详细的介绍Java堆吗?
>
> **候选人:**
>
> 好的~
>
> Java中的堆术语线程共享的区域。主要用来保存**对象实例，数组**等，当堆中没有内存空间可分配给实例，也无法再扩展时，则抛出OutOfMemoryError异常。
>
> 在JAVA8中堆内会存在年轻代、老年代
>
> 1）Young区被划分为三部分，Eden区和两个大小严格相同的Survivor区，其中，Survivor区间中，某一时刻只有其中一个是被使用的，另外一个留做垃圾收集时复制对象用。在Eden区变满的时候， GC就会将存活的对象移到空闲的Survivor区间中，根据JVM的策略，在经过几次垃圾收集后，任然存活于Survivor的对象将被移动到Tenured区间。
>
> 2）Tenured区主要保存生命周期长的对象，一般是一些老的对象，当一些对象在Young复制转移一定的次数以后，对象就会被转移到Tenured区。
>
> **面试官**：能不能解释一下方法区？
>
> **候选人:**
>
> 好的~
>
> 与虚拟机栈类似。本地方法栈是为虚拟机**执行本地方法时提供服务的**。不需要进行GC。本地方法一般是由其他语言编写。
>
> **面试官**：你听过直接内存吗？
>
> **候选人:**
>
> 嗯~~
>
> 它又叫做**堆外内存**，**线程共享的区域**，在 Java 8 之前有个**永久代**的概念，实际上指的是 HotSpot 虚拟机上的永久代，它用永久代实现了 JVM 规范定义的方法区功能，**主要存储类的信息，常量，静态变量**，即时编译器编译后代码等，这部分由于是在堆中实现的，受 GC 的管理，不过由于永久代有 -XX:MaxPermSize 的上限，所以如果大量动态生成类（将类信息放入永久代），很容易造成 OOM，有人说可以把永久代设置得足够大，但很难确定一个合适的大小，受类数量，常量数量的多少影响很大。
>
> 所以在 Java 8 中就把方法区的实现移到了本地内存中的元空间中，这样方法区就不受 JVM 的控制了,也就不会进行 GC，也因此提升了性能。
>
> **面试官**：什么是虚拟机栈
>
> **候选人:**
>
> 虚拟机栈是描述的是方法执行时的内存模型,是线程私有的，生命周期与线程相同,每个方法被执行的同时会创建**栈桢**。保存执行方法时的**局部变量、动态连接信息、方法返回地址信息**等等。方法开始执行的时候会进栈，方法执行完会出栈【相当于清空了数据】，所以这块区域**不需要进行 GC**。
>
> **面试官**：能说一下堆栈的区别是什么吗？
>
> **候选人:**
>
> 嗯，好的，有这几个区别
>
> 第一，栈内存一般会用来存储局部变量和方法调用，但堆内存是用来存储Java对象和数组的的。堆会GC垃圾回收，而栈不会。
>
> 第二、栈内存是线程私有的，而堆内存是线程共有的。
>
> 第三、两者异常错误不同，但如果栈内存或者堆内存不足都会抛出异常。
>
> 栈空间不足：java.lang.StackOverFlowError。
>
> 堆空间不足：java.lang.OutOfMemoryError。

### 2. 类加载器


> **面试官**：什么是类加载器，类加载器有哪些?
>
> **候选人:**
>
> 嗯，是这样的
>
> JVM只会运行二进制文件，而类加载器（ClassLoader）的主要作用就是将**字节码文件加载到JVM中**，从而让Java程序能够启动起来。
>
> 常见的类加载器有4个
>
> 第一个是启动类加载器(BootStrap ClassLoader)：其是由C++编写实现。用于加载JAVA_HOME/jre/lib目录下的类库。
>
> 第二个是扩展类加载器(ExtClassLoader)：该类是ClassLoader的子类，主要加载JAVA_HOME/jre/lib/ext目录中的类库。
>
> 第三个是应用类加载器(AppClassLoader)：该类是ClassLoader的子类，主要用于加载classPath下的类，也就是加载开发者自己编写的Java类。
>
> 第四个是自定义类加载器：开发者自定义类继承ClassLoader，实现自定义类加载规则。
>
> **面试官**：说一下类装载的执行过程？
>
> **候选人:**
>
> 嗯，这个过程还是挺多的。
>
> 类从加载到虚拟机中开始，直到卸载为止，它的整个生命周期包括了：加载、验证、准备、解析、初始化、使用和卸载这7个阶段。其中，验证、准备和解析这三个部分统称为连接（linking）
>
> 1.加载：查找和导入class文件
>
> 2.验证：保证加载类的准确性
>
> 3.准备：为类变量分配内存并设置类变量初始值
>
> 4.解析：把类中的符号引用转换为直接引用
>
> 5.初始化：对类的静态变量，静态代码块执行初始化操作
>
> 6.使用：JVM 开始从入口方法开始执行用户的程序代码
>
> 7.卸载：当用户程序代码执行完毕后，JVM 便开始销毁创建的 Class 对象，最后负责运行的 JVM 也退出内存
>
> **面试官**：什么是双亲委派模型？
>
> **候选人:**
>
> 嗯，它是是这样的。
>
> 如果一个类加载器收到了类加载的请求，它首先不会自己尝试加载这个类，而是把这请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传说到顶层的启动类加载器中，只有当父类加载器返回自己无法完成这个加载请求（它的搜索返回中没有找到所需的类）时，子类加载器才会尝试自己去加载
>
> **面试官**：JVM为什么采用双亲委派机制
>
> **候选人:**
>
> 主要有两个原因。
>
> 第一、通过双亲委派机制可以避免某一个类被重复加载，当父类已经加载后则无需重复加载，保证唯一性。
>
> 第二、为了安全，保证类库API不会被修改

### 3. 垃圾回收


> **面试官**：简述Java垃圾回收机制？（GC是什么？为什么要GC）
>
> **候选人:**
>
> 嗯，是这样~~
>
> 为了让程序员更专注于代码的实现，而不用过多的考虑内存释放的问题，所以，在Java语言中，有了自动的垃圾回收机制，也就是我们熟悉的GC(Garbage Collection)。
>
> 有了垃圾回收机制后，程序员只需要关心内存的申请即可，内存的释放由系统自动识别完成。
>
> 在进行垃圾回收时，不同的对象引用类型，GC会采用不同的回收时机
>
> **面试官**：强引用、软引用、弱引用、虚引用的区别？
>
> **候选人:**
>
> 嗯嗯~
>
> 强引用最为普通的引用方式，表示一个对象处于**有用且必须**的状态，如果一个对象具有强引用，则GC并不会回收它。即便堆中内存不足了，宁可出现OOM，也不会对其进行回收
>
> 软引用表示一个对象处于**有用且非必须**状态，如果一个对象处于软引用，在内存空间足够的情况下，GC机制并不会回收它，而在内存空间不足时，则会在OOM异常出现之间对其进行回收。但值得注意的是，因为GC线程优先级较低，软引用并不会立即被回收。
>
> 弱引用表示一个对象处于**可能有用且非必须**的状态。在GC线程扫描内存区域时，一旦发现弱引用，就会回收到弱引用相关联的对象。对于弱引用的回收，无关内存区域是否足够，一旦发现则会被回收。同样的，因为GC线程优先级较低，所以弱引用也并不是会被立刻回收。
>
> 虚引用表示一个对象处于**无用**的状态。在任何时候都有可能被垃圾回收。虚引用的使用必须和引用队列Reference Queue联合使用
>
> **面试官**：对象什么时候可以被垃圾器回收
>
> **候选人:**
>
> 思考一会~~
>
> 如果一个或多个对象没有任何的引用指向它了，那么这个对象现在就是垃圾，如果定位了垃圾，则有可能会被垃圾回收器回收。
>
> 如果要定位什么是垃圾，有两种方式来确定，第一个是引用计数法，第二个是可达性分析算法
>
> 通常都使用可达性分析算法来确定是不是垃圾
>
> **面试官**： JVM 垃圾回收算法有哪些？
>
> **候选人:**
>
> 我记得一共有四种，分别是标记清除算法、复制算法、标记整理算法、分代回收
>
> **面试官**： 你能详细聊一下分代回收吗？
>
> **候选人:**
>
> 关于分代回收是这样的
>
> 在java8时，堆被分为了两份：新生代和老年代，它们默认空间占用比例是1:2
>
> 对于新生代，内部又被分为了三个区域。Eden区，S0区，S1区默认空间占用比例是8:1:1
>
> 具体的工作机制是有些情况：
>
> 1）当创建一个对象的时候，那么这个对象会被分配在新生代的Eden区。当Eden区要满了时候，触发YoungGC。
>
> 2）当进行YoungGC后，此时在Eden区存活的对象被移动到S0区，并且**当前对象的年龄会加1**，清空Eden区。
>
> 3）当再一次触发YoungGC的时候，会把Eden区中存活下来的对象和S0中的对象，移动到S1区中，这些对象的年龄会加1，清空Eden区和S0区。
>
> 4）当再一次触发YoungGC的时候，会把Eden区中存活下来的对象和S1中的对象，移动到S0区中，这些对象的年龄会加1，清空Eden区和S1区。
>
> 5）对象的年龄达到了某一个限定的值（**默认15岁** ），那么这个对象就会进入到老年代中。
>
> 当然也有特殊情况，如果进入Eden区的是一个大对象，在触发YoungGC的时候，会直接存放到老年代
>
> 当老年代满了之后，**触发FullGC**。**FullGC同时回收新生代和老年代**，当前只会存在一个FullGC的线程进行执行，其他的线程全部会被挂起。 我们在程序中要尽量避免FullGC的出现。
>
> **面试官**：讲一下新生代、老年代、永久代的区别？
>
> **候选人:**
>
> 嗯！是这样的，简单说就是
>
> **新生代**主要用来存放新生的对象。
>
> **老年代**主要存放应用中生命周期长的内存对象。
>
> **永久代**指的是永久保存区域。主要存放Class和Meta（元数据）的信息。在Java8中，永久代已经被移除，取而代之的是一个称之为“元数据区”（**元空间**）的区域。元空间和永久代类似，不过元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。因此，默认情况下，元空间的大小仅受本地内存的限制。
>
> **面试官**：说一下 JVM 有哪些垃圾回收器？
>
> **候选人:**
>
> 在jvm中，实现了多种垃圾收集器，包括：串行垃圾收集器、并行垃圾收集器（JDK8默认）、CMS（并发）垃圾收集器、G1垃圾收集器（JDK9默认）
>
> **面试官**：Minor GC、Major GC、Full GC是什么
>
> **候选人:**
>
> 嗯，其实它们指的是不同代之间的垃圾回收
>
> Minor GC 发生在新生代的垃圾回收，暂停时间短
>
> Major GC 老年代区域的垃圾回收，老年代空间不足时，会先尝试触发Minor GC。Minor GC之后空间还不足，则会触发Major GC，Major GC速度比较慢，暂停时间长
>
> Full GC 新生代 + 老年代完整垃圾回收，暂停时间长，**应尽力避免**

### 4. JVM实践（调优）


> **面试官**：JVM 调优的参数可以在哪里设置参数值？
>
> **候选人:**
>
> 我们当时的项目是springboot项目，可以在项目启动的时候，java -jar中加入参数就行了
>
> **面试官**：用的 JVM 调优的参数都有哪些？
>
> **候选人:**
>
> 嗯，这些参数是比较多的
>
> 我记得当时我们设置过堆的大小，像-Xms和-Xmx
>
> 还有就是可以设置年轻代中Eden区和两个Survivor区的大小比例
>
> 还有就是可以设置使用哪种垃圾回收器等等。具体的指令还真记不太清楚。
>
> **面试官**：嗯，好的，你们平时调试 JVM都用了哪些工具呢？
>
> **候选人:**
>
> 嗯，我们一般都是使用jdk自带的一些工具，比如
>
> jps 输出JVM中运行的进程状态信息
>
> jstack查看java进程内**线程的堆栈**信息。
>
> jmap 用于生成堆转存快照
>
> jstat用于JVM统计监测工具
>
> 还有一些可视化工具，像jconsole和VisualVM等
>
> **面试官**：假如项目中产生了java内存泄露，你说一下你的排查思路？
>
> **候选人:**
>
> 嗯，这个我在之前项目排查过
>
> 第一呢可以通过jmap指定打印他的内存快照 dump文件，不过有的情况打印不了，我们会设置vm参数让程序自动生成dump文件
>
> 第二，可以通过工具去分析 dump文件，jdk自带的VisualVM就可以分析
>
> 第三，通过查看堆信息的情况，可以大概定位内存溢出是哪行代码出了问题
>
> 第四，找到对应的代码，通过阅读上下文的情况，进行修复即可
>
> **面试官**：好的，那现在再来说一种情况，就是说服务器CPU持续飙高，你的排查方案与思路？
>
> **候选人:**
>
> 嗯，我思考一下~~
>
> 可以这么做~~
>
> 第一可以使用使用top命令查看占用cpu的情况
>
> 第二通过top命令查看后，可以查看是哪一个进程占用cpu较高，记录这个进程id
>
> 第三可以通过ps 查看当前进程中的线程信息，看看哪个线程的cpu占用较高
>
> 第四可以jstack命令打印进行的id，找到这个线程，就可以进一步定位问题代码的行号
