---
layout: post
title: pods02-view-deletion
postTitle: 论文笔记 - PODS02 Peter Buneman
categories: [Paper Notes, Database, PODS]
description: 论文笔记 - PODS02, On Propagation of Deletions and Annotations Through Views, Peter Buneman
keywords: Paper Notes, Database, PODS, View Deletion
published: true
mathjax: true
typora-root-url: ..
---

[doi](https://doi.org/10.1145/543613.543633)

## 关系数据库基本概念

关系数据库（Relation Database）中，关系（Relation）就是表，元组（Tuple）指代一行，属性（Attribute）指代一列，位置（location）指代一个单元格：

<img src="https://raw.githubusercontent.com/ArmeriaWang/images-for-markdowns/master/20200730023025.png" style="zoom: 50%;" />

一个典型的关系如下：

<img src="https://i.loli.net/2020/08/01/AvC71w4xRI8tT3o.png" style="zoom:67%;" />

只看前两列。可以看到，如果把位置上的实体视作结点，那么一个tuple就代表着一条边。关系表通常记作$R(A_1, A_2, \dots, A_n)$，其中$A_1, A_2, \dots, A_n$是其attributes。位置location记作三元组$(R, t, A)$。

关系数据库的查询操作主要分为四种：S（select $\sigma$，选择行），P（projection $\Pi$，选择列），J（join $\bowtie$，自然连接，将两个关系中都出现的属性相等的tuple进行选择和合并），U（union $\cup$，并）。关系代数基础见Ullman的[课件](http://infolab.stanford.edu/~ullman/fcdb/aut07/slides/ra.pdf)。

## 几个经典NP问题

### 3-SAT问题

「Core-six」问题之首。

> Given a set of variables $U$, and a set $C$ of clauses, where each clause has **exactly** 3 elements, can we assign true/false values to the variables in $U$ that satisfies all of the clauses in $C$?

通过添加额外中间变量的方法，很容易将SAT问题归约到它。

### monotone 3-SAT问题

monotone的意思是，每个clause中的变量要么全negated，要么全non-negated：

> Given an formula of clauses $F^{\prime}=\wedge_{i=1}^{n} C_{i}^{\prime}$ where each clause in $F^{\prime}$ contains **all negated or non-negated** variables, and each clause $C_{i}$ contains **at most** 3 variables. Does there exist an assignment of the variables so that $F^{\prime}$ is satisfied?

从3-SAT问题规约而来。

### hitting set问题

> The input to the hitting set problem is same as the set cover problem, but the goal now is to find a smallest subset $X^{\prime} \subseteq X$ of elements such that $S_{i} \cap X^{\prime} \neq \emptyset$ for each set $S_{i}$ in the collection. The hitting set problem is a dual to the set cover problem and has the same approximability threshold.

也可以从超图的角度理解，

> If we regard $\Sigma$ as defining a hypergraph on $V$ (where each set in $\Sigma$ constituting a hyperedge) then we see that the hitting set problem is equivalent to the vertex cover problem on hypergraphs.

## 主要问题及结论

*On Propagation of Deletions and Annotations Through Views*（PODS 2002）主要讨论如下两个问题：

- **view deletion problem**。分为两个小问题：

  1. **view side-effect problem**。对于某个SPJU查询的结果（view），如何知道：能否删除原关系（source）中的一些tuple，使得view中的指定tuple也被删除，且对于view是side-effect-free的（或求view side-effect最小的删除方案）。
  2. **source side-effect problem**。对于某个SPJU查询的结果，要求删除source中尽量少的tuples，使得view中的指定tuple也被删除。这个问题中无视view中的side-effect。

  结论如下：

  - 对于包含PJ或JU的询问，上述两个问题都是NP的
  - 对于SPU或SJ询问，两个问题都是P的

-  **annotation placement problem**。对于某个SPJU查询的结果，寻找一个source中的位置，使得在该位置上放置标记（annotation）能传播（propagate）到view上的某一个指定位置，且view中的side-effect最小。

  结论如下：

  - 对于包含PJ的询问，是NP的
  - 对于SJU或SPU询问，是P的

直观上，这两个问题的共同点都是寻找查询结果的「源头」。

### 视图删除问题

证明的基本思路是：

- 要证明是NP的，可把monotone 3-SAT问题或hitting set问题归约到它
- 要证明是P的，给出一个P的算法即可

#### view side-effect problem

这一问题中，我们的目标是，在删除view中的指定tuple $t$的前提下，问是否存在对于view的side-effect-free的删除方案（或求view side-effect最小的删除方案），而不考虑source中删除了多少tuples。显然，求最小的难度大于等于判断是否存在side-effect-free的难度。这里，我们先讨论是否存在side-effect-free的删除方案。这是一个判断可行性问题，因此我们考虑用3-SAT问题进行规约。

##### PJ查询

考虑如下问题。有两个source关系：每个用户在若干群组里，每个群组中有若干文件。把这两个关系做join操作，再用project取USER和FILE两列，就形成「用户-文件」关系（view）。现在，如果我们想隔离某个用户和某个文件（不妨设为tuple $t$），就需要删除两个source关系中的某些tuple。现在我们想找到这么一种删除方案，它不仅能删除$t$，而且view中删除的tuple数量最少。如果除了$t$没有删除任何其他tuple，就称这种方案是side-effect-free的。

<img src="https://i.loli.net/2020/07/30/1kWGtlDAUj3pXmK.png" style="zoom: 50%;" />

直观地感觉这个问题，就能发现它是很难的。view中的关系是两个source关系的叠加，删去任何一个source tuple，都无法评估它对view产生的影响。

事实上，可以将monotone 3-SAT问题归约到它。设某个3-SAT实例有变元$x_i$，子句$C_i$，构造相应的view side-effect问题。

- 先构造两个关系$R_1(A, B)$和$R_2(B,C)$，分别包含形如$(a,x_i)$和$(x_i,c)$的所有tuple，其中$a$和$c$是两个辅助元素。
- 对于子句$C_i=(x_{i_1}, x_{i_2},x_{i_2})$，$R_1$会额外包含$(a_i,x_{i_1}),(a_i,x_{i_2}),(a_i,x_{i_3})$三个tuple；对于子句$C_{j}=(\overline{x_{j_{1}}}+\overline{x_{j_{2}}}+\overline{x_{j_{3}}})$，$R_2$会额外包含$\left(x_{j_{1}}, c_{j}\right),\left(x_{j_{2}}, c_{j}\right), \left(x_{j_{3}}, c_{j}\right)$三个tuple。
- 然后，我们询问$\Pi_{A, C}\left(R_{1} \bowtie R_{2}\right)$。显然，tuple $(a,c)$一定存在于view中。现要求把tuple $(a,c)$删除。

例如，对于monotone 3-SAT问题实例$(\overline{x_{1}}+\overline{x_{2}}+\overline{x_{3}})\left(x_{2}+\right.\left.x_{4}+x_{5}\right)(\overline{x_{4}}+\overline{x_{1}}+\overline{x_{3}})$，就有：

<img src="https://i.loli.net/2020/07/30/WaIkT4Qf1mJFlYS.png" style="zoom: 80%;" />

不难发现，「原3-SAT实例的解」与「把$(a,c)$删除的side-effect-free解」是等价的：如果$x_i$为true，就删除$R_1$中的$(a,x_i)$；否则删除$R_2$中的$(x_i, c)$。因此，我们就把monotone 3-SAT问题归约到了目标问题，从而证明了PJ查询的view side-effect problem是NP的。

##### JU查询

JU查询的view side-effect problem同样可以被monotone 3-SAT问题规约。

对于有$n$个变量$x_i$和$m$个子句的3-SAT问题，引入$2(m+n)$个关系。

- 对于每个变量$x_i$，构造两个单属性关系$R_i(A_1)$和$R_i'(A_2)$，前者只包含一个tuple $T$，后者只包含一个tuple $F$。

- 对于每个子句$C_i$，同样构造两个单属性关系$S_i(A_2)$和$S_i'(A_1)$，都各自只包含一个tuple $c_i$。

- 然后查询如下$m+n$个查询的并。U(S(P(J)))

  - 对于前$m$个查询，$Q_i$对应于子句$C_i=\left(x_{i_{1}}+x_{i_{2}}+x_{i_{3}}\right)$，有$Q_{i}=\left(R_{i_{1}} \bowtie S_{i}\right) \cup\left(R_{i_{2}} \bowtie S_{i}\right) \cup \left(R_{i_{3}} \bowtie S_{i}\right)$​。对于negated子句，把$R_{i_j}$和$S_i$分别替换为$R_{i_j}’$和$S_i'$即可。
  - 对于后$n$个查询，$Q_{m+j}$对应于变量$x_j$，有$Q_{m+j}=R_{j} \bowtie R_{j}^{\prime}$。

  显然，tuple $(T, F)$一定存在于view中。现要求把$(T, F)$删除。

例如，对于3-SAT实例$(\overline{x_{1}}+\overline{x_{2}}+\overline{x_{3}})\left(x_{2}+\right.\left.x_{4}+x_{5}\right)(\overline{x_{4}}+\overline{x_{1}}+\overline{x_{3}})$来说，就有：

<img src="https://i.loli.net/2020/07/30/ecVAtu9IKF37oRs.png" style="zoom: 80%;" />

同样地，「原3-SAT实例的解」与「把$(T,F)$删除的side-effect-free解」是等价的：如果$x_i$取true，就删除$R_i'$中tuple $F$，否则删除$R_i$中的tuple $T$。这样，JU问题的view side-effect problem也被3-SAT规约了。

在这两类询问的构造中，查询结果view需要由两部分组成：一是需要删除的tuple $t$，二是期望不受影响的其他tuples。其中第二部分是不可或缺的，否则会降低问题的难度（随便删掉一部分都能满足把$t$删掉的要求）。

##### SPU查询

SP查询就是选择原关系中的特定行和列。显然，要删除这类查询结果的某个tuple，只需要检查一遍source中的tuple是否满足select的条件即可，这是线性的时间复杂度。即使加上union操作，该算法也依然能保持多项式时间。

##### SJ查询

对于SJ查询形成的view，可以发现其中的任一tuple都只有一个witness（我理解为生成路径）。因此，只需切断witness路径上的一个tuple，就能删掉目标tuple。所以，依次检查这个witness路径上的各个tuple中，是否有不产生side-effect的即可。这也是一个多项式算法。

#### source side-effect problem

这一问题中，我们的目标是，要求删除source中尽量少的tuples，使得view中的指定tuple $t$也被删除。无视view中的side-effect。这是最优化问题，我们考虑用hitting set问题进行规约。基本思路是把

##### PJ查询

对于一个有$n$个元素、$m$个集合$S_i$的hitting set问题实例，我们构造其对应的source side-effect问题实例。设$S_i$由$i_k$个互不相同的元素$x_{i_1},x_{i_2},\ldots x_{i_k}$构成。

- 我们把每个$S_{i}$都编码为关系$R_0(S, A_1, \dots, A_n)$中的一个tuple$\left(s_{i}, d, \ldots, d, x_{i_{1}}, d, \ldots d, x_{i_{2}}, d, \ldots, d, x_{i_{k}}, d, \ldots, d\right)$，其中$d$是一个「虚元（dummy element）」。

  换句话说，$S_i$对应的特征tuple，就是对于自己包含的变元$x_i$，列$A_i$下面的值就是$x_i$；对于自己没有包含的变元$x_j$，列$A_j$下面就是$d$。

- 除了关系$R_0$，还有$n$个额外的关系$R_1, \dots, R_n$，每个都形如$R_i(A_i, B_i, C)$的形式，且都恰好包含$n+1$个tuples：$\left(x_{i}, \alpha_{0}, c\right),\left(d, \alpha_{1}, c\right), \ldots,\left(d, \alpha_{n}, c\right)$。即只有第一行是$(x_i, \alpha_0, c)$，下面的$n$行都是$(d, \alpha_i,c)$。

如上构造出的$n+1$个关系如下图所示。

<img src="https://i.loli.net/2020/07/30/7SLACaBMoyKpOZ4.png" style="zoom:80%;" />

接着构造PJ查询$\Pi_{C}\left(R_{0} \bowtie R_{1} \bowtie\ldots \bowtie R_{n}\right)$。查询结果只有一个tuple $(c)$，现在要求删除它，并使得source中删除的tuples数量尽量少。

考察tuple $(c)$是怎么来的。如果我们不是只选择$C$这个attribute，而是看整个$\left(R_{0} \bowtie R_{1} \bowtie\ldots \bowtie R_{n}\right)$，那么每个集合$S_i$都会生成$n^{n-\|S_i\|}$个tuples，也就是说有$n^{n-\|S_i\|}$个witness路径：对于$x_j \in S_i$的$R_j$，只有$x_j$一个选择；对于$x_j \notin S_i$的$R_j$，则有$n$种选择（$B$属性区别了各个虚元$d$）。对于前者，为了删除$t$，就需要删除$R_j$中$x_j$对应的那条tuple；对于后者，则需要完全删除$R_j$中$d$对应的$n$条tuple。

两个问题解的对应关系也非常明显了：在hitting set的最优解中，如果某个$x_i$被选中，就删去$R_i$中的$x_i$ tuple。显然，只删去这些tuple就足以删去view中的tuple $(c)$了。另一方面，一定也存在一个source side-effect的最优解，使得它仅删除形如$(x_p, \alpha_0, c)$的tuples，并对应于hitting set的最优解。这样，PJ查询的source side-effect问题就被规约为NP问题了。

但是，如果把join操作限制为chain join（即只有相邻两个关系之间才能共享相同的attribute），就可以转化为最小割模型，用多项式算法求解了。

##### JU查询

原论文中依赖于重命名（renaming $\delta$）操作完成证明，同样用hitting set问题进行规约。不失一般性地，假设每个集合$S_i$都恰好包含$k$个元素。对于每个元素$x_i$，都构造一个只包含一个tuple $(a)$的单属性关系$R_i(A)$。查询$Q$是$m$个子查询$Q_i$的并，子查询$Q_i$对应于集合$S_i$。如果$S_i$包含元素$x_{i_{1}}, \ldots, x_{i_{k}}$，就有 $Q_{i}=\delta_{A \mapsto A_{1}}\left(R_{i_{1}}\right) \bowtie \ldots \bowtie\delta_{A \mapsto A_{k}}\left(R_{i_{k}}\right)$。$Q$的查询结果只有一个tuple $(a, a, \dots, a)$（$k$个$a$），现要求删除这个tuple，并最小化source中删除的tuple数量。

考察这个tuple是怎么来的：每个子查询$Q_i$都为其提供了一条witness路径。因此，hitting set实例的最优解与本问题最优解的对应关系就很直观了。这样我们也证明了JU查询的source side-effect问题是NP难的。

##### SPU查询和SJ查询

与view side-effect问题中的处理完全类似。

### 标记放置问题

关于查询结果的「溯源」问题，本文提供了另一个有趣的维度：标记（annotation）。标记是作用在位置（location）上的，可以随着selection、projection、join、union、renaming操作传播。需要注意的是，标记传播中的equality并不是用实质相等定义的（类似OOP中的地址相等，而不是观察等价性）。

##### PJ查询

用3-SAT问题规约。设某个3-SAT实例有$m$个子句$C_i$。对于$C_i$（不妨设它包含$x_{i_1}, x_{i_2}, x_{i_3}$三个变量），构造关系$R_i(C_i, x_{i_1}, x_{i_2}, x_{i_3})$，每个$R_i$包含七个「赋值tuples」，每个tuple对应于一组使得$C_i$为真的取值。此外， 对$R_1, R_2, \dots, R_{m-1}$，有一个额外的dummy tuple $(c_i,d,d,d)$；对$R_m$有两个额外的dummy tuples $(c_m,d,d,d)$和$(c_m',d,d,d)$。 

现在查询$Q=\Pi_{C_{1}, \ldots, C_{m}}\left(R_{1} \bowtie \ldots \bowtie R_{m}\right)$。查询结果中恰好包含$(c_1, \dots, c_m)$和$(c_1, \dots, c_m')$两个tuples。不难看出，只有让原3-SAT实例成立的那组$x_i$的取值才会使得$(c_1, \dots, c_m)$出现在view中，反之亦然。

假设我们需要让$(Q(S), (c_1, \dots, c_m), C_1)$被传播上标记。如果原3-SAT实例是可满足的，那么只需要标记上source中的$(R_1, t, C_1)$位置即可（$t$是3-SAT解对应的取值tuple），这样就是side-effect-free的解。反之，如果我们有了side-effect-free的解，也必然说明3-SAT实例是可满足的。这样就完成了规约。

通过上述过程还可以得到推论：对于任意tuple $t\in Q(S)$，判断source中的某一tuple $t'$是否在$t$的witness路径中也是NP难的。进一步地，判断source中的某一标记是否会在output中出现也是NP难的。

##### SPU查询

对于SP查询view中的某一位置，source显然有唯一的side-effect-free的标记位置。对于SPU查询，分别考虑每个SP子查询，然后并起来即可。这是P的算法。

##### SJU查询

对于SJ查询，只需在目标location的tuple的witness路径上找到side-effcet最小的tuple即可。加上union操作后，还需统计对其他子查询造成的side-effect，取总和最小的那一个即可。这也是P的算法。

### 两个问题之间的联系

都是在「溯源」。deletion也可以理解为打在tuple上的annotation。

deletion问题是以tuple作为基本元素进行溯源，而annotation问题的基本元素是location。

## 参考资料

- [Relational Algebra](http://infolab.stanford.edu/~ullman/fcdb/aut07/slides/ra.pdf)
- [Relational Model](https://en.wikipedia.org/wiki/Relational_model)
- [Relation_(database)](https://en.wikipedia.org/wiki/Relation_(database))
- [monotone 3-SAT](https://npcomplete.owu.edu/tag/monotone-3sat/)
- [hitting set problem](https://theory.stanford.edu/~virgi/cs267/lecture5.pdf)
