---
title: "Packrat记忆化Parser与Recoverable Parser"
description: "本文主要介绍Tilly的前端所用到的技术。包括记忆化的Packrat Parser（包括它们带来的增量和解决左递归的方法）、Parser Combinator和基于PEG的Recoverable Parser。"
lead: "本文主要介绍Tilly的前端所用到的技术。包括记忆化的Packrat Parser（包括它们带来的增量和解决左递归的方法）、Parser Combinator和基于PEG的Recoverable Parser。"
date: 2021-06-29T17:19:38+08:00
lastmod: 2021-06-29T17:19:38+08:00
draft: false
weight: 50
images: []
contributors: [szp]
---

## Packrat Parser用于解决左递归[^warth2008packrat]

### 算法

全局变量：

- $Pos: \mathrm{P{\scriptsize OSITION}}$
- $\mathrm{H{\scriptsize EADS}}: \mathrm{P{\scriptsize OSITION}}\to\mathrm{H{\scriptsize EAD}}$
- $LRStack:\mathrm{LR}~\text{or}~\mathrm{\scriptsize NIL}$ <span class="comment">（链表）</span>
- $\mathrm{M{\scriptsize EMO}}:(\mathrm{R{\scriptsize ULE}},\mathrm{P{\scriptsize OSITION}})\rightarrow\mathrm{M{\scriptsize EMO}E{\scriptsize NTRY}}$

其中：

- $\mathrm{LR}=(seed:\mathrm{AST},rule:\mathrm{R{\scriptsize ULE}},head:\mathrm{H{\scriptsize EAD}}~\text{or}~\mathrm{\scriptsize NIL},next:\mathrm{lR})$
- $\mathrm{H{\scriptsize EAD}}=(rule: \mathrm{R{\scriptsize ULE}},involvedSet:\mathrm{S{\scriptsize ET}}~\text{of}~\mathrm{R{\scriptsize ULE}},evalSet:\mathrm{S{\scriptsize ET}}~\text{of}~\mathrm{R{\scriptsize ULE}})$
- $\mathrm{M{\scriptsize EMO}E{\scriptsize NTRY}}=(ans: \mathrm{AST}~\text{or}~\mathrm{LR},pos:\mathrm{P{\scriptsize OSITION}})$

#### $\mathrm{G{\scriptsize ROW}\text{-}LR}(R, P, M, H)$

参数：

- $R: \mathrm{R{\scriptsize ULE}}$
- $P: \mathrm{P{\scriptsize OSITION}}$
- $M: \mathrm{M{\scriptsize EMO}E{\scriptsize NTRY}}$
- $H: \mathrm{H{\scriptsize EAD}}$

返回值：

- $\mathrm{AST}$

过程：

- $\mathrm{H{\scriptsize EADS}}(P)\leftarrow H$
- $\mathbf{while}~\mathrm{\scriptsize TRUE}$
  - $Pos\leftarrow P$
  - $H.evalSet\leftarrow\mathrm{C{\scriptsize OPY}}(H.involvedSet)$
  - $\mathbf{let}~ans=\mathrm{E{\scriptsize VAL}}(R.body)$
  - $\mathbf{if}~ans=\mathrm{\scriptsize FAIL}~\text{or}~Pos\leq M.pos$
    - $\mathbf{break}$
  - $M.ans\leftarrow ans$
  - $M.pos\leftarrow Pos$
- $\mathrm{H{\scriptsize EADS}}(P)\leftarrow\mathrm{\scriptsize NIL}$
- $Pos\leftarrow M.pos$
- $\mathbf{return}~M.ans$
{.list-code}

#### $\mathrm{A{\scriptsize PPLY}\text{-}R{\scriptsize LUE}}(R, P)$

参数：

- $R: \mathrm{R{\scriptsize ULE}}$
- $P: \mathrm{P{\scriptsize OSITION}}$

返回值：

- $\mathrm{AST}$

过程：

- $\mathbf{let}~m=\mathrm{R{\scriptsize ECALL}}(R, P)$
- $\mathbf{if}~m=\mathrm{\scriptsize NIL}$
  - <span class="comment">$\triangleright$创建一个新的$\mathrm{LR}$，并入栈</span>
  - $\mathbf{let}~lr=\mathbf{new}~\mathrm{LR}(\mathrm{\scriptsize FAIL},R,\mathrm{\scriptsize NIL},LRStack)$
  - $LRStack\leftarrow lr$
  - <span class="comment">$\triangleright$记忆化$lr$，并解析$R$</span>
  - $m\leftarrow\mathbf{new}~\mathrm{M{\scriptsize EMO}E{\scriptsize NTRY}}(lr,P)$
  - $\mathrm{M{\scriptsize EMO}}(R,P)\leftarrow m$
  - $\mathbf{let}~ans=\mathrm{E{\scriptsize VAL}}(R.body)$
  - <span class="comment">$\triangleright lr$出栈</span>
  - $LRStack\leftarrow LRStack.next$
  - $m.pos\leftarrow Pos$
  - $\mathbf{if}~lr.head\neq\mathrm{\scriptsize NIL}$
    - $lr.seed\leftarrow ans$
    - $\mathbf{return}~\mathrm{LR}\text{-}\mathrm{A{\scriptsize NSWER}}(R,P,m)$
  - $\mathbf{else}$
    - $m.ans\leftarrow ans$
    - $\mathbf{return}~ans$
- $\mathbf{else}$
  - $Pos\leftarrow m.pos$
  - $\mathbf{if}~m.ans~\text{is}~\mathrm{LR}$
    - $\mathrm{S{\scriptsize ETUP}}\text{-}\mathrm{LR}(R,m.ans)$
    - $\mathbf{return}~m.ans.seed$
  - $\mathbf{else}$
    - $\mathbf{return}~m.ans$
{.list-code}

#### $\mathrm{S{\scriptsize ETUP}\text{-}LR}(R, L)$

参数：

- $R: \mathrm{R{\scriptsize ULE}}$
- $L: \mathrm{LR}$

过程：

- $\mathbf{if}~L.head=\mathrm{\scriptsize NIL}$
  - $L.head\leftarrow\mathbf{new}~\mathrm{H\scriptsize{EAD}}(R,\\{\\},\\{\\})$
- $\mathbf{let}~s=LRStack$
- $\mathbf{while}~s.head\neq L.head$
  - $s.head\leftarrow L.head$
  - $L.head.involvedSet\leftarrow L.head.involvedSet\cup\\{s.rule\\}$
  - $s\leftarrow s.next$
{.list-code}

#### $\mathrm{LR\text{-}A{\scriptsize NSWER}}(R, P, M)$

参数：

- $R: \mathrm{R{\scriptsize ULE}}$
- $P: \mathrm{P{\scriptsize OSITION}}$
- $M: \mathrm{M{\scriptsize EMO}E{\scriptsize NTRY}}$，其中$M.ans~\mathbf{is}~\mathrm{LR1}$

返回值：

- $\mathrm{AST}$

过程：

- $\mathbf{let}~h=M.ans.head$
- $\textbf{if}~h.rule\neq R$
  - $\textbf{return}~M.ans.seed$
- $\textbf{else}$
  - $M.ans\leftarrow M.ans.seed$
  - $\textbf{if}~M.ans=\mathrm{\scriptsize FAIL}$
    - $\textbf{return}~\mathrm{\scriptsize FAIL}$
  - $\textbf{else}$
    - $\textbf{return}~\mathrm{G{\scriptsize ROW}\text{-}LR}(R, P, M, h)$
{.list-code}

#### $\mathrm{R{\scriptsize ECALL}}(R, P)$

参数：

- $R: \mathrm{R{\scriptsize ULE}}$
- $P: \mathrm{P{\scriptsize OSITION}}$

返回值：

- $\mathrm{M{\scriptsize EMO}E{\scriptsize NTRY}}~\text{or}~\mathrm{\scriptsize NIL}$

过程：

- $\textbf{let}~m=\mathrm{M{\scriptsize EMO}}(R, P)$
- $\textbf{let}~h=\mathrm{H{\scriptsize EADS}}(P)$
- <span class="comment">$\triangleright$如果不是在扩展种子的阶段，直接返回存储的值</span>
- $\textbf{if}~h=\mathrm{\scriptsize NIL}$
  - $\mathbf{return}~m$
- <span class="comment">$\triangleright$不要计算那些这次左递归没涉及的规则</span>
- $\textbf{if}~m=\mathrm{\scriptsize NIL}~\text{and}~R\notin\\{h.read\\}\cup h.involvedSet$
  - $\mathbf{return}~\mathbf{new}~\mathrm{M{\scriptsize EMO}E{\scriptsize NTRY}}(\mathrm{\scriptsize FAIL},P)$
- <span class="comment">$\triangleright$只允许涉及的规则被计算一次</span>
- $\textbf{if}~R\in h.evalSet$
  - $h.evalSet\leftarrow h.evalSet\setminus\\{R\\}$
  - $\textbf{let}~ans=\mathrm{E{\scriptsize VAL}}(R.body)$
  - $m.ans\leftarrow ans$
  - $m.pos\leftarrow pos$
- $\textbf{return}~m$
{.list-code}

### 合并后的算法

参数：

- $R: \mathrm{R{\scriptsize ULE}}$
- $P: \mathrm{P{\scriptsize OSITION}}$

返回值：

- $\mathrm{AST}$

过程：

- $\textbf{let}~m=\mathrm{M{\scriptsize EMO}}(R, P)$
- $\textbf{let}~h=\mathrm{H{\scriptsize EADS}}(P)$
- <span class="comment">$\triangleright$如果不是在扩展种子的阶段，直接返回存储的值</span>
- $\textbf{if}~h\neq\mathrm{\scriptsize NIL}$
  - <span class="comment">$\triangleright$不要计算那些这次左递归没涉及的规则</span>
  - $\textbf{if}~m=\mathrm{\scriptsize NIL}~\text{and}~R\notin\\{h.read\\}\cup h.involvedSet$
    - $m\leftarrow~\mathbf{new}~\mathrm{M{\scriptsize EMO}E{\scriptsize NTRY}}(\mathrm{\scriptsize FAIL},P)$
  - <span class="comment">$\triangleright$只允许涉及的规则被计算一次</span>
  - $\textbf{elif}~R\in h.evalSet$
    - $h.evalSet\leftarrow h.evalSet\setminus\\{R\\}$
    - $\textbf{let}~ans=\mathrm{E{\scriptsize VAL}}(R.body)$
    - $m.ans\leftarrow ans$
    - $m.pos\leftarrow pos$
- $\mathbf{if}~m=\mathrm{\scriptsize NIL}$
  - <span class="comment">$\triangleright$创建一个新的$\mathrm{LR}$，并入栈</span>
  - $\mathbf{let}~lr=\mathbf{new}~\mathrm{LR}(\mathrm{\scriptsize FAIL},R,\mathrm{\scriptsize NIL},LRStack)$
  - $LRStack\leftarrow lr$
  - <span class="comment">$\triangleright$记忆化$lr$，并解析$R$</span>
  - $m\leftarrow\mathbf{new}~\mathrm{M{\scriptsize EMO}E{\scriptsize NTRY}}(lr,P)$
  - $\mathrm{M{\scriptsize EMO}}(R,P)\leftarrow m$
  - $\mathbf{let}~ans=\mathrm{E{\scriptsize VAL}}(R.body)$
  - <span class="comment">$\triangleright lr$出栈</span>
  - $LRStack\leftarrow LRStack.next$
  - $m.pos\leftarrow Pos$
  - $\mathbf{if}~lr.head\neq\mathrm{\scriptsize NIL}$
    - $lr.seed\leftarrow ans$
    - $\mathbf{let}~h=m.ans.head$
    - $\textbf{if}~h.rule\neq R$
      - $\textbf{return}~m.ans.seed$
    - $\textbf{else}$
      - $m.ans\leftarrow m.ans.seed$
      - $\textbf{if}~m.ans=\mathrm{\scriptsize FAIL}$
        - $\textbf{return}~\mathrm{\scriptsize FAIL}$
      - $\textbf{else}$
        - $\mathrm{H{\scriptsize EADS}}(P)\leftarrow h$
        - $\mathbf{while}~\mathrm{\scriptsize TRUE}$
          - $Pos\leftarrow P$
          - $h.evalSet\leftarrow\mathrm{C{\scriptsize OPY}}(h.involvedSet)$
          - $\mathbf{let}~ans=\mathrm{E{\scriptsize VAL}}(R.body)$
          - $\mathbf{if}~ans=\mathrm{\scriptsize FAIL}~\text{or}~Pos\leq m.pos$
            - $\mathbf{break}$
          - $m.ans\leftarrow ans$
          - $m.pos\leftarrow Pos$
        - $\mathrm{H{\scriptsize EADS}}(P)\leftarrow\mathrm{\scriptsize NIL}$
        - $Pos\leftarrow m.pos$
        - $\mathbf{return}~m.ans$
  - $\mathbf{else}$
    - $m.ans\leftarrow ans$
    - $\mathbf{return}~ans$
- $\mathbf{else}$
  - $Pos\leftarrow m.pos$
  - $\mathbf{if}~m.ans~\text{is}~\mathrm{LR}$
    - $\mathbf{let}~L=m.ans$
    - $\mathbf{if}~L.head=\mathrm{\scriptsize NIL}$
      - $L.head\leftarrow\mathbf{new}~\mathrm{H\scriptsize{EAD}}(R,\\{\\},\\{\\})$
    - $\mathbf{let}~s=LRStack$
    - $\mathbf{while}~s.head\neq L.head$
      - $s.head\leftarrow L.head$
      - $L.head.involvedSet\leftarrow L.head.involvedSet\cup\\{s.rule\\}$
      - $s\leftarrow s.next$
    - $\mathbf{return}~L.seed$
  - $\mathbf{else}$
    - $\mathbf{return}~m.ans$
{.list-code}

[^warth2008packrat]: Warth, Alessandro, James R. Douglass, and Todd Millstein. "Packrat parsers can support left recursion." *Proceedings of the 2008 ACM SIGPLAN symposium on Partial evaluation and semantics-based program manipulation*. 2008.

<style scoped>
ul.list-code, ul.list-code ul {
    list-style-type: none;
    padding-left: 2em;
}
ul.list-code {
    padding: 1em;
    border-radius: 0.5em;
    box-shadow: 0 0 3px rgba(0, 0, 0, 0.4);
}

.list-code .comment {
    opacity: 0.4;
}
</style>
