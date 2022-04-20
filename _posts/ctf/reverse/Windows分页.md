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

```
PDE = CR3 & 0xffffffe0 + PDI << 2
PTE = PDE & 0xfffff000 + PTI << 2
PA  = PTE & 0xfffff000 + OFFSET
```

CR3低5位、PDE低12位、PTE低12位均为属性

在WinDBG中调试，在调试前先将C:\boot.ini中的配置项改为

`multi(0)disk(0)rdisk(0)partition(1)\WINDOWS="Microsoft Windows XP Professional_debugf(10-10-12)" /execute=optin /fastdetect /debug /debugport=com1 /baudrate=115200`

配置中的 `execute `表示10-10-12分页方式，`noexecute`表示 2-9-9-12 分页方式， 和64位进行过渡

![IDTR中的值和物理IDTR对比](/images/image-20220418161423067.png)

## 内存跨页

当程序运行到页的边界时往往会出现跨页的情况，如`401ffe 401fff 402000 402001`这4个地址组成了一个完整的数据，由于页大小为0x1000，在40200处出现了跨页，此时需要拆分`401ffe `和`402000`

 有代码：

```assembly
_asm{
    mov eax, dword ptr ds:[0x401ffe];
    int 0x3;
}
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

## PDE属性

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

## PTE属性

和PDE的0相对应的位位D位，脏页位，即被使用过即为脏页



