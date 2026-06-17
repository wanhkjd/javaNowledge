1. JVM是什么

    Java Virtual Machine Java程序的运行环境（java二进制字节码的运行环境）

    好处： 
- 一次编写，到处运行
    
- 自动内存管理，垃圾回收机制


2. JVM由哪些部分组成，运行流程是什么？
![JVM组成|496](资源/JVM组成.PNG)

- ClassLoader（类加载器）
    
- Runtime Data Area（运行时数据区，内存分区）
    
- Execution Engine（执行引擎）
    
- Native Method Library（本地库接口）
    

运行流程：

（1）类加载器（ClassLoader）把Java代码转换为字节码

（2）运行时数据区（Runtime Data Area）把字节码加载到内存中，而字节码文件只是JVM的一套指令集规范，并不能直接交给底层系统去执行，而是有执行引擎运行

（3）执行引擎（Execution Engine）将字节码翻译为底层系统指令，再交由CPU执行去执行，此时需要调用其他语言的本地库接口（Native Method Library）来实现整个程序的功能。

2.  什么是程序计数器？

    程序计数器是线程私有的(线程安全)，每个线程一份，内部存储的是字节码的行号(记录上一次该线程执行到哪个行号了)，用于记录正在运行的字节码指令的地址。程序计数器不存在OOM，不会被GC回收。

3. 你能给我详细的介绍Java堆吗?