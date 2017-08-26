
title: introduction to algorithms note1
date: 2017/04/11
tags:
    - algorithm
    - data_structure

---

# ＜Introduction to algorithms＞笔记

### 摘要
　　＜introduction to algorithms＞覆盖全面,包含算法/数据结构/基本的离散数学知识;伪代码清晰简洁,适合有一定编程基础用算法实现;注重算法的推导和证明,以及对应的算法设计与分析能力;同时配套MIT课程,资料丰富.这里记录我CLRS的学习笔记,按主题章节对应分类, 以问题-答案形式总结.


### Part1 基础
算法定义/基本的算法设计/递归的复杂度/概率与随机算法

####  Chapter1 什么是算法?

>   算法:
    $\require{AMScd}$
    \begin{CD}
    input @>\text{computing procedure}>> ouput
    \end{CD}
> * 解决一类问题的计算过程, 一个具体问题-实例
> * 算法最重要的特性: 效率

<!-- more -->

#### Chapter2 如何证明一个算法的正确性?
> 1.循环不变式:
　　算法中大量的过程是对同类数据的重复处理,即循环是算法的基本结构.通过循环不变式可以证明经过大量复杂的步骤后,得到要求解的结果.它的结构如下:
    >> a. 初始化: 迭代之前是循环不变式是正确的(base case)
    >> b. 保持: 第n次迭代之前正确, 第n次之后保持正确(recursive formula)
    >> c. 终止: 最终算法正确

> 以插入排序为例：
\begin{align}
A[1]有序 \\\\
A[1] \ldots A[i]有序, 插入后A[1] \ldots A[i+1]有序 \\\\
\mapsto A[1] \ldots A[n] 有序
\end{align}
> 2.算法分析
>> a. 输入规模 n
>> b. 运行时间: 基本操作数(与n相关)
>> c. 最坏情况(上界), 平均情况(基于概率分析技术)
>> d. 运行增长率(rate of growth)

> 3.基本的算法设计方法
>> a. 增量(increment), 从零构造要求求解的结果(bottom up)
>> b. 分治(divide and conquer), 分解子问题递归解决合并(top down)
    divide -> conquer -> combine
以merge sort为例来说明分治策略:
\begin{align}
\{A[1] \ldots A[n]\}= \{A[1] \ldots A[n/2]\} + \{A[n/2+1] \ldots A[n]\} \\\\
recusively \, merge \{A[1] \ldots A[n/2]\} or \{A[n/2+1] \ldots A[n]\} \\\\
merge \{A[1] \ldots A[n/2]\} with \{A[n/2+1] \ldots A[n]\}
\end{align}

> 4.分治法的分析
　　分治法的基本理论是递归的解决子问题, 合并为最终解,即如下公式:
$$
T(n) =
\begin{cases}
\Theta(1), & \text {if n <= c} \\\\
aT(n/b) + D(n) + C(n), & else \\\\
\end{cases}
$$
　　以merge sort为例
$$
T(n) =
\begin{cases}
c, & \text {if n = 1} \\\\
2T(n/2) + cn, & \text {if n > 1} \\\\
\end{cases}
$$
如何求解递归式?
　　考虑到问题的分解, 类似从大的树根分叉出小的树枝, 构造递归树(等价形式),树节点表示时间代价,所有节点之和即为总的运行时间.这也是直观形象理解 **$log_m n$*在复杂度中出现的理由:
>> m表示问题被分解为m个子问题(m叉树), n表示问题规模, $log_m n$表示递归层数(m叉树的高度)


#### Chapter3 如何比较算法的优劣?(运行时间的阶数)

> 1.运行时间与数据规模之间的关系的表达:
>>渐进:
$\Theta(g(n)) = \\{f(n): \exists c\_1, c\_2, n\_0, \forall n \geq n\_0, 0 \leq c\_1g(n) \leq f(n) \leq c\_2g(n) \\}$
上界:
$O(g(n)) = \\{f(n): \exists c, n\_0, \forall n \geq n\_0, 0 \leq f(n) \leq cg(n)\\}$
下界:
$\Omega(g(n)) = \\{f(n): \exists c, n\_0, \forall n \geq n\_0, 0 \leq cg(n) \leq f(n)\\}$

> 2.把握阶数,忽略低阶项和系数:
>> $ 2n^2 + 3n + 1 = 2n^2 + \Theta(n) = \Theta(n^2)$

> 3.常用阶数:
>> $1 < \log n < n < n\log n < n^2 < n^3 < a^n < n!$

#### Chapter4 如何求解递归式(recurrence)?

> 1.定义:
　　递归式是一组等式或不等式,它所描述的函数是用在更小的输入下该函数的值来定义的.

> 2.把握大方向，忽略技术细节:
>> a. 假设自变量为整数: 如实际merge sort的递归式为:
$$
T(n) =
\begin{cases}
\Theta(1), & \text {if n = 1} \\\\
T(\lceil n/2 \rceil) + + T(\lfloor n/2 \rfloor) + \Theta(n), & \text {if n > 1} \\\\
\end{cases}
$$
>> b.向上向下取整, 简化边界条件

> 3.解递归式的三个方法: 代还法, 递归树, 主方法

> 3.1 代换法(Substitution method)
>> 定义:先猜测某个界的存在,再用数学归纳法去证明该猜测的正确性。
缺点:只能用于解的形式很容易猜的情形。
总结:这种方法需要经验的积累,可以通过转换为先前见过的类似递归式来求解。
例子1: 猜测对较大的n, $T(n)=2T(\lfloor n/2\rfloor + 17) + n$ 中与 $T(\lfloor n/2 \rfloor + 17)$ 与 $T(\lfloor n/2\rfloor)$接近
例子2:
\begin{align}
对T(n)=2T(\lfloor n/2\rfloor) + \log n, 做代换m=\log n \\\\
有T(2^m)=2T(2^{m/2}) + m, 设S(m) = T(2^m) \\\\
得 S(m) = 2S(m/2)+m \\\\
得该递归式的界是S(m) = O(m\log m) 代回得: \\\\
T(n) = T(2^m) = S(m) = O(m\log m) = O(\log n\log \log n)
\end{align}

> 3.2 递归树方法(Recursion-tree method)
>> 起因:代换法有时很难得到一个正确的好的猜测值。
用途:画出一个递归树是一种得到好猜测的直接方法。
分析(重点):在递归树中,每一个结点都代表递归函数调用集合中一个子问题的代价。将递归树中每一层内的代价相加得到一个每层代价的集合,再将每层的代价相加得到递归式所有层次的总代价。
总结:递归树最适合用来产生好的猜测,然后用代换法加以验证。递归树的方法非常直观,总的代价就是把所有层次的代价相加起来得到。但是分析这个总代价的规模却不是件很容易的事情,有时需要用到很多数学的知识,如数列求和, 不等式放缩等.

> 3.3 主方法(Master method)
>> 主方法是最好用的recipe 方法,可以很快估计出递归算法的时间复杂度,主方法总结了常见的情况并给出了一个公式.实际上主方法一直在比较 f(n)与 $N^{\log_b^a}$ 的规模,然后选取规模大的作为最后的递归式的规模
定义:
　　设 a≥1 和 b≥1 是常数f(n)是定义在非负整数上的一个确定的非负函数。又设T(n)也是定义在非负整数上的一个非负函数,且满足递归方程$T(n)=aT(n/b)+f(n)$ 。方程 $T(n)=aT(n/b)+f(n)$ 中的 $n/b$ 可以是$\lceil n/b\rceil$,也可以是 $\lfloor n/b \rfloor$。那么,在$f(n)$的三类情况下,我们有 T(n)的渐近估计式:
优点:针对形如 T(n)=aT(n/b)+f(n)的递归式:
\begin{align}
1) 对\epsilon > 0, f(n)=O(n^{\log_b^a-\epsilon}) \to T(n)=\Theta(n^{\log_b^a}) \\\\
2) f(n) = \Theta(n^{\log_b^a}) \to T(n)=\Theta(n^{\log_b^a}\log n) \\\\
3) 对\epsilon > 0,f(n)=O(n^{\log_b^a-\epsilon}) 且c < 1和 n > N, af(n/b) \leq cf(n) \to T(n)=\Theta(f(n))
\end{align}
缺点:并不能解所有形如上式的递归式的解。因为主方法在第 1 种情况与第2种情况之间、第2种情况与第3种情况之间都存在着一条沟,所以会存在着不能适用的情况。

#### Chapter5 如何将概率引入算法分析中?
> 1.概率分析: 确定input的分布或假设, 计算期望的运行时间 - 实际上是一个随机变量函数的求解过程
> 2.在不了解input分布的条件下,通过随机算法引入概率:
如果一个算法的行为不只是由输入决定,同时也由随机数生成器所产生的数值决定,则称这个算法是随机的。
> 3.指示器随机变量-- 概率与期望的转换
$$
事件A 对应的指示器随机变量的期望期等于事件 A 发生的概率。\\\\
I\\{A\\} =
\begin{cases}
    1, if \, A \, happens \\\\
    0, else
\end{cases}
$$
期望的线性性质:
\begin{align}
X = \sum\_{i=1}^n X\_i \\\\
E[x] = E[\sum\_{i=1}^n X\_i] = \sum\_{i=1}^n E[X\_i]
\end{align}

> 4.如何生成随机数组?
a. 随机优先级法:为数组的每个元素赋一个随机的优先级,再根据这个优先级对数组中的元素进行排序。可证这样得到的数字满足随机的性质。
b. 原地交换法:依次把A[i]与 A[Random(i+1, Length(A))]进行 swap,得到的新数组也满足随机性。
>> ```
for i ← 1 to n
    do swap A[i] ↔ A
```

>考虑到在真正的环境中的输入可能并不是随机的,所以我们可以采用先将输入进行随机打乱的方法来保证输入数据的随机性(方法b),这点在很多算法中得以体现,比如快排有其随机选取种子数来向输入中加入随机化的成分。

### Part2 排序与顺序统计

基本排序算法与对比/求解中集合第i小的数

> 1.排序问题
输入:n个数的序列$[a\_1,a\_2, \ldots, a\_n]$ \
输出:输入序列的一个重排$[a\_1',a\_2',\ldots a\_n'],使 a\_1' \leq a\_2' \leq \ldots \leq a\_n'$
2.原地排序算法:只有线性个数的元素会被移动到集合之外的排序算法。
3.第6章介绍堆排序,及其实现的优先级队列
4.第7章介绍快速排序没,及其对应的分割思想
5.第8章介绍了基于“比较”排序的算法的下界为$\Theta(n\log n)$。并介绍了几种不基于比较的排序方法,它们能突破$\Omega(n\log n)$的下界: 计数排序、基数排序、桶排序。
6.第9章介绍了顺序统计的概念:第i个顺序统计是集合中第i小的数。并介绍了两个算法:
a.最坏情况为$O(n^2)$但平均情况下为线性$O(n)$的算法
b.最坏情况下为线性$O(n)$的算法

#### Chapter6 如何结合数据结构和排序算法?

> 1.heapsort:$O(n\log n)$, 原地排序(in place)
    利用heap的性质(堆顶为max或min)来进行排序
> 2.heap:
>> a.逻辑结构为完全二叉树, 高度为$\log_2(n)$
>> b.物理结构为数组(非链表, 因为要求随机访问)
>> c.性质(以大顶堆为例):$A[Parent(i)] \geq A[i]$
>> d.方法:
* max-heapify(A, i): 递归向下调整以i为根的子树, 保持性质c, $O(\log n)$
* build-max-heap(A): 调用max-heapify建堆A, $O(n)$
* heapsort: 原地堆排序, $O(n\log n)$

>>```
max-heapify(A, i)
l <- Left(i)
r <- Right(i)
if l <= heap-size[A] and A[l] > A[i]
    then largest <- l
    else largest <- i
if r <= heap-size[A] and A[r] > A[largest]
    then largest <- r
if largest not = i
    then exchange A[i] <-> A[largest]
    max-heapify(A, largest)
```


>> ```
bulid-max-heap(A)
heap-size[A] <- length[A]
for i <- length[A] / 2 downto 1
    do max-heapity(A, i)
```

>> ```
heapsort(A)
build-max-heap(A)
for i <- length[A] downto 2
    do exchange A[i] <-> A[i]
    heap-size[A] <- heap-size[A] - 1
    max-heapify(A, 1)
```

> 3.优先级队列(用堆实现)
>> a. 方法:
    * maximum(S):返回最大key的元素 -> 大顶堆堆顶元素
    * insert(S, x): 插入x -> 插入堆尾并max-heapify(A, n + 1)
    * extract-max: 在maximum基础上删除max -> swap(A[1], A[n]), max-heaptify(A, 1)
b. 应用: 操作系统中优先级调度算法, 事件模拟

#### Chapter7 如何实现快速排序?
> 1.快排: 基于递归思想, 最坏$\Theta(n^2)$, 平均$O(n\log n)$, 且由于 $O(n\log n)$中隐含的常数因子很小,所以快排通常是用于排序的最佳的实用选择(因为其平均性能非常好).
特性:平均性能非常好、原地排序不需要额外的空间、代码实现机器简洁清晰

> 2.分割partition: 将A[p..r]分割为均小于等于A[q]的A[p..q-1] 和 均大于A[q]的 A[q+1..r]
>```
partition(A, p, r)
x <- A[r]
i <- p-1
for j <- p to r-1
    do if A[j] <= x
        then i <- i+1
            exchange A[i]<-> A[j]
exchange A[i+1] <-> A[r]
return i+1
```

> 3.快排, 递归实现
>```
quicksort(A, p, r)
if p < r
    then q <- partition(A, p, r)
        quicksort(A, p, q - 1)
        quicksort(A, q + 1, r)
```

> 4.快排性能:
划分是对称的, 生成的递归树亦对称,等同于merge sort;
划分极度不对称, 生成递归树为单支树, 树高为n, 等同于insert sort;
因此在真正的应用时很容易出现待排序的数组其实已经是有序的情况下,它在待排数组有序时的效率是最差的$O(n^2)$,所以需要随机化技术来使得划分尽可能均匀对称.
　　正如第5章所说的,由于工程中的输入可能不随机的,所以我们要将其随机化.有两种可选方案
>> a.直接对输入数据进行随机化排列
>> b.采用随机取样的随机化技术。
采用更高效的(2)随机取样(random sampling):随机选择key元素
>```
randomized-partition(A, p, r)
i <- random(p, r)
exchange A[r] <-> A[i]
return partition(A, p, r)
```

#### Chapter8 排序算法的下界在哪里?
> 1.任何比较的排序在最坏的情况下都要用$\Theta(n\log n)$次比较来进行排序,
采用决策树来证明:
\begin{align}
高度h, 叶节点个数l的决策树模拟n个元素的比较排序 \\\\
\because n元素有 n!个排列 \to  n! \leq l \\\\
又\because l \leq 2^h \\\\
\therefore n! \leq l \leq 2^h \\\\
\therefore h \geq \log(n!) = \Omega(n\log n)
\end{align}
所以合并排序和堆排序是渐近最优的。但要注意:快排不是渐近最优的,因为它在最坏的情况下是$O(n^2)$

> 2.三种以线性时间运行的排序算法:计数排序、基数排序和桶排序，皆基于非比较的模型．

> 2.1 计数排序: 统计元素个数, 从小到大输出, $\Theta(n)$
>```
counting-sort(A, B, k)
for i <- 0 to k
    do C[i] = 1
for j <- 1 to length[A]
    do C[A[j]] <- C[A[j]] + 1
C[i]: 等于i的元素个数
for i <- 1 to k
    do C[i] <- C[i] + C[i-1]
C[i]: 小于等于i的元素个数
for j <- lenght[A]
    do B[C[A[j]]] <- A[j]
        C[A[j]] <- C[A[j]] - 1
```
>a. 计数排序的一个重要性就是它是稳定的排序算法,这个稳定性是基数排序的基石。
b. 计数排序的思想简单、高效、可靠．
c. 缺点在于:
>>i. 需要很多额外的空间(当前类型的值的范围)
ii. 只能对离散的数据有效,比如int(double不行)
iii.基于假设:输入是小范围内的整数构成的。

> 2.2 基数排序: 基于数位(k), $\Theta(d(n+k))$
>```
radix-sort(A, d)
for i <- 1 to d
    do use a stable sort array A on digit i
```
>a. 低位相同时用高一位进行排序,要求基数排序时对每一位进行调用子排序算法时要求这个子排序算法必须是稳定的, 即同Key, 相对顺序不变.
b. 基数排序与直觉相反:它是按照从低位到高位的顺序排序的。我认为原因在于高有效位对低有效位有着决定性的作用。

> 2.3 桶排序:分割子区间(桶), 桶内排序后输出 $\Theta(n)$
>```
bucket-sort
n <- length[A]
for i <- 1 to n
    do insert A[i] into list B[nA[i]]
for i <- 0 to n-1
    do sort list B[i] with insert sort
output the lists B[0], B[1],..B[n-1] in order
```
>a.桶排序也只是期望运行时间能达到线性,对于最坏的情况,它的运行时间取决于它内部使用的子排序算法的运行时间,
一般为 O(nlgn)。
b.桶排序基于假设:输入的的元素均匀的分布在区间[0, 1]上。


> 3.总结: 所有的线性时间内的排序算法,都作出了一定的假设, 必须建立在一定的输入数据的条件上.


#### Chapter9 如何找到第i小的数?
> 1.the i-th order statistic: 第i小的元素
    直观的讲, 先排序然后直接选出A[i]就是该问题的解答.但实际上问题只要求第i小的数,排序则是求出了所有元素的次序, 那么是否不需要较大的消耗的排序,而去直接求解呢?

> 2.由简入深
>> min(max): 线性遍历, $\Theta(n)$
>>```
minimum(A)
min <- A[1]
for i <- 2 to length[A]
        do if min > A[i]
            then min <- A[i]
return min
```
>> i-th:随机选择与分割定位: 递归, $\Theta(n)$
>>```
randomized-select(A, p, r, i)
if p = r
        then return A[p]
q <- randomized-partition(A, p, r)
k <- q - p + 1
if i = k
        then return A[q]
elseif i < k
        then return randomized-select(A, p, q-1, i)
else
        return randomized-select(A, q+1, r, i-k)
```

> 3.在第8章,比较模型中平均情况下需要$\Theta(n\log n)$的时间, 输入满足一定假设情况下可以达到线性时间按$O(n)$.以期望线性时间选择i-th的方法以快速排序为模型,如同在快速排序中一样,此算法的思想也是对输入数组进行递归划分。但和快速排序不同的是,快速排序会递归处理划分的两边,而randomized-select只处理划分的一边,并由此将期望的运行时间由$O(n\log n)$下降到了$O(n)$。

### Part3 数据结构

####
