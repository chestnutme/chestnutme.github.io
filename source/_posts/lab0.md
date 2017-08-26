title: lab0-build-env
date: 2017/04/08
tags:
 - xv6
 - os

---
# lab 0: 实验环境搭建

### toolchain
> 在进行os实验之前, 要求有前置的基础:
> 1. C: 实现语言, 汇编: 阅读x86_64的汇编代码
> 2. linux: 实验系统环境, 主要是command line下的命令操作
> 3. git: 版本控制与代码提交
> 4. gcc, gdb, id: 编译源代码与调试
> 5. qemu: x86 硬件模拟器, 选择2为Bochs

### toolchain 安装
> 1.linux: 我的环境是linux mint 18.0 serena(基于ubuntu)
> 2.git: linux自带, 配置连接github
> ```bash
git -h
git add/commit/push/checkout
git remote add
```
> 3.gcc, gdb安装
>```bash
sudo apt-get install build-essential
sudo apt-get install gcc-multilib
gcc -h
gdb -h
```
>> 在Linux下已经一套适合6.828课程的工具链， 输入如下命令进行测试
>> ```bash
$ objdump -i
output: line2 elf32-i386
\
$ gcc -m32 -print-libgcc-file-name
output: /usr/lib/gcc/x86_64-linux-gnu/version/32/libgcc.a
```

> 4.qemu 安装
a. 依赖库安装:
`sudo apt-get install libsdl1.2-dev, libtool-bin, libglib2.0-dev, libz-dev, libpixman-1-dev.`
b.由于qemu编译xv6的问题, mit对qemu进行了修改, 用git下载源码:
`git clone https://github.com/geofft/qemu.git -b 6.828-2.3.0`
c.编译qemu
>```bash
cd qemu
./configure --disable-kvm --target-list="i386-softmmu x86_64-softmmu"
make && make install
```

> 5.其它问题
> a. Windows下可采用Cygwin或者virtualbox + ubuntu image, 但更推荐linux

### 参考链接
> 1. [6.828 toolchains](https://pdos.csail.mit.edu/6.828/2016/tools.html)
> 2. [lab guide](https://pdos.csail.mit.edu/6.828/2016/labguide.html)
> 3. [qemu manual](http://wiki.qemu.org/download/qemu-doc.html#pcsys_005fmonitor)
> 4. [gdb manual](http://sourceware.org/gdb/current/onlinedocs/gdb/)
