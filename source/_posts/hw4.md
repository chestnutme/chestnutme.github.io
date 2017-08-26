title: hw4-lazy-page-allocation
date: 2017/04/21
tags:
	- os
	- xv6

---
# hw4 lazy page allocation
## 目标
> OS可以使用页表硬件的技巧之一是 **堆内存的懒惰分配**.当进程需要更多的内存的时候,调用`malloc()`函数申请更多的堆内存,底层系统调用`sbrk`实际完成这个工作.`sbrk`系统调用分配物理内存,并映射到进程的虚拟地址.但是有的进程会一次申请大量的内存,但是又可能根本用不到,比如稀疏数组(sparse array).所以说复杂的内核涉及会将实际的allocation的工作推迟到应用程序尝试使用该页面的时候,即发生了`page fault`,然后再进行实际的分配.  -

## Part1 删除'sbrk'的内存分配
### 任务
> 从`sbrk(n)`系统调用实现中删除页分配,实现为`sysproc.c`中的`sys_sbrk()`函数.`sbrk(n)`系统调用将进程的内存大小增加n个字节,然后返回新分配区域的开始地址.

<!-- more -->

### 实现
> 对系统调用`sbrk`的实际实现`sys_sbrk()`进行修改,删除对`growproc()`的调用只将进程的内存空间大小增加n,而不进行实际的分配.修改代码如下
``` c
//sysproc.c
int
sys_sbrk(void)
{
  int addr, newsz;
  int n;

  if(argint(0, &n) < 0)
    return -1;
  addr = proc->sz;
  newsz = addr + n;
  if (newsz >= KERNBASE)
      return -1;
  proc->sz = newsz;
  return addr;
}
```
> 再次启动xv6,运行`echo hi`,得到如下错误提示：
``` bash
pid 3 sh: trap 14 err 6 on cpu 0 eip 0x12f1 addr 0x4004--kill proc
```
> 因为在shell中运行`echo`时,需要调用`malloc`函数.运行`malloc`的时候,虽然返回值是显示内存分配成功,但是当程序实际试图操作cmd指向的内存区域的时候,发现该内存区域不是当前进程所有的,因为在`sys_sbrk`中实际没有调用`growproc()`进行分配.

## Part2 懒惰分配
### 任务
> 通过在故障地址映射新分配的物理内存页面,然后返回到用户空间让程序继续执行,修改trap.c中的代码以响应用户空间的页面错误.代码不需要完美到照顾到每个细节,当前我们只需要能够执行`echo`等简单代码即可.　　

### 实现
> 在`trap.c`中,当发现是`page fault`错误的时候,可以按照当前进程的`proc->sz`来实际的分配内存.所以首先应该获取发生`page fault`时刻的虚地址,该虚地址之后的部分就应该是本来应该分配但是实际没有分配的(lazy allocation),而实际需要分配多少,应该根据`proc->sz`的大小来定.
因为在发生`page fault`的地址为在`malloc`之后返回给进程的地址,而这个地址正是`proc->sz`,所以该虚地址的大小为实际的内存的大小.
``` c
if (tf->trapno == T_PGFLT) {
      char* mem;
      uint a;

      a = PGROUNDDOWN(rcr2()); //rcr2()返回页错误时的地址
      mem = kalloc();
      if (mem == 0) {
          cprintf("run out of memory!\n");
          proc->killed = 1;
          break;
      }
      memset(mem, 0, PGSIZE);
      mappages(proc->pgdir, (char*)a, PGSIZE, v2p(mem), PTE_W|PTE_U);
      break;
}
```
