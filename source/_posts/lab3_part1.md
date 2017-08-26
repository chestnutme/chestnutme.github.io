title: lab3-user-environment-part1
date: 2017/04/26
tags:
- xv6
- os

---

# Lab3 用户环境 Part1
## 摘要
os运行分为两个基本状态: 内核态(kernel) 和 用户态(user).对应用进程而言, 运行在用户态下, 通过系统调用接口使用kernel提供的服务.
Lab3主要实现的功能是在用户环境下的进程的正常运行.包括三个方面:
> 1. 实现获得受保护的用户模式环境（即“进程”）运行所需的基本内核功能.
> 2. 创建数据结构以增强JOS内核,跟踪用户环境,创建单个用户环境,将程序映像加载到其中并运行.
> 3. 使JOS内核能够处理用户环境所产生的任何系统调用,并处理其导致的任何其他异常.

Lab3分为两个部分:
> part1. 用户环境和异常处理
> part2. 缺页中断, 断点异常和系统调用

<!-- more -->

## 准备
下载代码
``` bash
git pull
git checkout -b lab3 origin/Lab3
git merge lab2
```
lab3新增的实验相关文件:
``` c
inc/	env.h	      用户模式环境的公共定义
      trap.h	    陷阱处理的公共定义
      syscall.h	  从用户环境到内核的系统调用的公共定义
      lib.h	      用户模式支持库函数的公共定义
kern/	env.h	      用户模式环境的内核私有定义
      env.c	      实现用户模式环境的内核代码
      trap.h	    陷阱处理的内核私有定义
      trap.c	    陷阱处理代码
      trapentry.S	汇编陷阱处理程序入口
      syscall.h	  用于系统调用处理的内核私有定义
      syscall.c	  系统调用实现代码
lib/	Makefrag	  构建用户模式的库文件的makefile,obj/lib/libJOS.a
      entry.S	A   用户环境的汇编入口
      libmain.c	  从entry.S调用的用户模式库文件的启动代码
      syscall.c	  用户模式下系统调用的存根函数
      console.c	  putchar和getchar在用户模式下实现 ,提供控制台I/O
      exit.c	    用户模式下实现退出的代码
user/	*	          各种测试程序
```

## Part1 User Environment
新包含的文件`inc/env.h`里面包含了JOS内核的有关用户环境(User Environment)的基本定义.用户环境指的就是一个应用程序运行在系统中所需要的一个上下文环境,操作系统内核使用数据结构`Env`来记录每一个用户环境的信息.Lab3只会建一个用户环境,但是之后会把它拓展成能够支持多用户环境,即多个用户程序并发执行.

在`kern/env.c`文件中,操作系统一共维护了三个重要的和用户环境相关的全局变量：
``` c
struct Env *envs = NULL;    //所有的 Env 结构体l链表
struct Env *curenv = NULL;   //目前正在运行的用户环境
static struct Env *env_free_list;  //还没有被使用的 Env 结构体链表
```
一旦JOS启动,`envs`指针便指向了一个`Env`结构体链表,表示系统中所有的用户环境的`env`.JOS内核将支持同一时刻最多`NENV`个活跃的用户环境,系统会为每一个活跃的用户环境在`envs`链表中维护一个`Env`结构体.JOS内核用`env_free_list`链接起来把所有不活跃的`Env`结构体,方便进行用户环境`env`的分配和回收.另外,内核也会把`curenv`指针指向在任意时刻正在执行的用户环境的`Env`结构体.在内核启动时,并且还没有任何用户环境运行时,`curenv`的值为`NULL`.

### 环境状态 Environment Status
`Env`结构体定义在 inc/env.h`:
``` c
struct Env {
　　　　struct Trapframe env_tf;      //saved registers
　　　　struct Env * env_link;         //next free Env
　　　　envid_t env_id;　　            //Unique environment identifier
　　　　envid_t env_parent_id;        //envid of this env's parent
　　　　enum EnvType env_type;　　//Indicates special system environment
　　　　unsigned env_status;　　   //Status of the environment
　　　　uint32_t env_runs;         //Number of the times environment has run
　　　　pde_t \*env_pgdir;　　　　//Kernel virtual address of page dir.
};　　
```
具体含义如下:
`env_tf`:
　　定义在`inc/trap.h`文件中,里面存放着当用户环境暂停运行时,所有重要寄存器的值.内核也会在系统从用户态切换到内核态时保存这些值,这样的话用户环境可以在之后被恢复,继续执行.
`env_link`:
　　该指针指向在`env_free_list`中第一个空闲的`Env`结构体.前提是这个结构体还没有被分配给任意一个用户环境时.
`env_id`:
　　唯一的确定使用这个结构体的用户环境.当这个用户环境终止,内核会把这个结构体分配给另外一个不同的环境,这个新的环境会有不同的`env_id`值.
`env_parent_id`:
　　创建这个用户环境的父用户环境的`env_id`
`env_type`:
　　用于区别某个特定的用户环境.对于大多数环境来说,它的值都是 `ENV_TYPE_USER`.
`env_status`:
　　表示环境的状态,存放以下可能的值:
　　`ENV_FREE`: 结构体是空闲的,应该在链表`env_free_list`中.
　　`ENV_RUNNABLE`: 结构体对应的用户环境已经就绪,等待被分配处理机.
　　`ENV_RUNNING`: 结构体对应的用户环境正在运行.
　　`ENV_NOT_RUNNABLE`: 结构体所代表的是一个活跃的用户环境,但是它不能被调度运行,因为它在等待其他环境传递给它的消息.
　　`ENV_DYING`: 代表这个结构体对应的是一个僵尸环境.一个僵尸环境在下一次陷入内核时会被释放回收.
`env_pgdir`:
　　存放这个环境的页目录的虚拟地址

正如`Unix`中的进程一样,一个JOS环境中结合了“线程”和“地址空间”的概念.线程通常是由被保存的寄存器的值来定义的,而地址空间则是由`env_pgdir`所指向的页目录表还有页表来定义的.为了运行一个用户环境,内核必须设置合适的寄存器的值以及合适的地址空间.

### 分配环境数组
lab2中`mem_init()`函数中分配了`pages`数组的地址空间,用于记录内核中所有的页的信息.lab3需要进一步去修改`mem_init()`函数,分配`Env`结构体数组`envs`.

#### 练习1:
问:
> 修改`mem_init()`的代码,让它能够分配`envs`数组.这个数组是由`NENV`个`Env`结构体组成的.`envs`数组所在的内存空间在用户模式下是只读的,被映射到虚拟地址`UENVS`处.

解:
> 如同Lab2里面分配pages数组,分配一个`Env`数组给指针`envs`即可
``` c
//kern/pmap.c
//分配内存空间给envs
envs = (struct Env*)boot_alloc(NENV*sizeof(struct Env));
memset(envs, 0, NENV * sizeof(struct Env));
//页表中设置envs的映射关系
boot_map_region(kern_pgdir, UENVS, PTSIZE, PADDR(envs), PTE_U);
```

### 创建运行环境
编写 `kern/env.c`文件来运行一个用户环境了.由于现在没有文件系统,所以必须把内核设置成能够加载内核中的静态二进制程序映像文件.在`i386_init()`函数中,编码完成运行用户环境的功能.

#### 练习2:
问:
> 在文件`env.c`中,完成下列函数：
`env_init()`:
初始化所有的在`envs`数组中的`Env`结构体,并把它们加入到 `env_free_list`中. 还要调用`env_init_percpu`,这个函数要配置段式内存管理系统,让它所管理的段具有两种访问优先级的一种,一个是内核运行时的0优先级,以及用户运行时的3优先级.
`env_setup_vm()`:
为一个新的用户环境分配一个页目录表,并且初始化这个用户环境的地址空间中的和内核相关的部分.
`region_alloc()`:
为用户环境分配物理地址空间
`load_icode()`:
分析一个`ELF`文件,把它的内容加载到用户环境下.
`env_create()`:
利用`env_alloc`函数和`load_icode`函数,加载一个`ELF`文件到用户环境中
`env_run()`:
在用户模式下,开始运行一个用户环境.

解:
> `env_init`函数,通过遍历`envs`数组中的所有`Env`结构体,把每一个结构体的 `env_id`字段置0,因为要求所有的`Env`在`env_free_list`中的顺序,要和它在 `envs`中的顺序一致,采用头插法.　　
``` c
 void
env_init(void)
{
    // Set up envs array
    int i;
    env_free_list = NULL;
    for (i=NENV-1; i>=0; i--){
        envs[i].env_id = 0;
        envs[i].env_status = ENV_FREE;
        envs[i].env_link = env_free_list;
        env_free_list = &envs[i];
     }
     // Per-CPU part of the initialization
     env_init_percpu();
}
```
> `env_setup_vm()`初始化新的用户环境的页目录表,只设置页目录表中和操作系统内核跟内核相关的页目录项,用户环境的页目录项不要设置,因为所有用户环境的页目录表中和操作系统相关的页目录项都是一样的（除了虚拟地址`UVPT`,单独进行设置）,所以可以参照`kern_pgdir`中的内容来设置`env_pgdir`中的内容.
``` c
static int
env_setup_vm(struct Env *e)
{
    int i;
    struct PageInfo* p = NULL;

    // Allocate a page for the page directory
    if (!(p = page_alloc(ALLOC_ZERO)))
        return -E_NO_MEM;

     e->env_pgdir = (pde_t *)page2kva(p);
     p->pp_ref++;

     //Map the directory below UTOP.
     for (i = 0; i < PDX(UTOP); i++)
         e->env_pgdir[i] = 0;        

     //Map the directory above UTOP
     for (i = PDX(UTOP); i < NPDENTRIES; i++) {
         e->env_pgdir[i] = kern_pgdir[i];
     }

     // UVPT maps the env's own page table read-only.
     // Permissions: kernel R, user R
     e->env_pgdir[PDX(UVPT)] = PADDR(e->env_pgdir) | PTE_P | PTE_U;

     return 0;
 }
```
> `region_alloc()`为用户环境分配物理空间,这里注意我们要先把起始地址和终止地址进行页对齐,对其之后我们就可以以页为单位,为其一个页一个页的分配内存,并且修改页目录表和页表.
``` c
static void
region_alloc(struct Env *e, void *va, size_t len)
{
    void* start = (void *)ROUNDDOWN((uint32_t)va, PGSIZE);
    void* end = (void *)ROUNDUP((uint32_t)va+len, PGSIZE);
    struct PageInfo *p = NULL;
    void* i;
    int r;

    for(i = start; i < end; i += PGSIZE){
        p = page_alloc(0);
        if(p == NULL)
           panic(" region alloc failed: allocation failed.\n");

        r = page_insert(e->env_pgdir, p, i, PTE_W | PTE_U);
        if(r != 0)
            panic("region alloc failed.\n");
    }
}
```
>　`load_icode()`为每一个用户进程设置它的初始代码区,堆栈以及CPU标识.用户程序是`ELF`文件,解析`ELF`文件.
``` c
static void
load_icode(struct Env *e, uint8_t *binary)
{

    struct Elf* header = (struct Elf*)binary;

    if (header->e_magic != ELF_MAGIC)
        panic("load_icode failed: The binary we load is not elf.\n");

    if (header->e_entry == 0)
        panic("load_icode failed: The elf file can't be excuterd.\n");

   e->env_tf.tf_eip = header->e_entry;

   lcr3(PADDR(e->env_pgdir));   //load user pgdir

   struct Proghdr *ph, *eph;
   ph = (struct Proghdr* )((uint8_t *)header + header->e_phoff);
   eph = ph + header->e_phnum;
    for(; ph < eph; ph++) {
        if(ph->p_type == ELF_PROG_LOAD) {
            if(ph->p_memsz - ph->p_filesz < 0)
                panic("load icode failed : p_memsz < p_filesz.\n");

           region_alloc(e, (void *)ph->p_va, ph->p_memsz);
            memmove((void *)ph->p_va, binary + ph->p_offset, ph->p_filesz);
            memset((void *)(ph->p_va + ph->p_filesz), 0, ph->p_memsz - ph->p_filesz);
        }
     }
}
```
> `env_create`是利用`env_alloc`函数和`load_icode`函数,加载一个`ELF`文件到用户环境中
``` c
void
env_create(uint8_t *binary, enum EnvType type)
{
    struct Env *e;
    int rc;
    if ((rc = env_alloc(&e, 0)) != 0)
          panic("env_create failed: env_alloc failed.\n");

     load_icode(e, binary);
     e->env_type = type;
 }
```
> `env_run`开始运行一个用户环境
``` c
void
env_run(struct Env *e)
{

    if(curenv != NULL && curenv->env_status == ENV_RUNNING)
        curenv->env_status = ENV_RUNNABLE;

    curenv = e;
    curenv->env_status = ENV_RUNNING;
    curenv->env_runs++;
    lcr3(PADDR(curenv->env_pgdir));

    env_pop_tf(&curenv->env_tf);

    panic("env_run not yet implemented");
}
```

用户环境的代码被调用前,操作系统一共按顺序执行了以下几个函数：
``` c
* start (kern/entry.S)
* i386_init (kern/init.c)
cons_init
mem_init
env_init
trap_init （目前还未实现）
env_create
env_run
env_pop_tf
```
完成上述子函数的代码后,并且在`QEMU`下编译运行,系统会进入用户空间,并且开始执行`hello`程序,直到它做出一个系统调用指令`int`.但是这个系统调用指令不能成功运行,因为到目前为止,JOS还没有设置相关硬件来实现从用户态向内核态的转换功能.当CPU发现,它没有被设置成能够处理这种系统调用中断时,它会触发一个保护异常,然后发现这个保护异常也无法处理,从而又产生一个错误异常,然后又发现仍旧无法解决问题,所以最后放弃,这种异常被称为”triple fault”.通常来说,接下来CPU会复位,系统会重启.


### 处理中断和异常
到目前为止,当程序运行到第一个系统调用`int $0x30`时,就会进入错误的状态,因为现在系统无法从用户态切换到内核态.需要实现一个基本的异常处理机制,使得内核可以从用户态转换为内核态.学习X86的异常中断机制.

### 受保护的控制转移
异常(Exception)和中断(Interrupts)都是“受到保护的控制转移方法”,都会使CPU从用户态转移为内核态.在Intel的术语中,一个中断指的是由外部异步事件引起的CPU控制权转移,比如外部IO设备发送来的中断信号.一个异常则是由于当前正在运行的指令所带来的同步的CPU控制权的转移,比如除零异常.

为了能够确保这些控制的转移能够真正被保护起来,CPU的中断/异常机制通常被设计为：用户态的代码无权选择内核中的代码从哪里开始执行.CPU可以确保只有在某些条件下,才能进入内核态.在X86上,有两种机制配合工作来提供这种保护：
> 1. **中断向量表 The Interrupt Descriptor Table**：
CPU保证中断和异常只能够引起内核进入到一些特定的,被事先定义好的程序入口,而不是由触发中断的程序来决定中断程序入口.
X86有256个不同的中断和异常,对应特定的中断向量.一个向量指的就是0到255中的一个数.一个中断向量的值是根据中断源来决定的：不同设备/错误/对内核的请求,会产生出不同的中断和中断向量的组合.CPU将使用这个向量作为这个中断在中断向量表中的索引,这个表是由内核设置的,放在内核空间中.通过`IDT`中的任意一个表项,CPU得到以下信息:
>> * 加载到`EIP`寄存器中的值,这个值指向了处理这个中断的中断处理程序的位置.
>> * 加载到`CS`寄存器中的值,里面还包含了这个中断处理程序的运行特权级.

> 2. **任务状态段 The Task State Segment**
CPU还需要一个地方来存放异常/中断发生时CPU的状态,比如`EIP`和`CS`寄存器的值.这样的话,中断处理程序能够顺利重新返回到原来的程序中.这段内存自然也要保护起来,不能被用户态的程序所篡改.
正因为如此,当一个x86CPU要处理一个中断,异常并且使运行特权级从用户态转为内核态时,它也会把它的堆栈切换到内核空间中.一个叫做 “任务状态段（TSS）”的数据结构将会详细记录这个堆栈所在的段的段描述符和地址.CPU把`SS,ESP,EFLAGS,CS,EIP`以及一个可选错误码等等这些值压入到这个堆栈上.然后加载中断处理程序的`CS,EIP`值,并且设置`ESP,SS`寄存器指向新的堆栈.由于JOS中的内核态指的就是特权级0,所以CPU用`TSS`中的`ESP0,SS0`字段来指明这个内核堆栈的位置,大小.

### 中断/异常的类型
所有的由X86CPU内部产生的异常的向量值是0到31之间的整数.比如,页表错误所对应的向量值是14.而大于31号的中断向量对应的是软件中断,由int指令生成；或者是外部中断,由外部设备生成.

part1扩展JOS的功能,使它能够处理0~31号内部异常.part2让JOS能够处理48号软件中断,主要被用来做系统调用.在Lab4中会继续扩展JOS使它能够处理外部硬件中断,比如时钟中断.

实例:
> 假设CPU正在用户状态下运行代码,但是遇到了一个除法指令,并且除数为0.
```
         +--------------------+ KSTACKTOP             
         | 0x00000 | old SS   |     " - 4
         |      old ESP       |     " - 8
         |     old EFLAGS     |     " - 12
         | 0x00000 | old CS   |     " - 16
         |      old EIP       |     " - 20 <---- ESP
         +--------------------+
```
> 发生除零异常后,CPU运行如下:
> 1. CPU会首先切换自己的堆栈,切换到由`TSS`的`SS0,ESP0`字段所指定的内核堆栈区,这两个字段分别存放着`GD_KD`和`KSTACKTOP`的值.
> 2. CPU把异常参数压入到内核堆栈中,起始于地址`KSTACKTOP`：
> 3. 因为要处理的是除零异常,它的中断向量是0,CPU会读取`IDT`表中的0号表项,并且把`CS:EIP`的值设置为0号中断处理函数的地址值.
> 4. 中断处理函数开始执行,处理中断.

> 对于某些特定的异常,除了中要保存的五个值之外,还要再压入一个字,叫做错误码(error code).比如页表错误,就是其中一个实例.当压入错误码之后,内核堆栈的状态如下：
```
                     +--------------------+ KSTACKTOP             
                     | 0x00000 | old SS   |     " - 4
                     |      old ESP       |     " - 8
                     |     old EFLAGS     |     " - 12
                     | 0x00000 | old CS   |     " - 16
                     |      old EIP       |     " - 20
                     |     error code     |     " - 24 <---- ESP
                     +--------------------+             
```

### 中断/异常的嵌套
　　CPU在用户态下和内核态下都可以处理异常或中断.只有当CPU从用户态切换到内核态时,硬件才会自动地切换堆栈,并且把一些寄存器中的原来的值压入到堆栈上,并且触发相应的中断处理函数.但如果CPU已经由于正在处理中断而处在内核态下时,此时CPU会向内核堆栈压入更多的值.通过这种方式,内核就可处理嵌套中断.
　　如果CPU已经在内核态下并且遇到嵌套中断,因为它不需要切换堆栈,所以它不需要存储`SS,ESP`寄存器的值.此时内核堆栈如下:
```
                     +--------------------+ <---- old ESP
                     |     old EFLAGS     |     " - 4
                     | 0x00000 | old CS   |     " - 8
                     |      old EIP       |     " - 12
                     +--------------------+
```
但可能会发生一个问题: 如果CPU在内核态下捕捉一个异常,而且由于一些原因,比如堆栈空间不足,不能把当前的状态信息（寄存器的值）压入到内核堆栈中时,那么CPU是无法恢复到原来的状态,它会自动重启.

### 设置IDT
上文确定了基本信息, 可以设置IDT表,并在JOS处理异常了.Lab3只需要处理内部异常（中断向量号0~31）.在头文件`inc/trap.h和kern/trap.h`中包含了和中断异常相关的的定义.
最后要实现的效果如下：
```
IDT                   trapentry.S         trap.c

+----------------+                        
|   &handler1    |---------> handler1:          trap (struct Trapframe *tf)
|                |             // do stuff      {
|                |             call trap          // handle the exception/interrupt
|                |             // ...           }
+----------------+
|   &handler2    |--------> handler2:
|                |            // do stuff
|                |            call trap
|                |            // ...
+----------------+
 .
 .
 .
+----------------+
|   &handlerX    |--------> handlerX:
|                |             // do stuff
|                |             call trap
|                |             // ...
+----------------+
```

每一个中断/异常都有它自己的中断处理函数,定义在`trapentry.S`中.`trap_init()`将初始化`IDT`表.每一个处理函数都构建一个结构体 `Trapframe`在堆栈上,并且调用`trap()`函数指向这个结构体,`trap()`然后处理异常/中断,给它分配一个中断处理函数.
所以操作系统的中断控制流程为：
> 1. `trap_init()`先将所有中断处理函数的起始地址放到中断向量表`IDT`中.
> 2. 当中断发生时,不管是外部中断还是内部中断,CPU捕捉到该中断,进入核心态,根据中断向量去查询中断向量表,找到对应的表项
> 3. 保存被中断的程序的上下文到内核堆栈中,调用这个表项中指明的中断处理函数.
> 4. 执行中断处理函数.
> 5. 执行完成后,恢复被中断的进程的上下文,返回用户态,继续运行这个进程.

#### 练习4
问
> 修改`trapentry.S`和`trap.c`文件,实现上述的功能.在`trapentry.S`文件中为在`inc/trap.h`文件中的每一个`trap`加入一个入口指针, 修改`trap_init()`函数来初始化`IDT`表,使表中每一项指向定义在`trapentry.S`中的入口指针.
`_alltraps`要求:
> 1. 把值压入堆栈,使堆栈结构类似一个结构体`Trapframe`
> 2. 加载`GD_KD`的值到`%ds, %es`寄存器中
> 3. 压栈`%esp`的值,并且传递一个指向`Trapframe`的指针到`trap()``函数中.
> 4. 调用`trap`
解
> 在`trapentry.S`文件定义了两个宏,`TRAPHANDLER`,`TRAPHANDLER_NOEC`.其功能为：声明了一个全局符号name,并且这个符号是函数类型的,代表它是一个中断处理函数名.当系统检测到一个中断/异常时,首先需要完成的一部分操作,包括：中断异常码,中断错误码(error code).因为有些中断有中断错误码,有些没有,所以采用两个宏定义函数.
``` c
#define TRAPHANDLER(name, num)                        \
        .globl name;            /* define global symbol for 'name' */   \
        .type name, @function;  /* symbol type is function */           \
        .align 2;               /* align function definition */         \
        name:                   /* function starts here */              \
        pushl $(num);                                                   \
        jmp _alltraps
/* Use TRAPHANDLER_NOEC for traps where the CPU doesn't push an error code.
 * It pushes a 0 in place of the error code, so the trap frame has the same
 * format in either case.
 */
#define TRAPHANDLER_NOEC(name, num)                                     \
        .globl name;                                                    \
        .type name, @function;                                          \
        .align 2;                                                       \
        name:                                                           \
        pushl $0;                                                       \
        pushl $(num);                                                   \
        jmp _alltraps

.text

/*
 * Lab 3: Your code here for generating entry points for the different traps.
 */
TRAPHANDLER_NOEC(divide_entry, T_DIVIDE);
TRAPHANDLER_NOEC(debug_entry, T_DEBUG);
TRAPHANDLER_NOEC(nmi_entry, T_NMI);
TRAPHANDLER_NOEC(brkpt_entry, T_BRKPT);
TRAPHANDLER_NOEC(oflow_entry, T_OFLOW);
TRAPHANDLER_NOEC(bound_entry, T_BOUND);
TRAPHANDLER_NOEC(illop_entry, T_ILLOP);
TRAPHANDLER_NOEC(device_entry, T_DEVICE);
TRAPHANDLER(dblflt_entry, T_DBLFLT);
TRAPHANDLER(tss_entry, T_TSS);
TRAPHANDLER(segnp_entry, T_SEGNP);
TRAPHANDLER(stack_entry, T_STACK);
TRAPHANDLER(gpflt_entry, T_GPFLT);
TRAPHANDLER(pgflt_entry, T_PGFLT);
TRAPHANDLER_NOEC(fperr_entry, T_FPERR);
TRAPHANDLER(align_entry, T_ALIGN);
TRAPHANDLER_NOEC(mchk_entry, T_MCHK);
TRAPHANDLER_NOEC(simderr_entry, T_SIMDERR);
TRAPHANDLER_NOEC(syscall_entry, T_SYSCALL);
```
> 然后调用`_alltraps`函数,使程序在之后调用`trap.`c中的`trap`函数时,能够正确的访问到输入的参数,即`Trapframe`指针类型的输入参数`tf`.
``` c
_alltraps:
        pushl %ds
        pushl %es
        pushal

        movl $GD_KD, %eax
        movl %eax, %ds
        movl %eax, %es

        push %esp
        call trap
```
> 最后在`trap.c`中实现`trap_init`函数,即在`IDT`表中插入中断向量描述符,可以使用`SETGATE`宏实现,其定义如下：
``` c
//gate:IDT表的index入口
//istrap:判断是异常还是中断
//sel:代码段选择符
//off:对应的处理函数地址
//dpl:触发该异常或中断的用户权限.
#define SETGATE(gate, istrap, sel, off, dpl)
```
> `trap_init`实现：
``` c
void
trap_init(void)
{
        extern struct Segdesc gdt[];

        SETGATE(idt[T_DIVIDE], 0, GD_KT, divide_entry, 0);
        SETGATE(idt[T_DEBUG], 0, GD_KT, debug_entry, 0);
        SETGATE(idt[T_NMI], 0, GD_KT, nmi_entry, 0);
        SETGATE(idt[T_BRKPT], 0, GD_KT, brkpt_entry, 3);
        SETGATE(idt[T_OFLOW], 0, GD_KT, oflow_entry, 0);
        SETGATE(idt[T_BOUND], 0, GD_KT, bound_entry, 0);
        SETGATE(idt[T_ILLOP], 0, GD_KT, illop_entry, 0);
        SETGATE(idt[T_DEVICE], 0, GD_KT, device_entry, 0);
        SETGATE(idt[T_DBLFLT], 0, GD_KT, dblflt_entry, 0);
        SETGATE(idt[T_TSS], 0, GD_KT, tss_entry, 0);
        SETGATE(idt[T_SEGNP], 0, GD_KT, segnp_entry, 0);
        SETGATE(idt[T_STACK], 0, GD_KT, stack_entry, 0);
        SETGATE(idt[T_GPFLT], 0, GD_KT, gpflt_entry, 0);
        SETGATE(idt[T_PGFLT], 0, GD_KT, pgflt_entry, 0);
        SETGATE(idt[T_FPERR], 0, GD_KT, fperr_entry, 0);
        SETGATE(idt[T_ALIGN], 0, GD_KT, align_entry, 0);
        SETGATE(idt[T_MCHK], 0, GD_KT, mchk_entry, 0);
        SETGATE(idt[T_SIMDERR], 0, GD_KT, simderr_entry, 0);
        SETGATE(idt[T_SYSCALL], 0, GD_KT, syscall_entry, 3);

        trap_init_percpu();
}
```

### 问题
1. 每个异常/中断具有特定处理函数的目的是什么?（即如果所有异常/中断都传递给同一个处理程序,那么无法提供当前实现中存在的哪些功能?）
解:
　　不同的中断或者异常当然需要不同的中断处理函数,因为不同的异常/中断可能需要不同的处理方式,比如有些异常是代表指令有错误,则不会返回被中断的命令.而有些中断可能只是为了处理外部IO事件,此时执行完中断函数还要返回到被中断的程序中继续运行.

2. 对软中断程序,测试程序预计会产生一般的保护错误（陷阱13）,但软中断的代码说`int $14`.为什么要产生中断向量13?如果内核允许软中断程序的`int $14`指令调用内核的页面错误处理程序（中断向量14）,会发生什么?
解
　　因为当前的系统正在运行在用户态下,特权级为3,而`INT`指令为系统指令,特权级为0.特权级为3的程序不能直接调用特权级为0的程序,会引发一个`General Protection Exception`,即`trap 13
