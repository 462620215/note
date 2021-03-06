# Java代码是怎么运行的？

### Java 虚拟机具体是怎样运行 Java 字节码的？

#### 虚拟机视角

从虚拟机视角来看，执行 Java 代码首先需要将它编译而成的 class 文件加载到 Java 虚拟机中。加载后的 Java 类会被存放于方法区（Method Area）中。实际运行时，虚拟机会执行方法区内的代码。

如果你熟悉 X86 的话，你会发现这和段式内存管理中的代码段类似。而且，Java 虚拟机同样也在内存中划分出堆和栈来存储运行时数据。

不同的是，Java 虚拟机会将栈细分为面向 Java 方法的 **Java 方法栈**，面向本地方法（用 C++ 写的 native 方法）的**本地方法栈**，以及存放各个线程执行位置的 **PC 寄存器**。

![https://static001.geekbang.org/resource/image/ab/77/ab5c3523af08e0bf2f689c1d6033ef77.png](https://static001.geekbang.org/resource/image/ab/77/ab5c3523af08e0bf2f689c1d6033ef77.png)

在运行过程中，每当调用进入一个 Java 方法，Java 虚拟机会在当前线程的 Java 方法栈中生成一个栈帧，用以存放局部变量以及字节码的操作数。这个栈帧的大小是提前计算好的，而且 Java 虚拟机不要求栈帧在内存空间里连续分布。

当退出当前执行的方法时，不管是正常返回还是异常返回，Java 虚拟机均会弹出当前线程的当前栈帧，并将之舍弃。

#### 硬件视角
从硬件视角来看，Java 字节码无法直接执行。因此，Java 虚拟机需要将字节码翻译成机器码。

在 HotSpot 里面，上述翻译过程有两种形式：第一种是**解释执行**，即逐条将字节码翻译成机器码并执行；第二种是**即时编译（Just-In-Time compilation，JIT）**，即将一个方法中包含的所有字节码编译成机器码后再执行。

**前者的优势在于无需等待编译，而后者的优势在于实际运行速度更快。HotSpot 默认采用混合模式，综合了解释执行和即时编译两者的优点。它会先解释执行字节码，而后将其中反复执行的热点代码，以方法为单位进行即时编译。**



























