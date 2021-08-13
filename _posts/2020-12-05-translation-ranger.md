---
layout: post
title: "Translation Ranger: 通过操作系统支持连续性感知的旁路转换缓存(TLB)"
author: Weizhou Huang
tags:
 - ISCA
 - 2019
 - Virtual memory
 - Huge pages
 - TLB
---

# [Translation Ranger: Operating System Support for Contiguity-Aware TLBs](https://www.cs.yale.edu/homes/abhishek/ziyan-isca19.pdf) 

本文提供了一种通过迁移物理内存页来生成连续性映射的机制，用于提升连续性感知的TLB生成大页映射的概率，以提升TLB缓存和内存地址转换效率。

## 背景和问题：
当多个连续的4kb虚拟页映射到多个连续的物理页时，新型的连续性感知的TLB能够自动合并多个这样的映射条目为一个，从而增加TLB命中率.
如图1所示，左侧是连续性感知的TLB，右侧是传统的TLB，可以看到V0-V3的映射以及V12-V15的映射由于他们对应的物理页也是连续的，由于传统的TLB只能记录4KB,2MB,1GB的映射条目，图中的两段连续性映射需要占用8个TLB条目。
相比于传统TLB，新型的连续性感知的TLB可以用过将多个4KB映射为条目合并成为一个，使一个条目覆盖的内存段大小远超传统TLB，从而减少映射条目数量，增加TLB命中率。

<center>
<img src="/images/translation-ranger-contiguity-aware_TLB.png" width="70%" />

图 1  连续性感知的TLB
</center>

为了能够进一步提升虚拟内存机制的地址转换效率，许多研究提出通过创造逻辑到物理地址映射的连续性以充分利用这种连续性感知的TLB 减少映射条目数量，实现更高的地址转换效率

**问题在于**，现有研究只能在有限条件下创造这种连续性
表1列出了现有研究在创造连续性的条件，软件需求是对操作系统内存分配机制buddy allocator的依赖，是否需要预留内存空间用于分配大段物理内存，以及是否需要修改页表，硬件需求是对TLB功能的依赖，以及最右侧展示的是能提供怎样的连续性，下面我对其中四个主要的相关研究进行说明：

- 现有Linux操作系统已经支持的大页映射(THP)，但是它只对2MB和1GB粒度的连续映射进行合并，缺乏灵活性。
- Direct Segment难以应用于现实应用，因为他基于连续映射机制，而现实应用中的大段映射不一定连续，即无法完成多段内存映射的拼接，此外这种方法还需要显示的编程介入
- Redundant Memory Mappings的实现需要大量的操作系统修改
- Devirtualizing Memory的设计基于一个理想化的假设：操作系统总能提供较大的连续物理内存，因此在内存碎片化情况下表现不佳，但是随着系统运行时间增加，由于内存碎片化，可用的1GB连续空间数量大量减少，从而减少了创造连续性的机会.


<center>
<img src="/images/translation-ranger-对比.png" width="100%"  />

表 1  现有相关工作的局限性
</center>

## 研究动机：

本文的研究动机在于提出一种机制能够在任何实际应用场景下，为现有的连续性感知的TLB提供可以合并地址映射条目的机会，具体的要求同时也是设计的目标有：

- 为更大粒度的连续空间提供大页映射
- 在内存碎片化程度严重时也能够支持任一尺寸的连续性
- 不受负载的数量以及模式影响，在系统运行的任意时刻能够提供连续性

## 设计：

**Translation Ranger的设计思想：**
本设计要同时在虚拟和物理空间中创造连续性，在LINUX中，每一个虚拟地址范围有vm_area_struct(VMA)记录，应用通过mmap或malloc获取连续的虚拟地址，但是物理空间不一定是连续的。

本设计的**基本思想**是重新分配物理页的映射，在理想情况下，实现每一个VMA区域只使用一个PTE映射
实现方法基本分为三步：
1. 为每一个VMA分配一个anchor point，如图2中红色箭头所示，这个anchor point确定了虚拟页和物理页的起始地址
2. 基于每一个VMA的anchor point，动态合并物理内存:如图2左侧所示，VMA一开始指向分散的4个物理页，通过将P2迁移到P4，交换P5和P9，交换P7和P6，交换P7和P3，最终达到图2右侧所示的状态
3. 整个设计作为后台守护进程，定期迭代所有的仍然活跃的VMA，在vma的整个生命周期中， 为其维护地址映射的连续性 

基于以上设计思想，本文还解决了三个设计挑战：anchor point选取；在迁移VMA对应的物理内存页时，如何处理正在使用或者不迁移的页面；如何解决VMA的大小发生变化导致物理空间交叠或者浪费的问题。
具体细节请参考原文。



<center>
<img src="/images/translation-ranger-物理页迁移.png" width="75%" />

图 2  Translation Ranger工作原理
</center>

## 实验平台：
基于Linux kernel v4.16修改