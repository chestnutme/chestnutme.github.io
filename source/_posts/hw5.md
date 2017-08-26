title: hw5-cpu-alarm
date: 2017/04/24
tags:
	- os
	- xv6

---
# hw5 cpu alarm
## 目标
> 这次hw向xv6添加一个功能，以便在进程使用CPU时,定时提醒进程.这对计算敏感的进程很有帮助，限制他们CPU的使用时间，也让进程在计算的同时执行一些周期性任务.更通用来说来说，我们将实现1个用户级别的中断异常处理.
在这里会用到上次系统调用的实现机制.即增加1个`alarm(interval, handler)`系统调用.当1个应用调用`alarm(n, fn)`时，那么每隔n个CPU时钟节拍，内核将使应用调用`fn`函数.当`fn`函数返回时，应用从调用地址重新开始执行.

测试程序`alarmtest.c`

<!-- more -->

``` c
#include "types.h"
#include "stat.h"
#include "user.h"

void periodic();

int
main(int argc, char *argv[])
{
  int i;
  printf(1, "alarmtest starting\n");
  alarm(10, periodic);
  for(i = 0; i < 50*500000; i++){
    if((i++ % 500000) == 0)
      write(2, ".", 1);
  }
  exit();
}

void
periodic()
{
  printf(1, "alarm!\n");
}
```
其调用`alarm(10, periodic)`, 使得内核每隔10个tick调用一次`periodic`.输出形式如下:
``` bash
$alarmtest
alarmtest starting
.....alarm!
....alarm!
.....alarm!
......alarm!
.....alarm!
....alarm!
....alarm!
......alarm!
.....alarm!
...alarm!
...$
```

### 实现

实现步骤：

> 1.参照[hw3 system call](https://chestnutme.github.io/2017/04/19/hw3/),按照添加系统调用的方式添加用户态调用程序、系统调用号和系统调用程序
``` c
//user.h
int alarm(int ticks, void(*hander)());
//usys.S
SYSCALL(alarm)
//Makefile    UPROGS:
_alarmtest\
//syscall.h
#define SYS_alarm 23
//syscall.c
extern int sys_alarm(void);
//syscall.c syscalls[]
[SYS_alarm]   sys_alarm,
//sysproc.c
int
    sys_alarm(void)
    {
      int ticks;
      void (*handler)();

      if(argint(0, &ticks) < 0)
        return -1;
      if(argptr(1, (char**)&handler, 1) < 0)
        return -1;
      proc->alarmticks = ticks;
      proc->alarmhandler = handler;
      return 0;
    }
```


> 2.在`proc.c`的`proc`结构体中添加变量记录总`ticks`、当前`ticks`和对应的`handler`
``` c
//struct proc
int alarmticks;
int curalarmticks;
void (*alarmhander)();
```

> 3.在`trap.c`中处理时钟中断添加`handler`
``` c
case T_IRQ0 + IRQ_TIMER:
    if(cpu->id == 0){
       acquire(&tickslock);
       ticks++;

      wakeup(&ticks);
      release(&tickslock);
    }
      if(proc && (tf->cs & 3) == 3){
        proc->curalarmtick++;
        if(proc->alarmticks == proc->curalarmtick){  // tick到达了周期
          proc->curalarmtick = 0;

          //将eip压栈
          tf->esp -= 4;    
          *((uint *)(tf->esp)) = tf->eip;
          // 拷贝alarmhandler给eip，准备执行
          tf->eip =(uint) proc->alarmhandler;
        }
      }
    lapiceoi();
    break;
````
