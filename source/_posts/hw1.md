title: hw1-boot-xv6
date: 2017/04/09
tags: 
	- os
	- xv6

---

# hw1: boot xv6
### 摘要
>   6.828课程包括JOS实验和xv6系统两部分,其中JOS是从xv6中抽取的学习环境,学习xv6(阅读和修改)把握操作系统的全局, 实践JOS(project)则深入os的核心构建,相互辅助.
xv6学习主要通过xv6-book以及编译xv6实现, homework1即是启动xv6系统.

### answer
#### 1. xv6源码
```bash
git clone git://github.com/mit-pdos/xv6-public.git
```
#### 2. 编译
```bash
cd xv6
make
```
生成xv6.img镜像文件, 在qemu上模拟运行.

<!-- more -->

#### 3. 定位_start
```bash
$ nm kernel | grep _start
```
![_start.png](./_start.png)
定位_start的address为0x10000c
接着运行xv6, 在_start打断点,确定xv6启动时的系统信息
双开终端, 在一个终端`make qemu-gdb`, 另一个`gdb`,gdb成功连接上qemu的远程调试后,调试xv6
```
br * 0x10000c
0x10000c:	mov    %cr4,%eax
```
![br_start](./br_start.png)

#### 4. 启动时堆栈上有什么?
根据boot.S、boot.c和boot.asm中的代码，并结合四个问题
> 1.在0x7c00设置断点，单步指令调试，查看堆栈指针$esp在哪里被初始化。
2.单步调试，直到调用bootmain函数，观察堆栈内容。
3.在bootmain函数中第1条汇编指令做了什么。
4.在boot.asm中查找使eip变为0x10000c的指令，观察堆栈的变化。


1.在调用bootmain函数前，esp被初始化为0x7c00，即boot.s的入口。
```asm
boot.asm  mov    $start,  $esp
0x7c40:   mov    $0x7c00, $esp
```
２.在调用bootmain函数时，调用者保存调用地址，即下一条指令，并使$esp-4。
```asm
boot.asm
0x7c45:  call bootmain
spin:
0x7c4a:  jmp spin
```

3.在bootmain函数中，第1条汇编指令`为push     $ebp`，将$ebp压入堆栈，即保存调用者的栈帧。
```asm
boot.asm
0x7d0c:  push $ebp
```
４.当eip变为0x10000c时，bootmain函数将调用地址压入堆栈。 