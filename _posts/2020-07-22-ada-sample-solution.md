---
layout: post
title: ada-sample-solution
postTitle: 算法设计与分析2020 - 样例试题参考解答
categories: [Algorithms]
description: 算法设计与分析2020 - 样例试题个人解答
keywords: Algorithms
published: true
mathjax: true
typora-root-url: ..
---

# 算法样例试题个人解答

By [ArmeriaWang@github](https://armeriawang.github.io); [Online Version](http://armeria.wang/2020/07/22/ada-sample-solution/)

[TOC]



## T1

本算法的功能是计算

$$
\begin{aligned}
x
&= \sum_{i=1}^{n}\sum_{j=1}^{i}\sum_{k=1}^{i+j}1\\
&= \sum_{i=1}^{n}\sum_{j=1}^{i}i+j\\
&= \sum_{i=1}^n\ \frac{3i^2+i}{2}\\
\end{aligned}
$$

不难发现，$x$的值与时间复杂度是同阶的。由于$x=O(n^3)$，因此该算法的时间复杂度也是$O(n^3)$。

## T2

因为$f(n)=O(g(n))$，所以$\exists c_1>0, n_1, \forall n>n_1$，有$0\le f(n) \le c_1g(n)$。

因为$g(n)=O(h(n))$，所以$\exists c_2>0, n_2, \forall n>n_2$，有$0\le g(n) \le c_2h(n)$。

取$n' = \max\{n_1, n_2\}$，则$\forall n > n'$，有$0 \le f(n) \le c_1g(n) \le c_1\cdot c_2h(n)$。由于$c_1, c_2>0$，所以$c_1\cdot c_2 >0$，$f(n)=O(h(n))$。


## T3

在这里，$a = b = 2$，$f(n)=n^4$。因此$f(n)$的阶数更高，$T(n)=\theta(n^4)$。

## T4

### 基本思路

由于$S$有序，所以先对$S$划分，取出其中间元素$p_m$。然后，利用[快速选择算法](https://en.wikipedia.org/wiki/Quickselect)选出$A$中第$p_m$小的元素$a_m$，并以该元素为基准把$A$划分为两部分，其中$a_m$左边的都比它小，$a_m$右边的都比它大（同样用类似快速排序的方法实现）。这样，原问题就分解成两个规模为$k'=k/2$，$n'=n/2$（平均情况下）的完全类似的子问题。

### 伪代码

```pascal
// 快速选择算法
// 输入：数组A及其长度n，排名r
// 返回：A中第r小的元素
function QuickSelect(A, n, r) begin
	return rth smallest element of array A in O(n) time
end

// 分治算法
// 输入：数组A及其长度n，集合S及其长度k
// 返回：输出A中排名为S的元组
function PartitionSolve(A, n, S, k) begin
	if k = 1 then return QuickSelect(A, n, S[1])
	sMid = S[k/2]
	aMid = QuickSelect(A, n, Smid)
	i = 1
	j = n
	while (i < j) begin
		while (A[i] < aMid) i++
		while (A[j] > aMid) j--
		if (i < j) begin
			swap(A[i], A[j])
			i++
			j--
		end
	end
	ansL = PartitionSolve(A, i, S, k/2)
	ansR = PartitionSolve(A + i + 1, n - i, S + k/2, k - k/2)
	return merge(ansL, ansR)
end
```

### 时间复杂度

设$A$中元素数为$n$，$S$中元素数为$k$的问题的时间为$T(n,k)$，那么：

$$
\begin{aligned}
T(n,k)=2T(n/2, k/2) + n
\end{aligned}
$$

注意到$k\le n$，所以边界条件为$k\ge 1$。用代入法容易证明，$T(n,k)=O(n\log{k})$。

## T5

### 贪心策略

按加工时间$f$从大到小的方式将任务排序，按顺序依次做即可。

### 时间复杂度

算法时间复杂度就是排序的时间复杂度$O(n\log{n})$。

### 正确性证明

#### 引理

设$S$为任务集合$<J_1, J_2,\cdots,J_n$>，$T(S)$为把集合$S$中的任务全部做完所需的时间，任务$J_i$所需的设计时间、加工时间分别为$d_i$、$f_i$。则

$$
\begin{aligned}
T(S)=\min\{d_i+\max\{T(S-\{J_i\}), f_i\}
\end{aligned}
$$

其中$1\le i \le n$，$J_i$表示$S$中第一个做的任务，$\min$用于决策谁是第一个。

设$J_i$和$J_j$分别是是$S$中第一个、第二个做的任务，$T_{s_1s_2\dots s_k}(S)$为前$k$个任务顺序是$J_{s_1}J_{s_2}\dots J_{s_k}$的前提下所能取得的最小时间，那么：

$$
\begin{aligned}
T_{ij}(S) 
&= d_i + \max\{T_j(S-\{J_i\}), f_i\}\\
&= d_i + \max\{d_j+\max\{T(S-\{J_i, J_j\}), f_j\},f_i\}\\
&= d_i + \max\{T(S-\{J_i, J_j\})+d_j, f_j+d_j, f_i\}\\
&= \max\{T(S-\{J_i, J_j\})+d_j+d_i, f_j+d_j+d_i, f_i+d_i\}
\end{aligned}
$$

假设有$f_i\le f_j$。此时$\mathrm{max}$中只可能取到第一项和第二项。不难发现，若调换$J_i$和$J_j$，那么无论$T_{ij}(S)$取的是这两项中的哪一项，$T_{ji}(S)$都不会比$T_{ij}(S)$更大。

#### 贪心选择性

对于加工时间$f$最大的任务$J_i$来说，它要第一个做。否则，可以像冒泡排序一样，把它依次与相邻的之前的任务交换顺序，直到交换到第一个，且在此过程中总时间$T$不会变大（引理）。

#### 最优子结构

由引理中的第一个式子可知，子问题$S-\{J_i\}$取得最优解$T(S-\{J_i\})$时，原问题$T(S)$一定也取得最优解。

因此贪心策略成立。

## T6

### 基本思路与DP方程

首先预处理出所有的$\max(l, r)$和$\min(l, r)$，其中$\max(l, r)$表示$\{x_l, x_{l+1}, \dots, x_r\}$中的最大值，$\min(l,r)$表示最小值。

设$f[i][j]$为考虑到第$i$个数，已分成$j$段的最小代价。那么：

$$
\begin{aligned}
f[i][j] = \min_{j-1\le t\le i-1}\{f[t][j-1] + \mathrm{cost}(\{x_{t+1}, x_{t+2},\cdots, x_i\})\}
\end{aligned}
$$

即：枚举第$j-1$段的右端点$t$，取造成总代价最小者作为实际决策。

### 伪代码

```pascal
// 输入：x[1..n], k

for i = 1 to n do
	minv[i][i] = maxv[i][i] = x[i]

for i = 1 to n - 1 do
	for j = i + 1 to n do begin
		minv[i][j] = min(minv[i][j - 1], x[j])
		maxv[i][j] = max(maxv[i][j - 1], x[j])
	end

f[1][1] = 0
for i = 2 to n do
	for j = 1 to min(i, k) do
		f[i][j] = ∞
		for t = j - 1 to i - 1 do
			f[i][j] = min(f[i][j], f[t][j - 1] + (maxv[t + 1][i] - minv[t + 1][i])^2)

return f[n][k]
```

### 时间复杂度

预处理的过程是$O(n^2)$的，状态数是$O(nk)$个，每个状态的计算需要$O(n)$的时间，因此总的复杂度为$O(n^2 + n^2k) = O(n^2k)$。

拓展：本题可以采用[斜率优化](https://oi-wiki.org/dp/opt/slope/)的方法进一步压缩时间复杂度。

## T7

### 基本思路

如果没有RESET操作，就从低位到高位，按照从左到右（数组下标从小到大）的顺序存储二进制值即可。加入RESET操作后，需要额外记录其长度信息。由于题目限制只能使用一个bit数组，所以可以用如下方式存储：$D = a_0s_0a_1s_1\cdots a_ms_m$。其中，$a_0a_1\cdots a_m$为计数值，$s_i$是「有效位标记」：当且仅当$a_i$是最高位或低于最高位时为1，否则为0。初始时全部为0。

- **INCREMENT操作**。对于$a$，将从左到右将1置为0，直至遇到一个0，将其置为1。对于$s$，只需在最后的0置1的$a_i$对应的$s_i$置为1即可。
- **RESET操作**。从左到右把$a$和$s$置0，直至碰到某一位的$s$是0，说明已经超过了最高位，就不再继续了。

### 势能分析证明（未完成）

下面用势能分析方法证明，对于任意的长度为$n$的操作序列，上述方法的时间复杂度都是$O(n)$。

**<u>注意，下列证明是不完整的，RESET的均摊费用无法转化为常数，可能是势能函数设计得不对。请有办法的大佬不吝指点！</u>**

设势能函数$\phi(D)$为计数器中1的个数。

计数器初始状态$D_0$中1的个数为0，$\phi(D_0)=0$。显然第$i$个操作之后$D_i$满足$\phi(D_i)\ge 0=\phi(D_0)$。于是$n$个操作的平摊代价总和就表示了实际代价的一个上界。

- **INCREMENT操作**。设第$i$次INCREMENT操作对$t_i$个$a$中的位进行了置0，将一位置1；将一个$s$中的位进行了置1：
  - 该操作实际代价：$c_i = t_i + 2$
  - 在第$i$次操作后计数器中1的个数$b_i \le b_{i-1} - t_i+2$
  - 势差：$\phi(D_i) - \phi(D_{i-1}) \le (b_{i-1}-t_i+2)-b_{i-1}= 2-t_i$
  - 平摊代价：$c_i' = c_i + \phi(D_i) - \phi(D_{i-1}) \le (t_i+2) + (2-t_i) = 4$

- **RESET操作**。设第$i$次RESET操作对$D$中总共$k_i$位进行了清0：
  - 该操作实际代价：$c_i = k_i$
  - 第$i$次操作后计数器中1的个数$b_i = 0$
  - 势差：$\phi(D_i) - \phi(D_{i-1}) \le 0-b_{i-1} = -b_{i-1}$
  - 平摊代价：$c_i' = c_i + \phi(D_i) - \phi(D_{i-1}) \le k_i-b_{i-1}$

### 会计法证明

对每次INCREMENT操作征收4元，RESET操作征收1元即可。具体来说，数组里的每个1都会有3元的存款（由0变1消耗1元）。这3元存款里预留出1元作为以后该位翻转为零时用；再留出1元作为维护最高位的费用；再留出1元作为RESET时使用（同时最高位标记也都被清零）。

## T8

### 基本思路

这是一道经典的最小割题目。

把两个核心分别作为源点$S$和汇点$T$，每个例程模块是一个结点。从$S$向模块结点$i$连一条容量为$A_i$的边，从各个模块结点$i$向$T$连一条容量为$B_i$的边。如果模块$i$和模块$j$有数据交换，就在这两个结点间连容量为额外费用$P$的双向边。

假设我们已经有了这个图的最小割$C$。根据割的定义，每个模块结点要么与$S$在同一个集合中（与$S$连通），要么与$T$在同一个集合中（与$T$连通）。这分别代表在核心$S$上运行和在核心$T$上运行。

可以看到，如果两个模块在同一个核心上运行，那么它们之间就是连通的，无需割去容量为$P$的额外代价边。否则，为使二者不连通，就必须割去它们之间的边。由于割$C$是最小割，因此其容量就是我们所要的最小代价。

### 伪代码

```pascal
// 输入：流网络G<V, E>，源s，汇t
// 输出：G中从s到t的最大流
function FordFulkerson(G, s, t) begin
	for any (u, v) in E do begin
		f(u, v) = 0
		f(v, u) = 0
	end
	while G_f exist augmenting path p do begin
    	cap(p) = min{c_f(u, v) | (u, v) is an edge of p}
    	for every edge (u, v) in p do begin
    		f(u, v) = f(u, v) + cap(p)
    		f(v, u) = -f(u, v)
    	end
	end
end

// 输入：模块数N，运行成本A、B，M对需要数据交换的例程，数据交换成本P
// 输出：最低总费用
function SolveCore(N, A, B, M, P) begin
	G = new Graph with source S and target T
	for i = 1 to n do begin
		AddEdge(G, S, i, A_i)
		AddEdge(G, i, T, B_i)
	end
	for every pair (i, j) in M do begin
		AddEdge(G, i, j, P)
		AddEdge(G, j, i, P)
	end
	return FordFulkerson(G, S, T)
end
```

### 算法复杂度

本题等价于求解$O(n)$个结点、$O(n+m)$条边的网络流问题，因此时间复杂度取决于所采用的最大流算法。如果使用Edmonds-Karp算法，时间复杂度就是$O(n\cdot (n+m)^2)$。

## T9

维护一个长度为$n$的队列$Q$，初始时为前$n$个字符。从左到右依次扫描，每次把队头（最左侧字符）出队，新的字符（右侧读入的新字符）入队。这样，每次移动只需判断这两个字符是否为碱基字符即可。时间复杂度为$O(\mathrm{len}(s) + n)$。

## T10

设估价函数$h$为「当前结点的最小出边和终点$T$的最小入边之和」。注意，如果当前结点的出边包含$T$，就需要进行特殊检查。

图中点数过多，这里只以结点$S$、1、6、7、8、9、11形成的子图为例（终点为11）：

![image-20200721220554344](/images/asserts/image-20200721220554344.png)

| 访问节点 | 更新结点 | $g$  | $h$  | $f$  |
| :------: | :------: | :--: | :--: | ---- |
|   $S$    |    1     |  8   |  12  | 20   |
|   $S$    |    6     |  2   |  22  | 24   |
|   $S$    |    7     |  3   |  18  | 21   |
|    1     |    8     |  10  |  15  | 25   |
|    7     |    9     |  11  |  10  | 21   |
|    9     |    11    |  21  |  0   | 21   |

因此$S$到11的最短路为21。