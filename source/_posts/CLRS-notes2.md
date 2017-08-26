title: introduction to algorithms note2
tags:
  - algorithm
  - data_structure
date: 2017/04/17

---

# ＜Introduction to algorithms＞笔记

### Part3: 数据结构
> 1. 动态集合:随着时间改变而变化
> 2. 关键字(keyword) -> 全序集(排序)
> 3. 基本操作
>> * `search(S, k)`
>> * `insert(S, x)`
>> * `delete(S, x)`
>> * `maximum/minimum(S)`
>> * `successor/predecessor(S, x)`

<!-- more -->

#### Chapter10 基本数据结构实现动态集合?
1. stack
> - LIFO: 后进先出
> - data
>> ```
Sequence S; #可由数组或链表实现
int top;
```
> - method
>> ```
stack-empty(S): O(1)
  if top[S] = 0
    then return true
    else return false

push(S, x): O(1)
  top[S] <- top[S] + 1
  S[top[S]] <- x

pop(S): O(1)
   if stack-empty(S)
    then error "underflow"
    else top[S] <- top[S] - 1
      return S[top[S] + 1]

#若考虑实现stack的底层数据结构是静态的, 即栈容量有限,可能会上溢
stack-full(S): O(1)
 if top[S] == size(S)
    then return true
    else return false

```
> - apply:
>> 函数调用-递归
>> 括号匹配与表达式求值

2. queue
在第6章排序中由heap实现了priority_queue, 是特殊一类queue:在queue上实现了优先级.基本的queue:
> - FIFO: 先进先出
> - data:
>> ```
Sequence S;
int head, tail;
int length;
```
> - method
>> ```
enqueue(Q, x): O(1)
  Q[tail[Q]] <- x
  if tail[Q] = length[Q]
    then tail[Q] <- 1
    else tail[Q] <- tail[Q] + 1

dequeue(Q): O(1)
  x <- Q[head[Q]]
  if head[Q] = length[Q]
    then head[Q] <- 1
    else head[Q] <- head[Q] + 1
  return x
>> ```
> - apply:
>> * FIFO算法; 先来先服务
* 层次算法: 层次遍历, BFS

3. linkedlist
与数组的线性序是由下标决定(连续储存, 随机访问)不同, 链表的顺序由指针确定(易动态扩充)
> - data:
>> ```
Datatype data
pointer next, prev
```
> - method:
```
search(L, k): O(n)
  x <- head[L]
  while x  != NULL and key[x] != k
    do x <- next[x]
  return x

insert(L, x): O(1) 头插法
  next[x] <- head[L]
  if head[L] != NULL
    then prev[head[L]] <- x
  head[L] <- x
  prev[x] <- NULL

delete(L, x): x is pointer, O(1); x is key, O(n)
  if prev[x] != NULL
    then next[prev[x]] <- next[x]
    else head[L] <- next[x]
  if next[x] != NULL
    then prev[next[x]] <- prev[x]
```
> - techique: 哨兵(sentinel)
>> - 简化了边界条件, 代码实现更简洁,紧凑
>> ```
search(L, k)
  x <- next[nil[L]]
  while x != nil[L] and key[x] != k
    do x <- next[x]
  return x

insert(L, x)
  next[x] <- next[nil[L]]
  prev[next[nil[L]]] <- x
  next[nil[L]] <- x
  prev[x] <- nil[L]

delete(L, x)
  next[prev[x]] <- next[x]
  prev[next[x]] <- prev[x]
```
> - apply:
数组和链表作为两种基本的线性结构, 两者实现的功能基本相同, 但各有优劣:数组由下标实现随机访问, 可以实现二分,快排等高效算法, 但容量受限,动态增删复杂度高;链表只能线性遍历, 查找排序效率低, 但由于指针特性,易于动态增删, 常用于管理动态的内容.

4. 静态数组和分配释放
在某些编程语言(如fortran)中没有指针的概念, 可以添加2个域储存prev, next对象的下标,模拟指针的功能.
另外,动态增删元素时要向内存申请或释放内存, 一种机制是添加tag, 但最后销毁整个数据结构时处理, 一种则是C中的malloc/free机制,实现内存的分配和回收.

5. 有根树
将前驱后置的思想推广到任意同构的数据结构上, 自然导出了链接实现的树结构上, 这节只讨论树的基本结构:
> - binary tree
>> - node:
>> ```
data
pointer left, right
```

>> - multi tree
>> - node:
>>> * data:
>>> * pointer left-child, right-sibling #左孩子右兄弟表示法

#### Chapter10 如何实现映射关系的数据结构?
1. hash table: key ->(mapto) identifier
> 　在散列表中查找一个元素的时间与在链表中查找一个元素的时候相同,在最坏情况为 $O(n)$,但期望时间为 $O(1)$.在实践中,散列表的效率是很高的,一般可认为是$O(1)$, 基本的字典操作只需要$O(1)$的平均时间.
当待排序的关键字集合是静态的(即当关键字集合一旦存入后不再改变),"完全散列"能够在$O(1)$的最坏情况时间内支持关键字查找。当待排序的关键字的集合是静态的(即当关键字集合一旦存入后不需要再改变),“完全散列”能够在 O(1)的最坏时间内支持查找操作。

２. 散列方法
> 1. 直接寻址
  适用于关键字个数小于可能的关键字总数(一般是表长)
  但如果关键字的全集U很大,但实际关键字集合k很小, 则会造成大量空间的浪费.
> ```
search(T, k)
  return T[k]

insert(T, x)
  T[key[x]] <- x

delete(T, x)
  T[key[x]] <- NULL
```

> 2. 散列表
关键字集合k映射到长为m的散列表T上,即:
\begin{align}
hash-func: h: U -> \{0, 1, \ldots ,m-1\} \\\\
即 k \mapsto h(k)
\end{align}
问题: 碰撞(collision)two key -> one position
> 解决碰撞的方法:
>> 1. 好的hash-func, 尽可能减少碰撞
>> 2. 链接法(chaining): 散列到同一位置的存入链表中
>> ```
search(T, x): O(1)
  search for an element with key in list T[h(k)]

insert(T, x): O(1)
  insert x at the head of list T[h(key[x])]

delete(T, x): O(1)
  delete x from the list T[h(key[x])]
```
在众多的简单的解决碰撞的方法中,我觉得比较好的是通过链表法解决碰撞,虽然这个方法的理论最坏效率为 O(n),但是在平均情况下,它的性能也是非常好的,实现简单又高效。


3. hash-func
> 1. 好的hash-func
\begin{align}
对random \, k \, 0 \leq k < 1,\\\\
h(k) = \lfloor km \rfloor
\end{align}

> 2. 多数的散列函数都假定关键字域为自然数集$N$,如果所给关键字不是自然数,则必须有一种方法来将它们解释为自然数。
>> a. 除法散列法:
一般选取 m 的值为与2的整数幂不大接近的质数
>> $$h(k) = k\mod m$$
>> b. 乘法散列法:
构造散列函数的乘法方法包含两个步骤:首先用关键字剩上常数 A(0<A<1),并抽取 kA 的小数部分;然后用m剩以这个值,再取结果的底。
>> $$h(k) = \lfloor m(kA \mod 1) \rfloor$$
>> c.全域散列:全域散列的基本思想是在执行开始时,就从一族仔细设计的函数中,随机地选择一个作为散列函数。
i. 全域散列表是一种使用“链接法”来解决碰撞问题的散列表方法。
ii. 随机化保证了对于任何输入,算法都具有较好的平均性能。
iii. 全域的散列函数组:设$H$为一组散列函数,它将给定的关键字域$U$映射到{0,1,...,m-1}中,这样的一个函数组称为是全域的.如果从$H$中随机地选择一个散列函数,当关键字K≠J时,两者发生碰撞的概率不大于1/m。
iv.常用的一个全域散列函数类:
首先选择一个足够大的质数p,使得每一个可能的关键字k都落到0到p-1的范围内,包括首尾的0和p-1.这里我们假设全域是0–15, p为17。设集合Zp {0, 1, 2, ,...,p-1},集合Zp'为{1, 2, 3,...,p-1}。由于 p 是质数,
我们可以定义散列函数
$h(a, b, k) = ((a*k + b) \mod p) \mod m$
其中a属于Zp,b属于Zp'。由所有这样的a和b构成的散列函数,组成了函数簇,即全域散列。
v.明白这个散列函数的选取是在“执行开始”随机的选取一个是很重要的,要不然就会不明白到时候怎么进行查找。这里所谓的随机性应该这样理解:对于某一个散列表来说,它在初始化时已经把a,b固定了,但是对于一个还未初始化的全域散列表来说,a,b是随机选取的。

4. 开放寻址法:所有的元素都放在散列表里
> * 开放寻址法的好处就在于它根本不用指针,而是计算出要存取的各个pos。这样一来,由于不用存储指针就节省了空间,从而可以用同样的空间来提供更多的pos,其潜在的效果就是可以减少碰撞,提高查找速度。
> * 开放寻址法很像一个启发式的搜索,它的最坏性能也是 O(n),只不过散列函数为它提供了启发信息从而使得一般的平均性能会很好。
> * 在开放寻址法中,对散列元素的删除操作执行起来比较困难,因为删除操作会影响查找操作。解决办法是在pos里的值,被删除后置一个特定的值tag标记删除,而不是直接删除导致查找断开.
> * 探查法
线性探查、二次探查和双重散列都是对最基本的数组法的改进.
>> 1. 线性探查 linear probing
>> $$h(k,i) = (h'(k)+i) \mod m$$
>> $$h' = U \mapsto [0, 1, \ldots, m-1]$$
>> 2. 二次探查 quadrating probing
>> $$h(k,i) = (h'(k) + c_1i+ c_2i^2) \mod m$$
>> 3. 双重探查
双重散列是用于开放寻址的最好方法之一,因为它所产生的排列具有随机选择的排列的许多特性.
>> $$h(k,i) = (h_1(k) + ih_2(k)) \mod m$$


5. 完全散列:
> 如果某一种散列技术在进行查找时,其最坏情况内存访问次数为$O(1)$的话,则称其为完全散列。书上使用了一种两级的散列方案,每一级都采用全域散列.通常利用一种两级的散列方案,每一级上都采用全域散列.
完全散列的关键在于:二次散列表中要求没有碰撞.这是通过确保槽的个数是关键字的个数的平方来实现的。

#### Chapter12 如何用树结构实现动态集合?
1. 二叉查找树 binary search tree
> * define:
> $$key[left[x]] \leq key[x] \bigcup key[right[x]] \geq key[x]$$
> * method
> ```
inorder-walk(x): $O(n)$
  if x != NULL
    then inorder-walk(left[x])
      visit key[x]
      inorder-walk[right[x]]

search(x, k): $O(h)$
  if x = NULL or k = key[x]
    then return x
  if k < key[x]
    then return search(left[x], k)
    else return search(right[x], k)

minimum(x): $O(h)$
  while left[x] != NULL
    do x <- left[x]
  return x

maximum(x): $O(h)$
  while right[x] != NULL
    do x <- right[x]
  return x

successor(x): $O(h)$, 中序后继
  if right[x] != NULL
    then return minimum(right[x])
  y <- p[x]
  while y != NULL and x = right[y]
    do x <- y
      y <- p[y]
  return y

predecessor(x): $O(h)$ 中序前驱
  if left[x] != NULL
    then return maxinum(left[x])
  y < p[x]
  while y != NUll and x = left[y]
    do x <- y
      y <- p[y]
  return y

insert(T, k): $O(h)$
  y <- NUll
  x <- root[T]
  while x != NUll
    do y <- x
      if key[k] < key[x]
        then x <- left[x]
        else x <- right[x]
  p[x] <- u
  if y = NUll
    then root[T] <- x #tree T is empty
    else if key[k] < key[y]
      then left[y] <- k
      else right[y] <- k

delete(T, k): $O(h)$
  if left[k] = NUll or right[k] = NUll
    then y <- k
    else y <- successor(k)
  if left[y] != NUll
    then x <- left[y]
    else x <- right[y]
  if x != NUll
    then p[x] <- p[y]
  if p[y] = NUll
    then root[T] <- x
    else if y = left[p[y]]
      then left[p[y]] <- x
      else right[p[y]] <- de
  if y != k
    then key[k] <- key[y]
      copy y's satellite date into k
  return y
```
> BST的删除情况较复杂, 有难度的地方就是在删除同时存在左右子树的结点时需要进行处理.理解后可以概括为:对于这样的结点 x,找到 x 结点的前趋(或后继)y,将 x 的值替换为 y 的值,然后递归删除 y 结点就可以了。因为 y 一定没有右子树(后继对应没有左子树),所以递归删除的时候就是很简单的情况了。


#### Chapter13 如何降低树高,提高二叉树的操作效率?
从12章可以看到, BST的性能与树高直接相关.如果BST是单支树,则效率会降到$O(n)$, 如果树较为均匀平衡, 则效率提升到$O(\log n)$.红黑树即是一种自平衡的二叉查找树, 具有很好的时间性能.
> - define
>> * node: color, key, left, right, p
>> * 性质:
>>> 1. 每个结点或是红色,或是是黑色。
>>> 2. 根结点是黑的。
>>> 3. 所有的叶结点(NULL)是黑色的。(NULL 被视为一个哨兵结点,所有应该指向 NULL 的指针,都看成指向了 NULL 结点。)
>>> 4. 如果一个结点是红色的,则它的两个儿子节点都是黑色的。
>>> 5. 对每个结点,从该结点到其子孙结点的所有路径上包含相同数目的黑结点。
>>> 6. 黑高度的定义: 从某个结点出发(不包括该结点)到达一个叶结点的任意一条路径上,黑色结点的个数成为该结点x的黑高度.红黑树的黑高度定义为其根结点的黑高度.
> - 哨兵 nil[T] 代表所有的空节点, 其color=black

> - 性能:
红黑树是一种近似平衡的二叉树),它能保证在最坏的情况下,基本的动态集合操作的时间为 O(lgn)。通过对任何一条从根到叶子的路径上各个结点着色方式的限制,红黑树确保没有一条路径会比其它路径长出两倍,因而是接近平衡的。
注:全黑结点的满二叉树也满足红黑树的定义。满二叉树的效率本身就非常高,它是效率最好的二叉树之一.

> - method
>> *  旋转操作(左旋和右旋):
旋转操作是一种能保持二叉查找树性质的查找树局部操作。
所有对红黑树结构的修改都只能通过左右旋来完成,这样才能保证修改后的红黑树首先是一棵二叉查找树。
```
left-rotate(T, x)
  y <- right[x]
  right[x] <- left[y]
  p[left[y]] <- x
  p[y] <- p[x]
  if p[x] = nil[T]
    then root[t] <- y
    else if x = left[p[x]]
      then ledt[p[x]] <- y
      else right[p[x]] <- y
  left[y] <- y
  p[x] <- y
```
>> * 插入操作:
将结点Z插入树T中,就好像T是一棵普通的二叉查找树一样,然后将Z着为红色。为保证红黑性质能继续保持,我们调用一个辅助程序来对结点重新着色并旋转。插入结点Z的位置的确应该和普通二叉查找树一样,因为红黑树本身就首先是一棵二叉查找树;然后将Z着为红色,是为了保证性质5的正确性,因为性质5如果被破坏了是最难以恢复的;到这里,有可能被破坏的性质就只剩下性质2和性质4了,这都可以通过后来的辅助程序进行修复的。
插入操作可能破坏的性质:
i. 性质2:当被一棵空树进行插入操作时发生;
ii. 性质4:当新结点被插入到红色结点之后时发生;
```
insert(T, k)
  y <- nil[T]
  x <- root[T]
  while x != nil[T]
    do y <- x
      if key[k] < key[x]
        then x <- left[x]
        else x <- right[x]
  p[x] <- y
  if y = nil[T]
    then root[T] <- k
    else if key[k] < key[y]
      then left[y] <- k
      else right[y] <- k
  left[k] <- nil[T]
  right[k] <- nil[T]
  color[k] <- red
  insert-fixup(T, k)
```

>> * 删除操作:
和插入操作一样,先用BST的删除结点操作,然后调用相应的辅助函数做相应的调整。
首先只有被删除的结点为黑结点时才需要进行调整,理由如下:
i. 树中各结点的黑高度都没有变化
ii. 不存在两个相邻的红色结点
iii. 因为如果被删除的点是红色,就不可能是根,所以根仍然是黑色的
当被删除了黑结点之后,红黑树的性质5被破坏,上面说过了性质5被破坏后的修复难度是最大的。所以这里的修复过程使用了一个很新的思想,即视为被删除的结点的子结点有额外的一种黑色,当这一重额外的黑色存在之后,性质5就得到了继续。然后再通过转移的方法逐步把这一重额外的黑色逐渐向上转移直到根或者红色的结点,最后消除这一重额外的黑色。
删除操作中可能被破坏的性质:
i. 性质2:当y是根时,且 y 的一个孩子是红色,若此时这个孩子成为根结点;
ii. 性质4:当x和p[y]都是红色时;
iii.性质5:包含y的路径中,黑高度都减少了;
>> ```
delete(T, k)
  if left[k] = nil[T] or right[k] = nil[T]
    then y <- k
    else y <- successor(k)
  if left[y] != nil[T]
    then x <- left[y]
    else x <- right[y]
  p[x] <- p[y]
    then root[T] <- x
    else if y = left[p[y]]
      then left[p[y]] <- x
      else right[p[y]] <- x
  if y != k
    then key[k] <- key[y]
      copy y's satelite date into z
  if color[y] = black
    then delete-fixup(T, x)
  return y
```

> - 红黑树是真正的在实际中得到大量应用的复杂数据结构:C++STL中的关联容器 map,set都是红黑树的应用(所以标准库容器的效率太好了,;Linux 内核中的用户态地址空间管理也使用了红黑树。

#### Chapter14 数据规模扩大后,如何扩展数据结构?
> 1. 实际的工程中,极少会去创造新的数据结构,通常是对标准的数据结构附加一些信息,并添加一些新的操作以支持应用的要求。
> 2. 动态顺序统计(Dynamic order statistics)
关于顺序统计这个问题，在中位数和顺序统计量介绍了在O(n)时间内获取一组数据中第i小的数据。在算导第十四章介绍了另外一种方式来求第i小的数据，它的算法复杂度为$O(\log n)$，但却要依赖于另外一种数据结构顺序统计树(order statistic tree)。
顺序统计树，是从红黑树扩展而来。相较于红黑树，一个顺序统计树的结点x，比一个红黑树的结点要多拥有一个字段size.size为以x为根结点的子树所包含的所有结点的数目(也包括x本身)。可以得出一条结论：
> $$x.size=x.left.size+x.right.size+1$$
> 在一棵顺序统计树中，可以很轻便的求该树中第 i小的结点：
> ```
OS-SELECT(x, i)
    r = x.left.size + i
    if i == r
        return x
    else if i < r
        return OS-SELECT(x.left, i)
    else
        return OS-SELECT(x.right, i - r)
```
> 如上面的伪码所示，先求出 x 结点的排位r,因x不小于其左子树的所有结点，
所以r = x.left.size + 1。
若r等于i自不必说。若r大于i，说明所找的数，排在x之前，应从排在x之前的数中找第i小的数。即，从x的左子树中找第i小的数。若r小于i，则应该在比r大的数中找第i-r小的数，即在x的右子树中找第i-r小的数。同样也可以在O(lg n)的时间内求得指定结点的排位:
> ```
OS-RANK(T, x)
    r = x.left.size + 1
    y = x
    while y != T.root
        if y == y.p.right
            r = r+ y.p.left.size + 1
        y = y.p
    return r
```
> 因为size记录的是以当前结点为根结点的子树所包含的所有结点的数目(也包括当前节点本身)，所以左侧伪码通过统计从x到根结点这条路径中本身为右孩子的结点本身以及它们的的左孩子的size来求得x的排位。
顺序统计树以红黑树为基础进行扩展,所以红黑树的原有操作我们都可以继承下来。但是它增加了一个字段size,对于红黑树的删除和插入操作，我们不得不进

> 2. 数据结构的扩张:指在实际应用数据结构时对标准的数据结构中增加一些信息、编入一些新的操作等等。附加的信息必须能够为该数据结构上的常规操作所更新和维护。
> 3. 对一种数据结构的扩张过程可以分为四个步骤:
>> 1. 选择基础的数据结构
>> 2. 确定要在基础数据结构中添加哪些信息
>> 3. 验证可用基础数据结构上的基本修改操作来维护这些新添加的信息
>> 4. 设计新的操作
> 5. 红黑树的扩张定理:当结点中新添加的信息可以由该结点和它的左右子树来决定,那么就可以在不影响时间复杂度的前提下在插入和删除等操作中对红黑树的这些附加信息进行维护。
