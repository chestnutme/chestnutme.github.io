title: lab4-Preemptive Multitasking part23
date: 2017/05/13
tags:
- xv6
- os

---
# Lab4 抢占式多进程
## Part2 写入时复制 copy-on-write fork
如前所述,Unix将fork()系统调用作为进程创建原语.fork()系统调用将调用进程的地址空间（父进程）创建一个新的进程（子进程）.
xv6 Unix `fork()`通过将父进程页面中的所有数据复制到为子进程分配的新页面中,这个机制基本和`dumbfork()`相同.复制父进程的地址空间到子进程是`fork()`中代价最大的操作.
但是,调用`fork()`后经常在子进程中跟随`exec()`调用, 其会用加载新程序到子进程的内存.例如, 这是shell的常用机制.在这种情况下,复制父进程地址空间的时间大部分被浪费,因为子进程在调用`exec()`之前只使用很少的内存.

因此,Unix的更高版本利用虚拟内存硬件,允许父子进程**共享**映射到其各自地址空间的内存,直到其中一个进程实际修改这段内存.这种技术被称为**写时复制(copy-on-fork)**.为了做到这一点,调用`fork()`时,内核上将复制父进程的地址空间映射到进程,而不是复制映射页的内容,同时标记现在共享的页面为只读.当两个进程中的一个尝试写入其中一个共享页面时,该进程将出现页面错误.在这一点上,Unix内核意识到该页面真的是一个“虚拟”或“写时复制”的副本,因此它创建一个新的/私有的故障页面的可写副本.这样,单个页面的内容在实际写入之前实际上并不被复制.`fork()`的这种优化使随后的`exec()`代价小很多：在子进程调用`exec()`之前,可能只需要复制一个页面（它的堆栈的当前页面）.

在part2,实现一个 Unix `fork()`,其具有写时复制功能,并作为一个用户空间库例程来.另外,使得单个用户模式程序定义自己的`fork()`:一个程序想要一个稍微不同的`fork()`（如总是复制的`dumbfork()`,或者父子进程共享内存）可以很容易修改而实现.

### 用户级页面错误处理
用户级的写时复制`fork()`首先需要知道发生在写保护页面上的页面错误,这是首先要实现的.写时复制只是用户级页面故障处理的许多可能用途之一.
一般实现方式是:设置一个地址空间,以便页面错误指示出何时需要执行操作.例如,大多数Unix内核最初仅在新进程的堆栈区域中映射单个页面,并且随着进程的堆栈增加而随后分配和映射额外的堆栈页面,并导致尚未映射的堆栈地址发生页面错误,对此典型的Unix内核必须跟踪在进程空间的每个区域中出现页面错误时要执行的操作.又例如,堆栈区域中的故障通常会导致分配和映射物理内存新页面.程序BSS区域的故障通常会分配一个新的页面,以0填充,并在进行映射.
以上是内核需要跟踪的信息.不采用传统的Unix方法,而更好地处理用户空间中的每个页面错误,使得bug破坏性更小.这种设计具有额外的优点,允许程序在定义其内存区域方面具有很大的灵活性.
<!-- more -->

#### 设置页面错误处理器
为了处理自己的页面错误,用户环境将需要向JOS内核注册页面错误处理程序`entrypoint`.用户环境通过新的`sys_env_set_pgfault_upcall`系统调用注册其页面错误入口点.增加一个新成员`Env`结构`env_pgfault_upcall`来记录这些信息.

#### 练习8
问:
> 实现`sys_env_set_pgfault_upcall`系统调用.注:因为这个系统调用危险性很高,所以在查找目标环境的环境ID时要启用权限检查,

解:
> 实现如下:
``` c
static int
sys_env_set_pgfault_upcall(envid_t envid, void *func)
{
        struct Env *env;

        if (envid2env(envid, &env, 1) < 0)
                return -E_BAD_ENV;
        env->env_pgfault_upcall = func;
        return 0;
}
```

#### 用户环境的正常/异常堆栈
在正常运行期间,用户进程运行在用户栈上,栈顶寄存器`ESP`指向`USTACKTOP`处,堆栈数据位于`USTACKTOP-PGSIZE 与USTACKTOP-1`之间的页.当在用户模式发生页面错误时,内核将在专门处理页面错误的用户异常栈上重新启动进程. 异常栈正是为了上面设置的异常处理例程设立的.当异常发生时,而且该用户进程注册了该异常的处理例程,那么就会转到异常栈上,运行异常处理例程.
到目前位置出现了三个栈：
* 内核态系统栈 [KSTACKTOP, KSTACKTOP-KSTKSIZE]
* 用户态错误处理栈 [UXSTACKTOP, UXSTACKTOP - PGSIZE]
* 用户态运行栈 [USTACKTOP, UTEXT]

内核态系统栈是运行内核相关程序的栈,在有中断被触发之后,CPU会将栈自动切换到内核栈上来,而内核栈的设置是在`kern/trap.c`的`trap_init_percpu()`中设置的.
``` c
void
trap_init_percpu(void)
{
        // Setup a TSS so that we get the right stack
        // when we trap to the kernel.
        thiscpu->cpu_ts.ts_esp0 = KSTACKTOP - cpunum() * (KSTKGAP + KSTKSIZE);
        thiscpu->cpu_ts.ts_ss0 = GD_KD;

        // Initialize the TSS slot of the gdt.
        gdt[(GD_TSS0 >> 3) + cpunum()] = SEG16(STS_T32A, (uint32_t) (&thiscpu->cpu_ts),
                                        sizeof(struct Taskstate) - 1, 0);
        gdt[(GD_TSS0 >> 3) + cpunum()].sd_s = 0;

        // Load the TSS selector (like other segment selectors, the
        // bottom three bits are special; we leave them 0)
        ltr(GD_TSS0 + sizeof(struct Segdesc) * cpunum());

        // Load the IDT
        lidt(&idt_pd);
}
```
而用户态错误处理栈是用户定义注册了自己的中断处理程序之后,相应的例程运行时的栈.整个过程如下：
> 首先陷入到内核,栈位置从用户运行栈切换到内核栈,进入到`trap`中,进行中断处理分发,进入到`page_fault_handler()`.当确认是用户程序触发的`page fault`的时候(如果是内核触发的,会直接`panic`),为其在用户错误栈里分配一个`UTrapframe`的大小.把栈切换到用户错误栈,运行响应的用户中断处理程序中断处理程序可能会触发另外一个同类型的中断,这个时候就会产生递归式的中断嵌套处理.处理完成之后,返回到用户运行栈.

#### 调用用户页面故障处理程序
将用户自己定义的页面故障处理进程当作是一次函数调用看待,当错误发生的时候,调用一个函数,但实际上还是当前这个进程,并没有发生变化.所以当切换到异常栈的时候,依然运行当前进程,但只是运行的是中断处理函数,所以说此时的栈指针发生了变化,而且程序计数器`eip`也发生了变化,同时还需要知道的是引发错误的地址在哪.这些都是要在切换到异常栈的时候需要传递的信息.和之前从用户栈切换到内核栈一样,这里是通过在栈上构造结构体,传递指针完成的.
在`inc / trap.h`新定义了一个类似`struct Trapframe`结构体`struct UTrapframe`用来记录出现页面错误时候的信息：
``` c
struct UTrapframe {
        /* information about the fault */
        uint32_t utf_fault_va;  /* va for T_PGFLT, 0 otherwise */
        uint32_t utf_err;
        /* trap-time return state */
        struct PushRegs utf_regs;
        uintptr_t utf_eip;
        uint32_t utf_eflags;
        /* the trap-time stack to return to */
        uintptr_t utf_esp;
} __attribute__((packed));
```
相比于`Trapframe`,这里多了`utf_fault_va`,因为要记录触发错误的内存地址,同时还少了`es,ds,ss`等段记录.因为从用户态栈切换到异常栈,或者从异常栈再切换回去,实际上都是一个用户进程,所以不涉及到段的切换,不用记录.在实际使用中,`Trapframe`是作为记录进程状态的结构体存在的,也作为函数参数进行传递；而`UTrapframe`只在处理用户定义的异常时用到.
整体上讲,当正常执行过程中发生了页面错误,那么栈的切换是
**用户运行栈—>内核栈—>异常栈**
而如果在异常处理程序中发生了页面错误,那么栈的切换是
**异常栈—>内核栈—>异常栈**

#### 练习9
问:
> 实现`kern/trap.c`中`page_fault_handler`函数, 把页面错误分发到对应的用户态异常处理函数.

解:
> 如果当前已经在用户错误栈上了,那么需要留出4个字节,否则不需要,具体和跳转机制有关系.简单说就是在当前的异常栈栈顶的位置向下留出保存`UTrapframe`的空间,然后将`tf`中的参数复制过来.修改当前进程的程序计数器和栈指针,然后重启这个进程,此时就会在用户错误栈上运行中断处理程序了.中断处理程序运行结束之后,需要再回到用户运行栈中,这是异常处理程序需要做的.
``` c
void
page_fault_handler(struct Trapframe *tf)
{
        uint32_t fault_va;

        // Read processor's CR2 register to find the faulting address
        fault_va = rcr2();

        // Handle kernel-mode page faults.
        if (tf->tf_cs == GD_KT)
                panic("page_fault in kernel mode, fault address: %d\n", fault_va);

        struct UTrapframe *utf;

        if (curenv->env_pgfault_upcall) {
                if (UXSTACKTOP - PGSIZE <= tf->tf_esp && tf->tf_esp <= UXSTACKTOP - 1)
                        utf = (struct UTrapframe *)(tf->tf_esp - sizeof(struct UTrapframe) - 4);
                else
                        utf = (struct UTrapframe *)(UXSTACKTOP - sizeof(struct UTrapframe));
                user_mem_assert(curenv, (void *)utf, sizeof(struct UTrapframe), PTE_U | PTE_W);

                utf->utf_fault_va = fault_va;
                utf->utf_err = tf->tf_trapno;
                utf->utf_eip = tf->tf_eip;
                utf->utf_eflags = tf->tf_eflags;
                utf->utf_esp = tf->tf_esp;
                utf->utf_regs = tf->tf_regs;
                tf->tf_eip = (uint32_t)curenv->env_pgfault_upcall;
                tf->tf_esp = (uint32_t)utf;
                env_run(curenv);
        }

        // Destroy the environment that caused the fault.
        cprintf("[%08x] user fault va %08x ip %08x\n",
                curenv->env_id, fault_va, tf->tf_eip);
        print_trapframe(tf);
        env_destroy(curenv);
}
```

#### 用户模式页面错误的入口
接下来需要实现汇编程序,该程序将负责调用页面错误处理程序,并在原始故障指令下继续执行,其将调用处理程序`sys_env_set_pgfault_upcall()`向内核注册.

#### 练习10
问:
> 实现在`lib/pfentry.S`中的`_pgfault_upcall`调用.

解:
> `_pgfault_upcall`是所有用户页错误处理程序的入口,在这里调用用户自定义的处理程序,并在处理完成后,从异常栈中保存的`UTrapframe`结构体中恢复相应信息,然后跳回到发生错误之前的指令,恢复原来的进程运行.具体过程如下:
调用`_pgfault_handler`返回时的操作,此时的异常栈结构如下：
![UStack1](/images/lab4/UStack1.png)
这里`trap-time esp`上的空间有1个4字节的保留空间,是做为中断递归的情形. 然后将栈中的`trap-time esp`取出减去4,再存回栈中.此时如果是中断递归中,`esp-4`即是保留的4字节地址；如果不是则是用户运行栈的栈顶. 再将原来出错程序的`trap-time eip`取出放入保留的4字节,以便后来恢复运行.此时的异常栈布局如下：
![UStack2](/images/lab4/UStack2.png)
紧接着恢复通用寄存器和EFLAG标志寄存器,此时的异常栈结构如下：
![UStack3](/images/lab4/UStack3.png)
最后`pop esp`切换为原来出错程序的运行栈,最后使用`ret`返回出错程序.
``` c
.text
.globl _pgfault_upcall
_pgfault_upcall:
        // Call the C page fault handler.
        pushl %esp                      // function argument: pointer to UTF
        movl _pgfault_handler, %eax
        call *%eax
        addl $4, %esp                   // pop function argument

        movl 48(%esp), %ebp
        subl $4, %ebp
        movl %ebp, 48(%esp)
        movl 40(%esp), %eax
        movl %eax, (%ebp)

        // Restore the trap-time registers.  After you do this, you can no longer modify any general-purpose registers.
        addl $8, %esp
        popal

        // Restore eflags from the stack.  After you do this, you can
        // no longer use arithmetic operations or anything else that
        // modifies eflags.
        addl $4, %esp
        popfl

        // Switch back to the adjusted trap-time stack.

        popl %esp

        // Return to re-execute the instruction that faulted.
        ret
```

#### 练习11
问:
> 在`lib / pgfault.c`中完成`set_pgfault_handler()`,即实现用户级页面故障处理机制的C语言库函数.

解:
> 进程在运行前注册自己的页错误处理程序,重点是申请用户异常栈空间,最后添加上系统调用号.
``` c
void
set_pgfault_handler(void (*handler)(struct UTrapframe *utf))
{
        int r;

        if (_pgfault_handler == 0) {
                // First time through!
                if ((r = sys_page_alloc(thisenv->env_id, (void *)(UXSTACKTOP - PGSIZE), PTE_P | PTE_W | PTE_U)) < 0)
                        panic("set_pgfault_handler: %e", r);
                sys_env_set_pgfault_upcall(thisenv->env_id, _pgfault_upcall);

        }

        // Save handler pointer for assembly to call.
        _pgfault_handler = handler;
}
```

### 实现写入时复制 fork
接下来就是最重要的部分：实现`copy-on-write fork`.
与之前的`dumbfork()`不同,`fork()`出一个子进程之后,首先要进行的就是将父进程的页表的全部映射拷贝到子进程的地址空间中去.这个时候物理页会被两个进程同时映射,但是在写的时候是应该隔离的.采取的方法是在子进程映射的时候,将父进程空间中所有可以写的页表的部分全部标记为可读且COW(copy-on-write).而当父进程或者子进程任意一个发生了写的时候,因为页表现在都是不可写的,所以会触发异常,进入到我们设定的page fault处理例程,当检测到是对COW页的写操作的情况下,就可以将要写入的页的内容全部拷贝一份,重新映射.

#### 练习12
问:
> 实现在`lib/fork.c`的`fork,duppage和pgfault`.

解:
> 首先需要为父进程设定错误处理例程.调用`set_pgfault_handler()`是因为当前并不知道父进程是否已经建立了异常栈,没有的话就会建立一个,而`sys_env_set_pgfault_upcall`则不会建立异常栈.
接着调用`sys_exofork`准备一个和父进程状态相同的子进程,状态暂时设置为`ENV_NOT_RUNNABLE`.然后进行拷贝映射的部分,在当前进程的页表中所有标记为`PTE_P`的页的映射都需要拷贝到子进程空间中去.但是有一个例外,是必须要新申请一页来拷贝内容的,就是用户异常栈.因为copy-on-write就是依靠用户异常栈实现的,所以说这个栈要在fork完成的时候每个进程都有一个,要硬拷贝过来.
主要流程就是：
> 1. 申请新的物理页,映射到子进程的(UXSTACKTOP-PGSIZE)位置上去.
> 2. 父进程的PFTEMP位置也映射到子进程新申请的物理页上去,这样父进程也可以访问这一页.
> 3. 在父进程空间中,将用户错误栈全部拷贝到子进程的错误栈上去,也就是刚刚申请的那一页.
> 4. 然后父进程解除对PFTEMP的映射.
> 5. 最后把子进程的状态设置为可运行

具体实现如下:
首先是`pgfault`处理`page fault`时的写时复制.
``` c
static void
pgfault(struct UTrapframe *utf)
{
        int r;
        void *addr = (void *) utf->utf_fault_va;
        uint32_t err = utf->utf_err;

        if ((err & FEC_WR) == 0 || (uvpt[PGNUM(addr)] & PTE_COW) == 0)
                panic("pgfault: it's not writable or attempt to access a non-cow page!");
        // Allocate a new page, map it at a temporary location (PFTEMP),
        // copy the data from the old page to the new page, then move the new
        // page to the old page's address.

        envid_t envid = sys_getenvid();
        if ((r = sys_page_alloc(envid, (void *)PFTEMP, PTE_P | PTE_W | PTE_U)) < 0)
                panic("pgfault: page allocation failed %e", r);

        addr = ROUNDDOWN(addr, PGSIZE);
        memmove(PFTEMP, addr, PGSIZE);
        if ((r = sys_page_unmap(envid, addr)) < 0)
                panic("pgfault: page unmap failed %e", r);
        if ((r = sys_page_map(envid, PFTEMP, envid, addr, PTE_P | PTE_W |PTE_U)) < 0)
                panic("pgfault: page map failed %e", r);
        if ((r = sys_page_unmap(envid, PFTEMP)) < 0)
                panic("pgfault: page unmap failed %e", r);
}
```
在`pgfault()`中先判断是否页错误是由写时拷贝造成的,如果不是则`panic`.借用了一个一定不会被用到的位置`PFTEMP`,专门用来发生`page fault`的时候拷贝内容用的.先解除`addr`原先的页映射关系,然后将`addr`映射到`PFTEMP`映射的页,最后解除`PFTEMP`的页映射关系.
接下来是`duppage`函数,负责进行COW方式的页复制,将当前进程的第pn页对应的物理页的映射到`envid`的第pn页上去,同时将这一页都标记为COW.
``` c
static int
duppage(envid_t envid, unsigned pn)
{
        int r;

        void *addr;
        pte_t pte;
        int perm;

        addr = (void *)((uint32_t)pn * PGSIZE);
        pte = uvpt[pn];
        perm = PTE_P | PTE_U;
        if ((pte & PTE_W) || (pte & PTE_COW))
                perm |= PTE_COW;
        if ((r = sys_page_map(thisenv->env_id, addr, envid, addr, perm)) < 0) {
                panic("duppage: page remapping failed %e", r);
                return r;
        }
        if (perm & PTE_COW) {
                if ((r = sys_page_map(thisenv->env_id, addr, thisenv->env_id, addr, perm)) < 0) {
                        panic("duppage: page remapping failed %e", r);
                        return r;
                }
        }

        return 0;
}
```
最后是fork函数,将页映射拷贝过去,这里需要考虑的地址范围就是`从UTEXT到UXSTACKTOP`为止,而在此之上的范围因为都是相同的,在`env_alloc`的时候已经设置好了.
``` c
envid_t
fork(void)
{
        uint32_t addr;
        int i, j, pn, r;
        extern void _pgfault_upcall(void);
        if ((envid = sys_exofork()) < 0) {
                panic("sys_exofork failed: %e", envid);
                return envid;
        }
        if (envid == 0) {
                thisenv = &envs[ENVX(sys_getenvid())];
                return 0;
        }

        for (i = PDX(UTEXT); i < PDX(UXSTACKTOP); i++) {
                if (uvpd[i] & PTE_P) {
                        for (j = 0; j < NPTENTRIES; j++) {
                                pn = PGNUM(PGADDR(i, j, 0));
                                if (pn == PGNUM(UXSTACKTOP - PGSIZE))
                                        break;
                                if (uvpt[pn] & PTE_P)
                                        duppage(envid, pn);
                        }
                }

        }

        if ((r = sys_page_alloc(envid, (void *)(UXSTACKTOP - PGSIZE), PTE_P | PTE_U | PTE_W)) < 0) {
                panic("fork: page alloc failed %e", r);
                return r;
        }
        if ((r = sys_page_map(envid, (void *)(UXSTACKTOP - PGSIZE), thisenv->env_id, PFTEMP, PTE_P | PTE_U | PTE_W)) < 0) {
                panic("fork: page map failed %e", r);
                return r;
        }
        memmove((void *)(UXSTACKTOP - PGSIZE), PFTEMP, PGSIZE);
        if ((r = sys_page_unmap(thisenv->env_id, PFTEMP)) < 0) {
                panic("fork: page unmap failed %e", r);
                return r;
        }
        sys_env_set_pgfault_upcall(envid, _pgfault_upcall);
        if ((r = sys_env_set_status(envid, ENV_RUNNABLE)) < 0) {
                panic("fork: set child env status failed %e", r);
                return r;
        }

        return envid;
}
```

## Part3 抢占式调度和进程间通信
Lab4的最后一部分就是实现抢占式调度和进程间通信。

### 时钟中断和抢占
先前的调度是进程资源放弃CPU，但是实际中没有进程会这样做的，而为了不让某一进程耗尽CPU资源，需要抢占式调度，也就需要硬件定时。但是外部硬件定时在Bootloader的时候就关闭了，至今都没有开启。而JOS采取的策略是，在内核中的时候，外部中断是始终关闭的，在用户态的时候，需要开启中断。

#### 中断原则
外部中断称为IRQ。一共有16个可能的IRQ，编号为0到15.从IRQ号到IDT条目的映射不是固定的, `picirq.c`中的`pic_init`将IRQ 0-15映射到到IDT的`[IRQ_OFFSET,IRQ_OFFSET+15]`.
在`inc / trap.h`中， `IRQ_OFFSET`定义为十进制数32.因此IDT条目32-47对应于IRQ 0-15。例如，时钟中断是IRQ 0.因此，`IDT[IRQ_OFFSET + 0]`（即`IDT[32]`）包含内核中时钟的中断处理程序例程的地址。选择正确的`IRQ_OFFSET`使得设备中断不与处理器异常重叠,可以避免异常处理的混乱.
在JOS中，与xv6 Unix相比，做了一个关键的简化。在内核中，外部设备中断始终被禁用（和xv6一样，在用户空间中启用）。外部中断由寄存器的`FL_IF标志位%eflags`控制（见`inc/mmu.h`）。当该位置1时，外部中断被使能。虽然可以通过几种方式修改该位，但由于简化，可以通过在`%eflags`进入和离开用户模式时保存和恢复寄存器的过程来处理该位。必须确保`FL_IF`标志在用户环境中运行，以便当中断到达时，它将被传递到处理器并由中断代码处理。否则，中断会被屏蔽或忽略，直到重新启用中断。Lab1中用`bootloaded`的第一个指令屏蔽了中断，到目前为止还没开启过.

#### 练习12
问:
> 修改`kern/trapentry.S`和`kern/trap.c`来初始化`IDT中IRQs0-15`的入口和处理函数。然后修改`env_alloc`函数来确保进程在用户态运行时中断是打开的。

解:
> 模仿原先设置的默认中断向量即可，在`kern/trapentry.S`中定义`IRQ0-15`的处理例程。
``` c
TRAPHANDLER(irq0_entry, IRQ_OFFSET + 0, 0, 0);
TRAPHANDLER(irq1_entry, IRQ_OFFSET + 1, 0, 0);
TRAPHANDLER(irq2_entry, IRQ_OFFSET + 2, 0, 0);
TRAPHANDLER(irq3_entry, IRQ_OFFSET + 3, 0, 0);
TRAPHANDLER(irq4_entry, IRQ_OFFSET + 4, 0, 0);
TRAPHANDLER(irq5_entry, IRQ_OFFSET + 5, 0, 0);
TRAPHANDLER(irq6_entry, IRQ_OFFSET + 6, 0, 0);
TRAPHANDLER(irq7_entry, IRQ_OFFSET + 7, 0, 0);
TRAPHANDLER(irq8_entry, IRQ_OFFSET + 8, 0, 0);
TRAPHANDLER(irq9_entry, IRQ_OFFSET + 9, 0, 0);
TRAPHANDLER(irq10_entry, IRQ_OFFSET + 10, 0, 0);
TRAPHANDLER(irq11_entry, IRQ_OFFSET + 11, 0, 0);
TRAPHANDLER(irq12_entry, IRQ_OFFSET + 12, 0, 0);
TRAPHANDLER(irq13_entry, IRQ_OFFSET + 13, 0, 0);
TRAPHANDLER(irq14_entry, IRQ_OFFSET + 14, 0, 0);
TRAPHANDLER(irq15_entry, IRQ_OFFSET + 15, 0, 0);
```
然后在IDT中注册，修改`trap_init`，由于先前已经实现简化，故此无需做处理。
最后在`env_alloc`函数中打开中断。
``` c
        // kern/env_alloc.c
        // Also clear the IPC receiving flag.
        e->env_ipc_recving = 0;

        // Set FL_IF so that user environments run with interrupts enabled
        e->env_tf.tf_eflags |= FL_IF;

        // commit the allocation
        env_free_list = e->env_link;
        *newenv_store = e;
```

#### 处理时钟中断
现在虽然中断使能已经打开，在用户态进程运行的时候，外部中断会产生并进入内核，但是现在还没有能处理这类中断。所以需要修改`trap_dispatch`，在发生外部定时中断的时候，调用调度器，调度另外一个可运行的进程。

#### 练习14
问:
> 修改`trap_dispatch`h函数，当发生时钟中断时调用`sched_yield`函数来调度下一个进程。

解:
> 添加对应函数即可
```
// kern/trap.c

// Handle clock interrupts. Don't forget to acknowledge the
// interrupt using lapic_eoi() before calling the scheduler!
if (tf->tf_trapno == IRQ_OFFSET + IRQ_TIMER) {
                lapic_eoi();
                sched_yield();
                return;
 }
```

#### 进程间通信(IPC)
IPC是计算机系统中非常重要的一部分。在JOS实现IPC的方式是当两个进程需要通信的话，一方要发起`recv`，然后阻塞，直到有一个进程调用`send`向正在接受的进程发送了信息，阻塞的进程才会被唤醒。在JOS中，可以允许传递两种信息，一是一个32位整数，另外一个就是传递页的映射，在这个过程中，接收方和发送方将同时映射到一个相同的物理页，此时也就实现了内存共享。最后将这两个功能实现为一个系统调用。
　　
#### 实现IPC
在JOS的IPC实现机制中，修改`Env`结构体如下：
``` c
struct Env {
        struct Trapframe env_tf;        // Saved registers
        struct Env *env_link;           // Next free Env
        envid_t env_id;                 // Unique environment identifier
        envid_t env_parent_id;          // env_id of this env's parent
        enum EnvType env_type;          // Indicates special system environments
        unsigned env_status;            // Status of the environment
        uint32_t env_runs;              // Number of times environment has run
        int env_cpunum;                 // The CPU that the env is running on

        // Address space
        pde_t *env_pgdir;               // Kernel virtual address of page dir

        // Exception handling
        void *env_pgfault_upcall;       // Page fault upcall entry point

        // Lab 4 IPC
        bool env_ipc_recving;           // Env is blocked receiving
        void *env_ipc_dstva;            // VA at which to map received page
        uint32_t env_ipc_value;         // Data value sent to us
        envid_t env_ipc_from;           // envid of the sender
        int env_ipc_perm;               // Perm of page mapping received
};
```
其中增加了5个成员：
* env_ipc_recving：
当进程使用env_ipc_recv函数等待信息时，会将这个成员设置为1，然后堵塞等待；当一个进程向它发消息解除堵塞后，发送进程将此成员修改为0。
* env_ipc_dstva：
如果进程要接受消息并且是传送页，保存页映射的地址，且该地址<=UTOP。
* env_ipc_value：
若等待消息的进程接收到消息，发送方将接收方此成员设置为消息值。
* env_ipc_from：
发送方负责设置该成员为自己的envid号。
* env_ipc_perm：
如果进程要接收消息并且传送页，那么发送方发送页之后将传送的页权限赋给这个成员

#### 练习15
问:
> 实现在`kern/syscall.c`中的`sys_ipc_recv和sys_ipc_try_send`函数。最后实现用户态的`ipc_recv和ipc_send`。

解:
> 首先是`sys_ipc_recv`函数，其功能是当一个进程试图去接收信息的时候，应该将自己标记为正在接收信息，而且为了不浪费CPU资源，应该同时标记自己为`ENV_NOT_RUNNABLE`，只有当有进程向自己发了信息之后，才会重新恢复可运行。最后将自己标记为不可运行之后，调用调度器运行其他进程。
``` c
static int
sys_ipc_recv(void *dstva)
{
        if (dstva < (void *)UTOP && PGOFF(dstva))
                return -E_INVAL;
        curenv->env_ipc_recving = true;
        curenv->env_ipc_dstva = dstva;
        curenv->env_status = ENV_NOT_RUNNABLE;
        curenv->env_ipc_from = 0;
        sched_yield();
        return 0;
}
```
接着是`sys_ipc_try_send`函数，其实现相对来说麻烦很多，因为有很多的检测项，包括权限是否符合要求，要传送的页有没有，能不能将这一页映射到对方页表中去等等。如果`srcva`是在`UTOP`之下，那么说明是要共享内存，那就首先要在发送方的页表中找到`srcva`对应的页表项，然后在接收方给定的虚地址处插入这个页表项。接收完成之后，重新将当前进程设置为可运行，同时把`env_ipc_recving`设置为0，以防止其他的进程再发送，覆盖掉当前的内容。
``` c
static int
sys_ipc_try_send(envid_t envid, uint32_t value, void *srcva, unsigned perm)
{
        int r;
        pte_t *pte;
        struct PageInfo *pp;
        struct Env *env;

        if ((r = envid2env(envid, &env, 0)) < 0)
                return -E_BAD_ENV;
        if (env->env_ipc_recving != true || env->env_ipc_from != 0)
                return -E_IPC_NOT_RECV;
        if (srcva < (void *)UTOP && PGOFF(srcva))
                return -E_INVAL;
        if (srcva < (void *)UTOP) {
                if ((perm & PTE_P) == 0 || (perm & PTE_U) == 0)
                        return -E_INVAL;
                if ((perm & ~(PTE_P | PTE_U | PTE_W | PTE_AVAIL)) != 0)
                        return -E_INVAL;
        }
        if (srcva < (void *)UTOP && (pp = page_lookup(curenv->env_pgdir, srcva, &pte)) == NULL)
                return -E_INVAL;
        if (srcva < (void *)UTOP && (perm & PTE_W) != 0 && (*pte & PTE_W) == 0)
                return -E_INVAL;
        if (srcva < (void *)UTOP && env->env_ipc_dstva != 0) {
                if ((r = page_insert(env->env_pgdir, pp, env->env_ipc_dstva, perm)) < 0)
                        return -E_NO_MEM;
                env->env_ipc_perm = perm;
        }

        env->env_ipc_from = curenv->env_id;
        env->env_ipc_recving = false;
        env->env_ipc_value = value;
        env->env_status = ENV_RUNNABLE;
        env->env_tf.tf_regs.reg_eax = 0;
        return 0;
}
```
完成后需要要加上分发机制，将调用号加上。
最后是2个用户态库函数的实现。
``` c
int32_t
ipc_recv(envid_t *from_env_store, void *pg, int *perm_store)
{
        int r;

        if (pg == NULL)
                r = sys_ipc_recv((void *)UTOP);
        else
                r = sys_ipc_recv(pg);
        if (from_env_store != NULL)
                *from_env_store = r < 0 ? 0 : thisenv->env_ipc_from;
        if (perm_store != NULL)
                *perm_store = r < 0 ? 0 : thisenv->env_ipc_perm;
        if (r < 0)
                return r;
        else
                return thisenv->env_ipc_value;
}

void
ipc_send(envid_t to_env, uint32_t val, void *pg, int perm)
{
        int r;
        void *dstpg;

        dstpg = pg != NULL ? pg : (void *)UTOP;
        while((r = sys_ipc_try_send(to_env, val, dstpg, perm)) < 0) {
                if (r != -E_IPC_NOT_RECV)
                        panic("ipc_send: send message error %e", r);
                sys_yield();
        }
}
```
