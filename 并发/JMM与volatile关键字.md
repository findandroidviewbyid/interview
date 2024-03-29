## 现代计算机内存模型
现代计算机CPU执行指令的速断远远超出了内存的存储速度，计算机的存储设备与处理器的运算速度有着几个数量级的差距，所以现代计算机系统多不得不加入一层或多层读写速度读写速度尽可能接近处理器运算速度的高速缓存来作为内存与处理器之间的缓冲：将运算需要的数据复制到高速缓存中，让运算能快速进行，运算结束后再从高速缓存同步回内存，这样处理器就无需等待缓慢的内存的读写了。

基于高速缓存的存储交互很好的解决了处理器与内存速度之间的矛盾，但也引入了一个新的问题：缓存的一致性。在多路处理器系统中，每个处理器都有自己的高速缓存，而他们又共享同一主内存。当多个处理器的运算任务都涉及到同一块主内存区域的时候，将导致各自的缓存数据不一致的情况出现。为了解决这一问题，需要各个处理器访问缓存的时候都遵循一些协议（例如MSI、MESI、MOSI等），在读写的时候都根据协议来操作。而所谓的“内存模型”就可以理解为在特定的操作协议下，对特定的内存或高速缓存进行读写访问的过程抽象。不同架构的物理机器可以拥有不一样的内存模型,Java虚拟机也有自己的内存模型。

![操作系统内存模型](https://user-gold-cdn.xitu.io/2020/4/29/171c47561b7af1b1?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## Java虚拟机内存模型（JMM）
Java内存模型（以下简称JMM）用来屏蔽各种硬件和操作系统的内存访问差异，以实现让Java程序在各种平台下都能达到一致的缓存访问效果。JMM定义程序中各种变量的访问规则，即关注在虚拟机中把变量值存储到内存和从内存中取出变量值这样的底层细节。

JMM规定了所有的变量都存储在主内存中（这里所说的变量指的是实例变量和类变量，不包含局部变量，因为局部变量是线程是有的，因此不存在竞争问题）。每条线程还有自己的工作内存，线程的工作内存中保存了被线程使用的变量的主内存副本，线程对变量的所有操作都必须在工作内存中进行，不能直接读写主内存中的数据。不同的线程之间也无法直接访问对方工作内存中的变量。线程变量值的传递均需要通过主内存来完成，线程、主内存、工作内存三者的交互关系如下图：

![Java内存模型](https://user-gold-cdn.xitu.io/2020/4/29/171c47561d31ab88?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

与计算机的内存模型类似，Java内存模型同样也会带来缓存一致性问题，即假如一个线程通过自己的工作内存修改了主内存中的变量，那么对于另外一个线程由于有自己的工作内存，且没能立即同步主内存中的数据，就造成了数据数据错误的问题。即所谓的**可见性**问题。

> 可见性指的是一个线程对共享变量的写操作对其它线程后续的读操作可见,即当一个线程修改了共享变量后，其他线程能够立即得知这个修改。

## volatile关键字

volatile关键字是Java虚拟机提供的最轻量级的同步机制。当一个变量被定义为volatile之后，它具备两项特性：
- 1.保证此变量对所有线程的可见性。
- 2.禁止指令重排序优化。

另外，要注意的是volatile并不能保证原子性。

那volatile是如何保证共享变量的可见性的呢？

在计算机的内存模型中我们提到，为了解决缓存一致性问题，需要各个处理器访问缓存的时候都遵循一些协议。而JVM解决缓存一致性问题也是如此。

以Intel的MESI为例，当CPU写数据时，如果发现操作的是共享变量，会发出信号通知其他CPU将该变量的缓存行置为无效，当其他CPU需要读取这个变量时，发现自己缓存中的该变量的缓存行是无效的，那么就会从内存重新读取。

由于volatile的MESI缓存一致性协议需要不断的从主内存休干和CAS不断循环，无效交互会导致总线宽带达到峰值。因此不要大量使用volatile关键字。

### 指令重排序
为了提高性能，编译器和处理器常常会对既定的代码执行顺序进行指令重排序。一般重排序可以分为如下三种：
- 编译器优化的重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序;
- 指令级并行的重排序。现代处理器采用了指令级并行技术来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序;
- 内存系统的重排序。由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行的。






[面试官没想到一个Volatile，我都能跟他扯半小时](https://juejin.cn/post/6844904149536997384)