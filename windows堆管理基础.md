# Windows堆管理基础 #



## Windows堆的数据结构和管理 ##
程序员使用堆时要做三件事情，申请一定的内存，使用内存，释放内存；对于堆管理系统来说，要设计一套高效的数据结构来支持用户的操作。现代操作系统的堆数据结构一般包括堆块和堆表两类。

堆块：
出于性能考虑，堆区的内存块按不同大小组织成块，以堆块为单位进行标识。一个堆块包括两个部分：块首和块身。块首是堆块首部的几个字节，标识堆块本身的信息，例如大小，使用状态等；块身是紧跟在块首后面的部分，也是最终分配给用户使用的数据区。

堆表：堆表一般位于堆区的起始位置，用于索引堆区中所有堆块的重要信息，包括堆块的位置，堆块的大小，空闲还是占用。堆表的数据结构决定了整个堆区的组织方式。堆表的数据结构决定了整个堆区的组织方式，是快速检索空闲块，保证堆分配效率的关键。堆表在设计时可能会考虑使用平衡二叉树等高级数据结构用户优化查询效率。

在windows中，堆表只索引空闲状态的堆块，占用状态的堆块由使用的进程索引。最重要的堆表有两种，空闲双向链表Freelist(空表）和快速单向链表Lookaside（快表）.

空表：
堆区一开始的堆表区中有一个128项的指针数组，被称作空表索引，该数组每一项包括两个指针，用于标志一条空表。

快表：
快表是windows用来加速堆块分配而采用的一种堆表。之所以称为快表是因为这类单向链表中从来不发生堆块合并。快表也有128条，组织结构与空表类似，只是其中堆块按照单链表组织。快表总是被初始化为空，而且每条快表最多只有4个结点。

## Windows堆分配 ##


windows在创建一个新进程的时候，系统为其分配默认堆， 默认堆的句柄会被保存在进程环境块（PEB）的ProcessHeap中；可以通过GetProcessHeap()获取默认堆的句柄，然后调用HeapAlloc从默认堆分配空间了。

    HANDLE GetProcessHeap()

除了进程的默认堆，应用程序也可以调用HeapCreate函数创建私有堆。

    WINBASEAPI HANDLE WINAPI HeapCreate(DWORD,DWORD,DWORD);

进程的默认堆和私有堆没有什么区别，应用程序都可以通过HeapAlloc分配空间。

    WINBASEAPI PVOID WINAPI HeapAlloc(HANDLE,DWORD,DWORD);

堆管理器在创建堆时创建的第一个段，我们将其称为0号段。如果堆是可增长的，当一个段不能满足要求时，堆管理器会继续创建其他段。但最多可以有64个段。段内部又由堆块构成。
每个堆使用_HEAP结构来描述。

###WinXP/2000的堆结构###

![_HEAP](http://ooaecudja.bkt.clouddn.com/heap.png)


_HEAP结构记录该堆的属性和资产情况。因此该结构也被称为是堆的头结构。调用HeapCreate函数返回的句柄便是此结构的地址。

VirtualMemoryThreshold为虚拟内存分配阈值，表示可以在段中分配的堆块的最大有效（即应用程序可以实际使用的）值，该值为508kB。当应用程序从堆中分配的堆块的最大大小大于该值的申请，堆管理器会直接从内存管理器中分配，并不会从从空闲链表申请。同时将此空间添加到VirtualAllocdBlocks结构所指向的链表中。

VirtualAllocdBlocks是一个链表的头指针，该链表维护着所有大于VirtualMemoryThreshold直接从内存管理器申请的空间。

Segments是一个数组，它记录着堆拥有的所有段。每个元素类型为_HEAP_SEGMENT结构。
LastSegmentIndex表示堆中最后一个段的序号，加1便是总段数。

FreeLists是一个双向链表的头指针(就是我们所说的空表），该链表记录着所有空闲堆块的地址。链表元素为FREE_LIST结构.

###Win7堆结构###
我使用的是Win7系统，Win7的堆结构和WinXP/2000不一样：

    typedef struct _HEAP_ENTRY  
    {  
    UINT SubSegmentCode;  
    USHORT PreviousSize;  
    BYTE SegmentOffset;  
    BYTE UnusedBytes;  
    }HEAP_ENTRY;  
      
    
    typedef struct _HEAP_SEGMENT  
      
    {  
    HEAP_ENTRY Entry;  
    UINT   SegmentSignature;  
    UINT   SegmentFlags;  
    LIST_ENTRY SegmentListEntry; //各heap_segment通过此字段连接  
    PHEAP Heap;  //指向所属的heap  
    //...省略若干字段  
    LIST_ENTRY UCRSegmentList;  
    }HEAP_SEGMENT; 
    
    typedef struct _HEAP  
      
    {  
    HEAP_SEGMENT Segment;  
    UINT   Flags;  
    UINT   ForceFlags;  
    //...省略若干字段  
    LIST_ENTRY SegmentList;  //通过此字段找到各heap_segment,从0号段开始，自然首先同HEAP最开始处那个HEAP_SEGMENT的SegmentListEntry链接  
    //...省略若干字段  
    HEAP_TUNING_PARAMETERS TuningParameters;  
    }*PHEAP, HEAP;  

可以看出，最大区别是HEAP结构记录HEAP_SEGMENT的方式采用了链表，这样不再受数组大小的约束，同时将HEAP_SEGMENT字段包含进HEAP，这样各堆段的起始便统一为HEAP_SEGMENT，不再有xp下0号段与其他段那种区别，可以统一进行管理了。


## Windows堆调试 ##

用于调试的代码(Eclipse/Mingw环境，汇编采用AT&T GCC模式）：

    #include <windows.h>
    main(){
    	HLOCAL h1,h2,h3,h4,h5,h6;
    	HANDLE hp;
    	hp = HeapCreate(0, 0x1000,0x10000);
    	__asm__("int $3");
    	h1 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 3);
    	h2 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 5);
    	h3 = HeapAlloc(hp, HEAP_ZERO_MEMORY,6);
    	h4 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 8);
    	h5 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 19);
    	h6 = HeapAlloc(hp, HEAP_ZERO_MEMORY, 24);
    	HeapFree(hp, 0, h1);
    	HeapFree(hp, 0, h3);
    	HeapFree(hp, 0, h5);
    	HeapFree(hp, 0, h4);
    
    	return 0;
    
    }

调试环境：

|操作系统|win7|
|编译器|Eclipse+CDT|
|调试器|OllyDbg|
|build版本|Release版本|

调试堆与调试栈不同，不能直接使用Ollydbg来加载程序，否则堆管理函数会检查到当前进程处于调试状态，从而调整堆管理策略。为了避免程序检测出调试器而使用调试堆管理策略，需要在创建堆之后加入一个人工断点：__asm__("int $3")然后让程序单独执行。当程序把堆初始化完后，断点会中断程序，再使用调试器attach进程,就能看到真实的堆了。

设置Ollydbg为just-in-time debugger；在进程中断时，选择“调试程序”,Ollydbg就会自动attach进程并停止在int 3指令处。
![](http://ooaecudja.bkt.clouddn.com/debug4.png)

###识别堆表###
我们已经看到了进程的地址空间，那么我们的堆表在哪里呢？当HeapCreate()成功创建堆表之后，会把分配的堆区(_HEAP结构)的起始地址放在EAX寄存器，从上图可以看到EAX中存储的是0x00030000,我们到内存中找这个地址看一下：

![](http://ooaecudja.bkt.clouddn.com/headaddr1.png)

结合我们前面提供的Win7 HEAP数据结构来查看。

HEAP头部是第一个内存块：HEAP_SEGMENT[HEAP_ENTRY]

       +0x000 SubSegmentCode: 0xfb722164 
	   +0x004 PreviousSize:0xdfea
	   +0x006 SegmentOffset:0x0
	   +0x007 UnusedBytes:0x1  _HEAP_ENTRY
       +0x008 SegmentSignature : 0xffeeffee   
       +0x00c SegmentFlags : 0
       +0x010 SegmentListEntry : _LIST_ENTRY [ 0x6800a8 - 0x6800a8 ]
       +0x018 Heap : 0x00680000 _HEAP
       +0x01c BaseAddress  : 0x00680000 Void
       +0x020 NumberOfPages: 0x10
       +0x024 FirstEntry   : 0x00680588 _HEAP_ENTRY
       +0x028 LastValidEntry   : 0x00690000 _HEAP_ENTRY
       +0x02c NumberOfUnCommittedPages : 0x0f
       +0x030 NumberOfUnCommittedRanges : 0x01
       +0x034 SegmentAllocatorBackTraceIndex : 0
       +0x036 Reserved : 0
       +0x038 UCRSegmentList   : _LIST_ENTRY [ 0x680ff0 - 0x680ff0 ]

下面是剩下的部分：

       +0x040 Flags: 0x1000
       +0x044 ForceFlags   : 0
       +0x048 CompatibilityFlags : 0
       +0x04c EncodeFlagMask   : 0x100000
       +0x050 Encoding :0x4b7321d5-0x0000dfea _HEAP_ENTRY
       +0x058 PointerKey   : 0x3255a4cd
       +0x05c Interceptor  : 0
       +0x060 VirtualMemoryThreshold : 0xfe00
       +0x064 Signature: 0xeeffeeff
       +0x068 SegmentReserve   : 0x100000
       +0x06c SegmentCommit: 0x2000
       +0x070 DeCommitFreeBlockThreshold : 0x200
       +0x074 DeCommitTotalFreeThreshold : 0x2000
       +0x078 TotalFreeSize: 0x14b
       +0x07c MaximumAllocationSize : 0x7ffdefff
       +0x080 ProcessHeapsListIndex : 0x03
       +0x082 HeaderValidateLength : 0x138
       +0x084 HeaderValidateCopy : (null) 
       +0x088 NextAvailableTagIndex : 0
       +0x08a MaximumTagIndex  : 0
       +0x08c TagEntries   : (null) 
       +0x090 UCRList  : _LIST_ENTRY [ 0x680fe8 - 0x680fe8 ]
       +0x098 AlignRound   : 0xf
       +0x09c AlignMask: 0xfffffff8
       +0x0a0 VirtualAllocdBlocks : _LIST_ENTRY [ 0x6800a0 - 0x6800a0 ]
       +0x0a8 SegmentList  : _LIST_ENTRY [ 0x680010 - 0x680010 ]
       +0x0b0 AllocatorBackTraceIndex : 0
       +0x0b4 NonDedicatedListLength : 0
       +0x0b8 BlocksIndex  : 0x00680150 Void
       +0x0bc UCRIndex : (null) 
       +0x0c0 PseudoTagEntries : (null) 
       +0x0c4 FreeLists: _LIST_ENTRY [ 0x680590 - 0x680590 ]
       +0x0cc LockVariable : 0x00680138 _HEAP_LOCK
       +0x0d0 CommitRoutine: 0x3255a4cd long  +17c06e63
       +0x0d4 FrontEndHeap : (null) 
       +0x0d8 FrontHeapLockCount : 0
       +0x0da FrontEndHeapType : 0 ''
       +0x0dc Counters : 0x010000_HEAP_COUNTERS
       +0x130 TuningParameters : _HEAP_TUNING_PARAMETERS

###堆块分配###
经过逐步分配，内存数据如下图：

![](http://ooaecudja.bkt.clouddn.com/heapafteralloc.png)

其中h1=0x00680590 freelist=0x6805a0,
 h2=0x006805a0,freelist=0x6805b0,
 h3=0x006805b0, freelist=0x6805c0,
 h4=0x006805c0,freelist=0x6805d0,
 h5=0x006805d0,freelist=0x6805f0,
 h6=0x006805F0,freelist=0x680610; 
可以看到在逐步分配的过程中heap的FreeList始终指向下一个可用的地址。
经过上述调试可以理解：

1. 初始状态下，快表和空表都是空的，不存在精确分配。HEAP的FreeList指向可分配的用户区域。
2. 分配函数会陆续从尾块中切走一些小块，最后把Freelist[0]指向新的尾块。

###堆块释放###
经过前三次内存释放，内存数据如下：
![](http://ooaecudja.bkt.clouddn.com/heapfree1.png)

freelist的结构如下
freelist[0]->h3(6805b0)->h1(680590)->h5(6805d0)->freemem(0x680610)

###Win7堆疑问###
1. freelist并不是一个数组，没有看出来；但是freelist的确是一个双向链表；
2. 有必要采用windbg对进程的堆进行调试和分析其结构



	





















