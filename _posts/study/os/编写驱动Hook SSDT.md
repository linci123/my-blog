---
title: '编写驱动Hook SSDT'
author: Kevin。
tags:
  - os
  - Windows
categories:
  - study
date: 2022-05-14 13:18:00
img: /images/article-banner/QQ截图20220514202647.jpg
typora-root-url: ..\..\..\
---

# 编写驱动Hook SSDT

## 查找函数偏移

要hook SSDT中的函数，就必须先找到这个函数在`SSDT`表中的偏移才能修改这个偏移上的地址

随意打开一个程序，下个API调用时候的断点，这里下`writeFile`的断点

![下断点](/images/image-20220514200909573.png)

F8调试指导出现调用`ntdll`中的系统函数`NtWirteFile`，F7步入，`0x112`就是`WriteFile`在SSDT中的偏移

![F8调试](/images/image-20220514200948112.png)

![0x112偏移](/images/image-20220514201055813.png)

在Windbg中查看是否正确

![找到NtWriteFile](/images/image-20220514201211094.png)

## 编写驱动

这里Hook了`NtOpenProcess`函数，查找过程与上一节相似，不再赘述

```c
#include <ntddk.h>

typedef struct _ServiceDescriptorTable // SSDT结构体
{
	PVOID ServiceTableBase;
	PVOID ServiceCount;
	ULONG NumberOfService;
	PVOID PararmTableBase;
} *PServiceDescriptorTable;

//导出KeServiceDescriptorTable，使系统能够修改该结构体
extern PServiceDescriptorTable KeServiceDescriptorTable;  

ULONG FunAddress;
ULONG OldServiceTable;
ULONG JmpAddress;

VOID Hook();
VOID Unhook();
VOID DriverUnload(PDRIVER_OBJECT pDriverObject);

//新的NtOpenProcess函数，与原来的函数参数结构保持一样
_declspec(naked) VOID _stdcall newNtOpenProcess(
	_Out_ PHANDLE ProcessHandle,
	_In_ ACCESS_MASK DesiredAccess,
	_In_ POBJECT_ATTRIBUTES ObjectAttributes,
	_In_opt_ PCLIENT_ID ClientId
) {
    /*这里编写Hook内容*/
	KdPrint(("进入Hook函数\n"));
	/****************/
    
    
	//在执行完自己写的函数之后执行Hook前的函数并跳转到原来的地址
	_asm
	{
		push 0xc4  
		push 0x804f52d8
		jmp[JmpAddress]
	}
}

//main函数
NTSTATUS DriverEntry(PDRIVER_OBJECT pDriverObject, PUNICODE_STRING reg_path)
{
	KdPrint(("驱动加载成功\n"));
	Hook();
    pDriverObject->DriverUnload = DriverUnload; //驱动卸载函数
	return STATUS_SUCCESS;
}

_declspec(naked) VOID CLOSE_INT() {
	_asm
	{
		cli  //关中断
		mov eax, cr0
		and eax, 0xfffeffff //修改cr0写保护位为r3不可写
		mov cr0, eax
		ret
	}
}

_declspec(naked) VOID OPEN_INT() {
	_asm
	{
		mov eax, cr0
		or eax, 0x10000 //恢复cr0
		mov cr0, eax
		sti //开中断
		ret
	}
}

VOID DriverUnload(PDRIVER_OBJECT pDriverObject)
{
	KdPrint(("驱动卸载成功\n"));
	Unhook();
}

VOID Hook() {
	FunAddress = (ULONG)KeServiceDescriptorTable->ServiceTableBase + 0x7a * 4; //基址 + 函数编号 x 4
	KdPrint(("FunAddress = % x\n", FunAddress));

	OldServiceTable = *(ULONG*)FunAddress;  //保存原始地址，以便卸载后恢复原始地址
	KdPrint(("OldServiceTable = % x\n", OldServiceTable));

	//保存Hook执行完的跳转地址，定为NtOpenProcess第二条指令之后
	//8058270a 68c4000000      push    0C4h
	//8058270f 68d8524f80      push    offset nt!ObWatchHandles+0x25c (804f52d8)
	JmpAddress = (ULONG)NtOpenProcess + 10; 
	KdPrint(("JmpAddress = % x\n", JmpAddress));

	CLOSE_INT();

	*(ULONG*)FunAddress = (ULONG)newNtOpenProcess; //修改原来的地址为新的Hook地址

	OPEN_INT();
}

VOID Unhook() {
	FunAddress = (ULONG)KeServiceDescriptorTable->ServiceTableBase + 0x7a * 4;
	CLOSE_INT();

	*(ULONG*)FunAddress = (ULONG)OldServiceTable; //将地址恢复成原来的地址

	OPEN_INT();

	KdPrint(("Hook恢复成功"));
}
```

