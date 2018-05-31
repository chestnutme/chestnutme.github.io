---
title: go internal - memory model
tags: go
date: 2018/05/31
update: 2018/05/31
---
## what is memory model
From Wiki[3], the memory model is defined as follows:
> In computing, a memory model describes the interactions of threads through memory and their shared use of the data.

The context is **multi-thread**, which is sth about concurrency/synchronization.

At the time when a computer only has one core/cpu, memory is owned by the single thread, in which instructions has a well-defined order. As time goes by, we have multi-cpus/multi-cores-one-cpu allowing us to executing more than one thread at the same time.The problem is that we only have one memory, and the set of threads are always cooperating with each other to cope with a whole task instead of being independent.They need communication(known as **IPC: inter-process communicaiton**), which have two methods:
* share memory
* message passing

The latter is used in the network/distributed system, we will leave it aside.The formmer defines a n-to-1 model, which we need to deal with the logic of the program - the execution order of threads happening at the same time.That's when we need the memory model:

<!-- more -->

## memory model .vs. memory layout
Notice that memory model is different from the notion of memory layout.
The memory layout defines when a program is running, how the parts of the process is laid in the memory. 2 examples:
C memory layout[7]
![C memory layout](/images/imgs/memoryLayoutC.jpg)

Java memory architecture[8]
![Jave mem arc](/images/imgs/javaMemArc.png)


## golang memory:

### atomic operation
* a instruction can't be partitioned in mechine-level, which is the smallest element of a program.
* The atomic operation is low-level concepts, which means most statements we code in high-level language is not atomic.Don't take is for granted. eg: `a += 1` will be decomposed as `load a;a+1;store a`
* a atomic operation is not a instantaneous operation, it lasts for a while. As a result, we can regard it as a operation with a start point and a end point, which need some time to complete. 


### happens before
> happens before
> a partial order on the execution of memory operations in a Go program, to specify the requirements of reads and writes, we define happens before, .
>  If event e1 happens before event e2, then we say that e2 happens after e1.
>   Also, if e1 does not happen before e2 and does not happen after e2, then we say that e1 and e2 happen concurrently

* single goroutine
*Within a single goroutine, the happens-before order is the order expressed by the program.*

* multi goroutine/concurrency
> memory visibility
A read r of a variable v is allowed to observe a write w to v if both of the following hold:
	* r does not happen before w.
	* There is no other write w' to v that happens after w but before r.
To guarantee that a read r of a variable v observes a particular write w to v, ensure that w is the only write r is allowed to observe. That is, r is guaranteed to observe w if both of the following hold:
	* w happens before r.
	* Any other write to the shared variable v either happens before w or after r.
This pair of conditions is stronger than the first pair; it requires that there are no other writes happening concurrently with w or r.

 *When multiple goroutines access a shared variable v, they must use synchronization events to establish happens-before conditions that ensure reads observe the desired writes.*
 
2 point need notice:
* The initialization of variable v with the zero value for v's type behaves as a write in the memory model.
* Reads and writes of values larger than a single machine word behave as multiple machine-word-sized operations in an unspecified order.
 
 
 
### synchronization
 
 
#### `init()`
*If a package p imports package q, the completion of q's init functions **happens before** the start of any of p's.*

#### goroutine creation
*The `go` statement that starts a new goroutine **happens before** the goroutine's execution begins.*

#### goroutine destruction
*The exit of a goroutine is not guaranteed to happen before any event in the program.*


#### channel
4 rules
* *A send on a channel **happens before** the corresponding receive from that channel completes.*
* *The closing of a channel **happens before** a receive that returns a zero value because the channel is closed.*
* *A receive from an unbuffered channel **happens before** the send on that channel completes.*
* *The kth receive on a channel with capacity C **happens before** the k+Cth send from that channel completes.*


synced goroutine: `a = "hello, world"` - `<-c`-`print(a)`
```go
var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world"
	c <- 0
}

func main() {
	go f()
	<-c
	print(a)
}
```

bufferd channel implements a semphone:
```go
var limit = make(chan int, 3)

func main() {
	for _, w := range work {
		go func(w func()) {
			limit <- 1
			w()
			<-limit
		}(w)
	}
	select{}
}
```

#### lock
* *For any sync.Mutex or sync.RWMutex variable l and n < m, call n of l.Unlock() **happens before** call m of l.Lock() returns.*

* *For any call to l.RLock on a sync.RWMutex variable l, there is an n such that the l.RLock happens (returns) after call n to l.Unlock and the matching l.RUnlock **happens before** call n+1 to l.Lock.*


#### once
*A single call of f() from once.Do(f) **happens (returns) before** any call of once.Do(f) returns.*


### incorrect sync
All following error can be seen as incorrect implicit sync, don't take for granted. The solution is: **explict synchronizaion**.

#### Error1: 2 goroutine without sync has no happen-before relation
the program may print 2 1 or 2 0 
```go
var a, b int

func f() {
	a = 1
	b = 2
}

func g() {
	print(b)
	print(a)
}

func main() {
	go f()
	g()
}
```

#### Error2: double-checking locking make no happens before relation 
may print nothing
```go
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func doprint() {
	if !done {
		once.Do(setup)
	}
	print(a)
}

func twoprint() {
	go doprint()
	go doprint()
}
```

#### Error3: waiting for a value make no happen before relation
may print nothing or infinite loop
```go
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func main() {
	go setup()
	for !done {
	}
	print(a)
}
```


## Digression
It's in 1978 that 2 papers having a enourmous impact on concurrent/distributed system published. One is **< Time, clocks, and the ordering of events in a distributed system >** written by Leslie Lamport - father of Paxos and Latex, laying the foundation for analysis of dissys. Another is **< Communicating sequential processes>** by CAR Hoare - the creator of `quicksort`, CSP plays a huge role in concurrent system, which also is the theory idea behind goroutine and channel.What a year!

## refer
* 1.[golang : memory model](https://golang.org/ref/mem)
* 2.[go语言内存模型](https://studygolang.com/articles/01176)
* 3.[wiki: memory barrier](https://en.wikipedia.org/wiki/Memory_barrier)
* 4.[wiki: memory model](https://www.quora.com/What-is-a-memory-model-in-C)
* 5.[quora: what is memory model in C](https://www.quora.com/What-is-a-memory-model-in-C)
* 6.[lec: memory model](https://www.cs.princeton.edu/courses/archive/fall10/cos597C/docs/memory-models.pdf)
*  7.[Memory Layout of C Programs](https://www.geeksforgeeks.org/memory-layout-of-c-program/)
*  8.[Memory Architecture Of JVM(Runtime Data Areas)](https://hackthejava.wordpress.com/2015/01/09/memory-architecture-by-jvmruntime-data-areas/)
