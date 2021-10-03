---
title: "构建数据依赖的实现"
description: "这篇文章综合了先前的“从CFG直接构建GSA的算法”，梳理其核心思想，总结DFST、支配、归约的各种性质、算法，并且给出了我的更清晰的实现。"
lead: "这篇文章综合了先前的“从CFG直接构建GSA的算法”，梳理其核心思想，总结DFST、支配、归约的各种性质、算法，并且给出了我的更清晰的实现。"
date: 2021-09-28T17:20:26+08:00
lastmod: 2021-09-28T17:20:26+08:00
draft: false
weight: 50
images: []
contributors: [szp]
---

上一篇文章的链接在这里：[从CFG直接构建GSA的算法]({{< relref "/blog/build-gsa-from-cfg/index.md" >}})。

<x-card>

**注意：未经许可，禁止转载**。通过[OpenTimestamps](https://opentimestamps.org/)，我已经获得了该文最早的区块链时间戳。有且只有我（[me@szp.io](mailto:me@szp.io)）持有其证明。因此如果你能伪造区块链，请你投计算机顶会；如果不能，不要未经我的许可，转载文章内容，否则我保留追究权利。

<x-warning comment="引用不当"></x-warning>这篇文章尚在编写中，因而还没给出完整的引用。如果你很迫切想知道前人的工作，可以看上一篇文章“[从CFG直接构建GSA的算法]({{< relref "/blog/build-gsa-from-cfg/index.md" >}})”的底部。

</x-card>

## 数据依赖的定义

数据依赖是合并基本块<x-comment>（入边数目大于等于2的基本块）</x-comment>的入边上的，它表示的是**从该基本块的立即支配者到该基本块的无环路径成立所需要满足的所有条件**。

下图是一个简单的CFG，本文之后的所有示例都会基于这个图。本文中的CFG会被认为是**带入口节点、可以有重边、可以有自环的有向图**。此外满足**可达性**：存在从入口节点到任意节点的路径。

<x-figure src="./example-cfg.svg" id="fig:example-cfg">示例CFG</x-figure>

上图各个合并节点入边的数据依赖如下：

<x-table id="tab:example-cfg-data-dependency">

|边的ID|数据依赖条件|
|-:|:-|
|6|$(P\land\neg R)\lor(\neg P\land Q\land\neg R)$|
|7|$R$|
|9|$\neg P\land Q$|
|10|$\neg P\land\neg Q\land\neg T$|
|13|$\neg P\land\neg Q\land T$|
|17|$P$|

示例<x-ref-figure ref="fig:example-cfg"></x-ref-figure>中CFG的所有数据依赖条件

</x-table>

接下来，我将讲解现在是数据依赖算法的具体实现。

## 算法的步骤

本算法是个多步骤的算法。具体各个步骤的依赖关系如下：

<x-figure src="./algo-steps.svg" id="fig:algo-steps">数据依赖算法步骤</x-figure>

各个步骤的输出及目的如下：

<x-table id="tab:algo-steps">

|编号|名称|输出|
|-:|:-:|:-|
|1|深度优先生成树|以入口基本块为根的生成树，识别出前向边和回边|
|2|支配树|每个基本块的立即支配者|
|3|识别循环|每个基本块所属的最近循环头|
|4|归约序列|找到一个序列，序列的每个节点可以归约到后面的节点|
|5|路径摘要|按照归约序列，计算从立即支配者出发的路径条件|

算法的各个步骤

</x-table>

## 深度优先生成树（DFST）

### DFST边的类别

通过深度优先搜索构成的生成树，则原控制流图中的边可以被分成4种：

1. **前向边**：终点是起点的子孙，且不是自环边
2. **回边**：终点是起点的祖父，且不是自环边
3. **自环边**：起始与终止节点相同
4. **交叉边**：终点既不是起点的子孙，也不是起点的祖父

<x-comment>注：本文中的祖父是指***包含节点自己***的集合：节点、节点的父亲、节点的父亲的父亲等等组成的集合，其中也；子孙的概念类似。如果需要强调不包含自己，我会使用***严格祖先***、***严格子孙***。有些论文中的分类中会将生成树上的边称为***树边***，进而细分前向边，这里简化讨论不这么做。</x-comment>

形象的分类，及其重要的性质如下图：

<x-figure src="./dfst-edge.svg" id="fig:dfst-edge">DFST边的分类及性质</x-figure>

例子如下，节点上的数字为节点的逆后序遍历序号，要指出的一点是，**DFST的前序/后序/逆后序遍历特指一种遍历方式**，这个遍历是由生成方式决定的。因此当我们说DFST的逆后序遍历时，实际上是说CFG的逆后序遍历：

<x-figure src="./dfst-example.svg" id="fig:dfst-example">示例DFST，标注了逆后序遍历序号及边的类型</x-figure>

自环边和回边我们统称为**循环边**，而前向边和交叉边统称为**非循环边**。

### DFST边的性质

上图第1个性质显然，这里给出第2个性质的证明。

<x-card>
<x-theorem id="th:non-cycle-dag">令DFST上节点$x$的逆后序遍历序号为$\mathrm{RevPostOrder}(x)$，那么对于任何非循环边（尤其是交叉边）$e$有

$$\mathrm{RevPostOrder}(\mathrm{Dst}(e))>\mathrm{RevPostOrder}(\mathrm{Src}(e))$$

</x-theorem>
<x-proof for="th:non-cycle-dag"><x-wip></x-wip></x-proof>
</x-card>

上述的定理其实告诉我们，由非循环边组成的子图是一个DAG，其拓扑排序的一种结果是$\mathrm{RevPostOrder}$。这样各种边的性质就很清楚了：

<x-table id="tab:dfst-edge-properties">

|边的类别|边方向的序性质|数学表示|
|:-|:-|:-|
|非循环边|增加|$\mathrm{RevPostOrder}(\mathrm{Dst}(e))>\mathrm{RevPostOrder}(\mathrm{Src}(e))$|
|自环边|不变|$\mathrm{RevPostOrder}(\mathrm{Dst}(e))=\mathrm{RevPostOrder}(\mathrm{Src}(e))$|
|回边|减少|$\mathrm{RevPostOrder}(\mathrm{Dst}(e))<\mathrm{RevPostOrder}(\mathrm{Src}(e))$|

DFST中各种边的性质

</x-table>

这个性质很有用，比如：如果一条路径从树左边的节点前往了无子孙关系的右边的节点，那么一定是要经过回边的，因为只有回边能使拓扑序减少。

最后，我们很关心的一个问题是，回边集合和非循环边集合会不会因为DFST遍历时边的优先选取而发生变化。遗憾的是，确实可能会发生变化。最小的例子如下图：

<x-figure src="./variant-cycle-set.svg" id="fig:variant-cycle-set">展示了回边集合和非循环边集合会随DFST变化</x-figure>

但如果控制流图可归约，那么DFST对应的回边集合和非循环边集合是不会变的。这也就是第3个性质，具体细节会在归约章节讨论<x-wip></x-wip>。

## 支配树

### 支配的概念

CFG上，节点$x$**支配**节点$y$，是指从入口节点到$y$的每条路径都经过了$x$。由于具有自反性<x-comment>（任何节点都支配自己）</x-comment>、反对称性<x-comment>（否则到$x$需要经过$y$，到$y$需要经过$x$，结果到达$x$或$y$没有有穷的路径）</x-comment>、传递性，这是个偏序关系<x-comment>（故目前看支配关系可以对应到一个DAG）</x-comment>。为了方便，我们认为$x$小于$y$。节点$x$**严格支配**节点$y$就是$x$**支配**$y$且$x\neq y$。

非入口节点$x$的**立即支配者**$y$就是所有严格支配者中极大的<x-comment>（进一步是“最大的”，看接下来的“唯一”）</x-comment>，它是存在的<x-comment>（至少入口节点是其严格支配者）</x-comment>，且唯一的<x-comment>（如果不同的$y$和$z$同时出现在所有入口节点到$x$的路径上，那么一定有$y$严格支配$z$或$z$严格支配$y$。否则就会出现两条路径，一条$y$出现在了$z$之前，另一条$z$出现在了$y$之前，那么拼接一下就可以得到不经过$y$的路径）</x-comment>。

**非入口节点的立即支配者存在且唯一**，和**支配的偏序关系**是支配的两条独立性质。通过这两条性质，就能知道支配关系组成了一个带根树，根即为入口节点。这颗树我们称为**支配树**。

### 支配算法

一个节点的严格支配者满足下面的式子<x-comment>（集合交的单位元是全集，因而两个式子不能合并）</x-comment>：

<x-formula id="eq:dom-rec">$$\begin{cases}\mathrm{StrictDoms}(x)=\varnothing,&\text{if}~x=Entry\\\\\mathrm{StrictDoms}(x)=\bigcap_{y\in\mathrm{Pred}(x)}y\cup\mathrm{StrictDoms}(y),&\text{if}~x\neq Entry\end{cases}$$</x-formula>

#### DAG控制流图的支配算法

<x-ref-formula ref="eq:dom-rec"></x-ref-formula>似乎可以递归。对于DAG控制流图，我们发现这个定义是个**结构递归**<x-comment>（可以找到一种顺序，在求$\mathrm{StrictDoms}(x)$时，对任意$y\in\mathrm{Pred}(x)$，$\mathrm{StrictDoms}(y)$已经求出）</x-comment>，因而对于任意基本块$x$，满足上式的$\mathrm{StrictDoms}(x)$是唯一确定的，并且按照拓扑排序即可在确定的时间内完成计算。在稍后的章节中，我们将看到这个递归算法在忽略回边的情况下，同样适用于可归约图<x-wip></x-wip>。

实际计算中，我们会发现如果用bit vector作为集合，其集合的交并运算非常复杂，并且没有完全利用支配的“树”的性质。注意到：

$$\mathrm{StrictDoms}(x)=\\{\mathrm{idom}(x),\mathrm{idom}(\mathrm{idom}(x)),\dots\\}$$

且该集合是全序的，立即支配者为最大元素：

$$\mathrm{idom}(x)=\max(\mathrm{StrictDoms}(x))$$

因此就可以得到一个时间、空间上更有效的计算方法：

$$\begin{align*}
\mathrm{idom}(x)&=\max(\mathrm{StrictDoms}(x))\\\\&=\max(\bigcap_{y\in\mathrm{Pred}(x)}y\cup\mathrm{StrictDoms}(y))\\\\&=\max(\bigcap_{y\in\mathrm{Pred}(x)}\\{y,\mathrm{idom}(y),\mathrm{idom}(\mathrm{idom}(y)),\dots\\})\\\\&=\mathrm{LCA}(\mathrm{Pred}(x)),~~~~\text{if}~x\neq Entry\end{align*}$$

这里$\mathrm{LCA}$是指支配者树上的最低公共祖先。总结一下，就得到了下面的算法：

<x-card>
<x-algorithm id="lst:dag-dom-tree">DAG支配树算法。</x-algorithm>
<x-pseudo-code for="lst:dag-dom-tree">

1. 将$Entry$节点构成一颗单节点树$DomTree$
2. 循环：如果还存在一个基本块$x$，满足$x\notin DomTree\land\mathrm{Pred}(x)\subseteq DomTree$：
   1. 以$\mathrm{LCA}(\mathrm{Pred}(x))$为父亲，将$x$插入到$DomTree$上

</x-pseudo-code>
</x-card>

实际实现中，给每个节点一个计数器，初始化为入边的数量。使用一个队列，初始化只包含$Entry$。每次处理一个基本块后，遍历出边，减少终止节点的计数器，减到0了，就加入队列。所以不考虑$\mathrm{LCA}(\mathrm{Pred}(x))$的计算量时，复杂度为$\Omicron(|E|)$（$E$为CFG的边集）。

有趣的是，你会发现这个算法是F. Allen求interval算法的一个进阶版。<x-wip>值得深挖！</x-wip>

#### $\mathrm{LCA}$的计算

未加说明的话，以下讨论是对于一般的CFG，不仅限于DAG形式的。

多元素集合上的$\mathrm{LCA}(S)$可以归结为两个变量的$\mathrm{LCA}(a, b)$<x-comment>（因为$\mathrm{LCA}$有结合律）</x-comment>。单元素集合上的$\mathrm{LCA}(\\{x\\})=x$。空集上的$\mathrm{LCA}(\varnothing)$是ill-defined<x-comment>（因为$\mathrm{LCA}$无单位元）</x-comment>。当然，在所有基本块可达的情况下，如果$x\neq Entry$，则$\mathrm{Pred}(x)\neq\varnothing,\text{if}$，所以<x-ref-formula ref="eq:dom-rec"></x-ref-formula>定义良好。接下来就考虑如何快速地求解两个变量的$\mathrm{LCA}(a, b)$。

<x-card>
<x-theorem id="th:dom-post-order">支配树上的祖父子孙关系，在DFST上得到了保留。精确地来说：

$$\begin{cases}\mathrm{Ancestors}\_{DomTree}(x)=\mathrm{Doms}(x)\subseteq\mathrm{Ancestors}\_{DFST}(x)\\\\\mathrm{Descendants}\_{DomTree}(x)=\mathrm{Doms}^{-1}(x)\subseteq\mathrm{Descendants}\_{DFST}(x)\end{cases}$$

</x-theorem>
<x-proof for="th:dom-post-order">从支配的定义得到。</x-proof>
</x-card>

形象地来说，支配者树比DFST更加扁。一个等价的描述是：如果将树的自上而下视作一个偏序关系，那么支配树的偏序关系是DFST的偏序关系的子集；另一个更有趣的描述是：支配树的偏序关系是所有DFST的偏序关系的交。个人最喜欢的描述是：**支配树是DFST中的某些支干重新接到了祖先上组成的新树**。下图是一个例子：

<x-figure src="./dom-dfst-relation.svg" id="fig:dom-dfst-relation">DFST与支配树之间的关系</x-figure>

基于此，我们得到了下面的定理:

<x-card>
<x-theorem id="th:dom-post-order2">若DFST上对基本块$x$的逆后序遍历编号$\mathrm{RevPostOrder}(x)$，则支配树上，其子孙的$\mathrm{RevPostOrder}$大于等于$\mathrm{RevPostOrder}(x)$

$$\forall y(y\in\mathrm{Descendants}\_{DomTree}(x)\rightarrow\mathrm{RevPostOrder}(y)\geq\mathrm{RevPostOrder}(x))$$

</x-theorem>
<x-proof for="th:dom-post-order2">从<x-ref-theorem ref="th:dom-post-order"></x-ref-theorem>和逆后序遍历的性质得到。</x-proof>
</x-card>

有些时候我们就会想能不能得到更强的定理：支配树上也一定存在一种先序遍历<x-comment>（等价表述“逆后序遍历”）</x-comment>的结果和DFST的逆后序遍历一样。答案是否定的，见<x-ref-figure ref="fig:dom-dfst-relation"></x-ref-figure>。

基于<x-ref-theorem ref="th:dom-post-order2"></x-ref-theorem>就有了下面的算法。

<x-card>
<x-algorithm id="lst:dom-lca">支配树上$\mathrm{LCA}(a, b)$的算法。</x-algorithm>
<x-pseudo-code for="lst:dom-lca">

1. 依据DFST上的逆后序遍历对基本块进行编号，记这个编号为$\mathrm{RevPostOrder}(x)$
2. 循环：如果$a\neq b$：
   1. 如果$\mathrm{RevPostOrder}(a)<\mathrm{RevPostOrder}(b)$：
      1. $b\leftarrow\mathrm{Parent}\_{DomTree}(b)$<x-comment>（分支1）</x-comment>
   2. 否则：<x-comment>（一定有$\mathrm{RevPostOrder}(a)>\mathrm{RevPostOrder}(b)$）</x-comment>
      1. $a\leftarrow\mathrm{Parent}\_{DomTree}(a)$<x-comment>（分支2）</x-comment>
3. 返回$a$

</x-pseudo-code>
</x-card>

这个算法不仅简单，而且性能不错，因为能使用连续的数组提高缓存命中率。注意，分支1和分支2可能会交替执行，这其实是上面的“不能得到更强的定理”导致的：如<x-ref-figure ref="fig:dom-dfst-relation"></x-ref-figure>中，$N\_0,\dots,N\_4$插入到支配树后，计算$\mathrm{LCA}(\mathrm{Pred}(N\_5))$即$\mathrm{LCA}(N\_4,N\_3)$时，会：

1. 找到$N\_4$的父亲$N\_1$<x-comment>（分支2）</x-comment>
2. 找到$N\_3$的父亲$N\_0$<x-comment>（分支1）</x-comment>
3. 找到$N\_1$的父亲$N\_0$<x-comment>（分支2）</x-comment>
4. 得出：$\mathrm{LCA}(N\_4,N\_3) = N\_0$

不过似乎实验证明

#### 一般控制流图的支配算法

对于一般的控制流图，<x-ref-formula ref="eq:dom-rec"></x-ref-formula>不是结构递归，甚至单单这个式子，作为方程都无法确定唯一解，下面是个最小的例子：

<x-figure src="./non-dag-dom.svg" id="fig:non-dag-dom">一般控制流图<x-ref-formula ref="eq:dom-rec"></x-ref-formula>无唯一解示例</x-figure>

<x-ref-figure ref="fig:non-dag-dom"></x-ref-figure>中的例子提醒我们，如果采用迭代算法，算法的最终不动点和初值有关，且某些不动点是错误的解<x-comment>（不完整的解）</x-comment>。那么什么解是正确的呢？其实观察上面的两个解，你就可以猜到是那个偏序关系最丰富，树最高的那个解。这里给出其形式化描述以及证明。

<x-card>
<x-theorem id="th:dom-solution-max">对一般的CFG，求解下面的式子：

$$\begin{cases}\mathrm{X}(x)=\varnothing,&\text{if}~x=Entry\\\\\mathrm{X}(x)=\bigcap_{y\in\mathrm{Pred}(x)}y\cup\mathrm{X}(y),&\text{if}~x\neq Entry\end{cases}$$

会得到若干解，那么对于任意基本块$x$，每个解一定满足：

$$\mathrm{X}(x)\subseteq\mathrm{StrictDoms}(x)$$

</x-theorem>
<x-proof for="th:dom-solution-max"><x-comment>注意：证明非结构递归时，不能采用数学归纳法，因为那很可能是循环论证。</x-comment>反证法，假设存在$y$，满足$y\in\mathrm{X}(x)\land y\neq\mathrm{StrictDoms}(x)$。由于$y\neq\mathrm{StrictDoms}(x)$，存在路径$Entry=w\\_0\rightarrow w\\_1\rightarrow\cdots\rightarrow w\\_k=x$，且除终点外路径不经过$y$，即$y\notin\{w\\_i\mid 0\leq i\leq k-1\}$。通过有限次<x-comment>（强调！）</x-comment>的交换律和结合律，可以得到下面的式子：

$$\mathrm{X}(x)=(w\_{k-1}\cup((w\_{k-2}\cup((\cdots)\cap\cdots))\cap\cdots$$

因此，我们可以得到：

$$\mathrm{X}(x)\subseteq\\{w\_i\mid 0\leq i\leq k-1\\}$$

故$y\notin\mathrm{X}(x)$，与假设矛盾。

</x-proof>
</x-card>

因此我们可以给出一个迭代算法的思路。先用一个**偏大的集合**初始化所有$\mathrm{StrictDoms}$。而后不断迭代，不断缩小$\mathrm{StrictDoms}$，剔除其中肯定错误的元素。那么，这个**偏大的集合**可以是什么呢？所有CFG节点自然是一个很粗暴且正确的想法。不过你的大脑如果还在运作的话，可能会回想起<x-ref-theorem ref="th:dom-post-order"></x-ref-theorem>。是的，支配树的偏序关系是DFST的子集，用DFST作为初始的支配树是完全可行的。

<!--
          First Pass  Second Pass
{5}       {5}         {5}
{5,4}     {5,4}       {5,4}
{5,3}     {5,3}       {5,3}
{5,3,2}   {5,3,2}     {5,2}
{5,3,2,1} {5,1}       {5,1}

 5
/ \
4  3
   |
   2
   |
   1

  5
/ | \
4 3  1
  |
  2

   5
/ / \ \
4 3  1 2
-->

<!-- markdownlint-disable -->
<style scoped>
x-card {
    display: block;
    box-shadow: 0 0.5px 3px rgba(0,0,0,0.3);
    padding: 1rem;
    border-radius: .2rem;
    margin: 1rem 0;
}

x-card > *:last-child, x-card > * > *:last-child {
    margin-bottom: 0;
}

x-comment {
    opacity: .5;
}

x-wip, .warning {
    background-color: #ffeb3b;
    overflow: visible;
    padding: .1em;
    margin: 0 .1em;
    border-radius: .2em;
    box-shadow: 0 0 2px #9d8c00;
}

.warning sup {
    opacity: .5;
}

x-wip {
    color: blue !important;
}

x-wip::before {
    content: '🚧（施工中）'
}

.warning, .warning {
    color: red !important;
}

.warning::before {
    content: '⚠️'
}

x-theorem, x-proof, x-algorithm, x-pseudo-code {
    display: block;
    margin: .5em 0;
    padding-top: 6rem;
    margin-top: -6rem;
}

x-theorem .theorem-label, x-proof .proof-label {
    font-weight: bold;
    margin-right: .5em;
    color: darkblue;
}

x-proof .proof-qed {
    display: block;
    border: 1px solid black;
    width: .9em;
    height: .9em;
    margin : .3em .3em .3em auto;
}

x-ref-theorem, x-ref-algorithm, x-ref-figure, x-ref-table, x-ref-formula {
    display: inline-block;
}

x-ref-theorem .ref-theorem-label {
    color: darkblue;
}

x-algorithm .algorithm-label, x-pseudo-code .pseudo-code-label {
    font-weight: bold;
    margin-right: .5em;
    color: darkgreen;
}

x-ref-algorithm .ref-algorithm-label {
    color: darkgreen;
}

figure {
    padding-top: 6rem;
    margin-top: -4rem;
}

img + figcaption, table + figcaption {
    margin-top: 1em;
}

figure .figure-label {
    font-weight: bold;
    margin-right: .5em;
    color: darkmagenta;
}

x-ref-figure .ref-figure-label {
    color: darkmagenta;
}

figure .table-label {
    font-weight: bold;
    margin-right: .5em;
    color: darkred;
}

x-ref-table .ref-table-label {
    color: darkred;
}

table:not(.lntable) {
    display: block;
    margin: 0 auto;
    width: auto;
    max-width: 100%;
    overflow-x: auto;
}

table:not(.lntable) thead {
    border-top: 2px solid currentColor;
}

table:not(.lntable) tr:last-child {
    border-bottom: 2px solid currentColor;
}

.katex-display {
    overflow-x: auto;
}

.blog-content p {
    text-align: justify;
}

x-formula {
    display: flex;
    align-items: center;
    padding-top: 6rem;
    margin-top: -6rem;
    flex-wrap: wrap;
}

x-formula > span:first-child {
    display: block;
    flex-grow: 1;
    max-width: 100%;
}

x-formula .formula-label {
    display: block;
    white-space: nowrap;
    margin-left: auto;
    color: darkcyan;
}

x-formula .formula-label::before {
  content: "(";
  padding-right: .1em;
}

x-formula .formula-label::after {
  content: ")";
  padding-left: .1em;
}

x-ref-formula .ref-formula-label {
    color: darkcyan;
}
</style>

<script>
class XCard extends HTMLElement {
    constructor() {
        super();
    }
}

class XWip extends HTMLElement {
    constructor() {
        super();
    }
}

class XComment extends HTMLElement {
    constructor() {
        super();
    }
}

class XWarning extends HTMLElement {
    constructor() {
        super();
        this.classList.add('warning');
        const comment = this.getAttribute('comment');
        if (comment) {
            const sup = document.createElement('sup');
            sup.appendChild(document.createTextNode(`[${comment}]`));
            this.appendChild(sup);
        }
    }
}

const theoremNames = [];
const theoremIndices = {};

class XTheorem extends HTMLElement {
    constructor() {
        super();
        const index = theoremNames.length;
        const name = this.getAttribute('id') || `theorem-${index + 1}`
        this.setAttribute('id', name);
        theoremNames.push(name);
        theoremIndices[name] = index;
        const label = document.createElement('a');
        label.appendChild(document.createTextNode(`定理 ${index + 1}`));
        label.setAttribute('href', `#${name}`);
        label.classList.add('theorem-label');
        this.insertBefore(label, this.firstChild);
    }
}

class XProof extends HTMLElement {
    constructor() {
        super();
        const name = this.getAttribute('for');
        const label = document.createElement('a');
        label.classList.add('proof-label');
        if (name === null || theoremIndices[name] === undefined) {
            label.appendChild(document.createTextNode('未知证明'));
            label.classList.add('warning');
        } else {
            const index = theoremIndices[name];
            label.appendChild(document.createTextNode(`证明 ${index + 1}`));
            label.setAttribute('href', `#${name}`);
        }
        this.insertBefore(label, this.firstChild);
        // QED.
        const tail = document.createElement('div');
        tail.classList.add('proof-qed');
        this.appendChild(tail);
    }
}

class XRefTheorem extends HTMLElement {
    constructor() {
        super();
        const name = this.getAttribute('ref');
        const label = document.createElement('a');
        label.classList.add('ref-theorem-label');
        if (name === null || theoremIndices[name] === undefined) {
            label.appendChild(document.createTextNode('未知定理'));
            label.classList.add('warning');
        } else {
            const index = theoremIndices[name];
            label.appendChild(document.createTextNode(`定理 ${index + 1}`));
            label.setAttribute('href', `#${name}`);
        }
        this.appendChild(label);
        this.classList.add('ref-theorem');
    }
}

const algorithmNames = [];
const algorithmIndices = {};

class XAlgorithm extends HTMLElement {
    constructor() {
        super();
        const index = algorithmNames.length;
        const name = this.getAttribute('id') || `algorithm-${index + 1}`;
        this.setAttribute('id', name);
        algorithmNames.push(name);
        algorithmIndices[name] = index;
        const label = document.createElement('a');
        label.appendChild(document.createTextNode(`算法 ${index + 1}`));
        label.setAttribute('href', `#${name}`);
        label.classList.add('algorithm-label');
        this.insertBefore(label, this.firstChild);
    }
}

class XPseudoCode extends HTMLElement {
    constructor() {
        super();
        const name = this.getAttribute('for');
        const label = document.createElement('a');
        if (name === null || algorithmIndices[name] === undefined) {
            label.appendChild(document.createTextNode('未知伪码'));
            this.insertBefore(label, this.firstChild);
            label.classList.add('warning');
        } else {
            const index = algorithmIndices[name];
            label.appendChild(document.createTextNode(`伪码 ${index + 1}`));
            label.setAttribute('href', `#${name}`);
        }
        label.classList.add('pseudo-code-label');
        this.insertBefore(label, this.firstChild);
    }
}

class XRefAlgorithm extends HTMLElement {
    constructor() {
        super();
        const label = document.createElement('a');
        const name = this.getAttribute('ref');
        label.classList.add('ref-algorithm-label');
        if (name === null || algorithmIndices[name] === undefined) {
            label.appendChild(document.createTextNode('未知算法'));
            label.classList.add('warning');
        } else {
            const index = algorithmIndices[name];
            label.appendChild(document.createTextNode(`算法 ${index + 1}`));
            label.setAttribute('href', `#${name}`);
        }
        this.appendChild(label);
        this.classList.add('ref-algorithm');
    }
}

const figureNames = [];
const figureIndices = {};

class XFigure extends HTMLElement {
    constructor() {
        super();
        const index = figureNames.length;
        const name = this.getAttribute('id') || `figure-${index + 1}`;
        this.setAttribute('id', name);
        figureNames.push(name);
        figureIndices[name] = index;
        const figure = document.createElement('figure');
        const img = document.createElement('img');
        img.setAttribute('src', this.getAttribute('src'));
        img.setAttribute('alt', this.getAttribute('alt') || this.innerText);
        figure.appendChild(img);
        const figcaption = document.createElement('figcaption');
        const label = document.createElement('a');
        label.appendChild(document.createTextNode(`图片 ${index + 1}`));
        label.setAttribute('href', `#${name}`);
        label.classList.add('figure-label');
        figcaption.appendChild(label);
        const children = Array.from(this.childNodes);
        children.forEach(child => {
            child.remove();
            figcaption.appendChild(child);
        });
        figure.appendChild(figcaption);
        this.appendChild(figure);
    }
}

class XRefFigure extends HTMLElement {
    constructor() {
        super();
        const label = document.createElement('a');
        const name = this.getAttribute('ref');
        label.classList.add('ref-figure-label');
        if (name === null || figureIndices[name] === undefined) {
            label.appendChild(document.createTextNode('未知图片'));
            label.classList.add('warning');
        } else {
            const index = figureIndices[name];
            label.appendChild(document.createTextNode(`图片 ${index + 1}`));
            label.setAttribute('href', `#${name}`);
        }
        this.appendChild(label);
    }
}

const tableNames = [];
const tableIndices = {};

class XTable extends HTMLElement {
    constructor() {
        super();
        const index = tableNames.length;
        const name = this.getAttribute('id') || `table-${index + 1}`;
        this.setAttribute('id', name);
        tableNames.push(name);
        tableIndices[name] = index;
        const figure = document.createElement('figure');
        const figcaption = document.createElement('figcaption');
        const label = document.createElement('a');
        label.appendChild(document.createTextNode(`表格 ${index + 1}`));
        label.setAttribute('href', `#${name}`);
        label.classList.add('table-label');
        figcaption.appendChild(label);
        const children = Array.from(this.childNodes);
        children.forEach(child => {
            child.remove();
            if (child.tagName === 'TABLE') {
                figure.appendChild(child);
            } else if (child.tagName === 'P') {
                Array.from(child.childNodes).forEach(child => figcaption.appendChild(child));
            }
        });
        figure.appendChild(figcaption);
        this.appendChild(figure);
    }
}

class XRefTable extends HTMLElement {
    constructor() {
        super();
        const label = document.createElement('a');
        const name = this.getAttribute('ref');
        label.classList.add('ref-table-label');
        if (name === null || tableIndices[name] === undefined) {
            label.appendChild(document.createTextNode('未知表格'));
            label.classList.add('warning');
        } else {
            const index = tableIndices[name];
            label.appendChild(document.createTextNode(`表格 ${index + 1}`));
            label.setAttribute('href', `#${name}`);
        }
        this.appendChild(label);
    }
}

const formulaNames = [];
const formulaIndices = {};

class XFormula extends HTMLElement {
    constructor() {
        super();
        const index = formulaNames.length;
        const name = this.getAttribute('id') || `formula-${index + 1}`;
        this.setAttribute('id', name);
        formulaNames.push(name);
        formulaIndices[name] = index;
        const label = document.createElement('a');
        label.appendChild(document.createTextNode(`${index + 1}`));
        label.setAttribute('href', `#${name}`);
        label.classList.add('formula-label');
        this.appendChild(label);
    }
}

class XRefFormula extends HTMLElement {
    constructor() {
        super();
        const name = this.getAttribute('ref');
        const label = document.createElement('a');
        label.classList.add('ref-formula-label');
        if (name === null || formulaIndices[name] === undefined) {
            label.appendChild(document.createTextNode('未知公式'));
            label.classList.add('warning');
        } else {
            const index = formulaIndices[name];
            label.appendChild(document.createTextNode(`公式 ${index + 1}`));
            label.setAttribute('href', `#${name}`);
        }
        this.appendChild(label);
        this.classList.add('ref-formula');
    }
}

customElements.define('x-card', XCard);
customElements.define('x-wip', XWip);
customElements.define('x-comment', XComment);
customElements.define('x-warning', XWarning);
customElements.define('x-theorem', XTheorem);
customElements.define('x-proof', XProof);
customElements.define('x-ref-theorem', XRefTheorem);
customElements.define('x-algorithm', XAlgorithm);
customElements.define('x-pseudo-code', XPseudoCode);
customElements.define('x-ref-algorithm', XRefAlgorithm);
customElements.define('x-figure', XFigure);
customElements.define('x-ref-figure', XRefFigure);
customElements.define('x-table', XTable);
customElements.define('x-ref-table', XRefTable);
customElements.define('x-formula', XFormula);
customElements.define('x-ref-formula', XRefFormula);
</script>
