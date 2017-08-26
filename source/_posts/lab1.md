title: lab1-boot-pc
date: 2017/04/10
tags:
 - xv6
 - os

---

# Lab1 启动pc
## lab1分为三部分
> 1.熟悉x86汇编语言，qemu x86模拟器，启动pc
2.分学习6.828内核的boot loader部分。
3.深入研究6.828内核JOS的初始化部分，该部分代码在kernel目录下


## 1. 实验代码下载
```bash
git clone https://pdos.csail.mit.edu/6.828/2016/jos.git lab
cd lab
```
## 2. part1 启动pc
### 启动 qemu
> ` make qemu`
![make qemu](./boot.png)
现在JOS内核只有两条命令来监视内核。help和kerninfo
![kerninfo](./kerninfo.png)

<!-- more -->

### PC物理地址空间
```
+------------------+  <- 0xFFFFFFFF (4GB)
|      32-bit      |
|  memory mapped   |
|     devices      |
|                  |
/\/\/\/\/\/\/\/\/\/\

/\/\/\/\/\/\/\/\/\/\
|                  |
|      Unused      |
|                  |
+------------------+  <- depends on amount of RAM
|                  |
|                  |
| Extended Memory  |
|                  |
|                  |
+------------------+  <- 0x00100000 (1MB)
|     BIOS ROM     |
+------------------+  <- 0x000F0000 (960KB)
|  16-bit devices, |
|  expansion ROMs  |
+------------------+  <- 0x000C0000 (768KB)
|   VGA Display    |
+------------------+  <- 0x000A0000 (640KB)
|                  |
|    Low Memory    |
|                  |
+------------------+  <- 0x00000000
```
>　　早期基于16位Intel 8088处理器只能操作1MB物理内存，因此物理地址空间起始于0x00000000到0x000FFFFF，其中640KB为 Low memory，这只能被随机存储器(RAM)使用。
　　从 0x000A0000 到 0x000FFFFF的384KB留着给特殊使用，例如作为视频显示缓存或者储存在非易失存储器的硬件。从 0x000F0000 到 0x000FFFFF 占据64KB区域的部分是最重要的BIOS.
　　现在的x86处理器支持超过4GB的物理RAM，所以RAM扩展到了0xFFFFFFFF。当然，BIOS也流出了开始的32位寻址空间为了让32位的设备映射。JOS这里只用开始的256MB，所以假设PC只有32位地址空间。

### **BIOS**
> 在一个终端中输入 make qemu-gdb, 另一个终端输入 make gdb.开始调试程序。
> ```asm
[f000:fff0] 0xffff0: ljmp 0xf000, 0xe05b
> ```
> 上面是GDB反汇编出的第一条执行指令，这条指令表面了：
IBM PC 执行的起始物理地址为 0x000ffff0,PC 的偏移方式为 CS = 0xf000，IP = 0xfff0.第一条指令执行的是 jmp指令，跳转到段地址 CS = 0xf000，IP = 0xe05b
QEMU模拟了8088处理器的启动，当启动电源，BIOS最先控制机器，这时还没有其他程序执行，之后处理器进入实模式也就是设置 CS 为 0xf000，IP 为 0xfff0。在启动电源也就是实模式时，地址转译根据这个公式工作：物理地址 = 16 * 段地址 + 偏移量。所以 PC 中 CS 为 0xf000 IP 为 0xfff0 的物理地址为：
   16 * 0xf000 + 0xfff0   # 十六进制中乘16,左移4位
   = 0xf0000 + 0xfff0
   = 0xffff0
0xffff0 在 BIOS (0x100000) 的结束地址之前。
当BIOS启动，它设置了一个中断描述符表并初始化多个设备比如VGA显示器。在初始化PCI总线和所有重要的设备之后，它寻找可引导的设备，之后读取boot loader 并转移控制。

## Part 2: The Boot Loader
> 512 byte是区域的扇区是硬盘最小调度单位，每次读或写操作都至少是一个扇区，并且还会进行对齐。BIOS加载引导扇区到内存中是从物理地址0x7c00到0x7dff，然后使用jmp指令设置 CS:IP 为 0000:7c00。因此 boot loader 不能超过512字节，它执行两个功能：
>> 1. boot loader 切换处理器从实模式到保护模式，这样能访问大于1MB的物理地址空间。
2. boot loader 从硬盘中读取内核。

### Boot
> 通过 `b *0x7c00`设置断点，接着`c`运行到断点处，使用`x/i` 来查看当前的指令。

> 在哪执行了32位代码？
>>`[0:7c2d] => 0x7c2d: ljmp $0x8,$0x7c32 `这条指令之后，即boot.S 中的 `ljmp $PROT_MODE_CSEG, $protcseg` ，地址符号就变成` 0x7c32 `了。

> 最后一条 boot loader 指令后, 执行第一条内核指令在哪里？
>> boot loader 最后一步是加载kernel，所以在 boot/main.c 中可以找到 
>> ```c
\* call the entry point from the ELF header
((void (*)(void)) (ELFHDR->e_entry))()
```
>表明这是准备读取ELF头。
通过 `objdump -x obj/kern/kernel `可以查看kernel的信息，开头为` start address 0x0010000c`，通过 `b *0x10000c`然后在` c` 能得到执行的指令是 `movw $0x1234,0x472`.

>第一条kernel指令在哪里
>> 同上一条的问题，存在kern/entry.S中。

> 设置一个断点在地址0x7c00处，这是boot sector被加载的位置。然后让程序继续运行直到这个断点。跟踪/boot/boot.S文件的每一条指令，同时使用boot.S文件和系统为你反汇编出来的文件obj/boot/boot.asm.也可以使用GDB的x/i指令来获取去任意一个机器指令的反汇编指令，把源文件boot.S文件和boot.asm文件以及在GDB反汇编出来的指令进行比较。
追踪到bootmain函数中，而且还要具体追踪到readsect()子函数里面。找出和readsect()c语言程序的每一条语句所对应的汇编指令，回到bootmain()，然后找出把内核文件从磁盘读取到内存的那个for循环所对应的汇编语句。找出当循环结束后会执行哪条语句，在那里设置断点，继续运行到断点，然后运行完所有的剩下的语句。

> boot.S
```asm
start:
  .code16                     # 16位汇编模式
  cli                         # 关中断
  cld                         # 操作方向标志位DF，使DF=0。
```
>```asm
  lgdt    gdtdesc
  movl    %cr0, %eax
  orl     $CR0_PE_ON, %eax
  movl    %eax, %cr0
# 切换到保护模式后，加载GDT(Global Descriptor Table)，接着修改了cr0寄存器的值，$CR0_PE_ON值为0x1，代表启动保护模式的flag标志。
```
>```asm
  movl    $start, %esp
  call bootmain
  # 设置栈指针，接着开始调用bootmain函数。
```
>```asm
    7d15:    55                       push   %ebp
    7d16:    89 e5                    mov    %esp,%ebp
    7d18:    56                       push   %esi
    7d19:    53                       push   %ebx
    # 参数压栈, 准备进入函数
```
>```asm
    // read 1st page off disk
    readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);
    7d1a:    6a 00                    push   $0x0
    7d1c:    68 00 10 00 00           push   $0x1000
    7d21:    68 00 00 01 00           push   $0x10000
    7d26:    e8 b1 ff ff ff           call   7cdc <readseg>
    # 调用readseg函数，对应3个参数(物理地址，页的大小，偏移量)
```
>```asm
0x7ceb: shr $0x9,%edi 
# 执行 offset = (offset / SECTSIZE) + 1, 除法求出扇区号。
0x7cee: add %ebx,%esi 
# 执行 end_pa = pa + count, 计算这个扇区结束的物理地址。
0x7cf0: inc %edi 
# 执行了 offset = (offset / SECTSIZE) + 1中的加1。
0x7cf1: and $0xfffffe00,%ebx 
# 执行了 pa &= ~(SECTSIZE - 1);。
0x7cf7:    cmp    %esi,%ebx
0x7cf9:    jae    0x7d0d
# 执行 while (pa < end_pa) 循环判断语句。
```
>```asm
# 加载程序段
7d3a:    a1 1c 00 01 00           mov    0x1001c,%eax
7d3f:    0f b7 35 2c 00 01 00     movzwl 0x1002c,%esi
7d46:    8d 98 00 00 01 00        lea    0x10000(%eax),%ebx
7d4c:    c1 e6 05                 shl    $0x5,%esi
7d4f:    01 de                    add    %ebx,%esi
# 接着循环调用readseg函数，将Program Header Table中表项读入内存。
# 最后加载内核
((void (*)(void)) (ELFHDR->e_entry))();
```

### 加载内核
> 　　为了理解 boot/main.c，需要了解ELF二进制文件。编译并链接比如JOS内核这样的C程序，编译器会将源文件(.c)转为包含汇编指令的目标文件(.o)。接着链接器把所有的目标文件组合成一个单独的二进制镜像（binary image），比如 obj/kern/kernel，这种文件就是ELF(是可执行可链接形式的缩写)。
　　当前只需要知道，可执行的ELF文件由带有加载信息的头，多个程序段表组成。每个程序段表是一个连续代码块或者数据，它们要被加载到内存具体地址中。boot loader 不修改源码和数据，直接加载到内存中并运行。
　　ELF开头是固定长度的ELF头，之后是一个可变长度的程序头，它列出了需要加载的程序段。ELF头的定义在 inc/elf.h 中。主要学习以下3个程序段：
>>.text: 程序执行指令
.rodata:只读数据，比如ASCII字符串
.data: 存放程序初始化的数据段，比如有初始值的全局变量。

>　　当链接器计算程序内存布局时，会在内存里紧挨着.data段的.bss段中保留空间给未初始化的全局变量。C规定未初始化的全局变量为0。因此没必要在ELF的.bss段储存内容，链接器只储存了.bss段的地址和大小。
使用 `objdump -h obj/kern/kernel `可以查看ELF头的相关信息。
　　重点关注 .text段 的VMA(链接地址)和LMA(加载地址)，段的加载地址即加载进内存的地址。段的链接地址就是这个段预计在内存中执行的地址。
　　回到 boot/main.c， ph->p_pa是每个程序头包含的段目的物理地址。BIOS把引导扇区加载到内存地址0x7c00，这也就是引导扇区的加载地址和链接地址。在 boot/Makefrag 中，是通过传 -Ttext 0x7C00 这个参数给链接程序设置了链接地址，因此链接程序在生成的代码中产生正确的内存地址。


## Part 3: The Kernel
### 使用虚拟内存
> 　　boot loader 的链接地址和加载地址是一样的，然而 kernel 的链接地址和加载地址有些差异。查看 kern/kernel.ld 可以发现内核地址在 0xF0100000。
　　操作系统内核通常被链接并且运行在非常高的虚拟地址，比如文件里看到的 0xf0100000，为了让处理器虚拟地址空间的低地址部分给用户程序使用。
许多机器没有地址为 0xf0100000的物理内存，所以内核不能放在那儿。因此使用处理器内存管理硬件将虚拟地址 0xf0100000 (内核希望运行的链接地址)映射到物理地址 0x00100000 (boot loader加载内核后所放的物理地址)。尽管内核虚拟地址很高，但加载进物理地址位于1MB的地方仅仅高于BIOS的ROM。这需要PC至少有1MB的物理内存。
在下一个lab，会映射物理地址空间底部256MB，也就是 0x00000000 到 0x0fffffff，到虚拟地址0xf0000000~0xffffffff。所以JOS只使用物理内存开始的256MB。
　　目前，只是映射了物理内存开始的4MB， 使用手写的静态初始化页目录和也表在 kern/entrypgdir.c。当 kern/entry.S 设置 CR0_PG 标记，存储器引用就变为虚拟地址，即存储器引用是由虚拟存储器硬件转换为物理地址的虚拟地址。entry_pgdir 将虚拟地址 0xf0000000 ~ 0xf0400000 转换为物理地址 0x00000000 ~ 0x00400000，虚拟地址 0x00000000 ~ 0x00400000 也转换为物理地址 0x00000000 ~ 0x00400000。任何不在这两个范围内的虚拟地址会导致硬件异常。

### 堆栈
> C语言是如何在x86框架上使用堆栈的?需要查看指令寄存器(IP)的值的变化。
研究内核是在哪初始化堆栈，找出堆栈存放在内存的位置。内核是如何保存一块空间给堆栈的？堆栈指针指向这块区域的哪儿？
看了几个文件以后，发现在 kern/entry.S 中提到了设置堆指针和栈指针。
> ```asm
    # Clear the frame pointer register (EBP)
    # so that once we get into debugging C code,
    # stack backtraces will be terminated properly.
    movl    $0x0,%ebp            # nuke frame pointer

    # Set the stack pointer
    movl    $(bootstacktop),%esp
```
> 为了查看堆的位置，所以要使用gdb，同样还是` b *0x10000c `打断点进入 entry。 si 一步步执行，在 `0x10002d: jmp *%eax `之后，下一条指令变为 `0xf010002f <relocated>: mov $0x0,%ebp`。其实地址应该还是 `0x10002f`，所以这里的 `0xf010002f `是因为开启的虚拟地址。
通过 gdb 发现 `0xf0100034 <relocated+5>: mov $0xf0110000,%esp`， 也就是说%esp也就是bootstacktop的值为0xf0110000。其中 kern/entry.S 的 KSTKSIZE 应该就是堆栈的大小，通过跳转，发现在 inc/memlayout.h 里提到了堆栈。
```c
// Kernel stack.
#define KSTACKTOP    KERNBASE
#define KSTKSIZE    (8*PGSIZE)           // size of a kernel stack
#define KSTKGAP        (8*PGSIZE)           // size of a kernel stack guard
# PGSIZE 定义在 inc/mmu.h 中，值为 4096，所以 KSTKSIZE 为 32KB。 使用 info registers可以查出esp和ebp的值。最高地址为bootstacktop的值，也就是0xf0110000。
```

##  参考链接
> * [boot pc](https://pdos.csail.mit.edu/6.828/2016/labs/lab1/)