title: hw3-system-calls
date: 2017/04/19
tags:
	- os
	- xv6

---
# hw3 xv6 system calls
## 目标
在[hw1 boot pc]的基础上,学习xv6的系统调用:
> * 运用xv6的系统调用
> * 修改xv6 kernel, 添加新系统调用

## Part1: 系统调用追踪

### 任务
> 修改`systemcall.c`,在进行系统调用时,打印出系统调用的名字和返回值.实现后,在xv6启动时是打印如下内容:
``` bash
...
fork -> 2
exec -> 0
open -> 3
close -> 0
$write -> 1
 write -> 1
```
<!-- more -->

### 实现
> 系统调用的储存形式是: 在`syscall[]`数组中指向系统调用函数的指针.因此修改`syscall()`函数, 使其获取函数指针的索引, 而从索引到系统调用的映射在` syscall.h`头文件中.添加`syscall_name`字符串数组, 映射系统调用索引到其名称上, 然后在`syscall()`函数内添加`cprintf`输出:
``` c
static char syscalls_name[][8] = {
[SYS_fork]    "fork",
[SYS_exit]    "exit",
[SYS_wait]    "wait",
[SYS_pipe]    "pipe",
[SYS_read]    "read",
[SYS_kill]    "kill",
[SYS_exec]    "exec",
[SYS_fstat]   "fstat",
[SYS_chdir]   "chdir",
[SYS_dup]     "dup",
[SYS_getpid]  "getpid",
[SYS_sbrk]    "sbrk",
[SYS_sleep]   "sleep",
[SYS_uptime]  "uptime",
[SYS_open]    "open",
[SYS_write]   "write",
[SYS_mknod]   "mknod",
[SYS_unlink]  "unlink",
[SYS_link]    "link",
[SYS_mkdir]   "mkdir",
[SYS_close]   "close",
[SYS_date]    "date",
};

void
syscall(void)
{
  int num;

  num = proc->tf->eax;  //获取系统调用号
  if(num > 0 && num < NELEM(syscalls) && syscalls[num])
	{
    proc->tf->eax = syscalls[num]();  //存储系统调用返回值
    cprintf("%s -> %d\n", syscalls_name[num], proc->tf->eax);
  } else {
    cprintf("%d %s: unknown sys call %d\n",
            proc->pid, proc->name, num);
    proc->tf->eax = -1;
  }
}
```

## Part2 日期系统调用

### 任务
> 添加一个新的系统调用`date`,其获取当前的UTC时间并将其返回给用户程序.完成后,在xv6 shell提示符下输入`date`可打印当前的UTC时间.

### 实现
> 使用帮助函数`cmostime()`(定义在`lapic.c`)来读取实时时钟,然后根据`date.h`中定义的struct rtcdate结构体,作为一指针参数传递给`cmostime()`.
查看`uptime`系统调用的实现如下
``` c
grep -n uptime *.[chS]
#output
syscall.c: extern int sys_uptime(void);
syscall.c:[SYS_uptime] sys_uptime,
syscall.c:[SYS_uptime] "uptime",
syscall.h:#define SYS_uptime 14
sysproc.c:sys_uptime(void)
user.h:int uptime(void);
usys.S:SYSCALL(uptime)
```
> 归纳出添加系统调用的一般步骤:
* 在`syscall.h`中添加系统调用号
* 在`user.h`中添加用户态函数的定义
* 在`usys.S`中添加用户态函数的实现
* 在`syscall.c`中添加系统调用函数的外部声明
* 在`sysproc.c`中添加系统调用函数的实现

> 'date'系统调用函数具体实现代码：
``` c
int
sys_date(void)
{
  struct rtcdate *r;

  if (argptr(0, (void *)&r, sizeof(*r)) < 0)
          return -1;
  cmostime(r);
  return 0;
}
```
