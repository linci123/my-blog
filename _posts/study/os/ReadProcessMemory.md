---
title: 'ReadProcessMemory调用过程'
author: Kevin。
tags:
  - os
  - Windows
categories:
  - study
date: 2022-07-02 20:44:00
img: /images/article-banner/QQ截图20220702234227.jpg
typora-root-url: ..\..\..\
---



# ReadProcessMemory调用过程

## 基本用法

根据进程读句柄取内存空间

```c
BOOL ReadProcessMemory(
    HANDLE hProcess,  //进程句柄
    LPCVOID lpBaseAddress, // 内存地址
    LPVOID lpBuffer, // 接收的内容，缓冲区指针
    DWORD nSize, // 读取的字节数，如果为0直接返回
    LPDWORD lpNumberOfBytesRead //指向传输到指定缓冲区的字节数的指针
							 //如果lpNumberOfBytesRead为空，则忽略该参数
)//  //成功返回1 失败返回0
```

## 函数分析

![ReadProcessMemory函数](/images/image-20220702204623967.png)

由反汇编可以看出，`ReadProcessMemory`函数直接调用了`NtReadVirtualMemory`，而`NtReadVirtualMemory`在ntdll.dll中，`ntdll`又与`ntoskrnl.exe`对应，在`ntoskrnl.exe`中找到`NtReadVirtualMemory`

### NtReadVirtualMemory

经过一系列初始化，函数调用了`MmCopyVirtualMemory`

```c
004B154F                 push    eax             // HandleInformation
004B1550                 lea     eax, [ebp+Object]
004B1553                 push    eax             // Object
004B1554                 push    dword ptr [ebp+AccessMode] // AccessMode
004B1557                 push    _PsProcessType  // ObjectType
004B155D                 push    10h             // DesiredAccess
004B155F                 push    [ebp+ProcessHandle] // Handle
004B1562                 call    _ObReferenceObjectByHandle@24 // 跟据进程句柄获取EPROCESS结构的指针
004B1567                 mov     [ebp+var_Eprocess], eax
004B156A                 test    eax, eax 
004B156C                 jnz     short loc_4B1592 //如果_ObReferenceObjectByHandle返回0 则调用_MmCopyVirtualMemory
004B156E                 lea     eax, [ebp+var_size]
//--------------------------------------传入参数-------------------------------------
004B1571                 push    eax             // BytesRead
004B1572                 push    dword ptr [ebp+AccessMode] // AccessMode
004B1575                 push    esi             // NumberOfBytesToRead
004B1576                 push    [ebp+Buffer]    // Buffer
004B1579                 push    [edi+_KTHREAD.ApcState.Process] // CurrentProcess
004B157C                 push    [ebp+BaseAddress] // BaseAddress
004B157F                 push    [ebp+Object]    // Process
//--------------------------------------传入参数-------------------------------------
004B1582                 call    _MmCopyVirtualMemory@28 // // 调用_MmCopyVirtualMemory
004B1587                 mov     [ebp+var_Eprocess], eax
004B158A                 mov     ecx, [ebp+Object] // Object
004B158D                 call    @ObfDereferenceObject@4 // ObfDereferenceObject(x)
```

### MmCopyVirtualMemory

```c
004A22D5                 mov     edi, edi
004A22D7                 push    ebp
004A22D8                 mov     ebp, esp
004A22DA                 cmp     [ebp+Length], 0 //如果要读取的长度为0则直接返回
004A22DE                 jz      loc_526787
004A22E4                 push    ebx
004A22E5                 mov     ebx, [ebp+process]
004A22E8                 mov     ecx, ebx
004A22EA                 mov     eax, large fs:_KPCR.PrcbData.CurrentThread
004A22F0                 cmp     ebx, [eax+_KTHREAD.ApcState.Process] 
004A22F3                 jnz     short loc_4A22F8
004A22F5                 mov     ecx, [ebp+CurrentProcess]//如果目的线程和当前线程相同，将目标线程指针替换成当前线程
004A22F8
004A22F8 loc_4A22F8:                            
004A22F8                 add     ecx, 80h //    RunRef
004A22FE                 mov     [ebp+process], ecx
004A2301                 call    @ExAcquireRundownProtection@4
004A2306                 test    al, al
004A2308                 jz      loc_52678E
004A230E                 cmp     [ebp+Length], 1FFh
004A2315                 push    esi //保存esi环境
004A2316                 push    edi //保存edi环境
004A2317                 mov     edi, [ebp+arg_18]
004A231A                 ja      loc_4ADEF1  //如果要读取的长度大于1FF，则调用_MiDoMappedCopy
004A2320
004A2320 loc_4A2320:                             
004A2320                 push    edi             // BytesRead
004A2321                 push    dword ptr [ebp+AccessMode]
004A2324                 push    [ebp+Length]    // Length
004A2327                 push    [ebp+Address]   // Address
004A232A                 push    [ebp+CurrentProcess] // TAEGET_PROCESS
004A232D                 push    [ebp+BaseAddress] // int
004A2330                 push    ebx             // PRKPROCESS
004A2331                 call    _MiDoPoolCopy@28 // // 否则调用_MiDoPoolCopy
004A2336                 mov     esi, eax
004A2338
004A2338 loc_4A2338:                            
004A2338                 mov     ecx, [ebp+process] // RunRef
004A233B                 call    @ExReleaseRundownProtection@4 // ExReleaseRundownProtection(x)
004A2340                 pop     edi
004A2341                 mov     eax, esi
004A2343                 pop     esi
004A2344
004A2344 loc_4A2344:                           
004A2344                 pop     ebx
004A2345
004A2345 loc_4A2345:                            
004A2345                 pop     ebp
004A2346                 retn    1Ch
```

### MiDoMappedCopy

```c
004ADD5C                 push    0ACh
004ADD61                 push    offset stru_41C3D0
004ADD66                 call    __SEH_prolog
004ADD6B                 xor     edi, edi
004ADD6D                 mov     [ebp+Status], edi // STATUS_SUCCESS
004ADD70                 mov     eax, [ebp+arg_BaseAddress]
004ADD73                 mov     [ebp+CurrentAddress], eax
004ADD76                 mov     eax, [ebp+Address]
004ADD79                 mov     [ebp+TargetAddress], eax
004ADD7C                 mov     eax, 0E000h     // MI_MAPPED_COPY_PAGES * PAGE_SIZE
004ADD81 bufferSize = ecx
004ADD81 TotalSize = eax
004ADD81                 mov     bufferSize, [ebp+Length]
004ADD84                 cmp     bufferSize, TotalSize
004ADD86                 ja      short loc_4ADD8A // 如果要读取的长度小于等于0xe000，则将要读取的长度设定为0xe000
004ADD88                 mov     TotalSize, bufferSize
004ADD8A
004ADD8A loc_4ADD8A:                             
004ADD8A                 lea     ebx, [ebp+MemoryDescriptorList] // MDL是用来建立一块虚拟地址空间与物理页面之间的映射
004ADD90                 mov     [ebp+MDL], ebx
004ADD93                 mov     [ebp+remainingSize], bufferSize
004ADD96                 mov     [ebp+CurrentSize], TotalSize
004ADD99                 mov     [ebp+FailedInProbe], edi
004ADD9C                 mov     [ebp+BadAddress], edi // 0
004ADD9F                 mov     [ebp+HaveBadAddress], edi // 0
004ADDA2
004ADDA2 loc_4ADDA2:                            
004ADDA2                 mov     eax, [ebp+remainingSize] // while(remainingSize > 0)
004ADDA5                 cmp     eax, edi
004ADDA7                 jbe     loc_4ADEDF      // return
004ADDAD                 cmp     eax, [ebp+CurrentSize]
004ADDB0                 jb      loc_50DFB6      // 如果RemainingSize < CurrentSize，则CurrentSize = RemainingSize
004ADDB6
004ADDB6 loc_4ADDB6:                            
004ADDB6                 lea     eax, [ebp+ApcState]
004ADDB9                 push    eax             // ApcState
004ADDBA                 push    [ebp+PROCESS]   // PROCESS
004ADDBD                 call    _KeStackAttachProcess@8 // 将本进程附加到目标进程中
004ADDC2                 mov     [ebp+BaseAddress], edi
004ADDC5                 mov     [ebp+PagesLocked], edi
004ADDC8                 mov     [ebp+var_4C], edi
004ADDCB                 mov     [ebp+ms_exc.registration.TryLevel], edi
004ADDCE                 mov     eax, [ebp+arg_BaseAddress]
004ADDD1                 xor     esi, esi
004ADDD3                 inc     esi
004ADDD4                 cmp     [ebp+CurrentAddress], eax
004ADDD7 True = esi
004ADDD7                 jnz     short loc_4ADE05 // 初始化MemoryDescriptorList
004ADDD9                 cmp     [ebp+AccessMode], 0
004ADDDD                 jz      short loc_4ADE05 // 初始化MemoryDescriptorList
004ADDDF                 mov     [ebp+FailedInProbe], True
004ADDE2                 mov     eax, [ebp+Length]
004ADDE5                 cmp     eax, edi
004ADDE7                 jz      short loc_4ADE02
004ADDE9                 mov     ecx, [ebp+arg_BaseAddress]
004ADDEC                 add     eax, ecx
004ADDEE                 cmp     eax, ecx
004ADDF0                 jb      loc_50DFBE
004ADDF6                 cmp     eax, _MmUserProbeAddress
004ADDFC                 ja      loc_50DFBE
004ADE02
004ADE02 loc_4ADE02:                             
004ADE02                 mov     [ebp+FailedInProbe], edi
004ADE05
004ADE05 loc_4ADE05:                             
004ADE05                 mov     [ebx+_MDL.Next], edi // 初始化MemoryDescriptorList
004ADE07                 mov     eax, [ebp+CurrentAddress]
004ADE0A                 and     eax, 0FFFh
004ADE0F                 mov     ecx, [ebp+CurrentSize]
004ADE12                 lea     edx, [eax+ecx+0FFFh]
004ADE19                 shr     edx, 0Ch
004ADE1C                 lea     edx, [edx*4+1Ch]
004ADE23                 mov     [ebx+_MDL.Size], dx
004ADE27                 mov     [ebx+_MDL.MdlFlags], di
004ADE2B                 mov     edx, [ebp+CurrentAddress]
004ADE2E                 and     edx, 0FFFFF000h
004ADE34                 mov     [ebx+_MDL.StartVa], edx
004ADE37                 mov     [ebx+_MDL.ByteOffset], eax
004ADE3A                 mov     [ebx+_MDL.ByteCount], ecx
004ADE3D                 push    edi             // Operation
004ADE3E                 push    dword ptr [ebp+AccessMode] // AccessMode
004ADE41                 push    ebx             // MemoryDescriptorList
004ADE42                 call    _MmProbeAndLockPages@12 // MmProbeAndLockPages(x,x,x)
004ADE47                 mov     [ebp+PagesLocked], True
004ADE4A                 push    20h             // Priority -> HighPagePriority
004ADE4C                 push    edi             // BugCheckOnFailure -> False
004ADE4D                 push    edi             // RequestedAddress -> NULL
004ADE4E                 push    esi             // CacheType -> MmCached
004ADE4F                 push    edi             // AccessMode -> KernelMode
004ADE50                 push    ebx             // MemoryDescriptorList
004ADE51                 call    _MmMapLockedPagesSpecifyCache@24 
004ADE56                 mov     [ebp+BaseAddress], eax
004ADE59                 cmp     eax, edi
004ADE5B                 jz      loc_526621
004ADE61
004ADE61 loc_4ADE61:                        
004ADE61                 lea     eax, [ebp+ApcState]
004ADE64                 push    eax             // ApcState
004ADE65                 call    _KeUnstackDetachProcess@4 // KeUnstackDetachProcess(x)
004ADE6A                 lea     eax, [ebp+ApcState]
004ADE6D                 push    eax             // ApcState
004ADE6E                 push    [ebp+CurrentProcess] // PROCESS
004ADE71                 call    _KeStackAttachProcess@8 // KeStackAttachProcess(x,x)
004ADE76                 mov     eax, [ebp+arg_BaseAddress]
004ADE79                 cmp     [ebp+CurrentAddress], eax
004ADE7C                 jnz     short loc_4ADE96
004ADE7E                 cmp     [ebp+AccessMode], 0
004ADE82                 jz      short loc_4ADE96
004ADE84                 mov     [ebp+FailedInProbe], esi
004ADE87                 push    esi             // Alignment
004ADE88                 push    [ebp+Length]    // Length
004ADE8B                 push    [ebp+Address]   // Address
004ADE8E                 call    _ProbeForWrite@12 // ProbeForWrite(x,x,x)
004ADE93                 mov     [ebp+FailedInProbe], edi
004ADE96
004ADE96 loc_4ADE96:                             
004ADE96                                       
004ADE96                 mov     [ebp+var_4C], esi
004ADE99                 mov     ecx, [ebp+CurrentSize]
004ADE9C                 mov     esi, [ebp+BaseAddress]
004ADE9F                 mov     edi, [ebp+TargetAddress]
004ADEA2                 mov     eax, ecx
004ADEA4                 shr     ecx, 2
004ADEA7                 rep movsd
004ADEA9                 mov     ecx, eax
004ADEAB                 and     ecx, 3
004ADEAE                 rep movsb
004ADEB0                 or      [ebp+ms_exc.registration.TryLevel], 0FFFFFFFFh
004ADEB4                 lea     eax, [ebp+ApcState]
004ADEB7                 push    eax             // ApcState
004ADEB8                 call    _KeUnstackDetachProcess@4 // KeUnstackDetachProcess(x)
004ADEBD                 push    ebx             // MemoryDescriptorList
004ADEBE                 push    [ebp+BaseAddress] // BaseAddress
004ADEC1                 call    _MmUnmapLockedPages@8 // MmUnmapLockedPages(x,x)
004ADEC6                 push    ebx             // MemoryDescriptorList
004ADEC7                 call    _MmUnlockPages@4 // MmUnlockPages(x)
004ADECC                 mov     eax, [ebp+CurrentSize]
004ADECF                 sub     [ebp+remainingSize], eax
004ADED2                 add     [ebp+CurrentAddress], eax
004ADED5                 add     [ebp+TargetAddress], eax
004ADED8                 xor     edi, edi
004ADEDA                 jmp     loc_4ADDA2      // while(remainingSize > 0)
004ADEDF // ---------------------------------------------------------------------------
004ADEDF
004ADEDF loc_4ADEDF:                            
004ADEDF                 mov     eax, [ebp+BytesRead]
004ADEE2                 mov     ecx, [ebp+Length]
004ADEE5                 mov     [eax], ecx
004ADEE7                 xor     eax, eax
004ADEE9
004ADEE9 loc_4ADEE9:                            
004ADEE9                 call    __SEH_epilog
004ADEEE                 retn    1Ch
```

### MiDoPoolCopy

```c
004AD387 TotalSize = edi
004AD387 
004AD387                 push    248h
004AD38C                 push    offset stru_41D0D8
004AD391                 call    __SEH_prolog
004AD396                 mov     eax, large fs:_KPCR.PrcbData.CurrentThread
004AD39C                 mov     eax, [ebp+SourceAddress]
004AD39F                 mov     [ebp+CurrentAddress], eax
004AD3A2                 mov     eax, [ebp+TargetAddress]
004AD3A5                 mov     [ebp+CurrentTargetAddress], eax
004AD3A8                 mov     eax, 10000h     // MI_MAX_TRANSFER_SIZE = 64 * 1024
004AD3AD                 mov     TotalSize, eax
004AD3AF                 cmp     [ebp+BufferSize], eax // if (BufferSize <= MI_MAX_TRANSFER_SIZE) TotalSize = BufferSize//
004AD3B2                 ja      short loc_4AD3B7
004AD3B4                 mov     TotalSize, [ebp+BufferSize]
004AD3B7
004AD3B7 loc_4AD3B7:                             
004AD3B7                 and     [ebp+HavePoolAddress], 0
004AD3BB                 mov     esi, 200h       // MI_POOL_COPY_BYTES
004AD3C0                 cmp     [ebp+BufferSize], esi
004AD3C3                 ja      loc_5266C7      // ExAllocatePoolWithTag
004AD3C9
004AD3C9 loc_4AD3C9:                             
004AD3C9                 lea     eax, [ebp+StackBuffer]
004AD3CF                 mov     [ebp+PoolAddress], eax
004AD3D2
004AD3D2 loc_4AD3D2:                             
004AD3D2                 xor     eax, eax
004AD3D4                 mov     [ebp+Status], eax
004AD3D7                 mov     [ebp+_], eax
004AD3DA                 mov     ecx, [ebp+BufferSize]
004AD3DD                 mov     [ebp+RemainingSize], ecx
004AD3E0                 mov     ebx, TotalSize
004AD3E2                 mov     [ebp+FailedInProbe], eax // STATUS_SUCCESS
004AD3E5
004AD3E5 loc_4AD3E5:                             
004AD3E5                 xor     eax, eax
004AD3E7                 cmp     [ebp+RemainingSize], eax
004AD3EA                 ja      loc_4AD496      // RemainingSize < CurrentSize
004AD3F0                 cmp     [ebp+HavePoolAddress], eax
004AD3F3                 jnz     loc_526779      // return
004AD3F9
004AD3F9 loc_4AD3F9:                             
004AD3F9                 mov     eax, [ebp+ReturnSize]
004AD3FC                 mov     ecx, [ebp+BufferSize]
004AD3FF                 mov     [eax], ecx
004AD401                 xor     eax, eax
004AD403
004AD403 loc_4AD403:                             
004AD403                                         // MiDoPoolCopy(x,x,x,x,x,x,x)+793ED↓j
004AD403                 call    __SEH_epilog
004AD408                 retn    1Ch
004AD40B // ---------------------------------------------------------------------------
004AD40B
004AD40B loc_4AD40B:                            
004AD40B                                         // MiDoPoolCopy(x,x,x,x,x,x,x)+13C↓j ...
004AD40B                 mov     ecx, ebx        // RtlCopyMemory -> memcpy
004AD40D                 mov     esi, [ebp+CurrentAddress]
004AD410                 mov     edi, [ebp+PoolAddress]
004AD413                 mov     eax, ecx
004AD415                 shr     ecx, 2
004AD418                 rep movsd               // 把ESI的内容拷贝到EDI中
004AD418                                         // 把数据读到临时空间，这个临时空间是在内核堆栈或内核的内存中，即高2G的位置
004AD41A                 mov     ecx, eax
004AD41C                 and     ecx, 3
004AD41F                 rep movsb               // 对齐
004AD421                 lea     eax, [ebp+ApcState]
004AD424                 push    eax             // ApcState
004AD425                 call    _KeUnstackDetachProcess@4 // KeUnstackDetachProcess(x)
004AD42A                 lea     eax, [ebp+ApcState]
004AD42D                 push    eax             // ApcState
004AD42E                 push    [ebp+TAEGET_PROCESS] // PROCESS
004AD431                 call    _KeStackAttachProcess@8 // KeStackAttachProcess(x,x)
004AD436                 mov     eax, [ebp+SourceAddress]
004AD439                 cmp     [ebp+CurrentAddress], eax
004AD43C                 jnz     loc_4E24CD
004AD442                 cmp     [ebp+PreviousMode], 0
004AD446                 jz      loc_4E24CD
004AD44C                 xor     esi, esi
004AD44E                 inc     esi
004AD44F                 mov     [ebp+FailedInProbe], esi
004AD452                 push    esi             // Alignment
004AD453                 push    [ebp+BufferSize] // Length
004AD456                 push    [ebp+TargetAddress] // Address
004AD459                 call    _ProbeForWrite@12 // ProbeForWrite(x,x,x)
004AD45E                 and     [ebp+FailedInProbe], 0
004AD462
004AD462 loc_4AD462:                          
004AD462                 mov     [ebp+var_3C], esi
004AD465                 mov     ecx, ebx
004AD467                 mov     esi, [ebp+PoolAddress]
004AD46A                 mov     edi, [ebp+CurrentTargetAddress]
004AD46D                 mov     eax, ecx
004AD46F                 shr     ecx, 2
004AD472                 rep movsd
004AD474                 mov     ecx, eax
004AD476                 and     ecx, 3
004AD479                 rep movsb
004AD47B                 or      [ebp+Exception.registration.TryLevel], 0FFFFFFFFh
004AD47F                 lea     eax, [ebp+ApcState]
004AD482                 push    eax             // ApcState
004AD483                 call    _KeUnstackDetachProcess@4 // KeUnstackDetachProcess(x)
004AD488                 sub     [ebp+RemainingSize], ebx
004AD48B                 add     [ebp+CurrentAddress], ebx
004AD48E                 add     [ebp+CurrentTargetAddress], ebx
004AD491                 jmp     loc_4AD3E5
004AD496 // ---------------------------------------------------------------------------
004AD496
004AD496 loc_4AD496:                             
004AD496                 cmp     [ebp+RemainingSize], ebx
004AD499                 jb      loc_5266E7      // CurrentSize = RemainingSize//
004AD49F
004AD49F loc_4AD49F:                           
004AD49F 0 = esi
004AD49F                 lea     eax, [ebp+ApcState]
004AD4A2                 push    eax             // ApcState
004AD4A3                 push    [ebp+SourceProcess] // PROCESS
004AD4A6                 call    _KeStackAttachProcess@8 // KeStackAttachProcess(x,x)
004AD4AB                 xor     esi, esi
004AD4AD                 mov     [ebp+var_3C], 0
004AD4B0                 mov     [ebp+Exception.registration.TryLevel], 0
004AD4B3                 mov     ecx, [ebp+SourceAddress]
004AD4B6                 cmp     [ebp+CurrentAddress], ecx // (CurrentAddress == SourceAddress) && (PreviousMode != KernelMode)
004AD4B9                 jnz     loc_4AD40B      // RtlCopyMemory -> memcpy
004AD4BF                 cmp     [ebp+PreviousMode], 0 //  KernelMode
004AD4C3                 jz      loc_4AD40B      // RtlCopyMemory -> memcpy
004AD4C9                 mov     [ebp+FailedInProbe], 1
004AD4D0                 mov     eax, [ebp+BufferSize]
004AD4D3                 cmp     eax, 0
004AD4D5                 jz      short loc_4AD4ED
004AD4D7                 add     eax, ecx
004AD4D9                 cmp     eax, ecx
004AD4DB                 jb      loc_4E24D5
004AD4E1                 cmp     eax, _MmUserProbeAddress
004AD4E7                 ja      loc_4E24D5
004AD4ED
004AD4ED loc_4AD4ED:                            
004AD4ED                 mov     [ebp+FailedInProbe], esi
004AD4F0                 jmp     loc_4AD40B      // RtlCopyMemory -> memcpy
004AD4F0 
```
#### loc_4AD496
```c
004AD496                 cmp     [ebp+RemainingSize], ebx
004AD499                 jb      loc_5266E7      // CurrentSize = RemainingSize
004AD49F
004AD49F loc_4AD49F:                             
004AD49F o = esi
004AD49F                 lea     eax, [ebp+ApcState]
004AD4A2                 push    eax             // ApcState
004AD4A3                 push    [ebp+SourceProcess] // PROCESS
004AD4A6                 call    _KeStackAttachProcess@8 // KeStackAttachProcess(x,x)
004AD4AB                 xor     o, o
004AD4AD                 mov     [ebp+var_3C], o
004AD4B0                 mov     [ebp+Exception.registration.TryLevel], o
004AD4B3                 mov     ecx, [ebp+SourceAddress]
004AD4B6                 cmp     [ebp+CurrentAddress], ecx // (CurrentAddress == SourceAddress) && (PreviousMode != KernelMode)
004AD4B9                 jnz     loc_4AD40B      // RtlCopyMemory -> memcpy
004AD4BF                 cmp     [ebp+PreviousMode], 0 //  KernelMode
004AD4C3                 jz      loc_4AD40B      // RtlCopyMemory -> memcpy
004AD4C9                 mov     [ebp+FailedInProbe], 1
004AD4D0                 mov     eax, [ebp+BufferSize]
004AD4D3                 cmp     eax, o
004AD4D5                 jz      short loc_4AD4ED
004AD4D7                 add     eax, ecx
004AD4D9                 cmp     eax, ecx
004AD4DB                 jb      loc_4E24D5
004AD4E1                 cmp     eax, _MmUserProbeAddress
004AD4E7                 ja      loc_4E24D5
004AD4ED
004AD4ED loc_4AD4ED:
004AD4ED                 mov     [ebp+FailedInProbe], esi
004AD4F0                 jmp     loc_4AD40B      // RtlCopyMemory -> memcpy
```

### KeStackAttachProcess

```c
0041CF95                 mov     edi, edi
0041CF97                 push    ebp
0041CF98                 mov     ebp, esp
0041CF9A                 push    esi
0041CF9B                 push    edi
0041CF9C                 mov     eax, large fs:_KPCR.PrcbData.CurrentThread
0041CFA2                 mov     esi, eax
0041CFA4                 mov     eax, large fs:_KPCR.PrcbData.DpcRoutineActive
0041CFAA                 test    eax, eax
0041CFAC                 jnz     loc_44A944
0041CFB2                 mov     edi, [ebp+PROCESS]
0041CFB5                 cmp     [esi+_KTHREAD.ApcState.Process], edi
0041CFB8                 jz      loc_41D0C8
0041CFBE                 xor     ecx, ecx
0041CFC0                 call    ds:__imp_@KeAcquireQueuedSpinLockRaiseToSynch@4 
0041CFC6                 cmp     [esi+_KTHREAD.ApcStateIndex], OriginalApcEnvironment
0041CFCD                 mov     byte ptr [ebp+PROCESS], al
0041CFD0                 jnz     loc_431CA8 //如果当前线程以及被附加上，则直接将ApcState传入_KiAttachProcess，否则传入SavedApcState
0041CFD6                 lea     eax, [esi+_KTHREAD.SavedApcState]
0041CFDC                 push    eax
0041CFDD                 push    [ebp+PROCESS]
0041CFE0                 push    edi
0041CFE1                 push    esi
0041CFE2                 call    _KiAttachProcess@16 // 把其他进程加载到本进程
0041CFE7                 mov     eax, [ebp+ApcState]
0041CFEA                 and     [eax+_KAPC_STATE.Process], 0
0041CFEE
0041CFEE loc_41CFEE:                             
0041CFEE                 pop     edi
0041CFEF                 pop     esi
0041CFF0                 pop     ebp
0041CFF1                 retn    8
    
0041D0C8 loc_41D0C8:  
0041D0C8                 mov     eax, [ebp+ApcState]
0041D0CB                 mov     [eax+_KAPC_STATE.Process], 1
0041D0D2                 jmp     loc_41CFEE
```

### KiAttachProcess

只要操作进程内存就会调用KiAttachProcess函数，hook KiAttachProcess函数即可知道哪个线程中的进程进来的

```c
0041A4C1                 mov     edi, edi
0041A4C3                 push    ebp
0041A4C4                 mov     ebp, esp
0041A4C6                 push    ebx
0041A4C7                 push    esi
0041A4C8                 mov     esi, [ebp+Thread]
0041A4CB                 push    edi
0041A4CC                 push    [ebp+SaveApcState]
0041A4CF                 mov     edi, [ebp+Process]
0041A4D2                 inc     [edi+_EPROCESS.Pcb.StackCount] // 当前进程内存中线程的栈计数+1
0041A4D6                 lea     ebx, [esi+_KTHREAD.ApcState] // ApcListHead[KernelMode]
0041A4D9                 push    ebx
0041A4DA                 call    _KiMoveApcState@8 // SaveApcState = ApcState，备份ApcState
0041A4DF                 mov     [ebx+_LIST_ENTRY.Blink], ebx
0041A4E2                 mov     [ebx+_LIST_ENTRY.Flink], ebx
0041A4E4                 lea     eax, [esi+(_KTHREAD.ApcState.ApcListHead.Flink+8)] // ApcListHead[UserMode]  8 = sizeof(LIST_ENTRY) * 1
0041A4E7                 mov     [eax+_LIST_ENTRY.Blink], eax
0041A4EA                 mov     [eax+_LIST_ENTRY.Flink], eax
0041A4EC                 lea     eax, [esi+_KTHREAD.SavedApcState]
0041A4F2                 cmp     [ebp+SaveApcState], eax
0041A4F5                 mov     [esi+_KTHREAD.ApcState.Process], edi // 更改为要读取的进程
0041A4F8                 mov     [esi+_KTHREAD.ApcState.KernelApcInProgress], 0
0041A4FC                 mov     [esi+_KTHREAD.ApcState.KernelApcPending], 0
0041A500                 mov     [esi+_KTHREAD.ApcState.UserApcPending], 0
0041A504                 jnz     short loc_41A519 // 判断进程是否在内存中
0041A506                 mov     [esi+_KTHREAD.ApcStatePointer], eax // 14C SavedApcState 替换指针的位置
0041A50C                 mov     [esi+(_KTHREAD.ApcStatePointer+4)], ebx // 34 ApcState
0041A512                 mov     [esi+_KTHREAD.ApcStateIndex], AttechedApcEnvironment // 改成挂靠的APC
0041A519
0041A519 loc_41A519:                             // CODE XREF: KiAttachProcess(x,x,x,x)+43↑j
0041A519                 cmp     [edi+_KPROCESS.State], ProcessInMemory
0041A51D                 jnz     loc_44A8A2      // 判断是否在内存中
0041A523                 lea     esi, [edi+40h]
0041A526
0041A526 loc_41A526:                             // CODE XREF: KiAttachProcess(x,x,x,x)+303D7↓j
0041A526                 mov     eax, [esi+_LIST_ENTRY.Flink]
0041A528                 cmp     eax, esi
0041A52A                 jnz     loc_44A87F      // 判断线程的等待链表是否为空
0041A530                 mov     eax, [ebp+SaveApcState]
0041A533                 push    [eax+_KAPC_STATE.Process]
0041A536                 push    edi
0041A537                 call    _KiSwapProcess@8 // KiSwapProcess(x,x)
0041A53C                 mov     cl, [ebp+ApcLock]
0041A53F                 call    @KiUnlockDispatcherDatabase@4 // KiUnlockDispatcherDatabase(x)
0041A544
0041A544 loc_41A544:                             // CODE XREF: 0044A89D↓j
0041A544                                         // KiAttachProcess(x,x,x,x)+30463↓j
0041A544                 pop     edi
0041A545                 pop     esi
0041A546                 pop     ebx
0041A547                 pop     ebp
0041A548                 retn    10h
```



