---
layout: post
title: java 解释器与JIT编译器
date: 2018-01-30
categories: blog
tags: [jvm,jit]
subtitle: 但是如今的HotSpot VM中不仅内置有解释器，还内置有先进的JIT（Just In Time Compiler）编译器，在Java虚拟机运行时，解释器和即时编译器能够相互协作，各自取长补短
---

早在Java1.0版本的时候，Sun公司发布了一款名为Sun Classic VM的Java虚拟机，它同时也是世界上第一款商用Java虚拟机，在当时这款虚拟机内部只提供解释器，用今天的眼光来看待必然是效率低下的，因为如果Java虚拟机只能够在运行时对代码采用逐行解释执行，程序的运行性能可想而知。但是如今的HotSpot VM中不仅内置有解释器，还内置有先进的JIT（Just In Time Compiler）编译器，在Java虚拟机运行时，解释器和即时编译器能够相互协作，各自取长补短。在此大家需要注意，无论是采用解释器进行解释执行，还是采用即时编译器进行编译执行，最终字节码都需要被转换为对应平台的本地机器指令。或许有些开发人员会感觉到诧异，既然HotSpot VM中已经内置JIT编译器了，那么为什么还需要再使用解释器来“拖累”程序的执行性能呢？比如JRockit VM内部就不包含解释器，字节码全部都依靠即时编译器编译后执行，尽管程序的执行性能会非常高效，但程序在启动时必然需要花费更长的时间来进行编译。对于服务端应用来说，启动时间并非是关注重点，但对于那些看中启动时间的应用场景而言，或许就需要采用解释器与即时编译器并存的架构来换取一个平衡点。


既然HotSpot VM中采用了即时编译器，那么这就意味着将字节码编译为本地机器指令是一件运行时任务。在HotSpot VM中内嵌有两个JIT编译器，分别为Client Compiler和Server Compiler，但大多数情况下我们简称为C1编译器和C2编译器。开发人员可以通过如下命令显式指定Java虚拟机在运行时到底使用哪一种即时编译器，如下所示：

```
-client：指定Java虚拟机运行在Client模式下，并使用C1编译器；
-server：指定Java虚拟机运行在Server模式下，并使用C2编译器。
```

除了可以显式指定Java虚拟机在运行时到底使用哪一种即时编译器外，默认情况下HotSpot VM则会根据操作系统版本与物理机器的硬件性能自动选择运行在哪一种模式下，以及采用哪一种即时编译器。简单来说，C1编译器会对字节码进行简单和可靠的优化，以达到更快的编译速度；而C2编译器会启动一些编译耗时更长的优化，以获取更好的编译质量。不过在Java7版本之后，一旦开发人员在程序中显式指定命令“-server”时，缺省将会开启分层编译（Tiered Compilation）策略，由C1编译器和C2编译器相互协作共同来执行编译任务。不过在早期版本中，开发人员则只能够通过命令“-XX:+TieredCompilation”手动开启分层编译策略。


之前笔者曾经提及过，缺省情况下HotSpot VM是采用解释器与即时编译器并存的架构，当然开发人员可以根据具体的应用场景，通过命令显式地为Java虚拟机指定在运行时到底是完全采用解释器执行，还是完全采用即时编译器执行。如下所示：

```
-Xint：完全采用解释器模式执行程序；
-Xcomp：完全采用即时编译器模式执行程序；
-Xmixed：采用解释器+即时编译器的混合模式共同执行程序。
```


在此大家需要注意，如果Java虚拟机在运行时完全采用解释器执行，那么即时编译器将会停止所有的工作，字节码将完全依靠解释器逐行解释执行。反之如果Java虚拟机在运行时完全采用即时编译器执行，但解释器仍然会在即时编译器无法进行的特殊情况下介入执行，以确保程序能够最终顺利执行。


由于即时编译器将本地机器指令的编译推迟到了运行时，自此Java程序的运行性能已经达到了可以和C/C++程序一较高下的地步。这主要是因为JIT编译器可以针对那些频繁被调用的“热点代码”做出深度优化，而静态编译器的代码优化则无法完全推断出运行时热点，因此通过JIT编译器编译的本地机器指令比直接生成的本地机器指令拥有更高的执行效率也就理所当然了。比如使用Python实现的PyPy执行器，比使用C实现的CPython解释器更加灵活，更重要的是，在程序的运行性能上进行比较，PyPy将近是CPython解释器执行效率的1至5倍，这就是对JIT技术魅力的一个有力证明。并且Java技术自身的诸多优势同样也是C/C++无法比拟的，所谓各有所长就是这个道理。在此大家需要注意，世界上永远没有最好的编程语言，只有最适用于具体应用场景的编程语言。