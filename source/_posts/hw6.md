title: hw6-threads-and-locking
date: 2017/04/30
tags:
	- os
	- xv6

---
# hw6 threads and locking
## 目标
在多线程OS中，通常是在一个进程中包括多个线程，每个线程都是作为利用CPU的基本单位，是花费最小开销的实体.OS中实现进程和线程提高系统的并行性，hw6探索使用线程和锁来并行编程Hash表.

## 准备
hw6在多核硬件的os上即可编译运行，不依赖于课程的xv6.
``` bash
wget https://pdos.csail.mit.edu/6.828/2016/homework/ph.c
gcc -g -02 ph.c -pthread
```
源程序（`ph.c`)如下：
``` c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <assert.h>
#include <pthread.h>
#include <sys/time.h>

#define SOL
#define NBUCKET 5
#define NKEYS 100000

struct entry {
  int key;
  int value;
  struct entry *next;
};
struct entry *table[NBUCKET];
int keys[NKEYS];
int nthread = 1;
volatile int done;


double
now()
{
 struct timeval tv;
 gettimeofday(&tv, 0);
 return tv.tv_sec + tv.tv_usec / 1000000.0;
}

static void
print(void)
{
  int i;
  struct entry *e;
  for (i = 0; i < NBUCKET; i++) {
    printf("%d: ", i);
    for (e = table[i]; e != 0; e = e->next) {
      printf("%d ", e->key);
    }
    printf("\n");
  }
}

static void
insert(int key, int value, struct entry **p, struct entry *n)
{
  struct entry *e = malloc(sizeof(struct entry));
  e->key = key;
  e->value = value;
  e->next = n;
  *p = e;
}

static
void put(int key, int value)
{
  int i = key % NBUCKET;
  insert(key, value, &table[i], table[i]);
}

static struct entry*
get(int key)
{
  struct entry *e = 0;
  for (e = table[key % NBUCKET]; e != 0; e = e->next) {
    if (e->key == key) break;
  }
  return e;
}

static void *
thread(void *xa)
{
  long n = (long) xa;
  int i;
  int b = NKEYS/nthread;
  int k = 0;
  double t1, t0;

  //  printf("b = %d\n", b);
  t0 = now();
  for (i = 0; i < b; i++) {
    // printf("%d: put %d\n", n, b*n+i);
    put(keys[b*n + i], n);
  }
  t1 = now();
  printf("%ld: put time = %f\n", n, t1-t0);

  // Should use pthread_barrier, but MacOS doesn't support it ...
  __sync_fetch_and_add(&done, 1);
  while (done < nthread) ;

  t0 = now();
  for (i = 0; i < NKEYS; i++) {
    struct entry *e = get(keys[i]);
    if (e == 0) k++;
  }
  t1 = now();
  printf("%ld: get time = %f\n", n, t1-t0);
  printf("%ld: %d keys missing\n", n, k);
  return NULL;
}

int
main(int argc, char *argv[])
{
  pthread_t *tha;
  void *value;
  long i;
  double t1, t0;

  if (argc < 2) {
    fprintf(stderr, "%s: %s nthread\n", argv[0], argv[0]);
    exit(-1);
  }
  nthread = atoi(argv[1]);
  tha = malloc(sizeof(pthread_t) * nthread);
  srandom(0);
  assert(NKEYS % nthread == 0);
  for (i = 0; i < NKEYS; i++) {
    keys[i] = random();
  }
  t0 = now();
  for(i = 0; i < nthread; i++) {
    assert(pthread_create(&tha[i], NULL, thread, (void *) i) == 0);
  }
  for(i = 0; i < nthread; i++) {
    assert(pthread_join(tha[i], &value) == 0);
  }
  t1 = now();
  printf("completion time = %f\n", t1-t0);
}
```

## 并行
在4核机器上双线程运行程序：
``` bash
./a.out 2
# 2：n_threads为在hash表上执行put和get操作的线程数
```
产生如果如下：
```
1: put time = 0.007442
0: put time = 0.007474
1: get time = 6.559389
1: 1 keys missing
0: get time = 6.567974
0: 1. keys missing
completion time = 6.575672
```
> 每个线程分两个阶段运行.在第一阶段`put`，每个线程将`[NKEYS/nthread]`key放入hash表.在第二阶段`get`，每个线程从hash表中获取`NKEYS`.输入结果为每个线程每个阶段花费多长时间，底部的完成时间为应用程序的总运行时间.在上面的输出中，应用程序的完成时间约为6.5秒.

对比单线程，看双线程是否改进了性能：
```
./a.out 1
0: put time = 0.016793
0: get time = 5.447454
0: 0 keys missing
completion time = 5.464499
```
> 单线程情况（5.5s）的完成时间略小于双线程情况（6.5s），增加的运行时间应该在消耗线程切换上。但双线程在`get`阶段的总工作量是单线程的两倍.因此，双线程在两个内核上实现了两倍的并行加速，效果很好.`put`阶段实现了一些加速; 双线程并行地插入相同数量的key，比单线程多一倍.另外，还有一个问题；`1 keys missing`说明在双线程运行中，程序在阶段2找不到的阶段1中插入1个键.

在4核机器上运行结果如下：
```
23: put time = 0.014067
1: put time = 0.014450
2: put time = 0.014245
0: put time = 0.015714
3: get time = 6.202647
3: 45 keys missing
0: get time = 6.211748
0: 45 keys missing
2: get time = 6.287174
2: 45 keys missing
1: get time = 6.306257
1: 45 keys missing
completion time = 6.322159
```
> 4线程完成时间与双线程大致相同，但是运行时间是双线程的两倍，实现了良好的并行性.但也发现缺失了更多的key的问题，相比较于单线程不会出现缺少key的问题.

## 问题解决
### key缺失原因
为什么当两个或多个线程同时运行时，key开始丢失？考虑这样一种情况：假设线程A和B同时运行。如果两个线程同时插入同一个bucket的key，则可能会发生竞争。下面的事件概述将导致这样的结果的操作。
```
t-A: calls insert() on bucket 1 (key = 6, 6 % NBUCKETS = 1)

t-B: calls insert() on bucket 1 (key = 21, 21 % NBUCKETS = 1)

...

t-A: executes e->next = n

t-B: executes e->next = n

t-A: executes *p = e

t-B: executes *p = e
```
这样做的结果是执行的最后一个线程B有效地`*p = e`覆盖了前一个线程A的动作并设置了新的列表的头指针.按照上面的执行顺序，假设插入线程A的键6将会丢失，因为线程B覆盖了它。
### **锁机制**
为了避免上述事件的发生，即线程同时访问同一个资源，采用**锁机制**，给`put`和`get`加锁，实现线程的**互斥**.linux `pthread.h`提供的锁机制如下：
``` c
pthread_mutex_t lock;     // declare a lock
pthread_mutex_init(&lock, NULL);   // initialize the lock
pthread_mutex_lock(&lock);  // acquire lock
pthread_mutex_unlock(&lock);  // release lock
```
首先想到的是给每一个`put`和`get`操作加锁，但发现这种结果导致多线程变成了线性运行，使得并行性降为0.然后发现`get`实际是读操作，不会对hash表进行写操作，在经典的读者-写着进程互斥问题中学到，读操作无需加锁，允许多个读操作并行执行，这样就在保证不出现key缺失的情况下提高了并行性。最后考虑`put`操作是对整个hash表加锁，实际上只需要给`put`操作访问的bucket加锁，而不是全局加锁，这使得其他`put`操作可以同时执行bucket的插入进一步提高了并行性。运行效果如下：
```
0: put time = 0.022744
1: put time = 0.022521
1: get time = 6.309178
1: 0 keys missing
0: get time = 6.323614
0: 0 keys missing
completion time = 6.350177
```
`ph.c`修改结果如下：
``` c
...
int keys[NKEYS];
pthread_mutex_t bucket_locks[NBUCKET];
...
static
void put(int key, int value)
{
  int i = key % NBUCKET;
  pthread_mutex_lock(&bucket_locks[i]);
  insert(key, value, &table[i], table[i]);
  pthread_mutex_unlock(&bucket_locks[i]);
}
...
int
main(int argc, char *argv[])
{
  pthread_t *tha;
  void *value;
  long i;
  double t1, t0;

  if (argc < 2) {
    fprintf(stderr, "%s: %s nthread\n", argv[0], argv[0]);
    exit(-1);
  }
  nthread = atoi(argv[1]);
  tha = malloc(sizeof(pthread_t) * nthread);
  srandom(0);
  assert(NKEYS % nthread == 0);
  for (i = 0; i < NKEYS; i++) {
    keys[i] = random();
  }

  //init lock for every bucket
  for (i = 0; i < NBUCKET; i++) {
    pthread_mutex_init(&bucket_locks[i], NULL);
  }

  t0 = now();
  for(i = 0; i < nthread; i++) {
    assert(pthread_create(&tha[i], NULL, thread, (void *) i) == 0);
  }
  for(i = 0; i < nthread; i++) {
    assert(pthread_join(tha[i], &value) == 0);
  }
  t1 = now();
  printf("completion time = %f\n", t1-t0);
}
```
