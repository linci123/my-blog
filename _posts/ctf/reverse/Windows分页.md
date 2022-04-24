---
title: 'Windows分页'
author: Kevin。
tags:
  - reverse
categories:
  - ctf
date: 2022-04-20 13:18:00
img: /images/article-banner/QQ截图20220420160926.jpg
typora-root-url: ..\..\..\
---

# Windows分页

Windows32主要有两种分页方式，10-10-12 和 2-9-9-12，加起来就是32位，其中2，9，10，12代表2的x次方

## 10-10-12分页

![分页方式](/images/image-20220418134759160.png)

根据因特尔手册给出的图可以看出，当前程序的页目录索引存在CR3中，根据PDI的地址查找页表索引，再根据页表入口地址查找物理地址

具体的计算方法如下：

以线性地址`0x8003f400`为例，首先拆分此线性地址为：

```PDI:10 0000 0000      PTI:00 0011 1111      OFFSET:0100 0000 0000```

其中PDI、PTI分别为页目录、页表的首地址，OFFSET为最后物理地址的偏移

`Windbg`中用`r cr3 `展示CR3的值

```c
PDE = CR3 & 0xffffffe0 + PDI << 2
PTE = PDE & 0xfffff000 + PTI << 2
PA  = PTE & 0xfffff000 + OFFSET
```

CR3低5位、PDE低12位、PTE低12位均为属性

在WinDBG中调试，在调试前先将C:\boot.ini中的配置项改为

`multi(0)disk(0)rdisk(0)partition(1)\WINDOWS="Microsoft Windows XP Professional_debugf(10-10-12)" /execute=optin /fastdetect /debug /debugport=com1 /baudrate=115200`

配置中的 `execute `表示10-10-12分页方式，`noexecute`表示 2-9-9-12 分页方式， 和64位进行过渡

![IDTR中的值和物理IDTR对比](/images/image-20220418161423067.png)

## 2-9-9-12分页

由于技术的发展，4G的内存已经不够系统使用了，所以搞了个2-9-9-12分页模式扩展内存，当然物理内存还是那些，但将基址由32位扩展到了36位，同时为了内存对齐，PTE整体由原来的4字节扩大到了8字节

由于页的大小仍保持4KB，0x1000对齐，所以后12位依旧保留

但由于PTE的长度变成了8字节，所以4KB / 8 = 512B = 2 ^ 9，这便是9位PTI的由来，因为2的9次方个PDE就能找到所有的页，因此PDI=9，还剩下两位则被分配到了PDPI

![PTE结构](/images/image-20220424155421707.png)

![PDPE结构](/images/image-20220424155322901.png)

2-9-9-12分页方式相比于10-10-12的分页方式多了一个PDPI，即`pdpi：2  pdi：9  pti：9  offset：12`

计算方式为：

```c
PDPE = CR3 & 0xffffffe0 + PDPI << 3
PDE = PDPE & 0xfffff000 + PDI << 3
PTE = PDE & 0xfffff000 + PTI << 3
PA  = PTE & 0xfffff000 + OFFSET
```

`.process /i [ID]`切换一个进程，选择当前的EIP（实验中为`004011c8`）

分割字节为2-9-9-12方式，得出`pdpi：0  pdi：2  pti：1  offset：1c8`

使用`!dq`读取`CR3+0`八个字节

![读取CR3](/images/image-20220424151428095.png)

接下来的操作基本与10-10-12分页相似

![打印物理地址中的内存值](/images/image-20220424151522830.png)

可以看出与线性地址中的值相同

![物理地址与线性地址对比](/images/image-20220424151559938.png)

## 内存跨页

当程序运行到页的边界时往往会出现跨页的情况，如`401ffe 401fff 402000 402001`这4个地址组成了一个完整的数据，由于页大小为0x1000，在40200处出现了跨页，此时需要拆分`401ffe `和`402000`

 有代码：

```assembly
mov eax, dword ptr ds:[0x401ffe];
int 0x3;
```

Ctrl+F5运行后，成功在WinDBG中断下来

首先拆分`401ffe`成`0000000001   0000000001   ffe`

获取CR3（此处CR3值为23ea9000）计算各个偏移后获取到物理地址中相对应的数据

![物理地址中的数据](/images/image-20220420195652255.png)

可以看出和线性地址中的数据并不一样，在`402000`后的数据发生了变化，此时就产生了分页

![线性地址中的数据](/images/image-20220420195733487.png)

拆分`402000`为`0000000001   0000000010   000`，计算一系列地址后获取物理地址，找到了上面`402000`后消失的数据

![402000物理地址](/images/image-20220420195904596.png)

![eax的值也为上述内容](/images/image-20220420200239632.png)

## PDE

### 属性

![属性](/images/image-20220420200538312.png)

| 位数 | 名称 |     全名      | 值为1时 |    值为0时     |
| :--: | :--: | :-----------: | :-----: | :------------: |
|  0   |  P   |    存在位     |  存在   |     不存在     |
|  1   | R/W  |    读写位     |   写    |       读       |
|  2   | U/S  | 用户/超级用户 |  用户   | 超级用户(系统) |
|  3   | PWT  | write-through |    /    |       /        |
|  4   | PCD  |  不允许缓存   |    /    |       /        |
|  5   |  A   |   可访问位    | 访问过  |     没访问     |
|  6   |  0   |    保留位     |    /    |       /        |
|  7   |  PS  |    页尺寸     | **注1** |      4KB       |
|  8   |  G   |    全局页     |    /    |       /        |
|  9   | AVL  | 软件可利用位  |    /    |       /        |

**注1**：一个页为4MB且没有PTE，只有PDE和OFFSET



## PTE

### 属性

和PDE的0相对应的位位D位，脏页位，即被使用过即为脏页

### 修改PTE使程序访问0x0线性地址

执行`mov eax, dword ptr ds:[0]`的情况下无法访问内存，是因为该地址在PTE中为缺页状态，访问不到物理地址。

写中断，断到Windbg中

```c
unsigned int i;
_asm {
    int 0x3;
    mov eax, dword ptr ds:[0];
    mov i, eax;
}
printf("%x\n", i);
```

```bash
!dd cr3 #查看CR3地址中的值,实验时cr3为3c505000，其中的值为 3c2d6867 3c3d5867
!dd 3c2d6000 #找到线性地址为0时的PTE，发现并没有数据
```

![PTE为0](/images/image-20220421150353890.png)

选择PDI中第二个值，查看PTE，这与PE结构的开头相对应

![PTE加偏移后的值](/images/image-20220421150439783.png)

![可执行文件头](/images/image-20220421150527571.png)

修改第一个PTE的值为第二个PTE的值

![修改PTE](/images/image-20220421151001446.png)

输入g继续执行，输出i

![输出结果](/images/image-20220421151219944.png)

## 查找线性地址中的PDE

Windbg执行`!dd cr3 l400`查看CR3（此处为225d6000）地址中的值，去除低12位搜索相似值的地址，在`225d6c00`处找到相似值

![与CR3值相似的值](/images/image-20220421165720239.png)

取该数据对应的地址 - CR3，即`PDE = 225d6c00 - 225d6000 = c00`

由于`CR3+PDI * 4 = pde`，则`PDI = (PDE - CR3) / 4 = 300`

0x300地址的页比较特殊，既指向PDE又指向PTE也指向OFFSET，称为上帝页，所以可以得出如下结果

`PDI:300  PTI:300  OFFSET:000` => `1100 0000 0011 0000 0000 000` => `c0300000`

![CR3物理地址和线性地址中的值对比](/images/image-20220421170239132.png)

2-9-9-10分页模式中，如果找不到与CR3相似的内存区域，可以找CR3对应内存值的相似内存

## 查找线性地址中的PTE

由于上述的0x300被称为上帝页，则可根据分离的PDI、PTI以及OFFSET推测出PTI为0时的PTE，将PTI和OFFSET置0则可得出PTE 0的页地址`C0000000`

> PDI     PTI   OFFSET
>
> 300     000      000   ===>  C0000000

![PTI线性地址和PTI物理地址](/images/image-20220422205901711.png)

那么也可推测出PTI为1时，PTE为C0001000，以此类推，公式为`PTE = c0000000 + PDI*0x1000 + PTI * 0x4`

![PTI线性地址和PTI物理地址](/images/image-20220422210011550.png)

### 打印线性地址中的内容

```c
// page.cpp : Defines the entry point for the console application.
//

#include "stdafx.h"
#include <windows.h>

WORD w_cs;
WORD w_ss;

DWORD PDTable[0x400];

DWORD GetFunAddress(void* Fun){
	BYTE* p;
	p = (BYTE*)Fun;
	if ( *p == 0xe9){
		p = p+5+ *(DWORD*)&p[1];
	}
	return (DWORD)p;
}


_declspec(naked)void fun()
{
	_asm{
		push ebp
		mov ebp,esp
		sub esp,0x44
		push ebx
		push esi
		push edi
	}

	_asm{
		mov ax,cs
		mov cx,ss
		mov w_cs,ax
		mov w_ss,cx
	}

	DWORD* pPDIAddress;
	DWORD dwPDE;
	int i;
	pPDIAddress = (DWORD*)0xc0300000;
	//进入0环后修改每个PDE属性
	for ( i = 0; i < 0x400; i++){
		dwPDE = *(pPDIAddress+i);
		PDTable[i] = dwPDE;
		if( (dwPDE&0x1) != 0) //判断是否为有效PDE
		{
			*(pPDIAddress+i) = dwPDE | 4; //将U/S位修改为1
		}
	}
    
	_asm{
		pop edi
		pop esi
		pop ebx
		mov esp,ebp
		pop ebp
		retf
	}
}


int main(int argc, char* argv[])
{
	LPVOID pAddress;
	DWORD dwFunAddress;
	DWORD* pReadAddress;
	int i;

	memset(&w_cs,0,2);
	memset(PDTable,0x0,0x400);

	pAddress = VirtualAlloc((void*)0x60000000,0x4000,MEM_COMMIT|MEM_RESERVE,PAGE_EXECUTE_READWRITE);

	if ( pAddress != NULL ){
		dwFunAddress = GetFunAddress(fun);
		memcpy(pAddress,(void*)dwFunAddress,0x200);
	}

	printf("拷贝函数成功\n");
	//进入0环之前要先修改GDTR
	_asm{
		_emit(0x9a)

		_emit(0x10)
		_emit(0x10)
		_emit(0x10)
		_emit(0x10)

		_emit(0xc3)
		_emit(0x00)
	}

	printf("调用门函数调用成功\n");
	printf("w_cs = %x, w_ss = %x\n", w_cs,w_ss);
	printf("--------------PDE-----------\n");
	pReadAddress = (DWORD*)0xc0300000;

    
	for ( i = 0; i < 0x3; i++){
		printf("[0x%08x] = 0x%08x\n", pReadAddress+i, *(pReadAddress+i));
	}
    
	for ( i = 0; i < 0x100; i += 2){
		printf(" 0x%08x-------0x%08x\n", *(DWORD*)&PDTable[i+1],*(DWORD*)&PDTable[i]);
	}
	return 0;
}
```

