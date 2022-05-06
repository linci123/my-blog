---
title: 'int 0x2e和sysenter'
author: Kevin。
tags:
  - os
  - Windows
categories:
  - study
date: 2022-04-29 13:18:00
img: /images/article-banner/QQ截图20220506230902.jpg
typora-root-url: ..\..\..\
---

# int 0x2e和sysenter

## int 0x2e

`int 0x2e`是一个自陷（Trap）指令，告诉CPU将用户态切换成系统态。从`TSS`中装入本线程的系统空间堆栈段寄存器`SS`和堆栈指针寄存器`ESP`，再将SS、ESP、EFLAGS、CS、EIP压入用户系统空间堆栈，然后从`IDT`中`0x2e*8`的偏移调用程序入口，执行内核程序。执行完程序后，使用`ret`返回到用户态。调用的函数名为`_KiSystemService`

Windows定义`0x30`以上的中断向量用于外部设备，但外部设备用不了那么多中断，所以定成了`0x2e`，而Linux则用的时`int 0x80`实现系统调用

## _KiSystemService函数分析

### IDA简单操作

`shift + F1`：查找结构体并添加结构体

`t`： 选择结构体并将反汇编中的内存地址替换成结构体

`;`： 添加一行注释

`q`： 修复标红的反汇编

`x`： 代表谁调用了它

### Windbg简单操作

`bp ADDR`下断点

`bl`列出现有断点相关信息

`bc ID` 清除断点

### 入口函数

```assembly
004067C1  push    0               ; 压入_Trap_Frame结构体中的几个成员，push 0为压入ErrorCode
                                  ; +0x064 ErrCode          : Uint4B
004067C3  push    ebp			 ; +0x060 Ebp              : Uint4B
004067C4  push    ebx  			 ; +0x05c Ebx              : Uint4B
004067C5  push    esi			 ; +0x058 Esi              : Uint4B
004067C6  push    edi			 ; +0x054 Edi              : Uint4B
004067C7  push    fs			 ; +0x050 SegFs            : Uint4B
004067C9  mov     ebx, 30h
004067CE  mov     fs, ebx		  ;修改fs为r0
004067D0  push    large dword ptr fs:0 ; _KPCR.NtTib.ExceptionList
004067D7  mov     large dword ptr fs:0, 0FFFFFFFFh ; 把_KPCR.NtTib.ExceptionList值清0
004067E2  mov     esi, large fs:_KPCR.PrcbData.CurrentThread  ; 暂存线程指针
004067E9  push    dword ptr [esi+_KTHREAD.PreviousMode]
004067EF  sub     esp, 48h
004067F2  mov     ebx, [esp+_KTRAP_FRAME.SegCs] ; int 0x2e按照原来的函数调用方式执行，无需判断堆栈平衡
004067F6  and     ebx, 1          ; 0 = 进入调用前为r0
004067F6                          ; 1 = 进入调用前为r3
004067F9  mov     [esi+_KTHREAD.PreviousMode], bl
004067FF  mov     ebp, esp
00406801  mov     ebx, [esi+_KTHREAD.TrapFrame]
00406807  mov     [ebp+_KTRAP_FRAME._Edx], ebx
0040680A  mov     [esi+_KTHREAD.TrapFrame], ebp
00406810  cld									 ; 关中断
00406811  mov     ebx, [ebp+_KTRAP_FRAME._Ebp]      ;保存状态
00406814  mov     edi, [ebp+_KTRAP_FRAME._Eip]
00406817  mov     [ebp+_KTRAP_FRAME.DbgArgPointer], edx
0040681A  mov     [ebp+_KTRAP_FRAME.DbgArgMark], 0BADB0D00h ; 调试掩码
00406821  mov     [ebp+_KTRAP_FRAME.DbgEbp], ebx
00406824  mov     [ebp+_KTRAP_FRAME.DbgEip], edi
00406827  test    [esi+_KTHREAD.DebugActive], 0FFh
0040682B  jnz     Dr_kss_a
```

### Dr_kss_a

```assembly
004066BC  test    [ebp+_KTRAP_FRAME.EFlags], 20000h
004066C3  jnz     short loc_4066D2
004066C5  test    [ebp+_KTRAP_FRAME.SegCs], 1
004066CC  jz      loc_406831
004066D2 loc_4066D2:
004066D2  mov     ebx, dr0   ;保存调试寄存器状态
004066D5  mov     ecx, dr1
004066D8  mov     edi, dr2
004066DB  mov     [ebp+_KTRAP_FRAME.Dr0], ebx
004066DE  mov     [ebp+_KTRAP_FRAME.Dr1], ecx
004066E1  mov     [ebp+_KTRAP_FRAME.Dr2], edi
004066E4  mov     ebx, dr3
004066E7  mov     ecx, dr6
004066EA  mov     edi, dr7
004066ED  mov     [ebp+_KTRAP_FRAME.Dr3], ebx
004066F0  mov     [ebp+_KTRAP_FRAME.Dr6], ecx
004066F3  xor     ebx, ebx
004066F5  mov     [ebp+_KTRAP_FRAME.Dr7], edi
004066F8  mov     dr7, ebx
004066FB  mov     edi, large fs:_KPCR.Prcb
00406702  mov     ebx, [edi+_KPRCB.ProcessorState.SpecialRegisters.KernelDr0]
00406708  mov     ecx, [edi+_KPRCB.ProcessorState.SpecialRegisters.KernelDr1]
0040670E  mov     dr0, ebx
00406711  mov     dr1, ecx
00406714  mov     ebx, [edi+_KPRCB.ProcessorState.SpecialRegisters.KernelDr2]
0040671A  mov     ecx, [edi+_KPRCB.ProcessorState.SpecialRegisters.KernelDr3]
00406720  mov     dr2, ebx
00406723  mov     dr3, ecx
00406726  mov     ebx, [edi+_KPRCB.ProcessorState.SpecialRegisters.KernelDr6]
0040672C  mov     ecx, [edi+_KPRCB.ProcessorState.SpecialRegisters.KernelDr7]
00406732  mov     dr6, ebx
00406735  mov     dr7, ecx
00406738  jmp     loc_406831

00406831 loc_406831:
00406831  sti      ; 开中断
00406832  jmp     loc_406922 ; 跳转到fastcall代码中
```

### Windbg调试

`dq idtr l70`查看中断门和陷阱门，`dq 8003f400+2e*8`获取`int 0x2e`跳转的地址

![0x2e偏移对应的值](/images/image-20220429163156015.png)

将```804dee00`0008e7c1```分割出地址`804de7c1`，`u 804de7c1`找到入口函数反汇编

![nt!KiSystemService](/images/image-20220429163309441.png)

## sysenter

CPU为了实现`sysenter`，新增加了三个MSR寄存器：`SYSENTER_CS_MSR`、`SYSENTER_EIP_MSR`、`SYSENTER_ESP_MSR`，在执行`sysenter`时，CPU根据这三个寄存器的内容设置跳转目标和堆栈指针，只要预先设置这三个寄存器，就可快速执行系统调用，所以`sysenter`也被称为`快速系统调用`。调用的函数为`_KiFastCallEntry`

`MSR`寄存器只能通过`rdmsr`、`wrmsr`实现读写，该指令为特权指令，在0环执行。

> rdmsr 174 # 读取cs
>
> rdmsr 175 # 读取esp
>
> rdmsr 176 # 读取eip

## _KiFastCallEntry函数分析

当软件在3环调用sysenter后，系统便会调用_KiFastCallEntry函数，由于快速调用用的jmp指令，用寄存器传值，所以需要系统自己构建堆栈环境

### _KPCR

一个CPU只有一个，其中储存了CPU一些重要数据，在windbg中使用`dt _kpcr`命令显示结构体，fs只在0环时指向KPCR，fs:N后面的N指向KPCR中的偏移。fs在3环时指向TEB，本节暂不介绍

fs:0指向KPCR结构体的第一项`_NT_TIB`，`_NT_TIB`的第一项为`ExceptionList`，所以fs:0指向的就是异常列表

### 入口函数

初始化段寄存器以及保存3环状态

```assembly
00406E0F  mov     ecx, 23h
00406E14  push    30h             ; 修改FS
00406E16  pop     fs
00406E18  mov     ds, cx          ; 修改DS
00406E1A  mov     es, cx          ; 修改ES
00406E1C  mov     ecx, large fs:_KPCR.TSS ; fs:40在结构体中指向TSS
00406E23  mov     esp, [ecx+_KTSS.Esp0]
00406E26  push    23h             ; 由于上面修改了其他寄存器，这个寄存器只能是SS
00406E28  push    edx             ; 3环ESP
00406E29  pushf                   ; 保存3环EFLAGS状态
```

### loc_406E2A

```assembly
00406E2A  push    2               ; 初始化EFLAGS寄存器 第一位（首位从0开始计数）默认为1，其余位为0，所以是2
00406E2C  add     edx, 8          ; 3环的参数
00406E2F  popf					; 将push的2赋值给EFLAGS
00406E30  or      byte ptr [esp+1], 2 ; 3环的EFLAGS
00406E35  push    1Bh             ; 保存R3的CS
00406E37  push    dword ptr ds:0FFDF0304h ; 要返回的EIP
00406E3D  push    0               ; 错误码
00406E3F  push    ebp
00406E40  push    ebx
00406E41  push    esi
00406E42  push    edi
00406E43  mov     ebx, large fs:_KPCR.SelfPcr
00406E4A  push    3Bh             ; 保存R3的FS
00406E4C  mov     esi, [ebx+_KPCR.PrcbData.CurrentThread] ; 当前的线程
00406E52  push    [ebx+_KPCR.NtTib.ExceptionList] ; 把R3的异常链表保存一份
00406E54  mov     [ebx+_KPCR.NtTib.ExceptionList], 0FFFFFFFFh ; 把RO的异常初始化
00406E5A  mov     ebp, [esi+_KTHREAD.InitialStack] ; 初始化堆栈
00406E5D  push    1               ; 表示用户的状态： 1为用户，0 为系统
00406E5F  sub     esp, 48h        ; 到Ktrap_frame的首地址
00406E62  sub     ebp, 29Ch       ; 减去ktrap_frame结构体的大小8c以及存放浮点的空间
00406E68  mov     [esi+_KTHREAD.PreviousMode], 1 ; 当前线程模式置1
00406E6F  cmp     ebp, esp        ; 检查ESP和EBP是否相等
00406E71  jnz     loc_406DD7      ; 不相等就报错
00406E77  and     [ebp+_KTRAP_FRAME.Dr7], 0   ; 调试寄存器赋值
00406E7B  test    [esi+_KTHREAD.DebugActive], 0FFh ; 检查线程是不是调试状态
00406E7F  mov     [esi+_KTHREAD.TrapFrame], ebp
00406E85  jnz     Dr_FastCallDrSave
```

#### Dr_FastCallDrSave

```assembly
00406CB0  test    [ebp+_KTRAP_FRAME.EFlags], 20000h ; 检查是不是8086模式
00406CB7  jnz     short loc_406CC6
00406CB9  test    [ebp+_KTRAP_FRAME.SegCs], 1
00406CC0  jz      loc_406E8B ; 跳回上个函数下一步的EIP
00406CC6 loc_406CC6: ; 保存原来的调试寄存器并加载内核调试寄存器
00406CC6  mov     ebx, dr0
00406CC9  mov     ecx, dr1
00406CCC  mov     edi, dr2
00406CCF  mov     [ebp+_KTRAP_FRAME.Dr0], ebx
00406CD2  mov     [ebp+_KTRAP_FRAME.Dr1], ecx
00406CD5  mov     [ebp+_KTRAP_FRAME.Dr2], edi
00406CD8  mov     ebx, dr3
00406CDB  mov     ecx, dr6
00406CDE  mov     edi, dr7
00406CE1  mov     [ebp+_KTRAP_FRAME.Dr3], ebx
00406CE4  mov     [ebp+_KTRAP_FRAME.Dr6], ecx
00406CE7  xor     ebx, ebx
00406CE9  mov     [ebp+_KTRAP_FRAME.Dr7], edi
00406CEC  mov     dr7, ebx
00406CEF  mov     edi, large fs:_KPCR.Prcb
00406CF6  mov     ebx, [edi+_KPRCB.ProcessorState.SpecialRegisters.KernelDr0]
00406CFC  mov     ecx, [edi+_KPRCB.ProcessorState.SpecialRegisters.KernelDr1]
00406D02  mov     dr0, ebx
00406D05  mov     dr1, ecx
00406D08  mov     ebx, [edi+_KPRCB.ProcessorState.SpecialRegisters.KernelDr2]
00406D0E  mov     ecx, [edi+_KPRCB.ProcessorState.SpecialRegisters.KernelDr3]
00406D14  mov     dr2, ebx
00406D17  mov     dr3, ecx
00406D1A  mov     ebx, [edi+_KPRCB.ProcessorState.SpecialRegisters.KernelDr6]
00406D20  mov     ecx, [edi+_KPRCB.ProcessorState.SpecialRegisters.KernelDr7]
00406D26  mov     dr6, ebx
00406D29  mov     dr7, ecx
00406D2C  jmp     loc_406E8B ; 跳回上个函数下一步的EIP
```

### loc_406E8B

调试的一系列操作

```assembly
00406E8B  mov     ebx, [ebp+_KTRAP_FRAME._Ebp]
00406E8E  mov     edi, [ebp+_KTRAP_FRAME._Eip]
00406E91  mov     [ebp+_KTRAP_FRAME.DbgArgPointer], edx ; 参数开始的位置 ESP+8
00406E94  mov     [ebp+_KTRAP_FRAME.DbgArgMark], 0BADB0D00h ; 调试掩码
00406E9B  mov     [ebp+_KTRAP_FRAME.DbgEbp], ebx
00406E9E  mov     [ebp+_KTRAP_FRAME.DbgEip], edi
00406EA1  sti                     ; 开中断
```

#### sti和cli

这两个指令为中断操作，`sti(Set Interrupt)`开中断，`cli(Clear Interrupt)`关中断，只能在0环下执行

`sti`中断标志置1指令 使 IF = 1

`cli`中断标志置0指令 使 IF = 0

cli关闭中断后，该线程不允许外部线程打扰，sti开启后恢复

### loc_406EA2

```assembly
00406EA2  mov     edi, eax        ; EAX低12位是API的编号，eax是R3中传过来的
00406EA4  shr     edi, 8
00406EA7  and     edi, 30h        ; (编号 >> 8) & 30 求出在KeServiceDescriptorTable哪张表中，算出来就是KeServiceDescriptorTable[0]
00406EAA  mov     ecx, edi
00406EAC  add     edi, [esi+_KTHREAD.ServiceTable] ; +e0 是指向KeServiceDescriptorTable[0]的基址
00406EB2  mov     ebx, eax
00406EB4  and     eax, 0FFFh      ; 获取真正的编号
00406EB9  cmp     eax, [edi+8]
00406EBC  jnb     _KiBBTUnexpectedRange ; GUI
00406EC2  cmp     ecx, 10h
00406EC5  jnz     short loc_406EE2
00406EC7  mov     ecx, large fs:_KPCR.NtTib.Self
00406ECE  xor     ebx, ebx
00406ED0 loc_406ED0
00406ED0  or      ebx, [ecx+(_KPCR.PrcbData.ProcessorState.ContextFrame.FloatSave.DataSelector+0E00h)]
00406ED6  jz      short loc_406EE2
00406ED8  push    edx
00406ED9  push    eax
00406EDA  call    ds:_KeGdiFlushUserBatch
00406EE0  pop     eax
00406EE1  pop     edx
```

#### KeServiceDescriptorTable

`KeServiceDescriptorTable`简称SSDT，一共由16个字节，4个字节一项，依次为函数地址、访问次数、函数个数、参数表

计算函数地址的公式：`KeServiceDescriptorTable[0]+编号*4`

参数表查看参数所占字节数：`KeServiceDescriptorTable[3]+eax`

然而在有图形界面的`Windows`中，这里访问的是`KeServiceDescriptorTableShadow` ，也就是影子表，由win32k.sys调用，前16字节和`KeServiceDescriptorTable`一样，之后的16字节是GUI的信息

#### _KiBBTUnexpectedRange

```assembly
00406BE2  cmp     ecx, 10h
00406BE5  jnz     short loc_406C20 ; 报错
00406BE7  push    edx
00406BE8  push    ebx             ; 转换图形线程
00406BE9  call    _PsConvertToGuiThread@0 ; PsConvertToGuiThread()
00406BEE  or      eax, eax        ; 判断返回值是否为0
00406BF0  pop     eax
00406BF1  pop     edx
00406BF2  mov     ebp, esp
00406BF4  mov     [esi+_KTHREAD.TrapFrame], ebp
00406BFA  jz      loc_406EA2      ; 如果返回值为0，返回上一个函数头，相当于循环
00406C00  lea     edx, unk_48C4D0
00406C06  mov     ecx, [edx+8]
00406C09  mov     edx, [edx]
00406C0B  lea     edx, [edx+ecx*4] ; 获取函数
00406C0E  and     eax, 0FFFh
00406C13  add     edx, eax
00406C15  movsx   eax, byte ptr [edx] ; 获取传入参数的长度
00406C18  or      eax, eax ; 判断eax是否为0
00406C1A  jle     loc_406F11
00406C20 loc_406C20:
00406C20  mov     eax, 0C000001Ch ; 错误代码
00406C25  jmp     loc_406F11
```

### loc_406EE2

```assembly
00406EE2  inc     large dword ptr fs:_KPCR.PrcbData.KeSystemCalls
00406EE9  mov     esi, edx        ; R3 esp+8
00406EEB  mov     ebx, [edi+0Ch]  ; ServiceTable 参数表
00406EEE  xor     ecx, ecx
00406EF0  mov     cl, [eax+ebx]   ; 获取参数字节
00406EF3  mov     edi, [edi]      ; 函数地址表
00406EF5  mov     ebx, [edi+eax*4] ; 找到哪个函数地址
00406EF8  sub     esp, ecx        ; 留出栈空间，准备拷贝
00406EFA  shr     ecx, 2          ; 按照四个字节，四个字拷贝
00406EFD  mov     edi, esp
00406EFF  cmp     esi, ds:_MmUserProbeAddress ; 判断ESP+8 是否是 R3的线性地址 7fff0000
00406F05  jnb     loc_4070B3
00406F0B loc_406F0B:
00406F0B  rep movsd               ; 拷贝参数
00406F0D  call    ebx             ; 调用函数
```

### Windbg调试

随便在OD里面拖入一个应用程序，这里调试一个`notepad.exe`，在`WriteFile`中下个断点，往下调试发现已经调用了`ntdll`中的函数，接着单步进入发现调用了`0x7ffe0300`处的函数地址，

![调用ZwWriteFile](/images/image-20220429160331384.png)

![接着调用0x7ffe0300](/images/image-20220429160443971.png)

在windbg中断下来，切换到notepad.exe进程，使用`dt _kuser_shared_data 7ffe0000`（0环的地址为`0xffdf0000`）命令列出3环和0环交互的结构体，在`0x300`偏移处找到函数入口，`u 0x7c92e4f0`查看该地址的反汇编，找到快速调用入口

![_kuser_shared_data](/images/image-20220429160535475.png)

![查看反汇编](/images/image-20220429161013993.png)

`0x7ffe0300`存的时一个函数指针，指向了`KiIntSystemCall()`或`KiFastSystemCall()`两个函数之一。

其中`KiIntSystemCall`就是调用了`int 0x2e`进入内核态，代码为

```assembly
lea edx, [esp+8]
int 0x2e  #进入内核态
ret       #返回用户态
```

而`KiFastSystemCall`有两段代码

```assembly
_KiFastSystemCall:
	mov edx, esp
	sysenter

_KiFastSystemCallRet:
	ret
```

## 驱动编写

dt _DRIVER_OBJECT

dt _LDR_DATA_TABLE_ENTRY

bt 位测试  

cmc cf取反



