---
title: 'KiSwapThread线程切换'
author: Kevin。
tags:
  - os
  - Windows
categories:
  - study
date: 2022-05-23 20:44:00
img: /images/article-banner/QQ截图20220702234055.jpg
typora-root-url: ..\..\..\
---

# KiSwapThread线程切换

KiSwapThread是通过切换线程上下文来实现线程切换的函数

## KiSwapThread主函数

```assembly
0040AB8A        mov     edi, edi
0040AB8C        push    esi
0040AB8D        push    edi
0040AB8E        mov     eax, large fs:_KPCR.Prcb ; 获取KPRCB
0040AB94        mov     esi, eax
0040AB96        mov     eax, [esi+_KPRCB.NextThread] ; 获取下一个线程
0040AB99        test    eax, eax
0040AB9B        mov     edi, [esi+_KPRCB.CurrentThread] ; 获取当前线程
0040AB9E        jnz     loc_4109B9
0040ABA4        push    ebx
0040ABA5        movsx   ebx, [esi+_KPRCB.Number]
0040ABA9        xor     edx, edx
0040ABAB        mov     ecx, ebx
0040ABAD        call    @KiFindReadyThread@8 ; 找到一个准备执行的线程
0040ABB2        test    eax, eax
0040ABB4        jz      loc_4107BE      ; 获取空闲线程
0040ABBA
0040ABBA loc_40ABBA:
0040ABBA        pop     ebx
0040ABBB loc_40ABBB:
0040ABBB        mov     ecx, eax        ; 找到的线程
0040ABBD        call    @KiSwapContext@4 ; 执行线程上下文切换
0040ABC2        test    al, al
0040ABC4        mov     cl, [edi+58h]   ; NewIrql
0040ABC7        mov     edi, [edi+54h]
0040ABCA        mov     esi, ds:__imp_@KfLowerIrql@4 ; KfLowerIrql(x)
0040ABD0        jnz     loc_41BC56
0040ABD6
0040ABD6 loc_40ABD6:
0040ABD6        call    esi ; KfLowerIrql(x) ; KfLowerIrql(x)
0040ABD8        mov     eax, edi
0040ABDA        pop     edi
0040ABDB        pop     esi
0040ABDC        retn
```

## KiSwapContext

```assembly
.text:0040580E                 sub     esp, 10h        ; 栈分配空间
.text:00405811                 mov     [esp+0Ch], ebx  ; push
.text:00405815                 mov     [esp+8], esi
.text:00405819                 mov     [esp+4], edi
.text:0040581D                 mov     [esp], ebp
.text:00405820                 mov     ebx, large fs:_KPCR.SelfPcr ; 获取当前KPCR
.text:00405827                 mov     esi, ecx
.text:00405829                 mov     edi, [ebx+_KPCR.PrcbData.CurrentThread] ; 指向当前线程
.text:0040582F                 mov     [ebx+_KPCR.PrcbData.CurrentThread], esi ; 将找到的线程放入_kpcr中的当前线程
.text:00405835                 mov     cl, [edi+_KTHREAD.WaitIrql] ; 中断请求级别
.text:00405838                 call    SwapContext     ; esi新线程
.text:00405838                                         ; edi老线程
.text:0040583D                 mov     ebp, [esp]      ; 相当于POP新线程的寄存器
.text:00405840                 mov     edi, [esp+4]
.text:00405844                 mov     esi, [esp+8]
.text:00405848                 mov     ebx, [esp+0Ch]
.text:0040584C                 add     esp, 10h
.text:0040584F                 retn
```

## SwapContext

```assembly
.text:00405932                 or      cl, cl          ; 判断cl是否为0
.text:00405934                 mov     es:[esi+_KTHREAD.State], Running
.text:00405939                 pushf                   ; 保存原来线程的eflags
.text:0040593A                 lea     ecx, [ebx+(_KPCR.PrcbData.LockQueue.Next+8)]
.text:00405940                 call    @KeAcquireQueuedSpinLockAtDpcLevel@4 ; 锁队列，链表
.text:00405945                 lea     ecx, [ebx+_KPCR.PrcbData.LockQueue]
.text:0040594B                 call    @KeReleaseQueuedSpinLockFromDpcLevel@4 
.text:00405950 loc_405950:                             ; CODE XREF: KiIdleLoop()+7C↓j
.text:00405950                 mov     ecx, [ebx+_KPCR.NtTib.ExceptionList]
.text:00405952                 cmp     [ebx+_KPCR.PrcbData.DpcRoutineActive], 0
.text:00405959                 push    ecx             ; 保存OLD的异常链表
.text:0040595A                 jnz     loc_405AE2      ; 不为0就蓝屏
.text:00405960                 cmp     ds:_PPerfGlobalGroupMask, 0 ; 记录代码，通常情况不发生
.text:00405967                 jnz     loc_405AB9
.text:0040596D loc_40596D:
.text:0040596D                 mov     ebp, cr0        ; 读cr0
.text:00405970                 mov     edx, ebp
.text:00405972                 cmp     [edi+_KTHREAD.NpxState], 0 ; 检查浮点状态，浮点寄存器有没有用过
.text:00405976                 jz      loc_405A94      ; 如果为0就修改cr0
.text:0040597C loc_40597C:
.text:0040597C                 mov     cl, [esi+_KTHREAD.DebugActive] ; 新线程调试状态是否激活
.text:0040597F                 mov     [ebx+_KPCR.DebugActive], cl ; 将新线程的调试状态给处理器调试状态
.text:00405982                 cli
.text:00405983                 mov     [edi+_KTHREAD.KernelStack], esp ; 当前线程运行到最后一个地方的ESP，把老线程保存起来，老线程即将结束，新线程即将开始
.text:00405986                 mov     eax, [esi+_KTHREAD.InitialStack] ; 新线程的InitialStack
.text:00405989                 mov     ecx, [esi+_KTHREAD.StackLimit] ; 新线程的StackLimit
.text:0040598C                 sub     eax, 210h       ; 给浮点寄存器留空间
.text:00405991                 mov     [ebx+_KPCR.NtTib.StackLimit], ecx ; 设置为新的StackLimit
.text:00405994                 mov     [ebx+_KPCR.NtTib.StackBase], eax ; 设置为新的StackBase
.text:00405997                 xor     ecx, ecx
.text:00405999                 mov     cl, [esi+_KTHREAD.NpxState] ; 检查新线程的浮点状态
.text:0040599C                 and     edx, 0FFFFFFF1h ; EDX放的是原来cr0的值
.text:0040599F                 or      ecx, edx
.text:004059A1                 or      ecx, [eax+_FX_SAVE_AREA.Cr0NpxState] ; 检查浮点内容
.text:004059A7                 cmp     ebp, ecx
.text:004059A9                 jnz     loc_405A8C      ; 改变cr0，cr0中存有浮点状态
.text:004059AF                 lea     ecx, [ecx]
```

