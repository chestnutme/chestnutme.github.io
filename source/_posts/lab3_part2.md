title: lab3-user-environment-part2
date: 2017/04/29
tags:
- xv6
- os

---

# Lab3 用户环境 Part2

## Part2 缺页中断, 断点异常和系统调用
part1完成后,JOSkernel已经具备一定的异常处理能力了,part2将进一步完善它,使它能够处理不同类型的中断/异常.

### 处理缺页中断
　　缺页中断（中断向量14（`T_PGFLT`））是一个非常重要的中断,因为后续的实验中,非常依赖kernel能够处理缺页中断的能力.当缺页中断发生时,系统会把引起中断的线性地址存放到控制寄存器`CR2`中.在`trap.c`中,已经提供了一个能够处理这种缺页异常的函数`page_fault_handler()`.

<!-- more -->

#### 练习5
问
> 修改`trap_dispatch`函数,使系统能够把缺页异常引导到 `page_fault_handler()`上执行.在修改完成后,JOS可以成功运行测试程序`faultread,faultreadkernel,faultwrite,faultwritekernel`

解:
> 根据 `trapentry.S`文件中的`TRAPHANDLER`函数可知,这个函数会把当前中断的中断向量压入堆栈中,再根据`inc/trap.h`文件中的`Trapframe`结构体可以知道,`Trapframe`中的`tf_trapno`成员代表这个中断的中断向量.所以在 `trap_dispatch`函数中需要根据输入的`Trapframe`指针`tf`中的 `tf_trapno`成员来判断捕捉是什么中断.如果是是缺页中断,则执行 `page_fault_handler`函数,修改代码如下：
``` c
static void
trap_dispatch(struct Trapframe *tf)
{
    int32_t ret_code;
    // Handle processor exceptions.
    switch(tf->tf_trapno) {
        case (T_PGFLT):
            page_fault_handler(tf);
            break;
         default:
            // Unexpected trap: The user process or the kernel has a bug.
            print_trapframe(tf);
            if (tf->tf_cs == GD_KT)
                panic("unhandled trap in kernel");
            else {
                env_destroy(curenv);
                return;
            }
    }
}
```

### 断点异常
断点异常,中断向量3（`T_BRKPT`）.这个异常可以让调试器能够给程序加上断点.加断点的基本原理就是把要加断点的语句用一个`INT3`指令替换:执行到`INT3`时,会触发软中断.在JOS中通过把这个异常转换成一个伪系统调用,这样的话任何用户环境都可以使用这个伪系统调用来触发JOSkernel监视器`kernel monitor`.

### 练习6
问
> 修改`trap_dispatch()`使断点异常发生时,能够触发kernel监视器.要求修改后的 JOS 能够正确运行`breakpoint`测试程序.

解
> 这个练习其实和上一个练习是类似的,只不过是在这里需要处理断点异常 (`T_BRKPT`),kernel monitor 就是定义在`kern/monitor.c`文件中的`monitor` 函数,修改程序如下
``` c
static void
trap_dispatch(struct Trapframe *tf)
{
    int32_t ret_code;
    // Handle processor exceptions.
    switch(tf->tf_trapno) {
        case (T_PGFLT):
            page_fault_handler(tf);
            break;
        case (T_BRKPT):
            monitor(tf);        
            break;
         default:
            // Unexpected trap: The user process or the kernel has a bug.
            print_trapframe(tf);
            if (tf->tf_cs == GD_KT)
                panic("unhandled trap in kernel");
            else {
                env_destroy(curenv);
                return;
            }
    }
}
```

#### 问题
> 3. 在上面的断点异常(`break point exception`)测试程序中,如果在设置`IDT`时,对断点异常采用不同的方式进行设置,可能会产生触发不同的异常,有可能是断点异常,有可能是一般保护异常(`general protection exception`).这是为什么？应该怎么做才能得到一个想要的断点异常,而不是一般保护异常？
解:
　　通过实验发现出现这个现象的问题就是在设置`IDT`表中的断点异常的表项时,如果把表项中的`DPL`字段设置为3,则会触发断点异常,如果设置为0,则会触发一般保护异常.`DPL`字段代表的含义是段描述符优先级（`Descriptor Privileged Level`）,如果想要当前执行的程序能够跳转到这个描述符所指向的程序继续执行的话,要求当前运行程序的`CPL,RPL`的最大值需要小于等于`DPL`,否则就会出现优先级低的代码试图去访问优先级高的代码的情况,就会触发一般保护异常.那么的测试程序首先运行于用户态,它的`CPL`为3,当异常发生时,它希望去执行`int 3`指令,这是一个系统级别的指令,用户态命令的`CPL`一定大于`int 3`的`DPL`,所以就会触发一般保护异常,但是如果把`IDT`这个表项的`DPL`字段设置为3时,就不会出现这样的现象了,这时如果再出现异常,肯定是因为还没有编写处理断点异常的程序所引起的,所以是断点异常.

### 系统调用
用户程序会要求kernel帮助它完成系统调用.当用户程序触发系统调用,系统进入kernel态.CPU和操作系统将保存该用户程序当前的上下文状态,然后由kernel执行正确的代码完成系统调用,然后回到用户程序继续执行.而用户程序到底是如何得到操作系统的服务,以及它如何说明它希望操作系统如何服务的方法,有很多不同的实现方式.
在JOS中,采用`int`指令,这个指令会触发一个CPU的中断.特别的,用`int $0x30`来代表系统调用中断.注意,`int 0x30`不是通过硬件产生的.
应用程序会把系统调用号以及系统调用的参数放到寄存器中.通过这种方法,kernel就不需要去查询用户程序的堆栈了.系统调用号存放到`%eax`中,参数则存放在`%edx, %ecx, %ebx, %edi, 和 %esi`中.kernel会把返回值送到`%eax`中.在`lib/syscall.c`中已经写好触发一个系统调用的代码.　　

#### 练习7
问:
> 给中断向量`T_SYSCALL`编写一个中断处理函数.需要编辑`kern/trapentry.S`和`kern/trap.c`中的`trap_init()`函数,也需要修改`trap_dispatch()`函数,使其能够通过调用`syscall()`（在``kern/syscall.c``中定义的）函数处理系统调用中断.最后需要实现`kern/syscall.c`中的`syscall()`函数, 确保这个函数会在系统调用号为非法值时返回`-E_INVAL`.要求充分理解`lib/syscall.c`文件,处理在`inc/syscall.h`文件中定义的所有系统调用.
　　通过`make run-hello`指令来运行`user/hello`程序,它应该在控制台上输出 “hello, world”,然后引发一个缺页中断.

解:
> 需要了解一下系统调用的整个流程:如果现在运行的是kernel态的程序的话,此时调用了一个系统调用,比如`sys_cputs`函数时,此时不会触发中断,那么系统会直接执行定义在`lib/syscall.c`文件中的`sys_cputs`这个文件中定义了几个比较常用的系统调用,包括`sys_cputs`, `sys_cgetc`等等.还会发现他们都是统一调用一个 `syscall()`函数,通过这个函数的代码发现其实它是执行了一个汇编指令.所以最终是这个函数完成了系统调用.
以上是运行在kernel态下的程序,调用系统调用时的流程.但是如果是用户态程序呢？这个练习就是让编写程序使的用户程序在调用系统调用时,最终也能经过一系列的处理最终去执行`lib/syscall.c`中的`syscall`指令.
让看一下这个过程,当用户程序中要调用系统调用时,依然比如`sys_cputs`,从它的汇编代码中会发现,它会执行一个`int $0x30`指令,这个指令就是软件中断指令,这个中断的中断号就是`0x3`,即`T_SYSCALL`,所以题目要求首先为这个中断号编写一个中断处理函数,首先就要在`kern/trapentry.S`文件中为它声明它的中断处理函数,即`TRAPHANDLER_NOEC`,与其他中断号的声明一样.
``` c
//kern/trapentry.S
....
TRAPHANDLER_NOEC(t_fperr, T_FPERR)
TRAPHANDLER(t_align, T_ALIGN)
TRAPHANDLER_NOEC(t_mchk, T_MCHK)
TRAPHANDLER_NOEC(t_simderr, T_SIMDERR)
....
TRAPHANDLER_NOEC(t_syscall, T_SYSCALL)
....
```

> 然后在`trap.c`文件中声明`t_syscall()``函数.并且在`trap_init()`函数中为它注册
``` c
//kern/trap.c
....
void t_fperr();
void t_align();
void t_mchk();
void t_simderr();

void t_syscall();
.....
void
trap_init(void)
{
    extern struct Segdesc gdt[];

        .....
    SETGATE(idt[T_ALIGN], 0, GD_KT, t_align, 0);
    SETGATE(idt[T_MCHK], 0, GD_KT, t_mchk, 0);
    SETGATE(idt[T_SIMDERR], 0, GD_KT, t_simderr, 0);

    SETGATE(idt[T_SYSCALL], 0, GD_KT, t_syscall, 3);
    // Per-CPU setup
    trap_init_percpu();
}    
```

> 此时当系统调用中断发生时,系统就可以捕捉到这个中断了,中断发生时,系统会调用 `_alltraps`代码块,并且运行到`trap()`函数处,进入`trap()`函数后,经过一系列处理进入`trap_dispatch()`函数.题目中要求此时需要去调用`kern/syscall.c`中的`syscall`函数,注意到这个函数不是`lib/syscall.c1`中的`syscall`函数,但是通过阅读`kern/syscall.c`中的`syscall`程序发现,它的输入和 `lib/syscall.c`中的`syscall`很像,对比如下

>`kern/syscall.c`中的`syscall`：
``` c
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
```
>  `lib/syscall.c`中的`syscall`：
``` c
syscall(int num, int check, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
```
> 可以假设,`kern/syscall.c`中的`syscall`就是一个外壳函数,它的存在就是为了能够调用`lib/syscall`.所以按照这个思路继续进行下去,再继续观察 `kern/syscall.c`中的其他函数,会发现`kern/syscall.c`中的所有函数和 `lib/syscall.c`中的所有函数都是一样的.比如 在这两个文件中都有`sys_cputs` 函数,但仔细观察可以发现这两个同名的函数,发现实现方式却不一样.以`sys_cputs`函数为例：
> `kern/syscall.c`中的`sys_cputs`：
``` c
static void
sys_cputs(const char *s, size_t len)
{
    // Check that the user has permission to read memory [s, s+len).
    // Destroy the environment if not:.

    user_mem_assert(curenv, s, len, 0);
    // Print the string supplied by the user.
    cprintf("%.*s", len, s);
}
```
> `lib/syscall.c`中的`sys_cputs`:
``` c
void
sys_cputs(const char *s, size_t len)
{
    syscall(SYS_cputs, 0, (uint32_t)s, len, 0, 0, 0);
}
```
> 可见在`lib/syscall.c`中,直接调用`syscall`,但是注意观察 `kern/syscall.c`中的`sys_cputs`,它调用了`cprintf`,这个调用其实就是为了完成输出的功能.但要注意,当程序运行到这里时,系统已经工作在kernel态了,而`cprintf`函数其实就是通过调用`lib/syscall.c`中的`sys_cputs`来实现的,由于此时系统已经处于kernel态了,所以这个`sys_cputs`可以被执行了.所以 `kern/syscall.c`中的`sys_cputs`函数通过调用`cprintf`实现了调用 `lib/syscall.c`中的`syscall`.
所以剩下的部分是如何在`kern/syscall.c`中的`syscall()`函数中正确的调用`sys_cputs`函数了,当然 `kern/syscall.c` 中其他的函数也能完成这个功能.所以必须根据触发这个系统调用的指令到底想调用哪个系统调用来确定该调用哪个函数.

> 如何确定这个指令是要调用哪个系统调用呢？答案是根据`syscall`函数中的第一个参数`syscallno`.这个值要手动传递进去的,它存储在哪里？通过阅读 `lib/syscall.c` 中的`syscall`函数,可以知道它存放在`%eax`寄存器中,所以最后完成`trap_dispatch`和`kern/syscall.c`中的`syscall`函数的代码.
``` c
static void
trap_dispatch(struct Trapframe *tf)
{
    int32_t ret_code;
    // Handle processor exceptions.
    switch(tf->tf_trapno) {
        case (T_PGFLT):
            page_fault_handler(tf);
            break;
        case (T_BRKPT):
            monitor(tf);        
            break;
        case (T_SYSCALL):
    //        print_trapframe(tf);
            ret_code = syscall(
                    tf->tf_regs.reg_eax,
                    tf->tf_regs.reg_edx,
                    tf->tf_regs.reg_ecx,
                    tf->tf_regs.reg_ebx,
                    tf->tf_regs.reg_edi,
                    tf->tf_regs.reg_esi);
            tf->tf_regs.reg_eax = ret_code;
            break;
         default:
            // Unexpected trap: The user process or the kernel has a bug.
            print_trapframe(tf);
            if (tf->tf_cs == GD_KT)
                panic("unhandled trap in kernel");
            else {
                env_destroy(curenv);
                return;
            }
    }
}
```
> `kern/syscall.c` 中的 `syscall`():
``` c
int32_t
syscall(uint32_t syscallno, uint32_t a1, uint32_t a2, uint32_t a3, uint32_t a4, uint32_t a5)
{
    // Call the function corresponding to the 'syscallno' parameter.
    // Return any appropriate return value.

    // panic("syscall not implemented");

    switch (syscallno) {
        case (SYS_cputs):
            sys_cputs((const char *)a1, a2);
            return 0;
        case (SYS_cgetc):
            return sys_cgetc();
        case (SYS_getenvid):
            return sys_getenvid();
        case (SYS_env_destroy):
            return sys_env_destroy(a1);
        default:
            return -E_INVAL;
    }
}
```
　
### 启动用户模式
用户程序真正开始运行的地方是在`lib/entry.S`文件中.该文件中,首先会进行一些设置,然后就会调用`lib/libmain.c`文件中的`libmain()`函数.要求修改一`libmain()`函数,使它能够初始化全局指针`thisenv` ,让它指向当前用户环境的 `Env`结构体.
然后`libmain()`函数就会调用`umain()`.`umain()`函数恰好是`user/hello.c`中被调用的函数.在之前的实验中发现,`hello.c`程序只会打印 “hello, world”,然后就会报出`page fault`(缺页异常),原因是`thisenv->env_id`语句没有被成功初始化..现在已经正确初始化了`thisenv`的值,再次运行就应该不会报错了.

### 练习8
问：
> 把上述的要求补全的代码补全,然后重新启动kernel,此时应该看到`user/hello`程序会打印 “hello, world”, 然后再打印出来 “i am environment 00001000”.然后`user/hello`通过调用`sys_env_destroy()（lib/libmain.c lib/exit.c）`尝试退出.由于kernel目前仅仅支持一个用户运行环境,所以它应该报告唯一的用户环境已被销毁的消息,然后退回kernel监控器.

解：
> 这个练习是通过程序获得当前正在运行的用户环境的`env_id`, 以及这个用户环境所对应的`Env`结构体的指针.`env_id`可以通过调用`sys_getenvid()`函数来获得.那么如何获得它对应的`Env`结构体指针呢？
　　通过阅读`lib/env.h`文件知道,`env_id`的值类型为`int32_t`,按位模式分为三部分
> * 第31位被固定为0
> * 第10~30这21位是标识符,标示这个用户环境
> * 第0~9位代表这个用户环境所采用的`Env`结构体,在`envs`数组中的索引.
所以只需知道`env_id`的低 0~9 位,就可以获得这个用户环境对应的`Env`结构体了.代码如下：
``` c
void
libmain(int argc, char **argv)
{
    // set thisenv to point at our Env structure in envs[].
    thisenv = &envs[ENVX(sys_getenvid())];

    // save the name of the program so that panic() can use it
    if (argc > 0)
        binaryname = argv[0];

    // call user main routine
    umain(argc, argv);

    // exit gracefully
    exit();
}
```

### 页错误和内存保护
内存保护是操作系统的非常重要的一项功能,它可以防止由于用户程序崩溃对操作系统带来的破坏与影响.
操作系统通常依赖于硬件的支持来实现内存保护.操作系统可以让硬件能够始终知晓哪些虚拟地址是有效的,哪些是无效的.当程序尝试去访问一个无效地址,或者尝试去访问一个超出它访问权限的地址时,处理器会在这个指令处终止,并且触发异常,陷入内核态,与此同时把错误的信息报告给内核.如果这个异常是可以被修复的,那么内核会修复这个异常,然后程序继续运行.如果异常无法被修复,则程序永远不会继续运行.
> 一个可修复异常的典型是可自动扩展的堆栈.在许多系统中,内核在初始情况下只会分配一个内核堆栈页,如果程序想要访问这个内核堆栈页之外的堆栈空间的话,就会触发异常,此时内核会自动再分配一些页给这个程序,程序就可以继续运行了.

系统调用也为内存保护带来了问题.大部分系统调用接口让用户程序传递一个指针参数给内核.这些指针指向的是用户缓冲区.通过这种方式,系统调用在执行时就可以解引用这些指针.但是这里有两个问题：
1. 在内核中的页错误（page fault)要比在用户程序中的页错误更严重.如果内核在操作自己的数据结构时出现页错误,这是一个内核的bug,而且异常处理程序会中断整个内核.但是当内核在解引用由用户程序传递来的指针时,它需要一种方法去记录此时出现的页错误都是由用户程序带来的.
2. 内核通常比用户程序有着更高的内存访问权限.用户程序很有可能要传递一个指针给系统调用,这个指针指向的内存区域是内核可以进行读写的,但是用户程序不能.此时内核必须小心不要去解析这个指针,否则的话内核的重要信息很有可能被泄露.

现在需要通过仔细检查所有由用户传递来指针所指向的空间来解决上述两个问题.当一个程序传递给内核一个指针时,内核会检查这个地址是否在整个地址空间的用户地址空间部分,如果是将允许页表进行内存操作.

#### 练习9
问：
> 修改`kern/trap.c`文件,使其能够实现当在内核模式下发现页错误,`trap.c`将转到`panic`.阅读`user_mem_assert`（在`kern/pmap.c`）,并且实现`user_mem_check`；修改`kern/syscall.c`，检查输入参数.
启动内核后,运行`user/buggyhello`程序,用户环境可以被销毁,内核不可以`panic`,输出应该是：
```
[00001000] user_mem_check assertion failure for va 00000001
[00001000] free env 00001000
Destroyed the only environment - nothing more to do!
```

解：
> 首先确定应该根据什么来判断当前运行的程序时处在内核态下还是用户态下？根据`CS`段寄存器的低2位,这两位的名称叫做`CPL`位,表示当前运行的代码的访问权限级别,0代表是内核态,3代表是用户态.
题目要求在检测到页错误是出现在内核态时,通过`panic`跳出来,所以我们把`page_fault_handler`文件修改如下：
``` c
void
page_fault_handler(struct Trapframe *tf)
{
    uint32_t fault_va;

    // Read processor's CR2 register to find the faulting address
    fault_va = rcr2();

    // Handle kernel-mode 页错误s.
    if(tf->tf_cs && 0x01 == 0) {
        panic("page_fault in kernel mode, fault address %d\n", fault_va);
    }

    // We've already handled kernel-mode exceptions, so if we get here,
    // the page fault happened in user mode.

    // Destroy the environment that caused the fault.
    cprintf("[%08x] user fault va %08x ip %08x\n",
        curenv->env_id, fault_va, tf->tf_eip);
    print_trapframe(tf);
    env_destroy(curenv);
}
```
> 然后根据题目的要求,继续完善`kern/pmap.c`文件中的`user_mem_assert()`, `user_mem_check` 函数,通过观察 `user_mem_assert()`函数发现,它调用了 `user_mem_check()`函数.而 `user_mem_check`函数的功能是检查一下当前用户态程序是否有对虚拟地址空间`[va, va+len]`的`perm| PTE_P`访问权限.
然后要做的事情是,先找到这个虚拟地址范围对应于当前用户态程序的页表中的页表项,然后再去看一下这个页表项中有关访问权限的字段,是否包含`perm|PTE_P`,只要有一个页表项是不包含的,就代表程序对这个范围的虚拟地址没有`perm|PTE_P`的访问权限.代码实现如下：
``` c
int
user_mem_check(struct Env *env, const void *va, size_t len, int perm)
{
    char * end = NULL;
    char * start = NULL;
    start = ROUNDDOWN((char *)va, PGSIZE);
    end = ROUNDUP((char *)(va + len), PGSIZE);
    pte_t * cur = NULL;

    for(; start < end; start += PGSIZE) {
        cur = pgdir_walk(env->env_pgdir, (void *)start, 0);
        if((int)start > ULIM || cur == NULL || ((uint32_t)(*cur) & perm) != perm) {
              if(start == ROUNDDOWN((char *)va, PGSIZE)) {
                    user_mem_check_addr = (uintptr_t)va;
              }
              else {
                      user_mem_check_addr = (uintptr_t)start;
              }
              return -E_FAULT;
        }
    }

    return 0;
}
```

> 最后按照题目要求，补全`kern/syscall.c`文件中的一部分内容,即`sys_cputs` 函数,这个函数要求检查用户程序对虚拟地指空间`[s, s+len]`是否有访问权限,所以可以使用刚刚写好的函数`user_mem_assert`来实现.
``` c
static void
sys_cputs(const char *s, size_t len)
{
    // Check that the user has permission to read memory [s, s+len).
    // Destroy the environment if not:.

    user_mem_assert(curenv, s, len, 0);
    // Print the string supplied by the user.
    cprintf("%.*s", len, s);
}
```
> 最后在`kern/kdebug`中修改`debuginfo_eip`函数,对用户空间的数据使用`user_mem_check`函数检查当前用户空间是否对其有`PTE_U`权限.
``` c
...
    const struct UserStabData *usd = (const struct UserStabData *) USTABDATA;
    // Make sure this memory is valid.
    // Return -1 if it is not.  Hint: Call user_mem_check.
    // LAB 3: Your code here.
    if (user_mem_check(curenv, usd, sizeof(*usd),PTE_U) <0 )
            return -1;
    stabs = usd->stabs;
    stab_end = usd->stab_end;
    stabstr = usd->stabstr;
    stabstr_end = usd->stabstr_end;
    // Make sure the STABS and string table memory is valid.
    // LAB 3: Your code here.a
    if (user_mem_check(curenv, stabs, stab_end-stabs,PTE_U) <0 || `user_mem_check`(curenv, stabstr, stabstr_end-stabstr,PTE_U) <0)
            return -1;
```

## 参考链接
[lab3:user environment](https://pdos.csail.mit.edu/6.828/2016/labs/lab3/)
