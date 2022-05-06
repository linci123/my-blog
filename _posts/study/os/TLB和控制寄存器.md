---
title: 'TLB和控制寄存器'
author: Kevin。
tags:
  - os
  - Windows
categories:
  - study
date: 2022-04-27 13:18:00
img: /images/article-banner/QQ截图20220427213008.jpg
typora-root-url: ..\..\..\
---
# TLB和控制寄存器

## TLB

TLB（Translation Lookaside Buffer）中文一般翻译为**转译后备缓冲区**，也就是**页表缓存**，用来存放线性地址相对应的物理地址及其相关信息。TLB是在MMU中包括的一段小的缓存，比内存快了十几倍不等，但造价更贵，设计TLB时需要权衡**命中速度**与**线性地址转物理地址速度**还有**成本**之间的关系， 一般128、256、512条页表条目。

线性地址转物理地址先要从CR3读取PDE，再从PDE读取PTE，最后获取内存数据，总共要访问3次内存，而命中TLB后只需从中获取物理地址即可。

### 查询TLB

`cpuid`是查询CPU信息的指令，按照最初在EAX中提供的值，在EAX、EBX、ECX、EDX中显示出处理器包括系列、型号、分级、功能等信息。

当EAX为2时，`cpuid`返回TLB相关信息

```assembly
mov eax, 0x2;
cpuid;
```

每个寄存器的最高有效位（位 31）表示寄存器是包含有效信息（设置为 0）还是保留的（设置为 1）。

### 刷新

TLB可以自动刷新也可手动刷新，有以下几条规则

> 每当CR3发生变化，线性地址对应的物理地址发生变化，TLB自动刷新
>
> 当PDE属性中G位=1时，此条地址不刷新
>
> `INVLPG`指令可以刷新TLB, CPL为0（即内核态）才能执行

## 控制寄存器

以下列出了各寄存器一些相对重要的属性

### CR0

![CR0结构](/images/image-20220427212013152.png)

CR0包含了控制操作模式和处理器状态的系统控制标志

PE：是否启用保护模式 ，0=实模式， 1=保护模式

WP：写保护位，0 = 3环不可写，1 = PTE、PDE中的RW表示可读可写

AM：为了页对齐，Windows下默认为0

NW：禁止写通（Not Write-through）

CD：CPU是否开启内部缓存， 0 = 关闭，1 = 开启

PG：PE = 0并且PG = 0，线性地址就是物理地址，不用页表，PG = 1用页表

### CR1

保留

![CR1结构](/images/image-20220427212715869.png)

### CR2

![CR2结构](/images/image-20220427212735561.png)

cr2 存放缺页的线性地址

### CR3

![CR3结构](/images/image-20220427212755832.png)

cr3为物理寄存器，PCD PWT访问TLB缓存的标识

### CR4

![CR4结构](/images/image-20220427212903957.png)

PSE：控制PDE中的PS位是否有效， 0 = 无效，1 = 有效

PAE：0 = 按10-10-12拆分 1 = 按2-9-9-12拆分 

PGE：控制PDE G位，0 = G位不生效，1 = G位生效

TSD：控制RDTSC，0 = 3环皆可执行，1= 只能在0环执行

PWT：同时改变缓存和内存需将此位置1













