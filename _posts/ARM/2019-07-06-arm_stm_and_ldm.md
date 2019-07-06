---
layout: post

title:  ARM中的STM和LDM指令的解析
date:   2019-07-06 14:00:00 +0800
categories: arm
tag: arm
---

* content
{:toc}


STM指令
=================
STM是Store Multipile registers到连续的存储空间中，并且是按寄存器编号从低到高放置在存储空间的从低到高的位置中。
> 格式： STM[xx] Rn[!], registers

Rn为base地址，后面！表示是否将放置后的地址回写到base寄存器中。registers是要放置的寄存器组，xx表示放置的方式。

xx用于确认放置根据下面两个条件区分：
1.base地址之前和之后;
2.地址是放置前增加还是后增加。

由于都是放置在连续空间中，寄存器的放置顺序固定，所以xx就是用于确定放置空间的起始地址，该地址用address表示，根据xx来看下address的计算方法，下面len(registers)表示registers中的个数。

| xx | name | 介绍 | 
|:---:|:-----:|:-----:|
|IA(EA) | Increment After(Empty Ascending) | 放置后地址增加，address = Rn |
|DA(ED) | Decrement After(Empty Descending) | 放置后地址减少，address = Rn - len(registers) + 4 |
|DB(FD) | Decrement Before(Full Descending) | 放置前地址减少，addrss = Rn - len(registers) |
|IB(FA) | Increment Before(Full Ascending) | 放置前地址增加，address = Rn + 4 |

## STMDA(STMED)
来看下为什么 address = Rn - len(registers) + 4 ?

* 放置后地址减少;
* 寄存器编号按从低到高放置在存储空间的从低到高的位置;

所以先倒数第一的寄存器需要放置在base地址中，即mem[Rn] = R[end], 接下来倒数第二为 mem[Rn - 4] = R[end-1], 类似第一个就是mem[Rn - len(registers) + 4] = R[first]了。 

## 回写base寄存器

若需要回写base寄存器
* IA和IB是增加的方向，不管是放置后还是前增加都是Rn = Rn + len(registers) + 4
* DA和DB是减少的方向，不管是放置后还是前增加都是Rn = Rn - len(registers) + 4

LDM指令
==================
和STM相反，从连续的存储空间中读取内容到寄存器中。并且确认起始地址的方式也一样的，区别是放入到存储还是从存储中读。

