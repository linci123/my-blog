---
title: 'Windows APC机制'
author: Kevin。
tags:
  - os
  - Windows
categories:
  - study
date: 2022-06-24 20:44:00
img: /images/article-banner/QQ截图20220702234129.jpg
typora-root-url: ..\..\..\
---

# Windows APC机制

APC(Asynchronous Procedure Call)，即异步过程调用，也称不确定调用。Windows的APC机制本质上是一种对于应用软件（线程）的“软件中断”机制。与DPC不同，APC是针对线程的，每个线程都有自己的APC链表，同一个线程的APC也是排队执行。

## IRQL

**IRQL**是**Interrupt Request Level**的缩写，即中断请求级别。是Windows操作系统使用的处理器中断级别。

硬件产生信号发给可编程**中断控制器**（Programmable Interrupt Controller），中断控制器发送**中断请求**（Interrupt request(IRQ))及相应的优先级给`CPU`，CPU设置一个掩码屏蔽低优先级的其他中断请求到**挂起状态**(pending state),直到CPU**释放控制**给中断控制器。如果到来的中断有更高优先级，那么当前中断被挂起，CPU处理高优先级的中断。

Windows把**硬件中断**与**软件中断**都映射到内部的`中断表内`，这就是中断请求级别`IRQL`。多核处理器的每个内核有自己单独的`IRQL`。异步过程调用、用户态线程、内核模式操作都可以被中断，因此它们的IRQL低于线程调度器（或称分派器）。

常见的中断级别有`PASSIVE_LEVEL`、`APC_LEVEL`和`DISPATCH_LEVEL`，`APC_LEVEL`介于另外两者之间，是专为APC的软件中断保留的IRQL。

 `PASSIVE_LEVEL`：最低级别, 没有被屏蔽的中断。线程执行用户模式，可以访问分页内存。

`APC_LEVEL`：异步调用层。当有异步过程调用APC发生时，处理器提升到APC级别，因而就屏蔽了其它APC。可以访问分页内存。

`DISPATCH_LEVEL`：分发派遣层。DPC和更低的中断被屏蔽，不能访问分页内存，因为缺页中断也是在这个层。线程调度器也在此层，调度时只考虑优先级，因此APC_LEVEL上的线程被阻塞后，可以调度执行PASSIVE_LEVEL线程。

由上述优先级可知，APC的IRQL高于PASSIVE_LEVEL，所以优先于普通的线程代码。当一个线程获得控制是，它的APC过程会被立即执行。

## APC结构体

```c
_KAPC{
   +0x000 Type          //类型，应为KObjects enum类型的ApcObject
   +0x002 Size          //KAPC结构的大小 0x30
   +0x004 Spare0        //未发现被使用                            
   +0x008 Thread        //该APC属于哪个线程对象                                 
   +0x00c ApcListEntry  //APC被加入到线程APC双向链表中的节点对象，KiInsertQueueApc函数将KAPC挂到对应的队列中（挂到KAPC的成员ApcListEntry指向处）
   +0x014 KernelRoutine    //指向一个函数指针，该函数将在内核模式的APC_LEVEL上被执行 (调用ExFreePoolWithTag 释放APC结构所占的内存)
   +0x018 RundownRoutine   //也是一个函数指针，当一个线程终止时如果它的APC链表中还有APC对象，那么，若RundownRoutine成员非空，则调用它所指的函数
   +0x01c NormalRoutine    //用户APC ：指向用户APC处理函数入口  或者 内核apc ：指向真正的内核apc函数；指向在PASSIVE_LEVEL上执行的函数，如果此项为空，则NormalContext和ApcMode也将被忽略
   +0x020 NormalContext    //内核APC：NULL  用户APC：真正的APC函数
   +0x024 SystemArgument1  //APC函数的参数   
   +0x028 SystemArgument2  //APC函数的参数
   +0x02c ApcStateIndex    //APC对象环境，时KAPC_ENVIRONMENT enum成员，一旦APC对象被插入到线程的APC链表中，则ApcStateIndex域只是了它位于KTHREAD对象的哪个APC链表中
   +0x02d ApcMode     //当前的APC是用户APC还是内核APC
   +0x02e Inserted    //当前的KAPC结构体是否已经插入到APC队列 挂入前：0  挂入后  1
}
```

```c
_KAPC_STATE
   +0x000 ApcListHead        : [2] _LIST_ENTRY //两个APC队列
   +0x020 Process          : Ptr64 _KPROCESS //所属或所挂靠的进程
   +0x028 KernelApcInProgress   : UChar //内核APC正在执行
   +0x029 KernelApcPending     : UChar  //有内核APC正在等待得到执行
   +0x02a UserApcPending      : UChar //有用户APC正在等待得到执行
```

```c
_KAPC_ENVIRONMENT {
	OriginalApcEnvironment, 
	AttachedApcEnvironment,
	CurrentApcEnvironment,   //采用目标线程当前的环境
	InsertApcEnvironment
} KAPC_ENVIRONMENT;
```

```c
_KThread
    ...
    +0x034 ApcState         : _KAPC_STATE  //存储APC的所有信息
    ...
    +0x14c SavedApcState    : _KAPC_STATE  //保存APC的状态
    ...
```

在`kthread`结构体中有`ApcState`和`SavedApcState`。由于Windows内核允许一些跨进程的操作，需要把当时的用户空间切换到别的进程的用户控件，所以一个线程可以暂时挂靠（attach）到另一个进程的地址空间，当当前线程挂靠于另一个进程期间，就必须把这些当前环境的数据转移到`SavedApcState`中，在回到原进程的用户空间时再恢复。

## 执行流程

### KeInitializeApc   

**初始化APC的结构体** -> _KAPC

比较目标环境是否为当前APC

把传入的参数赋给当前的APC



### KeInsertQueueApc  

**插入APC**

先判断_KThread.ApcQueueable可不可用，并且Apc还没有被插入

如果不可用且没有被插入，则调用KiInsertQueueApc



### KiInsertQueueApc

判断APC对象是否已经插入

_KAPC.NormalRoutine != 0 判断内核APC函数是否为0

如果NormalRoutine等于0，将APC对象插入到前面，特殊APC _KAPC.Inserted 置1

test dl, dl  判断APC是用户模式还是内核模式 0 = 内核   1 = 用户

如果是内核模式，插到链表后面

​			如果是用户模式，判断是否为退出函数，不是则把用户APC置1

​			有用户APC等待执行，把用户APC取出来插到链表前面

​			_KAPC.NormalRoutine == 0 && 内核的APC链表 != 0

​			QueueUserAPC --> NtQueueApcThread --> KeInitializeApc --> KeInsertQueueApc

​			NtQueueApcThread 在内核里面分配一个kapc结构体，插入到APC链表里面

​			cmp [eax+_KAPC.KernelRoutine], offset _PsExitSpecialApc

​			把用户的APC设置1，有用户APC正在等待执行

​			把用户链表取出来，插入前面

### KiFastCallEntry

​	把线程唤醒关掉

​	cmp [ebx+_KTHREAD.ApcState.UserApcPending], 0; 有没有用户APC正在等待执行   = 1有用户正在等待执行  = 0 没有

### KiDeliverApc

通过KiDeliverApc去调用APC

​	取内核的APC判断APC链表是否为NULL

​	如果不为空 --> 执行0环的APC函数 

​		获取APC对象

​		获取APC结构的成员，包括参数

​		判断KAPC.NormalRoutine是否为空

​		移除当前要执行的APC对象

​		APC插入状态设置为0

​		call [ebp+var_KernelRoutine] 执行内核APC

判断KAPC.NormalRoutine不是空的

移除当前要执行的APC对象

APC插入状态设置为0

cmp [ebp+var_KernelRoutine]，0

如果不为空，call [ebp+var_KernelRoutine] 执行APC

​	循环检查内核APC队列，判断正常和特殊的内核APC，分别取执行内核APC函数，一直到内核APC执行完为止

​	执行完内核APC之后，开始执行用户APC

​		获取用户APC链表

​		判断用户APC链表是否为空

​		判断是否为用户APC

​		有没有需要这些的用户APC

​		获取APC对象

​		获取APC结构的成员包括参数

​		移除当前要执行的APC对象

​		APC插入状态设置为0

​		call ebx 释放APC函数

### KiInitializeUserApc

​		把Trap_Frame转换成Context结构

​		把3环的堆栈检查2DC放Context结构和四个参数

​		_ProbeForWrite测试地址是否可读可写

​		把Trap_Frame的段寄存器改成3环

​		把3环Context中的ESP保存到Trap_Frame中的ESP

​		把0环的Trap_Frame的EIP改成_KeUserApcDispatcher

​		把参数放进3环的Context的ESP中

​		
