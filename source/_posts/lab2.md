title: lab2-memeory-management
date: 2017/04/18
tags:
- xv6
- os

---

# Lab2 内存管理
## 摘要
> * 内存管理有两个组件:
> 1.  第一个组件是内核的物理内存分配器，因此内核可以分配内存并稍后释放它。.分配器将以4096个字节为单位进行操作，称为页面。代码要实现维护记录哪些物理页面是空闲的，哪些被分配的数据结构以及共享每个分配的页面的进程数量,另外还将编写例程以分配和释放内存页面。
> 2. 第二个组件是虚拟内存，它将内核和用户软件使用的虚拟地址映射到物理内存中的地址。 当指令使用内存，咨询一组页表时，x86硬件的内存管理单元（MMU）执行映射。 lab2根据提供的规范修改JOS以设置MMU的页表。

> - lab1分为三部分
> 1. part1: 物理页面管理 physical page management
> 2. part2: 虚拟内存 virtual memory
> 3. part3: 内核地址空间 kernel address space

<!-- more -->

## 实验代码
> ```bash
 cd ~/6.828/lab
 git checkout -b lab2 origin/lab2
 git merge lab1
```
> Lab 2 添加了以下文件:
```bash
inc/memlayout.h
kern/pmap.c
kern/pmap.h
kern/kclock.h
kern/kclock.c
```
> `memlayout.h` 描述了虚拟地址空间的结构，通过修改`pmap.c/memlayout.h` 和`pmap.h`来实现`PageInfo`结构，这个是为了记录哪些物理内存的page是空闲的。`kclock.c` 和 `kclock.h` 管理PC的时钟和CMOS RAM硬件，这个设备记录了物理内存的数量。`pmap.c`需要读这个设备来确定内存大小。

## Part 1：Physical Page Management
> 操作系统必须要追踪记录哪些内存区域是空闲的，哪些是被占用的。JOS内核是以页(page)为最小粒度来管理内存的，它使用MMU来映射，保护每一块被分配出去的内存。这里要具体实现一下物理内存页的分配子函数。它利用一个结构体`PageInfo`的链表来记录哪些页是空闲的，链表中每一个结点对应一个物理页。

### Exercise 1.
> 在文件 kern/pmap.c 中，完成以下几个子函数:
> * boot_alloc()
> * mem_init()
> * page_init()
> * page_alloc()
> * page_free()
> * check_page_free_list()和check_page_alloc()两个函数将会检测页分配器代码的正确性.


> 查看`pmap.c`中的代码，其中最重要的函数就是`mem_init`了，在内核刚开始运行时就会调用这个子函数，对整个操作系统的内存管理系统进行一些初始化的设置，比如设定页表等等操作。下面进入这个函数，首先这个函数调用 `i386_detect_memory` 子函数，这个子函数的功能就是检测现在系统中有多少可用的内存空间。
  之前有讲，jos把整个物理内存空间划分成三个部分：
>> 1. 从0x00000~0xA0000，这部分也叫basemem，是可用的。
>> 2. 紧接着是0xA0000~0x100000，这部分叫做IO hole，是不可用的，主要被用来分配给外部设备了。
>> 3. 再紧接着就是0x100000~0x，这部分叫做extmem，是可用的，这是最重要的内存区域。

> 这个子函数中包括三个变量，其中`npages`记录整个内存的页,`npages_basemem`记录basemem的页数，`npages_extmem`记录extmem的页数。
下一条指令为：
> ```c
kern_pgdir = (pde_t *) boot_alloc(PGSIZE);
memset(kern_pgdir, 0, PGSIZE);
```
> 其中`kern_pgdir`是一个指针，`pde_t *kern_pgdir`，它是指向操作系统的页目录表的指针，操作系统之后工作在虚拟内存模式下时，就需要这个页目录表进行地址转换。我们为这个页目录表分配的内存大小空间为`PGSIZE`，即一个页的大小。并且首先把这部分内存清0。

> 通过查看 `mem_init` 函数可以知道，`boot_alloc` 是用来初始化页目录(page directory)。在 `boot_alloc` 中，`nextfree` 为下一个空闲内存的虚拟内存地址，当 `nextfree` 为空时会先初始化。用到了`ROUNDUP`，这个`ROUNDUP`在 ``/inc/types.h` 中，因为内存区块是对齐的，所以每块都是固定的大小。npages 是页数量，可使用的内存大小是 `npages × PGSIZE` ，根据lab1提到的，KERNBASE是分配内存的起始地址，若`nextfree` 大于 `KERNBASE + npages × PGSIZE` 的值，就是指针地址溢出了。
所以只需要添加上这部分代码:
> ```c
result = nextfree;
nextfree = ROUNDUP(nextfree+n, PGSIZE);
if((uint32_t)nextfree > KERNBASE + (npages * PGSIZE)) {
    panic("Out of memory!\n");
}
```
> `mem_init()`在执行完上面的函数以后，会给`kern_pgdir`加上权限位。之后就是要初始化所有的`struct PageInfo`为 0。首先确定`PageInfo`的大小，然后用`boot_alloc()`分配内存，接着用`memset()`初始化。
> ```c
size_t PageInfo_size = sizeof(struct PageInfo);
pages = (struct PageInfo *)boot_alloc(npages * PageInfo_size);
memset(pages, 0, npages * PageInfo_size);
```

> 接着调用`page_init()`来初始化`page`结构和内存空闲链表。
`page_init()`，这个子函数的功能包括：
>> 1. 初始化pages数组
>> 2. 初始化pages_free_list链表，这个数组中存放着所有空闲页的信息

> 可以到这个函数的定义处具体查看，整个函数是由一个for循环构成，它会遍历所有内存页所对应的在`npages`数组中的`PageInfo`结构体，并且根据这个页当前的状态来修改这个结构体的状态，如果页已被占用，那么要把`PageInfo`结构体中的`pp_ref`属性置一；如果是空闲页，则要把这个页送入`pages_free_list`链表中。根据注释中的提示，第0页已被占用，`io hole`部分已被占用，还有在extmem区域还有一部分已经被占用，代码如下：
> ```c
size_t i;
page_free_list = NULL;

//num_alloc：在extmem区域已经被占用的页的个数
int num_alloc = ((uint32_t)boot_alloc(0) - KERNBASE) / PGSIZE;
//num_iohole：在io hole区域占用的页数
int num_iohole = 96;

for(i=0; i<npages; i++)
{
    if(i==0)
    {
        pages[i].pp_ref = 1;
    }    
    else if(i >= npages_basemem && i < npages_basemem + num_iohole + num_alloc)
    {
        pages[i].pp_ref = 1;
    }
    else
    {
        pages[i].pp_ref = 0;
        pages[i].pp_link = page_free_list;
        page_free_list = &pages[i];
    }
}
```

> 先实现`page_alloc()`函数，通过注释我们可以知道这个函数的功能就是分配一个物理页。而函数的返回值就是这个物理页所对应的`PageInfo`结构体。
所以这个函数的大致步骤应该是：
>> 1. 从free_page_list中取出一个空闲页的PageInfo结构体
>> 2. 修改free_page_list相关信息，比如修改链表表头
>> 3. 修改取出的空闲页的PageInfo结构体信息，初始化该页的内存

> 代码如下：
> ```c
struct PageInfo *
page_alloc(int alloc_flags)
{
    struct PageInfo *result;
    if (page_free_list == NULL)
        return NULL;

      result= page_free_list;
      page_free_list = result->pp_link;
      result->pp_link = NULL;

    if (alloc_flags & ALLOC_ZERO)
        memset(page2kva(result), 0, PGSIZE);

      return result;
}
```
> `实现page_free()`方法，根据注释可知，这个方法的功能就是把一个页的`PageInfo`结构体再返回给`page_free_list`空闲页链表，代表回收了这个页。
主要完成以下几个操作：
>> 1. 修改被回收的页的PageInfo结构体的相应信息。
>> 2. 把该结构体插入回page_free_list空闲页链表。

> 代码如下：
> ```c
void page_free(struct PageInfo *pp)
{
    // Fill this function in
    // Hint: You may want to panic if pp->pp_ref is nonzero or
    // pp->pp_link is not NULL.
      assert(pp->pp_ref == 0);
      assert(pp->pp_link == NULL);

      pp->pp_link = page_free_list;
      page_free_list = pp;
}
```

## Part 2: Virtual Memory
> 在x86体系中，一个虚拟地址(Virtual Address)是由两部分组成，一个是段选择子(segment selector)，另一个是段内偏移(segment offset)。一个线性地址(Linear Address)指的是通过段地址转换机构把虚拟地址进行转换之后得到的地址。一个物理地址(Physical Addresses)是分页地址转换机构把线性地址进行转换之后得到的真实的内存地址，这个地址将会最终送到你的内存芯片的地址总线上。
我们所编写的C语言程序中的指针的值是虚拟地址中段内偏移部分的值。在boot/boot.S文件中，我们引入了一个全局描述符表，这个表通过把所有的段的基址设置为0，界限设置为0xffffffff的方式，关闭了分段管理的功能。因此虚拟地址中的段选择子字段的内容已经没有任何意义，线性地址的值总是等于虚拟地址中段内偏移的值。
回顾一下lab1中的part 3，我们引入了一个简单的页表，使得内核可以运行与0xf0100000的虚拟地址空间，尽管它所在的真实位置是物理地址0x00100000处，刚刚好在ROM BIOS之上。这个页表仅仅映射了4MB的内存空间。在我们这个JOS操作系统中，我们希望把这种映射扩展到物理内存的头256MB空间上，并且把这部分物理空间映射到从0xf0000000开始的虚拟空间中，以及一些其他的虚拟地址空间中。
### Exercise 2
> 熟悉关于分页地址转换(page translation)和基于页的保护(page-based protection)。
首先介绍一下80386将逻辑地址转为物理地址的方法。
>> * 分段地址转换，由段选择子和段偏移量构成的逻辑地址转为线性地址。
>> * 分页地址转换，线性地址转为物理地址。

#### 区别虚拟地址，线性地址，物理地址
虚拟地址是有段选择子和段偏移构成。线性地址是经过分段地址转换单没进行分页地址转换。物理地址是两种转换之后最终通过硬件总线到RAM的地址。C 指针是虚拟地址的偏移部分。在 `boot/boot.S`中，引入全局描述符表(GDT)将所有段基址设为0到 0xffffffff。因此线性地址等于虚拟地址的偏移量。
```
           Selector  +--------------+         +-----------+
          ---------->|              |         |           |
                     | Segmentation |         |  Paging   |
Software             |              |-------->|           |---------->  RAM
            Offset   |  Mechanism   |         | Mechanism |
          ---------->|              |         |           |
                     +--------------+         +-----------+
            Virtual                   Linear                Physical
```

### Exercise 3`
> 在QEMU中使用 `xp 可以查看物理内存，PD虚拟机对于lab手册上调出QEMU monitor的方法没用，查看
指令为
```bash
qemu-system-i386 -hda obj/kern/kernel.img -monitor stdio -gdb tcp::26000 -D qemu.log。
```

> 在QEMU monitor中使用 `info pg` 查看当前页表， `info mem` 查看虚拟内存的范围。
进入保护模式以后，所有地址引用都是虚拟地址，由MMU转换，也就是说 C 指针都是虚拟地址。JOS内核经常需要操作地址通过整数而不解引用。JOS为了区分两种情况：类型 `uintptr_t` 代表虚拟地址，`physaddr_t` 代表物理地址。虽然都是32位整数，但是不能直接解引用，需要先转换类型。JOS需要读取或修改内存，尽管只知道物理地址。给页表添加映射需要分配无力内存去储存一个页目录，然后才能初始化内存。然而内核不能绕过虚拟地址转换，因此不能直接加载和储存物理地址。为了将物理地址转为虚拟地址，内核需要在物理地址加上`0xf0000000`从而找到相关的虚拟地址，可以使用`KADDR(pa)`完成这个操作。
同样，如果内核需要通过虚拟地址去找物理地址，就需要减去`0xf0000000`，可以使用`PADDR(va)`完成这个操作。

> 引用计数
之后实验经常需要将多个虚拟地址同时映射到同一块物理页上，因此需要给每一个物理页计数引用次数，这个值位于物理页 `struct PageInfo` 中的 `pp_ref` 字段中。

### Exercise 4
> 实现`kern/pmap.c`里的`pgdir_walk()，boot_map_region()，page_lookup()，page_remove()，page_insert()`这几个函数。`check_page()`会测试是否写的正确。

> 首先是 `pgdir_walk()`，参考注释可以得知，这个函数获得指向线性地址页表项的指针，传入的参数是页目录指针，线性地址和另外一个参数。
![page](http://xinqiu.me/2016/12/09/MIT-6.828-2/2.png)
> 由上面的图可以知道二级分页模式下线性地址到物理地址的转换。所以首先要获得页目录地址，判断是否指向的页表项存在，不存在则新建一个页表。这里有两个注意点。
>> * 第一个地方是要注意判断页表是否存在，根据下图页目录/表的结构，可以知道这里的P位代表Present，用来判断对应的物理页是否存在，存在则为1，所以通过与运算来判断。
>> * 另外一个注意点是新建页。为新建的物理页设置页目录时，需要添加上权限位

> 代码如下:
> ```c
pte_t *
pgdir_walk(pde_t *pgdir, const void *va, int create)
{
    // Fill this function in
    pde_t *pt = pgdir + PDX(va);
    pde_t *pt_addr_v;

    if (*pt & PTE_P) {
        pt_addr_v = (pte_t *)KADDR(PTE_ADDR(*pt));
        return pt_addr_v + PTX(va);
    } else {
        struct PageInfo *newpt;
        if (create == 1 && (newpt = page_alloc(ALLOC_ZERO)) != 0) {
            memset(page2kva(newpt), 0, PGSIZE);
            newpt->pp_ref ++;
            *pt = PADDR(page2kva(newpt))|PTE_U|PTE_W|PTE_P;
            pt_addr_v = (pte_t *)KADDR(PTE_ADDR(*pt));
            return pt_addr_v + PTX(va);
        }
    }
    return NULL;
}
```

> 接着是`boot_map_region`函数，这个函数将虚拟地址`[va, va+size)`映射到物理地址`[pa, pa+size)`，注释中提到可以使用上面写的`pgdir_walk`，获取页表地址，接着将物理地址的值与上权限位赋给页表地址。
```c
static void
boot_map_region(pde_t *pgdir, uintptr_t va, size_t size, physaddr_t pa, int perm)
{
    // Fill this function in
    int offset;
    pte_t *pt;
    for (offset = 0; offset < size; offset += PGSIZE) {
        pt = pgdir_walk(pgdir, (void *)va, 1);
        *pt = pa|perm|PTE_P;
        pa += PGSIZE;
        va += PGSIZE;
    }
}
```

> 之后是`page_lookup`函数，查找线性地址va对应的物理页面，找到就返回这个物理页，否则返回`NULL`。首先如果`pte_store`非0，则储存这个页的页表地址，这一步是为了之后的`page_remove`用的。
> ```c
struct PageInfo *
page_lookup(pde_t *pgdir, void *va, pte_t **pte_store)
{
    // Fill this function in
    pte_t *pte = pgdir_walk(pgdir, va, 0);
    if (pte_store != 0) {
        *pte_store = pte;
    }
    if (pte != NULL && (*pte & PTE_P)) {
        return pa2page(PTE_ADDR(*pte));
    }
    return NULL;
}
```

> `page_remove`实现参考注释里的提示，先通过`page_lookup`获得物理页，如果存在则执行删除工作`page_decref`，同时也要将`va`地址的页表项设为0，最后就是验证有效性。
> ```c
void
page_remove(pde_t *pgdir, void *va)
{
    // Fill this function in
    pte_t *pte;
    struct PageInfo *page = page_lookup(pgdir, va, &pte);
    if (page) {
        page_decref(page);
        *pte = 0;
        tlb_invalidate(pgdir, va);
    }
}
```
> 最后一步就是`page_insert`函数，将页面管理结构 `pp` 所对应的物理页面分配给线性地址 `va`。同时,将对应的页表项的 `permission` 设置成 `PTE_P&perm`。 注意:一定要考虑到线性地址 `va` 已经指向了另外一个物理页面或者干脆就是这个函数要指向的物理页面的情况。如果线性地址 `va` 已经指向了另外一个物理页面,则先要调用 `page_remove` 将该物理页从线性地址 `va` 处删除,再将 `va` 对应的页表项的地址赋值为 `pp` 对应 的物理页面。如果 `va` 指向的本来就是参数 `pp` 所对应的物理页面,则将 `va` 对应的页表项中 的物理地址赋值重新赋值为 `pp` 所对应的物理页面的首地址即可。
> ```c
int
page_insert(pde_t *pgdir, struct PageInfo *pp, void *va, int perm)
{
    // Fill this function in
    pte_t *pte = pgdir_walk(pgdir, va, 1);
    if (!pte) {
        return -E_NO_MEM;
    }  
    if (*pte & PTE_P) {
        if (PTE_ADDR(*pte) == page2pa(pp)) {
            tlb_invalidate(pgdir, va);
            pp->pp_ref--;
        }
        else {
            page_remove(pgdir, va);
        }
    }
    *pte = page2pa(pp) | perm | PTE_P;
    pp->pp_ref++;
    pgdir[PDX(va)] |= perm;
    return 0;
}
```

## Part 3: 内核地址空间

#### 线性地址的两部分
> JOS将处理器32位线性地址划分为占低地址的用户环境(进程)和占高地址的内核。 分界线是`inc/memlayout.h`中的变量 `ULIM`。内核保留了大约256MB的虚拟地址空间，lab1中内核设在那么高的地址就是因为要留一部分空间给用户环境。

#### 访问权限和故障隔离
> 内核和用户内存都在各自的环境地址空间中，必须在x86页表中使用访问权限位(`Permissions bits`)来使用户代码只访问用户的地址空间，否则用户的代码bug会覆盖内核数据，造成系统崩溃。值得注意的是可写权限位(`PTE_W`)可以同时影响用户和内核代码。
高于`ULIM`的内存内核可以读写，而用户环境没有权限。内核和用户在地址`[UTOP,ULIM)`有同样的权限：可读但不可写，这部分地址空间通常是一些特定的内核数据，让用户环境可以读取。最后，地址`UTOP`之下的是用户环境。

#### 初始化内核地址空间
> 设置`UTOP`之上的地址空间。在`inc/memlayout.h`中显示了布局。

#### Exercise 5
> 完成`mem_init()`中缺少的部分。
> 因为`mem_init`开头创建了初始化页目录`kern_pgdir`，首先是将`pages`数组映射到线性地址`UPAGES`，权限是内核只读。
> ```c
boot_map_region(kern_pgdir, UPAGES, PTSIZE, PADDR(pages),PTE_U);`
```
> 接着是映射物理地址到内核栈，也就是从地址范围`[KSTACKTOP-KSTKSIZE, KSTACKTOP)`映射到`bootstack`开始的物理地址页上，注释中提到了，只要映射`[KSTACKTOP-KSTKSIZE, KSTACKTOP)`， `[KSTACKTOP-PTSIZE, KSTACKTOP-KSTKSIZE)`不映射，权限位是内核读写。
> ```c
boot_map_region(kern_pgdir, KSTACKTOP-KSTKSIZE, KSTKSIZE, PADDR(bootstack), PTE_W);
```
> 最后是映射虚拟地址`[KERNBASE, 2^32)`到物理地址`[0, 2^32 - KERNBASE)`，权限位是内核读写。
> ```c
boot_map_region(kern_pgdir, KERNBASE, (0xffffffff-KERNBASE), 0, PTE_W);
```

## 参考链接
[memory management](https://pdos.csail.mit.edu/6.828/2016/labs/lab2/)
