---
title: 'TSS 任务状态段描述符'
author: Kevin。
tags:
  - os
categories:
  - study
date: 2022-04-15 13:18:00
img: /images/article-banner/QQ截图20220424203532.jpg
typora-root-url: ..\..\..\
---

# TSS 任务状态段描述符

​	TSS（Task State Segment）任务状态段描述符用于描述保存任务重要信息的系统段，权限发生变化要借用TSS。任务状态段寄存器TR的可见部分为当前任务状态段描述符的选择子，不可见部分是当前任务状态段的段基地址和段界限等信息，TR只装载一次，`TR.Base`指向的地址即TSS。

​	操作系统通过TSS实现任务的挂起和恢复，在切换任务的过程中，处理器中的各寄存器的当前值会被自动地保存到TR指定的TSS中，接着下一个任务的TSS的选择子被装入TR，最后从TR所指定的TSS中取出各寄存器的值送到处理器的各寄存器中。

## TSS基本格式

![任务状态段基本格式](/images/image-20220415230204679.png)

其中：

> ESP0-ESP2对应0环-2环的ESP，由于SS跟ESP相对应，所以SS段也相同
>
> CR3：每一个进程都有一个CR3，里面存的是页目录表基址，为物理地址 
>
> LDT：局部描述符表
>
> 链接字段：安排在TSS 0偏移处，其中高16位未使用，低16位保存前一任务的TSS选择子

​	TSS的地址映射寄存器位于`0x1C（CR3）`和`0x60（LDT）`中，在任务切换时，处理器自动从TSS中取出这两个字段，分别装入到CR3和LDTR寄存器中，这样就改变了虚拟地址空间到物理地址空间的映射。但由于在任务切换时，处理器**不会**把换出任务方式的寄存器CR3和LDTR的内容保存到TSS中的地址映射寄存器区域，所以如果改变了LDTR或CR3，就必须把新值保存到TSS中的地址映射寄存器区域中。

## WinDBG测试

windbg中 dd、dw、dq只能查看线性地址，查看物理地址需在dd前加上`!`

`!process 0 -1` 查看当前执行的进程

`!process 0 0`遍历进程，其中的DirBase就是CR3

![进程列表](/images/image-20220416110659573.png)

### 进程下WinDBG断点

使用前文的进入0环的方式，在进入0环后用硬编码下一个断点，系统就断到了WinDBG里面

```c
_declspec(naked)void fun(){
	_asm {
		push ebp
		mov ebp,esp
		sub esp,0x44
		push ebx
		push esi
		push edi
	}

 	_asm {
        mov ax,cs
        mov cx,ss
        mov w_cs,ax
        mov w_ss,cx
 	}
    
    _asm{
        int 0x3 //断到windbg，查找当前进程CR3
        //windows设定 int 0x8 时出现蓝屏
    }

	_asm{
		pop edi
		pop esi
		pop ebx
		mov esp,ebp
		pop ebp
		iretd
	}
}
```

![进程断到windbg中](/images/image-20220416110918439.png)

输入`u`可以查看当前断点以下的汇编，也可以在汇编的窗口中查看。

输入`!process 0 -1`查看当前进程信息

![当前进程信息](/images/image-20220416111156852.png)

输入`.process /i [PROCESS后面的ID]`可以切换进程

![切换回System进程](/images/image-20220416111332014.png)

在切换到System进程后，输入g运行后可以打印详细信息

### WinDBG查看TSS

由于TR的选择子是28，直接定位到GDT中0x28偏移的位置

![GDT](/images/image-20220416113031266.png)

按照描述符格式分割出TSS地址 ```80 008b 04`2000 20ab ==> 80042000```

`dd 80042000`查看TSS内容，`dt _KTSS 80042000`查看结构

![TSS内容](/images/image-20220416120112790.png)

下面乱记的，不知道啥意思

中断门或陷阱门

当权限不变时

int

pushfd

push cs

push eip



当权限变化时

push ss

push esp

pushfd

push cs

push eip



调用门

权限不变

push cs

push eip



权限改变

push ss

push esp

push cs

push eip























































