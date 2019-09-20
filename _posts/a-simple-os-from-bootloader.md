---
title: 从启动盘开始写一个简易操作系统
date: 2017-07-09 19:02:08
tags: Systems
categories: cs
---

## 前言&工具

&emsp; 我使用的是AT&T语法汇编，使用GCC编译，在qemu上跑这个简易的操作系统。GCC要求汇编代码是AT&T格式的，如果使用你想使用NASM编译的话，则可以使用intel语法。ld是GNU汇编器的链接器，GNU汇编器使用 AT&T 样式的语法。

 <!-- more -->

&emsp; **书籍推荐**： [《Orange's : 一个操作系统的实现》]() ， 这本书使用的是intel语法，非常详细地讲述了如何从一个启动盘写到一个具有进程调度、内存管理、文件系统、输入输出系统和进程间通信功能的os。书后附有代码，即使只是阅读代码也能有很多收获。

&emsp; 今天我要实现的是一个时间片轮转调度的简易os，并未实现文件系统和输入输出、进程通信。在这篇博客里我会列出学习操作系统时比较重要的知识点。我不会贴出我所有的代码，因为a)Oranges's里面已经很完备了，b)节省空间；我也不会深入地讲每一个知识点，google is always your friend :-D

**一些可能有用的资料**:

[[1] x86汇编数据类型](http://www.c-jump.com/CIS77/ASM/DataTypes/lecture.html#T77_0010_arithmetic)  

[[2] AT&T语法](http://csiflabs.cs.ucdavis.edu/~ssdavis/50/att-syntax.html) 

[[3] Linux 汇编器：对比 GAS 和 NASM](https://www.ibm.com/developerworks/cn/linux/l-gas-nasm.html)  # 善用ctrl+F

[[4] BIOS是什么](http://baike.baidu.com/item/bios?fr=aladdin) 

[[5] BIOS中断](https://en.wikipedia.org/wiki/INT_10H) 

[[6]Simple Linker Script Example](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/4/html/Using_ld_the_GNU_Linker/simple-example.html)  

[[7] 宏](https://sourceware.org/binutils/docs/as/Macro.html)  

[[8] GDT,LDT,GDTR,LDTR 详解](http://www.techbulo.com/708.html)  

[[9] Initrd是什么](http://baike.baidu.com/link?url=BE46Q77Bzdi8kRcBAiTT1cQ_2UimKCV-yworYMFYxShDcEjriOfIPr2RZK_lTyrApIvi31Mc3cKOMMk4QZTVBq)  

[[10] 实模式](http://www.cnblogs.com/chengxuyuancc/archive/2013/05/12/3073738.html)  

[[11] 寄存器相关](http://blog.csdn.net/yang_yulei/article/details/22613327)  

[[12] c语言环境初始化&c语言和汇编混合编程](http://www.cnblogs.com/defen/p/4755809.html)  

## 引导扇区

### 词汇解释

#### BIOS

&emsp;BIOS是英文"Basic Input Output System"的缩略词，直译过来后中文名称就是"基本输入输出系统"。其实，它是一组固化到计算机内主板上一个ROM芯片上的程序，它保存着计算机最重要的基本输入输出的程序、系统设置信息、开机后自检程序和系统自启动程序。 其主要功能是为计算机提供最底层的、最直接的硬件设置和控制。  

#### 启动协议

资料：[Linux内核启动协议](http://larmbr.com/2013/09/28/the-evolution-of-x86-real-mode-to-protected-mode-and-of-booting-protocol/)  (小内核时代与大内核时代的对比)

##### zImage：

* 从0x00000至0x01000这4k空间主要是保留给BIOS使用，存放加电自检期间检查到的系统硬件配置。比如BIOS在初始化硬件时，会把中断向量初始化放在地址0开始的物理内存处。
* 接下来从0x01000开始，就是可以存放引导程序的地方。引导程序负责把内核映像加载到内存，再将控制权交给内核。BIOS在完成其加电自检(POST), 检测外围设备等工作后,　就把磁盘设备上的MBR(Master Boot Record)加载到物理地址0x07c00处(而不是恰好在0x01000)。其实MBR正是存放着引导程序。MBR块是硬盘第一个扇区，它的大小只有512字节。如此小的引导程序能把内核映像加载到内存吗？ 在Linux初期，内核映像(zImage,有别于后来的大内核bImage)是很小的, 比如0.1.1版本内核的大小才196KB。所以，512byte的引导程序是可以把内核映像加载到内存的，这部分代码对应于之前的/boot/boot_sect.S这个文件，从名字可以看出，boot sector意即启动扇区，正是指MBR块。它在编译后大小刚好为512字节，能放在MBR块处。

##### bzImage

&emsp;随着内核的发展，内核体积也越来越庞大，进入了大内核时代。像旧时代靠内核自身的512字节的引导程序已经无法完成如此复杂的功能，需要引入专门的引导程序。



* **首先执行基本的引导装载过程**，该程序通常位于主引导记录(MBR)中，大小为512字节，由BIOS将其装入RAM中物理地址0x00007c00处，这就替代了旧内核时代内核自身的引程序，它的任务是建立实模式栈并利用BIOS指令将第二引导加载过程装入内存。



* **第二引导加载过程(又称次引导过程)随后从磁盘读取可用操作系统的映射表**，并提供给用户一个提示符，当用户选择需要被载入的操作系统(如双系统的时候选择Lunix或是windows)，或是经过一段时间的延迟自动选择一个默认值之后，次引导过程便将相应分区下的内核映像以及initrd*（Linux初始RAM磁盘（initrd）是在系统引导过程中挂载的一个临时根文件系统，用来支持两阶段的引导过程。initrd文件中包含了各种可执行程序和驱动程序，它们可以用来挂载实际的根文件系统，然后再将这个 initrd RAM磁盘卸载，并释放内存。）*装载到内存中，而前述的映射表是GRUB(GRand Unified Bootloader）通过读取/boot/grub/grub.conf文件中所设置的内容生成的。另外次引导过程还包括对特定文件系统(如ext2，ext3等)的支持以及对内核启动代码的初始化等职责，这就决定了次引导过程将占用较大的存储空间——连续多个扇区，从而无法装进单个扇区中，因此GRUB通常将该过程放在特定的文件系统中（通常是boot所在的根分区）。



* 次引导过程拷贝到内存的目标中包括一个名为initrd的文件，该文件的全称为boot loader initialized RAM disk，即bootloader初始化内存盘。它主要用于实现一些模块的加载以及文件系统的安装等功能。在次引导过程完成相应文件的加载之后将会执行一个长跳转指令，该指令跳过实模式内核代码的前512个字节，也即跳到由前述链接脚本所指定的执行入口_start处开始执行，而所跳过的512字节正是我们之前剖析的Linux内核自带的引导程序，整个的衔接过程可谓天衣无缝。而内核自带的boot_sect模块已经失去了它的作用，所以，从2.6.24版本的内核开始，已经把boot_sect.S和setup.S文件合并成为一个header.S文件。

对于我们接下来写的引导程序(boot loader)，理解启动协议是非常必要的。

#### 中断

&emsp;关中断是为了保护一些不能中途停止执行的程序而设计的，计算机的CPU进行的是时分复用。在多道程序设计的环境下，CPU是不断地交替地将这些程序的指令一条一条的分别执行。而CPU在这些指令之间的切换就是通过中断来实现的。关中断就是为了让CPU在一段时间内执行同一程序的多条指令而设计的，比如在出现了非常事件后又恢复正常时，CPU就会忙于恢复现场，在恢复现场的时候，CPU是不允许被其他的程序打扰的，此时就要启动关中断，不再相应其他的请求。当现场恢复完毕后，CPU就启动开中断，其他等待着的程序的指令就开始被CPU执行，计算机恢复正常。 

&emsp;boot只是完成硬件初始化，环境参数设置，代码搬运等工作，用不到中断。屏蔽中断是为了避免因为意外中断使得boot失败，最重要的是，对应中断代码还未准备好，所以启动时不能关中断。

#### 实模式

&emsp; 实模式（Real mode）是Intel 80286和之后的x86兼容CPU的操作模式。实模式的特性是一个20bit的区块存储器地址空间（即只有1 MB的存储器可以被定址），可以**直接软件访问BIOS例程以及周边硬件**，没有任何硬件档次的存储器保护观念或多任务。所有的80286系列和之后的x86 CPU都是以实模式下开机；80186和早期的CPU仅仅只有一种操作模式，也就是相当于后来芯片的这种实模式。
&emsp; 实模式下80386不支持优先级，所 有的指令相当于工作在特权级（优先级0）。

#### 链接脚本

&emsp; 在编译器产生可重定向的代码后，需要由链接脚本指出该程序将被加载到内存的什么地方。在输入文件在进行链接的时，每个链接都由链接脚本控制着，脚本由链接器命令语言组成。脚本的主要目的是描述如何把输入文件中的节（sections）映射到输出文件中，并控制输出文件的存储布局。  

#### 寄存器

AX――累加器（ Accumulator） ， 使用频度最高（ AH是它的高八位， AL是它的低八位， 其他同理）
BX――基址寄存器（ Base Register） ， 常存放存储器地址
CX――计数器（ Count Register） ， 常作为计数器
DX――数据寄存器（ Data Register） ， 存放数据
SI――源变址寄存器（ Source Index） ， 常保存存储单元地址
DI――目的变址寄存器（ Destination Index） ， 常保存存储单元地址
BP――基址指针寄存器（ Base Pointer） ， 表示堆栈区域中的基地址
SP――堆栈指针寄存器（ Stack Pointer） ， 指示堆栈区域的栈顶地址
IP――指令指针寄存器（ Instruction Pointer） ， 指示要执行指令所在存储单元的地址。 IP寄存器是一个专用寄存器。 

在8086中设置4个16位的段寄存器， 用于管理4种段：
CS(code segment)是代码段(取指令所用的段寄存器和偏移量一定是用CS和IP)
DS是数据段(data segment)
SS(stack segment)是堆栈段(SP是用来指向该堆栈的栈顶， 把它们合在一起可访问栈顶单元)
ES(extra segment)是附加段(串操作的目标操作数所用的段寄存器和偏移量一定是ES和D)
把内存分段后， 每一个段就有一个段基址， 段寄存器保存的就是这个段基址的高16位， 这个16位的地址左移四位（ 后面加上4个0） 就可构成20位的段
基址。 

### 代码

#### 启动扇区

```asm
vga_sec = 0Xb800		# 1*
clear = 4000
.code16					# assembler directive to start 16-bit real mode 
.text
.org 0x00				# the starting address of MBR

.global _start  
.section "text","ax"        
_start:
	cli					# clear interupt
	mov %cs,%ax			# real-mode register
	mov %ax,%ds
	mov 0x4000, %ax		# the physical address loaded by the bootloader
	mov %ax, %ss
	xor %sp,%sp			# clear screen
	call _boot
	ret
_boot:
	cld					# clear direction flag
	mov $vga_sec,%bx
	mov %bx,%es
	xor %di,%di
	mov $0000,%ax
	mov $clear,%cx		# clear screen
	rep stosb

	mov $msg,%si		# move the address of the string to the reg
	mov $0x7,%ah		# COLOR
	mov $vga_sec,%bx
	mov %bx,%es
	mov $986,%edx		# 2*
	call _disp			# output
	ret
_disp:
	lodsb
	or %al,%al			# check if the string has been completely outputed
	jz _endprint
	mov %ax,%es:0(,%edx,2)
	incl %edx
	jmp _disp
	ret
_endprint:
	hlt
.section ".data","a"
msg:
    .asciz "Hello, World!"
.section ".signature", "a"	
.globl	sig
sig:	.word 0xaa55
```

[1] 1* & 2* : 

&emsp;有两个寄存器：si和di(源寄存器、目的寄存器)，分别对应着指令stosb和lodsb,他们的区别在于：使用di时，是输出di寄存器里存放的ascii码所对应的字符，而使用si时，是输出si寄存器里存放的地址所对应的内存位置中所存放的ascii码对应的字符。  
&emsp; 输出时，先将vga_sec装入es(extra segment)寄存器中，然后使用stosb/lodsb指令，完成输出。因为这次我实现了清屏，所以使用di存放null，完成清屏；使用si完成字符串输出：在ax寄存器中存放着属性和ascii码，用

``` asm
mov %ax,%es:0(,%edx,2)
```

这条指令，通过段偏移的方式将输出字符的信息存放在屏幕上特定位置所对应的内存中去。  
内存中0x8b00开头的位置相当于是“特定”划给vga的(这个位置相当于LC-3里的DDR，将属性和ASCII码写入，即可输出)，这个实际上是映射，这样得以实现计算机与外部设备的数据交换。  

*关于颜色之类的参数调整请自行搜索

[2] CLD：

&emsp; 将标志寄存器Flag的方向标志位DF清零。
&emsp; 在字串操作中使变址寄存器SI或DI的地址指针自动增加， 字串处理由前往后 。

[3] 大小写后缀的区别:
&emsp; .s: 汇编语言源程序;汇编
&emsp; .S: 汇编语言源程序;预处理,汇编&emsp; 
&emsp; 小写的s文件， 在后期阶段不在进行预处理操作， 所以我们不能在这里面写预处理的语句在里面.
&emsp; 大写的S文件， 还会进行预处理、 汇编等操作， 所以我们可以在这里面加入预处理的命令.
ld命令是GNU的连接器， 将目标文件连接为可执行程序。 

#### 链接脚本

``` asm
OUTPUT_FORMAT("elf32-i386", "elf32-i386", "elf32-i386")

OUTPUT_ARCH(i386)

ENTRY(_start)


SECTIONS
{
	
	. = 0x7C00;
	
	.text		: { *(.text) }
	
	.data		: { *(.data) }

	
	. = 0x7C00 + 510;
	
	.signature		: { *(.signature) }		# 4kb = 512 bytes

}
```

#### 编译并执行

```shell
#!/bin/sh
gcc -c -m32 start16_hello.S -o start16_hello.o
ld -T start16_hello.ld start16_hello.o -o start16_hello.elf # -T for <filename>.ld
objcopy -O binary start16_hello.elf start16_hello.bin # or anything else will do
dd if=/dev/zero of=a_start16_hello.img bs=512 count=2880 # start making booting floppy
sudo losetup /dev/loop4 a_start16_hello.img
sudo dd if=start16_hello.bin of=/dev/loop4 bs=512 count=1
sudo losetup -d /dev/loop4
qemu-system-i386 -fda a_start16_hello.img
```

&emsp; 到这一步为止，我们已经完成了一个启动扇区，并得到了一个启动软盘，并直接使用vga接口输出了Hello world的问候语。 

## 保护模式

### 名词解释

#### BIOS中断

   CPU是根据中断号获取中断向量值，即对应中断服务程序的入口地址值。（类似于LC-3的trap，对应的向量表里存放着指令块的地址，转跳到对应的地址就能看见指令，实际上和上一次我们手动存到vga_seg的指令原理上差不多）。因此为了让CPU由中断号查找到对应的中断向量，就需要在内存中建立一张查询表，即中断向量表(在32位保护模式下该表称为中断描述符表)。80x86微机支持256个中断，对应每个中断需要安排一个中断服务程序。
  在80x86实模式运行方式下，每个中断向量由4字节组成。这4字节指明了一个中断服务程序的段值和段内偏移值。因此整个向量表的长度为1KB。当80x86微机启动时，ROM BIOS中的程序会在物理内存开始地址0x0000:0x0000处初始化并设置中断向量表，而各中断的默认中断服务程序则在BIOS中给出。由于中断向量表中的向量是按中断号顺序排列，因此给定一个中断号N，那么它对应的中断向量在内存中的位置就是0x0000:N×4，即对应的中断服务程序入口地址保存在物理内存0x0000:N×4位置处。
  若要在实模式下使用BIOS中断，需要在ah中存入想要使用的模式，然后调用int 0x10，选择0xe（put character）和直接写字符进入vga section的方式很像，不赘述。使用0x13(put string)则需要在cx中存储字符串的长度。
  BIOS中断只能在实模式中使用，所以进入保护模式后，如果要使用BIOS的程序，需要完成保护模式到实模式的转跳。

#### 实模式与保护模式

##### 实模式（20位）：  

&emsp; 16位段寄存器只记录段基址的高16位，因此段基址必须4位对齐（末4位为0），不采用虚拟地址空间，直接采用物理地址=段寄存器值*16+段内偏移。

##### 保护模式（32位）

 &emsp; 16位段寄存器无法直接记录段的信息，因此需要与全局描述符表GDT配合使用。GDT中记录了每个段的信息（段描述符），段寄存器只需记录段在GDT中的序号。  

##### 综述  

&emsp; 在IA32下， CPU有两种工作模式： 实模式和保护模式。 直观地看， 当我们打开自己的PC，开始时CPU是工作在实模式下的，经过某种机制之后，才进入保护模式。在保护模式下，CPU有着巨大的寻址能力，并为强大的32位操作系统提供了更好的硬件保障。     

&emsp; 在实模式下，16位的寄存器需要用“ 段： 偏移” 这种方法才能达到1MB的寻址能力，如今我们有了32位寄存器，一个寄存器就可以寻址4GB的空间， 是不是从此段值就被抛弃了呢？ 实际上并没有，新政策下的地址仍然用“ 段:偏移” 这样的形式来表示，只不过保护模式下“ 段” 的概念发生了根本性的变化。实模式下，段值还是可以看做是地址的一部分的，段值为XXXXh表示以XXXX0h开始的一段内存。而保护模式下，虽然段值仍然由原来16位的cs、 ds等寄存器表示，但此时它仅仅变成了一个索引，这个索引指向一个数据结构的一个表项，表项中详细定义了段的起始地址、界限、 属性等内容。 这个数据结构，就是GDT（实际上还可能是LDT）。GDT中的表项也有一个专门的名字，叫做描述符（Descriptor）。

##### 转变过程

* 保存实模式下的SP内的值
* 初始化段描述符
* 加载GDTR
* 关中断
* 打开地址线A20
* 准备切换到保护模式，置cr0的末位为1
* 跳转到保护模式

&emsp; 寄存器cr0的第0位是PE位， 此位为0时， CPU运行于实模式， 为1时， CPU运行于保护模式。 原来我们已经闭合了进入保护模式的开关， 也就是说， “ mov cr0, eax” 这一句之后， 系统就运行于保护模式之下了。 但是， 此时cs的值仍然是实模式下的值， 我们需要把代码段的选择子装入cs。 所以， 我们需要jmp指令：

```asm
jmp	dword SelectorCode32:0		;INTEL
ljmp $SelectorCode32,$0			#AT&T
```

&emsp; 注：长转移指令，能无条件在64KB内跳转。  

&emsp; 根据寻址机制我们知道，这个跳转的目标将是描述符DESC_CODE32对应的段的首地址，即标号LABEL_SEG_CODE32处。  

### GDT表

#### 简介

&emsp; 每个段有8位的段描述符(segment descriptor),存储在内存里。 寄存器GDTR存储着全局描述符表的基址，长48bits：  

1. 低16位是GDT的大小  

2. 高32位是GDT在内存中的位置  



&emsp; 全局描述符表GDT（Global Descriptor Table）在整个系统中，全局描述符表GDT只有一张(一个处理器对应一个GDT)，GDT可以被放在内存的任何位置，但CPU必须知道GDT的入口，也就是基地址放在哪里，Intel的设计者门提供了一个寄存器GDTR用来存放GDT的入口地址，程序员将GDT设定在内存中某个位置之后，可以通过LGDT指令将GDT的入口地址装入此寄存器，从此以后，CPU就根据此寄存器中的内容作为GDT的入口来访问GDT了。GDTR中存放的是GDT在内存中的基地址和其表长界限。

指令：

```asm
lgdt src
```

#### 实现

##### 段选择子

1. Index: 13 bits, 相应段描述符在GDT中的基址（相当于偏移量）  

2. Table Indicator, TI-bit: 1 bit, 选择GDT和LDT(局部描述符表，主要存放各个任务的私有描述符，如本任务的代码段描述符和数据段描述符等)  

3. Request privilege level, RPL-bits: 2 bits



&emsp; （这次我们只实现一个简单的GDT,一共有三部分：dummy，CD(code descriptor)，VD(vedio descriptor)）  

&emsp; “ 段:偏移” 形式的逻辑地址（Logical Address）经过段机制转化成“ 线性地址” （ Linear Address），而不是“ 物理地址” （ Physical Address）。 在上面的程序中， 线性地址就是物理地址。 另外， 包含描述符的， 不仅可以是GDT， 也可以是LDT。

### 代码

&emsp;我们只需要在上一次的基础上更改，并注意进入保护模式的输出方式与原来的有所不同。

```asm
reset_floppy:
	xorw	%ax, %ax				    # int 13h reset floppy
	xorb	%dl, %dl
	int     $0x13
	ret
```

！请自行查阅int 13h相关资料

```asm
# GDT
	.p2align 4
gdt:
	.word	0,0,0,0
	
	# CS
	.word	0xffff, 0
	.byte	0, 0x9a, 0xcf, 0

     # DS
	.word	0xffff, 0
	.byte	0, 0x92, 0xcf, 0
	# DUMMY
	.word	0,0,0,0
```

```asm
load_gdt:
	lgdt	gdtptr						

										# turn on protect-mode
	movw	$1, %ax
	lmsw	%ax      

	jmp	flush_instr						# reflesh cache
flush_instr:
	movl	$ProtectMode_DS, %eax		# reinitialize registers
	movw	%ax, %ds
	movw	%ax, %es
	movw	%ax, %fs
	movw	%ax, %gs

	movw	%ax, %ss
	movl	$0x4000, %ebp
	movl	$0x4000, %esp

										# set code segment
	ljmp	$ProtectMode_CS, $start32
```

## 进程调度

### 名词解释

#### os与扇区加载

&emsp; 在进入保护模式前，先加载OS。  

&emsp; 从第二个扇区开始读若干个扇区。 加载七个扇区。 

&emsp; MBR 中的主引导加载程序是一个 512 字节大小的映像，其中包含程序代码和一个小分区表。前 446 个字节是主引导加载程序，其中包含可执行代码和错误消息文本。接下来的 64 个字节是分区表，其中包含 4 个分区的记录（每个记录的大小是 16 个字节）。MBR 以两个特殊数字的字节（0xAA55）结束。这个数字会用来进行 MBR 的有效性检查。  

&emsp; 主引导加载程序的工作是查找并加载次引导加载程序（第二阶段）。它是通过在分区表中查找一个活动分区来实现这种功能的。当找到一个活动分区时，它会扫描分区表中的其他分区，以确保它们都不是活动的。当这个过程验证完成之后，就将活动分区的引导记录从这个设备中读入 RAM 中并执行它。  

#### FAT12  

&emsp; FAT12 是DOS时代就开始使用的文件系统（File System），直到现在仍然在软盘上使用。 几乎所有的文件系统都会把磁盘划分为若干层次以方便组织和管理， 这些层次包括：

1. 扇区（Sector）：磁盘上的最小数据单元。
2. 簇（Cluster）：一个或多个扇区。
3. 分区（Partition）：通常指整个文件系统。

&emsp; 我们已经接触过引导扇区，就让我们从这里开始。引导扇区是整个软盘的第0个扇区，在这个扇区中有一个很重要的数据结构叫做BPB（BIOS ParameterBlock），紧接着引导扇区的是两个完全相同的FAT表， 每个占用9个扇区。 第二个FAT之后是根目录区的第一个扇区。 根目录区的后面是数据区。根目录区位于第二个FAT表之后， 开始的扇区号为19， 它由若干个目录条目（ Directory Entry） 组成， 条目最多有BPB_RootEntCnt个。 由于根目录区的大小是依赖于BPB_RootEntCnt的， 所以长度不固定。

#### 从汇编进入C

&emsp; 需要为c的执行设立栈：位置一般在0x4000,BSS段里是未初始化的数据，清零的好处有很多，它使得代码的运行结果可以复现（如果未初始化的值每次都不一样，每次跑的结果也不一样）。

&emsp;使用C语言编写写VGA缓存和汇编写VGA缓存的本质都是一样的，是向x8b000这个位置写入信息(ascii码、颜色等)，不同的是c语言中直接使用的就是0x8b000这个地址，而汇编中用的是段偏移的方法。  

### 代码

```asm
	call	myMain			# .global myMain
```

！Attention:

&emsp; “ 一致” :  当转移的目标是一个特权级更高的一致代码段， 当前的特权级会被延续下去， 而向特权级更高的非一致代码段的转移会引起常规保护错误（ general-protection exception， #GP），除非使用调用门或者任务门。 如果系统代码不访问受保护的资源和某些类型的异常处理（比如，除法错误或溢出错误），它可以被放在一致代码段中。 为避免低特权级的程序访问而被保护起来的系统代码则应放到非一致代码段中。

&emsp;如果目标代码的特权级低的话，无论它是不是一致代码段， 都不能通过call或者jump转移进去，尝试这样的转移将会导致常规保护错误。

&emsp;所有的数据段都是非一致的， 这意味着不可能被低特权级的代码访问到。与代码段不同的是，数据段可以被更高特权级的代码访问到， 而不需要使用特定的门。

```c
void stack_init(unsigned long** stk,void (*task)(void))
{
  *(*stk)-- = (unsigned long) 0x08;        // CS 高地址
  *(*stk)-- = (unsigned long) task;        // eip
  *(*stk)-- = (unsigned long) 0xAAAAAAAA;  // EAX 
  *(*stk)-- = (unsigned long) 0xCCCCCCCC;  // ECX
  *(*stk)-- = (unsigned long) 0xDDDDDDDD;  // EDX 
  *(*stk)-- = (unsigned long) 0xBBBBBBBB;  // EBX
  *(*stk)-- = (unsigned long) 0x44444444;  // ESP
  *(*stk)-- = (unsigned long) 0x55555555;  // EBP
  *(*stk)-- = (unsigned long) 0x66666666;  // ESI
  *(*stk) = (unsigned long) 0x77777777;    // EDI 低地址    
}
```

```asm
CTX_SW: 
	pusha 
	movl %esp, prevSP 
	movl nextSP, %esp 
	popa 
	ret 
```

！请自行查阅上下文切换相关内容

```c
typedef struct myTCB {
	//unsigned long state; //PUT IN STACK[0]  
	//int tcbIndex;		//PUT IN STACK[1]
	struct myTCB * next;
	unsigned long* stkTop;
	unsigned long* stack;  
}myTCB,*TNode;
```

一个简单的调度只需要完成从等待队列中装载进程和上下文切换即可。

## 内存管理

！进入C后的工作比写汇编时简单很多，我就简单地给出需要实现的模块：

 ![img](http://drive.google.com/uc?export=view&id=0B_0Ovo-1o-jAUk9zLUlaSGNSajg) 

![img](http://drive.google.com/uc?export=view&id=0B_0Ovo-1o-jAcVo2dFAyUk94RUk)

## 时间片轮转

### 名词解释&代码

#### 中断初始化8259A

&emsp;在CPU中有两个用来控制中断的控制器：可编程中断控制器8259A。  

&emsp;可屏蔽中断与CPU的关系是通过对可编程中断控制器8259A建立起来的。它根据优先级在同时发生中断的设备中选择应该处理的请求， 而且可以通过对其寄存器的设置来屏蔽或打开相应的中断。  

&emsp;两片级联的8259A与CPU相连。 在BIOS初始化它的时候， IRQ0～IRQ7被设置为对应向量号08h～0Fh， 而通过表3.8我们知道，在保护模式下向量号08h-0Fh已经被占用了，所以我们不得不重新设置主从8259A。  

&emsp;8259A是可编程中断控制器， 对它的设置并不复杂， 是通过向相应的端口写入特定的ICW（Initialization Command Word） 来实现的。 主8259A对应的端口地址是20h和21h， 从8259A对应的端口地址是A0h和A1h。 ICW共有4个， 每一个都是具有特定格式的字节。   

&emsp;ICW的格式初始化过程：  

1. 往端口20h（ 主片） 或A0h（ 从片） 写入ICW1。  



2. 往端口21h（ 主片） 或A1h（ 从片） 写入ICW2。  



3. 往端口21h（ 主片） 或A1h（ 从片） 写入ICW3。  



4. 往端口21h（ 主片） 或A1h（ 从片） 写入ICW4。  



```asm
Init8259A:  

    mov    $0x11, %al
    out    %al, $0x20    ; 主8259, ICW1.

    out     %al, $0xA0   ; 从8259, ICW1.
    mov    &0x20, %al    ; IRQ0 对应中断向量 0x20

    out    %al,$0x21     ; 主8259, ICW2.
    mov    $0x28, %al    ; IRQ8 对应中断向量 0x28

    out    %al,$0xA1    ; 从8259, ICW2.
    mov    $0x04h, %al   ; IR2 对应从8259

    out    %al, $0x21    ; 主8259, ICW3.
    mov    $0x02, %al    ; 对应主8259的 IR2

    out    %al,$0xA1h    ; 从8259, ICW3.
    mov    $0x01, %al

    out    %al,$0x21        ; 主8259, ICW4.
    out     %al,$0xA1   ; 从8259, ICW4.

    mov    $0xff,%al    ; 仅仅开启定时器中断
    out    %al,$0x21    ; 主8259, OCW1.

    mov    $0xff,%al    ; 屏蔽从8259所有中断
    out    %al,$0A1h    ; 从8259, OCW1.

    ret
```

#### 时钟初始化  

&emsp; 8253 芯片接收主板上一个石英震荡器产生的频率，石英震荡器每秒震荡1193180次，所以8253 芯片的主频就是1.193180Mhz。我们又知道计数器从65536递减到0就会产生一个方波，所以OUT0 引脚会每秒产生1.193180Mhz/65536 = 18.2 次的方波信号。我们用晶振信号作为时钟信号。

```asm
init8253

    mov $0x34,%al

    out %al,$0x43

    mov $(11932 & 0xff), %al

    out %al,$0x40

    mov $(11932>>8),%al

    out %al,$0x40

    in $0x21,%al

    andb $0xFE,%al

    out %al,$0x21

    ret
```

#### 开关中断

```asm
enable_interrupt:
    sti        //开中断
    ret
```

```asm
disable_interrupt:
    cli        //关中断
    ret
```

#### 接口

```c
//=======interrupt & timer manager============  
void init8259A();      //中断

void init8253();    //时钟  

void enable_interrupt();      //开中断

void disable_interrupt();      //关中断

void tick();  
volatile int tick_number;
```

![img](http://drive.google.com/uc?export=view&id=0B_0Ovo-1o-jAUk9zLUlaSGNSajg)

也许之后我会写上文件系统部分

