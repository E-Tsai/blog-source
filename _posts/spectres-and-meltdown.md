---
title: Spectres And Meltdown
date: 2019-11-21 13:17:58
tags: [CN, security, systems]
categories: CS
---

从流水线 (pipeline) 到乱序执行 (out-of-order)，处理器的指令级并行保证了其性能上的绝大提升。但是在性能提升的同时，安全性的问题也随之出现。这篇博客主要介绍预测执行攻击。

<!--more-->

本文是在USTCLUG weekly party上的讲稿，直接使用了很多博客里的图片，引用源都列在了参考章节。

## 词汇解释

### 乱序 (out-of-order)

**循序执行 (in-order)**

在早期的处理器中，指令的执行一般在以下的步骤中完成：

1. 指令获取。
2. 如果输入的运算对象是可以获取的（比如已经存在于寄存器中），这条指令会被发送到合适的功能单元。如果一个或者更多的运算对象在当前的时钟周期中是不可获取的（通常需要从主存获取），处理器会开始等待直到它们是可以获取的。
3. 指令在合适的功能单元中被执行。
4. 功能单元将运算结果写回寄存器。

**乱序执行 (o3)**

这种范式通过以下步骤挑选可执行的指令先运行：

1. 指令获取。
2. 指令被发送到一个指令序列中（也称执行缓冲区或者保留站（reservation station））。
3. 指令将在序列中等待直到它的数据运算对象是可以获取的。然后指令被允许在先进入的、旧的指令之前离开序列缓冲区。
4. 指令被分配给一个合适的功能单元并由之执行。
5. 结果被放到一个序列中。
6. 仅当所有在该指令之前的指令都将他们的结果写入寄存器后，这条指令的结果才会被写入寄存器中。这个过程被称为毕业或者退休（retire）周期。

乱序执行的重要概念是实现了避免计算机在用于运算的对象不可获取时的大量等待。在上述文字的要点中，乱序执行处理器避免了在顺序执行处理器处理过程第二步中当指令由于运算数据未到位所造成的等待。

![](http://renesasrulz.com/cfs-file.ashx/__key/communityserver-blogs-components-weblogfiles/00-00-00-00-67/RX_5F00_pipeline_5F00_550.gif)

**Tomasulo算法**

在处理器中，先后执行的指令之间经常具有相关性（例如后一条指令用到前一条指令向寄存器写入的结果），因此早期简单的处理器使后续指令停顿，直到其所需的资源已经由前序指令准备就绪。Tomasulo算法则通过动态调度的方式，在不影响结果正确性的前提下，重新排列指令实际执行的顺序（乱序执行），提高时间利用效率。

![](https://www.researchgate.net/profile/Dimitris_Kehagias/publication/318502489/figure/fig1/AS:533824906170368@1504285183057/1-The-basic-structure-of-a-MIPS-floating-point-unit-using-Tomasulos-algorithm.png) 

### 旁路攻击 (side-channel attacks)

较之暴力破解攻击和密码算法的理论性弱点分析，旁路攻击侧重的是物理信息，例如时间，功率消耗，点此邪路或是声音。它的信息来源于计算机系统如何被implement，而不是被implement的算法或是设计。这些是“额外的信息”，所以被称为side-channel.

根据借助的介质，旁路攻击分为多个大类，包括：

- [缓存攻击](https://zh.wikipedia.org/w/index.php?title=快取旁路攻擊&action=edit&redlink=1)，通过获取对缓存的访问权而获取缓存内的一些敏感信息，例如攻击者获取云端主机物理主机的访问权而获取存储器的访问权，我们接下来会讲到的大部分**预测执行攻击**都属于缓存攻击；
- [计时攻击](https://zh.wikipedia.org/w/index.php?title=計時攻擊&action=edit&redlink=1)，通过设备运算的用时来推断出所使用的运算操作，或者通过对比运算的时间推定数据位于哪个存储设备，或者利用通信的时间差进行数据窃取，[NetSpectre](https://mlq.me/download/netspectre.pdf)就是计时攻击的一个例子。

除此之外，还有[基于功耗监控的旁路攻击](https://zh.wikipedia.org/wiki/功耗分析)，[电磁攻击](https://zh.wikipedia.org/w/index.php?title=電磁攻擊&action=edit&redlink=1)（如[TEMPEST](https://zh.wikipedia.org/w/index.php?title=TEMPEST&action=edit&redlink=1)攻击），[差别错误分析](https://zh.wikipedia.org/wiki/差別錯誤分析)，[数据残留](https://zh.wikipedia.org/w/index.php?title=数据残留&action=edit&redlink=1)等等，但是在这种攻击下在预测执行的前提下很难实现，就不展开讲了。

所有的攻击类型都利用了加密/解密系统在进行加密/解密操作时算法逻辑没有被发现缺陷，但是通过物理效应提供了有用的额外信息（这也是称为“旁路”的缘由），而这些物理信息往往包含了密钥、密码、密文等隐密数据。

### 虚拟内存 (virtual memory)

在现代计算机中，将虚拟地址转换为物理地址是非常常见的操作。如果把这个工作交给操作系统，那就太慢了。现代CPU硬件提供了一种称为TLB (Translation Lookaside Buffer) 的缓存，该设备可缓存最近使用的映射。这样，CPU大部分时间都可以直接在硬件中直接执行地址转换。

![](https://miro.medium.com/max/3598/1*-EkJxntbntPppqfPgTjbWg.png)



从上图中我们可以看到一个内存地址的翻译过程：首先在TLB中查找，如果在TLB中没有找到对应条目，则在页表中去寻找，如果成功了，则返回地址，并将它加入TLB，如果也失败了，操作系统会raise一个page fault.

![](https://miro.medium.com/max/3601/1*R3SKGOakznWoCMp76TGebg.png)

上图是现代计算机中虚拟内存的一个示例 (pre-meltdown)，红色是内核内存（只有操作系统可以访问），蓝色是用户程序的内存，蓝色是未分配的内存。

每一个页面 (page, 常见的page size是4KB) 都有一个permission bit，标志他是否是内核内存。如果一个用户程序师徒访问内核内存，就会导致page fault，操作系统在此时会终止此次访问。

### 缓存 (cache)

在计算机系统中，CPU高速缓存（英语：CPU Cache，在本文中简称缓存）是用于减少处理器访问内存所需平均时间的部件。

![](https://softwarerajivprab.files.wordpress.com/2019/07/cache.png)

### 预测执行 (speculation execution attack)

预测执行是优化技术的一类，采用这个技术的计算机系统会根据现有信息，利用空转时间提前执行一些将来可能用得上，也可能用不上的指令。如果指令执行完成后发现用不上，系统会抛弃计算结果，并回退 (squash) 执行期间造成的副作用。 推测执行的目标是在处理器系统资源过剩的情况下并行处理其他任务。推测执行无处不在。流水处理器的分支预测、数值预测、 预读取内存和文件、以及数据库系统的乐观并发控制等机能中都采用到了推测执行。

![img](https://miro.medium.com/max/751/1*xPqqyrbiNO7yrAsu9_VxWw.png)

上图是一个现代处理器的执行引擎。我们知道，处理器有多级缓存，每一级的缓存miss都会延迟程序执行的时间，为了优化时间上的性能，现代处理器引入了预测执行，举一个简单的例子：

```c
if(x < array_size){
    y = probe_array[array0[x] * 256];
}
```

想象这样一个情形：当执行这个代码时，`array_size`并不在内存中，与其原地空等它的值，不如让分支预测单元 (BPU, Branch Prediction Unit) 预测一下这个if判断是1还是0，然后根据它的预测，在等待`array_size`的时间内执行BPU选择的代码块的指令。如果后来发现BPU判断错误，也没有关系，处理器只要抛弃此时预测执行的指令所产生的计算结果，回退到if判断指令就行了。此时的时间性能仍然不会比空等更糟。

另一类常见的预测执行被称为间接分支预测 (indirect branch predict)，由于动态调度 (virtual dispatch)，这个情形非常普遍。

```c++
class Base {
 public:
  virtual void Foo() = 0;
};
class Derived : public Base {
 public:
  void Foo() override { … }
};
Base* obj = new Derived;
obj->Foo();
```

在机器代码中，这一代码段的实现方式是让目标文件 (obj) 从指向的内存位置加载虚拟调度表 (v-table)，然后调用它。由于此操作非常普遍，因此现代CPU具有各种内部缓存，并且经常会猜测间接分支将到达的位置并在该点继续执行。同样，如果处理器猜对了，它可以继续节省大量时间。如果不是这样，它可能会丢弃推测性的计算并重新开始。

## 熔断 (Meltdown): 流水线带来的挑战

```c
//assume probe array is flushed
secret_byte = *kernel_address;			// 指令1
tmp = probe_array[secret_byte* 512];	// 指令2

for (guess = 0; guess < possible_guesses; guess++){
    start_time = rdtscp();
    tmp = probe_array[guess * 512];
    times[guess] = rdtscp() - start_time;
}

// restore secret
secret = get_min_index(times);
```

**注释**

这里的`rdtscp()`是intel 64和IA-32架构上的一个Read Time-Stamp Counter and Processor ID函数。处理器中有一个时间戳寄存器 (time-stamp register)，这个函数会读取它的值，提供处理器时钟周期级别的时间 (GHz instead of ms)。

```c
static inline __attribute__((always_inline))  uint64_t rdtscp(void)
{
    unsigned int low, high;
    asm volatile("rdtscp" : "=a" (low), "=d" (high) :: "rcx" );
    return low | ((uint64_t)high) << 32; 
}
```

Meltdown出现不是因为乱序执行 (out-of-order)，流水线处理器都有可能有此安全缺陷。指令1和2虽然有数据依赖，但只要在流水线情形下，就可能在page fault之前把内核地址内容取进缓存里。



## 参考

[Pipeline and out-of-order instruction execution optimize performance]()

[乱序执行](https://zh.wikipedia.org/wiki/乱序执行) 

[Tomasulo算法](https://zh.wikipedia.org/wiki/Tomasulo算法) 

[旁路攻击](https://zh.wikipedia.org/wiki/旁路攻击) 

[缓存](https://zh.wikipedia.org/wiki/CPU缓存) 

[推测执行](https://zh.wikipedia.org/wiki/推测执行)

[The medium post](https://medium.com/@mattklein123/meltdown-spectre-explained-6bc8634cc0c2) 

[rdtscp()](https://www.felixcloutier.com/x86/rdtscp) 