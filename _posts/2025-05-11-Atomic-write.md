---
layout: post
title: 从CPU看原子操作
categories: [OS]
description: 从CPU看原子操作
keywords: os, atomic
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

### 前言

今天看到一篇文章里面提到，**16bit/32bit的写可以保证原子性。**自己对这个知识点不理解，所以上网找些资料进行总结。

### 正文

所谓原子操作，就是“不可中断的一个或一系列操作”。在多线程程序中，原子操作是一个非常重要的概念，它常常用来实现一些同步机制，同时也是一些常见的多线程Bug的源头。

例如：x++，在不同编译系统中会按多条指令的形式来处理这种语句。从内存中读x的值到寄存器中，对寄存器加1，再把新值写回x所处的内存地址

但今天要讨论的是**每条指令是否可以认为是原子操作**。

要彻底理解这个问题，我们首先需要从硬件讲起。以常见的X86 CPU来说，根据Intel的[参考手册](http://download.intel.com/design/processor/manuals/253668.pdf)，它基于以下三种机制保证了多核中加锁的[原子操作：

（1）Guaranteed atomic operations （注：8.1.1节有详细介绍）

（2）Bus locking, using the LOCK# signal and the LOCK instruction prefix

（3）Cache coherency protocols that ensure that atomic operations can be carried out on cached data structures (cache lock); this mechanism is present in the Pentium 4, Intel Xeon, and P6 family processors



这三个机制相互独立，相辅相承。简单的理解起来就是

（1）一些基本的内存读写操作是本身已经被硬件提供了原子性保证（例如读写单个字节的操作）；

（2）一些需要保证原子性但是没有被第（1）条机制提供支持的操作（例如read-modify-write）可以通过使用”LOCK#”来锁定总线，从而保证操作的原子性

（3）因为很多内存数据是已经存放在L1/L2 cache中了，对这些数据的原子操作只需要与本地的cache打交道，而不需要与总线打交道，所以CPU就提供了cache coherency机制来保证其它的那些也cache了这些数据的processor能读到最新的值



#### CPU对原子操作的影响

那么CPU对哪些（1）中的基本的操作提供了原子性支持呢？根据Intel手册8.1.1节的介绍：

从Intel486 processor开始，以下的基本内存操作是原子的：

**读写一个byte**
Reading or writing a word aligned on a 16-bit boundary

**读写16bit(2byte)内存对齐的字(word)**
Reading or writing a doubleword aligned on a 32-bit boundary

**读写32bit(4byte)内存对齐的双字(dword)**



从Pentium processor开始，除了之前支持的原子操作外又新增了以下原子操作：

Reading or writing a quadword aligned on a 64-bit boundary

**对齐到64位边界的四字的读写**

16-bit accesses to uncached memory locations that fit within a 32-bit data bus

**未缓存且在32位数据总线范围之内的内存地址的访问**



从P6 family processors开始，除了之前支持的原子操作又新增了以下原子操作：

**P6系列处理器**（以及以后生产的处理器）确保以下对基本存储器的操作行为为原子操作：

Unaligned 16-, 32-, and 64-bit accesses to cached memory that fit within a cache line

**对单个cache line中缓存地址的未对齐的16/32/64位访问（非对齐的数据访问非常影响性能）**



那么哪些操作是非原子的呢？

Accesses to cacheable memory that are split across bus widths, cache lines, and page boundaries are not guaranteed to be atomic by the Intel Core 2 Duo, Intel®Atom™, Intel Core Duo, Pentium M, Pentium 4, Intel Xeon, P6 family, Pentium, and
Intel486 processors.（说点简单点，那些被总线带宽、cache line以及page大小给分隔开了的内存地址的访问不是原子的，你如果想保证这些操作是原子的，你就得求助于机制（2），对总线发出相应的控制信号才行）。

需要注意的是尽管从P6 family开始对一些非对齐的读写操作已经提供了原子性保障，但是非对齐访问是非常影响性能的，需要尽量避免。当然了，对于一般的程序员来说不需要太担心这个，因为大部分编译器会自动帮你完成内存对齐。
