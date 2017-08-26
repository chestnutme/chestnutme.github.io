title: lab4-Preemptive Multitasking
date: 2017/05/04
tags:
- xv6
- os

---
# Lab4 抢占式多进程
## 摘要
Lab4目标是在多用户环境下实现抢占式多进程的支持，分解为三个部分：
* part1: 在JOS中添加对多处理器的支持，实现循环调度算法，并添加基本的环境管理的系统调用（创建/销毁环境，分配/映射内存）
* part2：实现类似Unix的`fork`机制，允许在用户模式下创建副本
* part3：添加对进程间通信(IPC)的支持，允许不同的用户环境进行进程间的通信和同步。并添加对硬件时钟中断和进程抢占的支持。

## 准备
下载实验代码：
``` bash
git pull
git chekout -b lab4 origin/lab4
git merge Lab3
```
Lab4 新增的文件：
```
kern/cpu.h	多处理器支持的内核私有定义
kern/mpconfig.c	读取多处理器配置的代码
kern/lapic.c	驱动处理器中的本地APIC单元的内核代码
kern/mpentry.S	非启动CPU的汇编代码入口
kern/spinlock.h	包括大内核锁在内的自旋锁的内和私有定义
kern/spinlock.c	自旋锁的内核代码实现
kern/sched.c	调度器的代码框架
```

<!-- more -->

## part1 多处理器支持和多任务协同
Lab4 part1将首先扩展JOS以在多处理器系统上运行，然后实现新的JOS内核系统调用，以允许用户级环境创建其他新的环境。另外实现协同循环调度，当当前环境自动放弃CPU（或退出）时，允许内核从一个环境切换到另一个环境。之后在part3中，还会实现抢占式调度，即使在环境不允许的情况下，允许内核在一定时间内重新从环境中重新控制CPU。

### 多处理器支持
目标使JOS支持对称多处理器(SMP)。`SMP`是一种多处理器架构，所有的CPU对等地访问系统资源。在SMP中所有CPU的功能是相同的，但是在启动过程中会被分为2类：引导处理器(`BSP`)负责初始化系统来启动操作系统，当操作系统被启动后，应用处理器(`AP`)被引导处理器激活。引导处理器是由硬件和BIOS决定的，目前所有的代码都是运行在BSP上。
在个SMP系统中，每个CPU有1个附属的`Local APIC（LAPIC）`单元。LAPIC单元负责处理系统中的中断，同时为它关联的CPU提供独一无二的标识符。在part1，将使用LAPIC单元以下基本功能(在`/kern/lapic.c`)：
* 读取LAPIC的标识符(APIC ID)来告诉我们正在哪个CPU上运行代码(`cpunum()`).
* 从BSP发送STARTUP的处理器间中断(`IPI`) 到APs来唤醒其它CPU(`lapic_startup()`).
* 在part3部分，将编程LAPIC内置的计时器来触发时钟中断来支持多任务抢占(`apic_init()`).

处理器访通过内存映射I/O(MMIO)的方式访问它的LAPIC。在MMIO中，一部分物理地址被硬连接到一些IO设备的寄存器上，导致操作内存的指令`load/store`可以直接操作设备的寄存器。我们已经看到过1个`IO hole`在物理地址`0xA0000`(用来写入VGA显示缓存)。LAPIC的hole开始于物理地址`0xFE000000`(4GB之下地32MB)，但是这地址过高，导致无法访问通过过去的直接映射(虚拟地址`0xF0000000`映射0x0，即只有256MB)。但是JOS虚拟地址映射预留了4MB空间在`MMIOBASE`处，需要分配空间一映射设备。

#### 练习1
问
> 在`kern/pmap.c`中实现`mmio_map_region`。查看`lapic_init`函数(`kern/lapic.c`)来确定使用方式

解：
> 在`lapic_init`函数的开头就会调用`mmio_map_region`
``` c
// lapicaddr is the physical address of the LAPIC's 4K MMIO
// region.  Map it in to virtual memory so we can access it.
lapic = mmio_map_region(lapicaddr, 4096);
```
> 在`kern/pmap.c`中，有具体的提示，设置个静态变量记录每次变化后的虚拟基地址，使用`boot_map_region`函数将`[pa,pa+size)的`物理地址映射到`[base,base+size)`，把`size`向上进位到`PGSIZE`。由于这是设备内存并不是正常的DRAM，所以使用cache缓存访问是不安全的，可以用页的标志位来实现。
``` c
//kern/pmap.c
void *
mmio_map_region(physaddr_t pa, size_t size)
{
    static uintptr_t base = MMIOBASE;
    void *ret = (void *)base;

    size = ROUNDUP(size, PGSIZE);
    if (base + size > MMIOLIM || base + size < base)
                panic("mmio_map_region: reservation overflow\n");

    boot_map_region(kern_pgdir, base, size, pa, PTE_P | PTE_PCD | PTE_PWT);
    base += size;

    return ret;
}
```

### 引导应用处理器(Application Processor)
在启动AP之前，BSP应该先收集关于多处理器系统的配置信息，比如CPU总数，CPUs的APIC ID和LAPIC单元的MMIO地址等。在`kern/mpconfig`文件中的`mp_init()`函数通过读BIOS设定的`MP配置表`获取这些信息。
`boot_aps(kern/init.c)`函数驱动AP引导程序。AP开始于实模式，跟BSP的开始相同，故此`boot_aps`函数拷贝AP入口代码(`kern/mpentry.S`)到实模式下的内存寻址空间。但是跟BSP不一样的是，当AP开始执行时，需要有一些控制将拷贝入口代码到`0x7000(MPENTRY_PADDR)`。
之后，`boot_aps`函数通过发送`STARTUP的IPI`(处理器间中断)信号到AP的LAPIC单元来一个个地激活AP。在`kern/mpentry.S`中的入口代码跟`boot/boot.S`中的代码类似。在一些简短的配置后，它使AP进入开启分页机制的保护模式，调用C语言的setup函数`mp_main`（`kern/init.c`)。`boot_aps()`等待AP `CPU_STARTED`在`cpu_status`其字段中设定出一个flag，设定`struct CpuInfo`然后继续唤醒下一个。

结合前几个lab，在i386 `init`函数中进行BSP启动的一些配置，经由lab2的`mem_init`，lab3的`env_init`和`trap_init`，lab4的`mp_init`和`lapic_init`，然后`boot_aps`函数启动所有的CPU
``` c
// lab2 memory management initialization functions
mem_init();

// lab3 user environment initialization functions
env_init();
trap_init();

// lab4 multiprocessor initialization functions
mp_init();
lapic_init();

// lab4 multitasking initialization functions
pic_init();

// Acquire the bit kernel lock before waiting AP
lock_kernel();

// starting non-boot APs
boot_aps();
```
多核处理器的初始化都在mp_init函数中完成，首先是调用mpconfig函数，主要功能是寻找一个MP 配置条目，然后对所有的CPU进行配置，找到启动的处理器。
``` c
void
mp_init(void)
{
    struct mp *mp;
    struct mpconf *conf;
    struct mpproc *proc;
    uint8_t *p;
    unsigned int i;

    bootcpu = &cpus[0];
    if ((conf = mpconfig(&mp)) == 0)
        return;
    ismp = 1;
    lapicaddr = conf->lapicaddr;

    for (p = conf->entries, i = 0; i < conf->entry; i++) {
        switch (*p) {
		    case MPPROC:
        proc = (struct mpproc *)p;
			  if (proc->flags & MPPROC_BOOT)
				    bootcpu = &cpus[ncpu];
			  if (ncpu < NCPU) {
				    cpus[ncpu].cpu_id = ncpu;
				    ncpu++;
			  } else {
            cprintf("SMP: too many CPUs, CPU %d disabled\n",
            proc->apicid);
			  }
        p += sizeof(struct mpproc);
        continue;
        case MPBUS:
        case MPIOAPIC:
        case MPIOINTR:
        case MPLINTR:
        p += 8;
        continue;
        default:
            cprintf("mpinit: unknown config type %x\n", *p);
        ismp = 0;
        i = conf->entry;
	      }
	 }
```
在启动过程中，`mp_init`和`lapic_init`是和硬件以及体系架构紧密相关的，通过读取某个特殊内存地址（当然前提是能读取的到，所以在`mem_init`中需要修改进行相应映射），来获取CPU的信息，根据这些信息初始化CPU结构。
在`boot_aps`函数中首先找到一段用于启动的汇编代码，该代码和lab3是嵌入在内核代码段之上的一部分，其中`mpentry_start`和`mpentry_end`是编译器导出符号，代表这段代码在内存（虚拟地址）中的起止位置，接着把代码复制到`MPENTRY_PADDR`处。随后调用`lapic_startap`来命令特定的AP去执行这段代码。
``` c
// Start the non-boot (AP) processors.
static void
boot_aps(void)
{
    extern unsigned char mpentry_start[], mpentry_end[];
  	void *code;
  	struct CpuInfo *c;

  	// Write entry code to unused memory at MPENTRY_PADDR
  	code = KADDR(MPENTRY_PADDR);
  	memmove(code, mpentry_start, mpentry_end - mpentry_start);

  	// Boot each AP one at a time
  	for (c = cpus; c < cpus + ncpu; c++) {
  		if (c == cpus + cpunum())  // We've started already.
  			continue;

  		// Tell mpentry.S what stack to use
  		mpentry_kstack = percpu_kstacks[c - cpus] + KSTKSIZE;
  		// Start the CPU at mpentry_start
  		lapic_startap(c->cpu_id, PADDR(code));
  		// Wait for the CPU to finish some basic setup in mp_main()
  		while(c->cpu_status != CPU_STARTED)
  			;
  	}
}
```

#### 练习2
问
> 修改`page_init`函数(`kern/page.c`)的实现，来避免将`MPENTRY_PADDR`处的物理页加入到空闲链表中,使得能安全地拷贝和运行AP的启动代码。

解：
> 在`boot_aps`函数中将启动代码放到了`MPENTRY_PADDR`处，而代码的来源则是在`kern/mpentry.S`中，功能与`boot.S`中的非常类似，主要就是开启分页模式，转到内核栈上去，实际上这时内核栈还没建好。在执行完`mpentry.S`中的代码之后，将会跳转到`mp_main`函数中去。而这里需要提前做的，就是将`MPENTRY_PADDR`处的物理页表标识为已用，这样不会将这一页放在空闲链表中分配出去。只需要在`page_init`中添加一个判断就可以。
``` c
//kern/pmap.c
void page_init(void)
{
    ......
    for (i = 0; i < npages; i++) {
        ......
        else if (i == MPENTRY_PADDR / PGSIEZ)
            continue;
        ......
    }
}
```

### 问题1
问
> 仔细比较`kern/mpentry.S`与`boot/boot.S`，`kern/mpentry.S`是被编译链接来运行在`KERNBASE`之上的，那么定义`MPBOOTPHYS宏``的目的是什么？为什么在kern/mpentry.S中是必要的，在boot/boot.S中不是呢？换句话说，如果在kern/mpentry.S中忽略它，会出现什么错误？

解
>
```
#define MPBOOTPHYS(s) ((s) - mpentry_start + MPENTRY_PADDR))))
// MPBOOTPHYS is to calculate symobl address relative to MPENTRY_PADDR. The ASM is executed in the load address above KERNBASE, but JOS need to run mp_main at 0x7000 address! Of course 0x7000’s page is reserved at pmap.c.
```
> 在AP的保护模式打开之前，是没有办法寻址到3G以上的空间的，因此用`MPBOOTPHYS`是用来计算相应的物理地址的。但是在`boot.S`中，由于尚没有启用分页机制，所以能够指定程序开始执行的地方以及程序加载的地址；但是，在`mpentry.S`的时候，由于主CPU已经处于保护模式下了，因此是不能直接指定物理地址的，而给定线性地址映射到相应的物理地址是允许的。

### CPU的状态和初始化
在编写多处理器操作系统时，区分每个处理器私有的每个CPU状态以及整个系统共享的全局状态是非常重要的。 `kern / cpu.h`定义了大多数CPU状态，包括`struct CpuInfo`存储CPU变量。 `cpunum()`返回调用它的CPU的ID，可以用作数组的索引`cpus`。
每个CPU独有的变量应该有：
* 内核栈
因为多个CPU可以同时陷入内核，所以我们需要为每个处理器分配一个内核栈，以防止它们干扰对方的执行。数组`percpu_kstacks[NCPU][KSTKSIZE]`为`NCPU`的内核栈提供了空间。
* TSS和TSS描述符
每个CPU任务状态段（TSS）也是需要的，以便指定每个CPU内核栈的位置。存储CPU i的`TSS cpus[i].cpu_ts`，并在GDT条目中定义相应的TSS描述符`gdt[(GD_TSS0 >> 3) + i]`。`ts`在`kern / trap.c`中定义的全局变量将不再有用。
* CPU的当前环境指针
由于每个CPU可以同时运行不同的用户进程中，因此重新定义了符号`curenv`来指代`cpus[cpunum()].cpu_env`（或`thiscpu->cpu_env`），它指向环境当前的上执行当前CPU（在其上运行代码的CPU）。
* CPU的系统寄存器
所有寄存器（包括系统寄存器）对于CPU都是专用的。因此，初始化这些寄存器的指令，例如`lcr3()， ltr()，lgdt()，lidt()`等，必须进行一次各CPU上执行。功能`env_init_percpu()`和`trap_init_percpu()`执行此功能。

#### 练习3
问
> 修改`mem_init_mp`函数来映射开始于`KSTACKTOP`的CPU栈。

解
> 多CPU的内存分布如下：
```
KERNBASE ----> +-------------------------+ 0xf0000000
KSTACKTOP      |   CPU0's Kernel Stack   | RW/-- KSTKSIZE
               |-------------------------|
               |   Invalid Memory(*)     | --/-- KSTKGAP
               +-------------------------+
               |    CPU1's Kernel STACK  | RW/-- KSTKSIZE
               |-------------------------|
               |   Invalid Memory(*)     | --/-- KSTKGAP
               +-------------------------+
               .             .           .
               .             .           .

```
需要为每个核都分配一个内核栈，每个内核栈的大小是`KSTKSIZE`，而内核栈之间的间距是`KSTKGAP`，起到保护作用。
``` c
//kern/pmap.c
static void
mem_init_mp(void)
{
    int i;
    uintptr_t kstacktop_i;

    for (i = 0; i < NCPU; i++) {
        kstacktop_i = KSTACKTOP - i * (KSTKGAP + KSTKSIZE);
        boot_map_region(kern_pgdir,
                        kstacktop_i - KSTKSIZE,
                        ROUNDUP(KSTKSIZE, PGSIZE),
                        PADDR(&percpu_kstacks[i]),
                        PTE_W | PTE_P);
        }
}
```

#### 练习4
问：
> 在`trap_init_percpu`函数中为BSP初始化TSS和TSS描述符，这在Lab3中是可行的，但是在这里会有问题，修改代码使它可以运行在所有CPU上。

解：
> 由于有多个CPU，所以在这里不能使用原先的全局变量`ts`，应该利用`thiscpu`指向的`CpuInfo`结构体和`cpunum`函数来为每个核的`TSS`进行初始化。
``` c
void
trap_init_percpu(void)
{
    thiscpu->cpu_ts.ts_esp0 = KSTACKTOP - cpunum() * (KSTKGAP + KSTKSIZE);
    thiscpu->cpu_ts.ts_ss0 = GD_KD;

    // Initialize the TSS slot of the gdt.
    gdt[(GD_TSS0 >> 3) + cpunum()] = SEG16(STS_T32A, (uint32_t) (&thiscpu->cpu_ts), sizeof(struct Taskstate) - 1, 0);
    gdt[(GD_TSS0 >> 3) + cpunum()].sd_s = 0;

    // Load the TSS selector (like other segment selectors, the
    // bottom three bits are special; we leave them 0)
    ltr(GD_TSS0 + sizeof(struct Segdesc) * cpunum());

    // Load the IDT
    lidt(&idt_pd);
}
```

### 锁
在`mp_main`函数中初始化AP后，代码就会进入自旋。在让AP进行更多操作之前，首先要解决多CPU同时运行在内核时产生的竞争问题。最简单的办法是实现1个大内核锁（big kernel lock)，1次只让一个进程进入内核模式，当CPU离开内核时释放锁。
在`kern/spinlock.h`中声明了大内核锁，提供了`lock_kernel`和`unlock_kernel`函数来快捷地获得和释放锁。总共有四处用到大内核锁：
* 在启动的时候，BSP启动其余的CPU之前，BSP需要取得内核锁
* `mp_main`中，也就是CPU被启动之后执行的第一个函数，这里应该是调用调度函数，选择一个进程来执行的，但是在执行调度函数之前，必须获取锁
* `trap函数``也要修改，因为可以访问临界区的CPU只能有一个，所以从用户态陷入到内核态的话，要加锁，因为可能多个CPU同时陷入内核态
* `env_run`函数，也就是启动进程的函数，之前在lab3中实现的，在这个函数执行结束之后，就将跳回到用户态，此时离开内核，也就是需要将内核锁释放

加锁后，将原有的并行执行过程在关键位置变为串行执行过程，整个启动过程大概如下：
> i386_init–>BSP获得锁–>boot_ap–>(BSP建立为每个cpu建立idle任务、建立用户任务，mp_main)—>BSP的sched_yield–>其中的env_run释放锁–>AP1获得锁–>执行sched_yield–>释放锁–>AP2获得锁–>执行sched_yield–>释放锁…..

#### 练习5
问
> 在上述位置应用大内核锁。

解
``` c
//i386_init
lock_kernel();
boot_aps();

//mp_main
lock_kernel();
sched_yield();

//trap
if ((tf->tf_cs & 3) == 3) {
    lock_kernel();
    assert(curenv);
    ......
}

//env_run
lcr3(PADDR(curenv->env_pgdir));
unlock_kernel();
env_pop_tf(&(curenv->env_tf));
```

### 问题2
问：
> 既然大内核锁保证了只有1个CPU能运行在内核，为什么我们还要为每个CPU准备1个内核栈。

解:
> 因为不同的内核栈上可能保存有不同的信息，当1个CPU从内核退出来之后，有可能在内核栈中留下了一些将来还有用的数据，所以一定要有单独的栈。

### 挑战1
问:
> 大内核锁简单便于应用，但是它取消了内核模式的并行。大多数现代操作系统使用不同的锁来保护共享状态的不同部分，这称之为细粒度锁。细粒度锁能有效地提高性能，但是也更困难地去实现和检测错误。所以你可以去掉大内核锁，在JOS中实现内核并发。

解
> 思路:实验指导中提供了一些JOS内核中的共享结构，具体实现就是在保证在使用这些结构体时保证互斥。
　　
### 循环调度(Round-Robin Scheduling)
接下来的任务是改变JOS内核，实现`round-robin`调度算法。
主要是在`sched_yield`函数内实现，从该CPU上一次运行的进程开始，在进程描述符表中寻找下一个可以运行的进程，如果没找到而且上一个进程依然是可以运行的，那么就可以继续运行上一个进程，同时将这个算法实现为了一个系统调用，使得进程可以主动放弃CPU。

#### 练习6
问:
> 实现`sched_yield`函数，并添加系统调用。

解:
> 修改代码如下.然后修改`kern/syscall.c`,添加相关的系统调用机制。
```
void
sched_yield(void)
{
        uint32_t i, j, start;
        struct Env *runenv;

        idle = thiscpu->cpu_env;
        start = (idle != NULL) ? ENVX(idle->env_id) : 0;
        runenv = NULL;

        for (i = 0; i < NENV; i++) {
                j = (start + i) % NENV;
                if (envs[j].env_status == ENV_RUNNABLE) {
                        if (runenv == NULL || envs[j].env_priority < runenv->env_priority)
                                runenv = &envs[j];
                }
        }
        if ((idle && idle->env_status == ENV_RUNNING) && (runenv == NULL || idle->env_priority < runenv->env_priority)){
                env_run(idle);
                return;
        }
        if (runenv) {
                env_run(runenv);
                return;
        }
        // sched_halt never returns
        sched_halt();
}
```
　　
#### 问题3
问:
> 在`lcr3()`运行之后，这个CPU对应的页表就立刻被换掉了，但是这个时候的参数e，也就是现在的`curenv`，为什么还是能正确的解引用？

解:
> 因为当前是运行在系统内核中的，而每个进程的页表中都是存在内核映射的。每个进程页表中虚拟地址高于`UTOP`之上的地方，只有`UVPT`不一样，其余的都是一样的，只不过在用户态下是看不到的。所以虽然这个时候的页表换成了下一个要运行的进程的页表，但是`curenv`的地址没变，映射也没变，还是依然有效的。

#### 问题4
问:
> 在用户环境进行切换时，为什么旧进程的寄存器一定要被保存以便之后重新装载？在哪里发生这样的操作？

解:
> 因为如果不进行保存，旧进程运行时的状态就丢失了，运行就不正确了。每次进入到内核态的时候，当前的运行状态都是在一进入的时候就保存了的。如果没有发生调度，那么之前`trapframe`中的信息还是会恢复回去，如果发生了调度，恢复的就是被调度运行的进程的上下文了。

#### 挑战2
问:
> 添加固定优先级的调度策略，确保高优先级的进程总是先于低优先级的进程。

解：
> 思路:为每个进程的结构体添加1个标志优先级的变量，在`sched_yield`时遍历链表选取优先级最高的可运行进程作为下一个运行进程。
> 1. 在`inc/env.h`中给`Env`结构添加1个成员`int env_priority`
> 2. 在`kern/env.c`中的`env_alloc`中给进程赋值为默认优先级`ENV_PRIOR_DEFAULT`，一共设立了四个基本的优先级，数字越小优先级越高
``` c
//inc/env.h
#define ENV_PRIOR_SUPER 0
#define ENV_PRIOR_HIGH 10
#define ENV_PRIOR_NORMAL 100
#define ENV_PRIOR_LOW 1000
```
> 3. 修改`kern/sched.c`中`sched_yield`函数的调度策略
``` c
//kern/sched.c  sched_yield()
        idle = thiscpu->cpu_env;
        start = (idle != NULL) ? ENVX(idle->env_id) : 0;
        runenv = NULL;

        for (i = 0; i < NENV; i++) {
                j = (start + i) % NENV;
                if (envs[j].env_status == ENV_RUNNABLE) {
                        if (runenv == NULL || envs[j].env_priority < runenv->env_priority)
                                runenv = &envs[j];
                }
        }
        if ((idle && idle->env_status == ENV_RUNNING) && (runenv == NULL || idle->env_priority < runenv->env_priority)){
                env_run(idle);
                return;
        }
        if (runenv) {
                env_run(runenv);
                return;
        }
```
> 4. 添加1个系统调用`sys_env_set_priority(envid, priority)`，允许进程的父进程或者自己修改自己的优先级。这里的修改与前面添加系统调用类似。
``` c
static int
sys_env_set_priority(envid_t envid, int priority)
{
        int ret;
        struct Env *env;

        if ((ret = envid2env(envid, &env, 1)) < 0)
                return ret;
        env->env_priority = priority;
        return 0;
}
```

#### 挑战3
问:
> 现在的JOS内核不支持应用来使用x86处理器的浮点数单元(FPU)，MMX指令和SSE扩展指令。扩展Env结构提供保存处理器浮点数状态的空间，扩展进程上下文交换代码来保存和恢复浮点数状态 。

解:
> 给中断加入保存浮点寄存器的功能。
> 1. 给`inc/trap.h`文件中的`Trapframe`结构新增`char tf_fpus[512]``成员，并增加`uint32_t tf_padding0[3]``来对齐
> 2. 修改`kern/trapentry.S`文件中的`_alltraps`函数，加入保存`fpu`寄存器的功能
```c
_alltraps:
        pushl %ds
        pushl %es
        pushal

        // save FPU
        subl $524, %esp
        fxsave (%esp)

        movl $GD_KD, %eax
        movl %eax, %ds
        movl %eax, %es

        push %esp
        call trap
```
> 3. 修改`kern/env.c`文件中的`env_pop_tf`函数，加入恢复`fpu`功能
``` c
 __asm __volatile("movl %0,%%esp\n"
                "\tfxrstor (%%esp)\n"
                "\taddl $524,%%esp\n"
                "\tpopal\n"
                "\tpopl %%es\n"
                "\tpopl %%ds\n"
                "\taddl $0x8,%%esp\n" /* skip tf_trapno and tf_errcode */
                "\tiret"
                : : "g" (tf) : "memory");
```

### 创建环境的系统调用
虽然当前内核现在有能力运行和切换多用户级进程，但是它仍然只能跑内核初始创建的进程。现在将实现必要的JOS系统调用来运行用户进程来创建和启动其它新的用户进程。
Unix提供了`fork`系统调用来创建进程，它拷贝父进程的整个地址空间到新创建的子进程。两个进程之间唯一的区别是它们的进程ID，在父进程fork返回的是子进程ID，而在子进程fork返回的是0。
参照Unix,实现1个不同的更原始的JOS系统调用来创建进程。利用这些系统调用能实现类似Unix的fork函数。
用户级fork函数在`user/dumbfork.c`中的`dumbfork()`中，该函数将父进程中所有页的内容全部复制过来，唯一不同的地方就是返回值不同。

### 练习7
问:
> 实现`ker/syscall.c`中的系统调用。

解:
> 首先是`sys_exofork`函数，这个系统调用将创建1个新的空白进程，没有映射的用户空间且无法运行。在调用函数时新进程的寄存器状态与父进程相同，但是在父进程会返回子进程的ID，而子进程会返回0。通过设置子进程的eax为0，来让系统调用的返回值为0。
``` c
static envid_t
sys_exofork(void)
{
        int ret;
        struct Env *env;

        if ((ret = env_alloc(&env, sys_getenvid())) < 0)
                return ret;
        env->env_status = ENV_NOT_RUNNABLE;
        env->env_tf = curenv->env_tf;
        env->env_tf.tf_regs.reg_eax = 0;
        return env->env_id;
}
```
接着是`sys_env_set_status`函数，设置进程的状态为`ENV_RUNNABLE`或者`ENV_NOT_RUNNABLE`。
``` c
static int
sys_env_set_status(envid_t envid, int status)
{
implemented");
        int ret;
        struct Env *env;

        if (status != ENV_RUNNABLE && status != ENV_RUNNING)
                return -E_INVAL;
        if ((ret = envid2env(envid, &env, 1)) < 0)
                return ret;
        env->env_status = status;
        return 0;
}
```
然后是`env_page_alloc`函数，分配1个物理页并映射到给定进程的进程空间的虚拟地址。
``` c
static int
sys_page_alloc(envid_t envid, void *va, int perm)
{
        struct Env *env;
        struct PageInfo *pp;

        if (envid2env(envid, &env, 1) < 0)
                return -E_BAD_ENV;
        if ((uintptr_t)va >= UTOP || PGOFF(va))
                return -E_INVAL;
        if ((perm & PTE_U) == 0 || (perm & PTE_P) == 0)
                return -E_INVAL;
        if ((perm & ~(PTE_U | PTE_P | PTE_W | PTE_AVAIL)) != 0)
                return -E_INVAL;
        if ((pp = page_alloc(ALLOC_ZERO)) == NULL)
                return -E_NO_MEM;
        if (page_insert(env->env_pgdir, pp, va, perm) < 0) {
                page_free(pp);
                return -E_NO_MEM;
        }
        return 0;
}
```
最后是s`ys_page_unmap`函数，解除指定进程中的1个页映射。
```
static int
sys_page_unmap(envid_t envid, void *va)
{
        struct Env *env;

        if (envid2env(envid, &env, 1) < 0)
                return -E_BAD_ENV;
        if ((uintptr_t)va >= UTOP || PGOFF(va))
                return -E_INVAL;

        page_remove(env->env_pgdir, va);
        return 0;
}
```
至此所有系统调用都完成了，可以运行一下`usr/umbfork.c`。首先看一下主函数，逻辑是父进程创建1个子进程，然后每次打印1条信息后交出控制权，并且让父进程重复10次而子进程重复20次。
``` c
void
umain(int argc, char **argv)
{
        envid_t who;
        int i;

        // fork a child process
        who = dumbfork();

        // print a message and yield to the other a few times
        for (i = 0; i < (who ? 10 : 20); i++) {
                cprintf("%d: I am the %s!\n", i, who ? "parent" : "child");
                sys_yield();
        }
}
```
