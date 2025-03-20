# 1. Heap Memory Management

1. 为什么要对内存进行管理？
	内核对象，如：任务、队列、信号量、时间组等需要在编译时静态分配RAM空间，或者在运行时动态分配空间
	- 动态分配：减少了设计和规划工作，简化了应用程序接口，并最大限度地减少了 RAM 占用
	- 静态分配：更具有确定性，无需处理内存分配失败，并消除了堆碎片风险

2. FreeRTOS中的内存管理
	考虑到嵌入式系统的性能问题，不使用C语言自带的malloc()和free()，而是使用pvPortMalloc()和vPortFree()
	pvPortMalloc()和vPortFree()是公共函数，可以在应用程序代码中调用

# 2. Example Memory Allocation Schemes

FreeRTOS 提供了五个 pvPortMalloc() 和 vPortFree() 的实现示例，本章将对它们进行详细介绍。FreeRTOS 应用程序可以使用其中一个示例实现，也可以提供自己的实现。
## 2.1 Heap_1

- 应用于小型嵌入式系统，只在启动FreeRTOS scheduler之前创建任务和其它内核对象
- 内核只会在应用程序开始执行任何实时功能之前（动态）分配内存，而且在应用程序的整个生命周期内，内存一直处于分配状态
- Heap_1是确定的，不会造成内存碎片
### Block Manager

```c title:Heap_Size_Config
/* A few bytes might be lost to byte aligning the heap start address. */

#define configADJUSTED_HEAP_SIZE        ( configTOTAL_HEAP_SIZE - portBYTE_ALIGNMENT )

/* Max value that fits in a size_t type. */

#define heapSIZE_MAX                    ( ~( ( size_t ) 0 ) )

/* Check if adding a and b will result in overflow. */

#define heapADD_WILL_OVERFLOW( a, b )   ( ( a ) > ( heapSIZE_MAX - ( b ) ) )
```
configTOTAL_HEAP_SIZE：实际堆空间的内存大小，受制于硬件性能
portBYTE_ALIGNMENT：堆空间对齐，在TOTAL_HEAP_SIZE的基础上减去了portBYTE_ALIGNMENT字节用于保证对齐
heapSIZE_MAX：由数据类型决定的理论最大堆空间大小（如最大数据类型是unsigned long long，则该值=unsigned long long的最大值）

```c title:Heap_Memory_Manager
/* Allocate the memory for the heap. */
#if ( configAPPLICATION_ALLOCATED_HEAP == 1 )

/* The application writer has already defined the array used for the RTOS
 * heap - probably so it can be placed in a special segment or address. */
    extern uint8_t ucHeap[ configTOTAL_HEAP_SIZE ];
#else
    static uint8_t ucHeap[ configTOTAL_HEAP_SIZE ];
#endif /* configAPPLICATION_ALLOCATED_HEAP */

/* Index into the ucHeap array. */
static size_t xNextFreeByte = ( size_t ) 0U;
```
static uint8_t ucHeap[configTOTAL_HEAP_SIZE]：
- 这里static关键字的作用：保持ucHeap的生命周期覆盖整个程序运行期间，并且让全局变量ucHeap仅在当前源文件中可见，保持封装不暴露给其它模块
	补充：
	![[Pasted image 20250118133353.png|800]]
- 堆空间的总大小为configTOTAL_HEAP_SIZE个uint8_t（也就是unsigned char）
xNextFreeByte：下一块空白堆空间的地址，初始设置为0
### Malloc

``` c title:Malloc
void * pvPortMalloc( size_t xWantedSize )
{
    void * pvReturn = NULL;
    static uint8_t * pucAlignedHeap = NULL;

    /* Ensure that blocks are always aligned. */
    #if ( portBYTE_ALIGNMENT != 1 )
    {
        size_t xAdditionalRequiredSize;
		
        if( ( xWantedSize & portBYTE_ALIGNMENT_MASK ) != 0x00 )
        {
            /* Byte alignment required. */
            xAdditionalRequiredSize = portBYTE_ALIGNMENT - ( xWantedSize & portBYTE_ALIGNMENT_MASK );

            if( heapADD_WILL_OVERFLOW( xWantedSize, xAdditionalRequiredSize ) == 0 )
            {
                xWantedSize += xAdditionalRequiredSize;
            }
            else
            {
                xWantedSize = 0;
            }
        }
    }
    #endif /* if ( portBYTE_ALIGNMENT != 1 ) */

    vTaskSuspendAll();
    {
        if( pucAlignedHeap == NULL )
        {
            /* Ensure the heap starts on a correctly aligned boundary. */
            pucAlignedHeap = ( uint8_t * ) ( ( ( portPOINTER_SIZE_TYPE ) &( ucHeap[ portBYTE_ALIGNMENT - 1 ] ) ) &
                                             ( ~( ( portPOINTER_SIZE_TYPE ) portBYTE_ALIGNMENT_MASK ) ) );
        }

        /* Check there is enough room left for the allocation. */
        if( ( xWantedSize > 0 ) &&
            ( heapADD_WILL_OVERFLOW( xNextFreeByte, xWantedSize ) == 0 ) &&
            ( ( xNextFreeByte + xWantedSize ) < configADJUSTED_HEAP_SIZE ) )
        {
            /* Return the next free byte then increment the index past this
             * block. */
            pvReturn = pucAlignedHeap + xNextFreeByte;
            xNextFreeByte += xWantedSize;
        }

        traceMALLOC( pvReturn, xWantedSize );
    }
    ( void ) xTaskResumeAll();

    #if ( configUSE_MALLOC_FAILED_HOOK == 1 )
    {
        if( pvReturn == NULL )
        {
            vApplicationMallocFailedHook();
        }
    }
    #endif

    return pvReturn;
}
```
3-4行：
pvReturn：储存分配内存的起始地址
pucAlignedHeap：指向堆起始地址的指针，用 static 修饰，保证它在多次调用中保留其值，初始化为 NULL

6-26行：
- 如果需要考虑内存对齐（portBYTE_ALIGNMENT != 1），先判断要对齐多少字节
- xAdditionalRequiredSize里记录由于内存对齐需要额外申请的字节数
- ( xWantedSize & portBYTE_ALIGNMENT_MASK ) != 0x00：portBYTE_ALIGNMENT_MASK一般等于portBYTE_ALIGNMENT-1，这样取&就类似于计算WantedSize能否整除portBYTE_ALIGNMENT。若不能整除，则结果!=0x00，即需要额外的字节来对齐
- xAdditionalRequiredSize = portBYTE_ALIGNMENT - ( xWantedSize & portBYTE_ALIGNMENT_MASK )：结果即xAdditionalRequiredSize%portBYTE_ALIGNMENT
- 若不会造成理论上的堆溢出，那么就把对齐所需的字节加进总字节里，否则总字节=0

28-50行：
- vTaskSuspendAll()：进入临界区，暂停任务切换，保护内存分配过程免受中断影响
- 若pucAlignedHeap为NULL，初始化pucAlignedHeap
``` c
pucAlignedHeap = ( uint8_t * ) ( ( ( portPOINTER_SIZE_TYPE ) &( ucHeap[ portBYTE_ALIGNMENT - 1 ] ) ) &
                                             ( ~( ( portPOINTER_SIZE_TYPE ) portBYTE_ALIGNMENT_MASK ) ) )
```
ucHeap[ portBYTE_ALIGNMENT - 1 ]：向高位偏移地址（不到一个对齐的长度），再和~portBYTE_ALIGNMENT_MASK做&操作：MASK=ALIGN（2的整数倍）-1。~MASK的低位全部为0，即去掉不对齐的低位地址。

``` c title:Malloc
/* Check there is enough room left for the allocation. */
if( ( xWantedSize > 0 ) &&
	( heapADD_WILL_OVERFLOW( xNextFreeByte, xWantedSize ) == 0 ) &&
	( ( xNextFreeByte + xWantedSize ) < configADJUSTED_HEAP_SIZE ) )
{
	/* Return the next free byte then increment the index past this
	 * block. */
	pvReturn = pucAlignedHeap + xNextFreeByte;
	xNextFreeByte += xWantedSize;
}
traceMALLOC( pvReturn, xWantedSize );
#if ( configUSE_MALLOC_FAILED_HOOK == 1 )
    {
        if( pvReturn == NULL )
        {
            vApplicationMallocFailedHook();

        }
    }
#endif
return pvReturn;
```
xNextFreeByte为堆空间第一块空白区域的索引
判断再申请xWantedSize块空间后是否会溢出，若不溢出则修改pvReturn和xNextFreeByte
完成错误处理后返回pvReturn，实现内存申请
### Free
Heap1没有实现Free的功能
### 总结
由于使用静态分配数组的方式实现，堆成为了FreeTOS的一部分，会看起来消耗大量RAM。
每个动态分配的任务都会导致两次调用 pvPortMalloc()，第一次给任务控制块（TCB）分配空间，第二次给task stack分配空间。
![[Pasted image 20250216211114.png|725]]
## 2.2 Heap_2

- Heap_2已经被Heap_4取代，后者包含增强的功能，但为了向后兼容，保留了Heap_2，实际进行开发时不建议再使用。
- 与Heap_1一样，由于使用了静态数组的方式实现，堆空间成为了FreeRTOS数据的一部分，因此看起来会占用大量的ROM。
- 对于堆空间分配，Heap_2使用了<span style="background:rgba(240, 107, 5, 0.2)">best-fit algorithm（最佳匹配算法）</span>，优先分配大小与申请空间大小最接近的free block。
- 与Heap_4不同，Heap_2不会将相邻的空闲块合并成一个较大的块，因此更容易出现空间碎片。
- Heap_2对于空闲块的管理：使用了一个LinkList，从小到大排列。
### Block_Manager

![[Pasted image 20250217133239.png|1100]]
- heapBLOCK_ALLOCATED_BITMASK：size_t位的数，最高位为1，作为掩码
- heapBLOCK_SIZE_IS_VALID( xBlockSize )：若xBlockSize最高位为0，则有效
- heapBLOCK_IS_ALLOCATED( pxBlock )：若xBlockSize为1，说明块已经被分配
- heapALLOCATE_BLOCK( pxBlock )、heapFREE_BLOCK( pxBlock )：分配时置最高位为1，释放时置最高位为0

``` c title:Heap_Manage_LinkList
/* Define the linked list structure.  This is used to link free blocks in order
 * of their size. */
typedef struct A_BLOCK_LINK
{
    struct A_BLOCK_LINK * pxNextFreeBlock; /*<< The next free block in the list. */
    size_t xBlockSize;                     /*<< The size of the free block. */
} BlockLink_t;


static const size_t xHeapStructSize = ( ( sizeof( BlockLink_t ) + ( size_t ) ( portBYTE_ALIGNMENT - 1 ) ) & ~( ( size_t ) portBYTE_ALIGNMENT_MASK ) );
#define heapMINIMUM_BLOCK_SIZE    ( ( size_t ) ( xHeapStructSize * 2 ) )

/* Create a couple of list links to mark the start and end of the list. */
PRIVILEGED_DATA static BlockLink_t xStart, xEnd;

/* Keeps track of the number of free bytes remaining, but says nothing about
 * fragmentation. */
PRIVILEGED_DATA static size_t xFreeBytesRemaining = configADJUSTED_HEAP_SIZE;

/* Indicates whether the heap has been initialised or not. */
PRIVILEGED_DATA static BaseType_t xHeapHasBeenInitialised = pdFALSE;

/*-----------------------------------------------------------*/

/*
 * Initialises the heap structures before their first use.
 */
static void prvHeapInit( void ) PRIVILEGED_FUNCTION;

/*-----------------------------------------------------------*/

/* STATIC FUNCTIONS ARE DEFINED AS MACROS TO MINIMIZE THE FUNCTION CALL DEPTH. */

/*
 * Insert a block into the list of free blocks - which is ordered by size of
 * the block.  Small blocks at the start of the list and large blocks at the end
 * of the list.
 */
#define prvInsertBlockIntoFreeList( pxBlockToInsert )                                                                 
    {                                                                                                                 
        BlockLink_t * pxIterator;                                                                                     
        size_t xBlockSize;                                                                                            
        xBlockSize = pxBlockToInsert->xBlockSize;
        /* Iterate through the list until a block is found that has a larger size */                                  
        /* than the block we are inserting. */
        for( pxIterator = &xStart; pxIterator->pxNextFreeBlock->xBlockSize < xBlockSize; 
	    pxIterator = pxIterator->pxNextFreeBlock )
        {                                                                                                             
            /* There is nothing to do here - just iterate to the correct position. */                                 
        }
        /* Update the list to include the block being inserted in the correct */                                      
        /* position. */                                                                                               
        pxBlockToInsert->pxNextFreeBlock = pxIterator->pxNextFreeBlock;
        pxIterator->pxNextFreeBlock = pxBlockToInsert;
    }
/*-----------------------------------------------------------*/
```
- 3-7行：定义了控制块的链表结构，pxNextFreeBlock为指针，指向下一个块；xBlockSize代表块大小
- xHeapStructSize：取得比sizeof( BlockLink_t )大的最小的portBYTE_ALIGNMENT倍数，保证对齐
- 39-55行：<span style="background:rgba(240, 107, 5, 0.2)">#define prvInsertBlockIntoFreeList( pxBlockToInsert )</span>为宏定义，定义了插入一个新块的方法。链表头插法，块从小到大排列
### Malloc

```c title:Calculate_Total_Size
void * pvPortMalloc( size_t xWantedSize )
{
    BlockLink_t * pxBlock;
    BlockLink_t * pxPreviousBlock;
    BlockLink_t * pxNewBlockLink;
    void * pvReturn = NULL;
    size_t xAdditionalRequiredSize;
    size_t xAllocatedBlockSize = 0;

    if( xWantedSize > 0 )
    {
        /* The wanted size must be increased so it can contain a BlockLink_t
         * structure in addition to the requested amount of bytes. */
        if( heapADD_WILL_OVERFLOW( xWantedSize, xHeapStructSize ) == 0 )
        {
            xWantedSize += xHeapStructSize;

            /* Ensure that blocks are always aligned to the required number
             * of bytes. */
            if( ( xWantedSize & portBYTE_ALIGNMENT_MASK ) != 0x00 )
            {
                /* Byte alignment required. */
                xAdditionalRequiredSize = portBYTE_ALIGNMENT - ( xWantedSize & portBYTE_ALIGNMENT_MASK );

                if( heapADD_WILL_OVERFLOW( xWantedSize, xAdditionalRequiredSize ) == 0 )
                {
                    xWantedSize += xAdditionalRequiredSize;
                }
                else
                {
                    xWantedSize = 0;
                }
            }
            else
            {
                mtCOVERAGE_TEST_MARKER();
            }
        }
        else
        {
            xWantedSize = 0;
        }
    }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }
```
14-43行：
- 若xWantSize>0，则判断加上xHeapStructSize是否溢出。不溢出就给xWantSize再加上xHeapStructSize（即一个链表节点对齐后的大小）
- 判断新的xWantSize是否对齐，若不对齐则再次对齐，加上xAdditionalRequiredSize的大小
- 若对齐了，或者xWantSize大小=0，则mtCOVERAGE_TEST_MARKER();（好像是用来测试分支覆盖率的，不用管）
- 如果判断出会溢出，则xWantSize=0

```c title:Init_Heap
    vTaskSuspendAll();
    {
        /* If this is the first call to malloc then the heap will require
         * initialisation to setup the list of free blocks. */
        if( xHeapHasBeenInitialised == pdFALSE )
        {
            prvHeapInit();
            xHeapHasBeenInitialised = pdTRUE;
        }
```
vTaskSuspendAll()：进入临界区，停止任务切换
若堆空间未初始化，进行初始化

```c title:prvHeapInit()
static void prvHeapInit( void ) /* PRIVILEGED_FUNCTION */
{
    BlockLink_t * pxFirstFreeBlock;
    uint8_t * pucAlignedHeap;

    /* Ensure the heap starts on a correctly aligned boundary. */
    pucAlignedHeap = ( uint8_t * ) ( ( ( portPOINTER_SIZE_TYPE ) & ucHeap[ portBYTE_ALIGNMENT - 1 ] ) & ( ~( ( portPOINTER_SIZE_TYPE ) portBYTE_ALIGNMENT_MASK ) ) );

    /* xStart is used to hold a pointer to the first item in the list of free
     * blocks.  The void cast is used to prevent compiler warnings. */
    xStart.pxNextFreeBlock = ( void * ) pucAlignedHeap;
    xStart.xBlockSize = ( size_t ) 0;

    /* xEnd is used to mark the end of the list of free blocks. */
    xEnd.xBlockSize = configADJUSTED_HEAP_SIZE;
    xEnd.pxNextFreeBlock = NULL;

    /* To start with there is a single free block that is sized to take up the
     * entire heap space. */
    pxFirstFreeBlock = ( BlockLink_t * ) pucAlignedHeap;
    pxFirstFreeBlock->xBlockSize = configADJUSTED_HEAP_SIZE;
    pxFirstFreeBlock->pxNextFreeBlock = &xEnd;
}
```
7行：与Heap_1一致，让堆开始地址对齐
11-22行：对堆空间以及链表初始化
- xStart：头节点，size=0，next指向堆空间起始地址
- xEnd：尾节点，size=configADJUSTED_HEAP_SIZE（整个堆空间大小），next=NULL
- pxFirstFreeBlock：第一个空闲块的指针，size=configADJUSTED_HEAP_SIZE，next=&xEnd

```c title:Allocate_Block
if( heapBLOCK_SIZE_IS_VALID( xWantedSize ) != 0 )
{
	if( ( xWantedSize > 0 ) && ( xWantedSize <= xFreeBytesRemaining ) )
	{
		/* Blocks are stored in byte order - traverse the list from the start
		 * (smallest) block until one of adequate size is found. */
		pxPreviousBlock = &xStart;
		pxBlock = xStart.pxNextFreeBlock;

		while( ( pxBlock->xBlockSize < xWantedSize ) && ( pxBlock->pxNextFreeBlock != NULL ) )
		{
			pxPreviousBlock = pxBlock;
			pxBlock = pxBlock->pxNextFreeBlock;
		}

		/* If we found the end marker then a block of adequate size was not found. */
		if( pxBlock != &xEnd )
		{
			/* Return the memory space - jumping over the BlockLink_t structure
			 * at its start. */
			pvReturn = ( void * ) ( ( ( uint8_t * ) pxPreviousBlock->pxNextFreeBlock ) + xHeapStructSize );

			/* This block is being returned for use so must be taken out of the
			 * list of free blocks. */
			pxPreviousBlock->pxNextFreeBlock = pxBlock->pxNextFreeBlock;

			/* If the block is larger than required it can be split into two. */
			if( ( pxBlock->xBlockSize - xWantedSize ) > heapMINIMUM_BLOCK_SIZE )
			{
				/* This block is to be split into two.  Create a new block
				 * following the number of bytes requested. The void cast is
				 * used to prevent byte alignment warnings from the compiler. */
				pxNewBlockLink = ( void * ) ( ( ( uint8_t * ) pxBlock ) + xWantedSize );

				/* Calculate the sizes of two blocks split from the single
				 * block. */
				pxNewBlockLink->xBlockSize = pxBlock->xBlockSize - xWantedSize;
				pxBlock->xBlockSize = xWantedSize;

				/* Insert the new block into the list of free blocks.
				 * The list of free blocks is sorted by their size, we have to
				 * iterate to find the right place to insert new block. */
				prvInsertBlockIntoFreeList( ( pxNewBlockLink ) );
			}

			xFreeBytesRemaining -= pxBlock->xBlockSize;

			xAllocatedBlockSize = pxBlock->xBlockSize;

			/* The block is being returned - it is allocated and owned
			 * by the application and has no "next" block. */
			heapALLOCATE_BLOCK( pxBlock );
			pxBlock->pxNextFreeBlock = NULL;
		}
	}
}
```
检查：
- xWantedSize是否合法（即xWantedSize的最高位不等于1）
- 目前的堆中是否有xWantedSize大小的空闲空间
若满足以上条件，则开始查询链表并分配空间
7-14行：
- prev指针：指向头节点xStart
- block指针：指向xStart.next
- 向后移动两个指针，直到block->size大于xWantedSize，或者block->next为NULL时停下
- 此时block指向的要么是能够分配的空闲块，要么是xEnd（尾节点）
17-43行：
- 若block指针不指向xEnd，说明目前指向的是可以分配出去的块
- `pvReturn = ( void * ) ( ( ( uint8_t * ) pxPreviousBlock->pxNextFreeBlock ) + xHeapStructSize )`：pvReturn返回的是申请的地址，前xHeapStructSize是用来存链表节点的，因此要跳过去，所以加上xHeapStructSize
- `pxPreviousBlock->pxNextFreeBlock = pxBlock->pxNextFreeBlock`：由于块已经分配出去，所以从记录free block的链表中去掉该块
- 若该分配的块BlockSize-xWantedSize比最小块尺寸heapMINIMUM_BLOCK_SIZE大，说明该块可分，小的块再插回链表中
- 调用函数 prvInsertBlockIntoFreeList( pxBlockToInsert )插入新块，代码如下：

```c title:prvInsertBlockIntoFreeList
/*
 * Insert a block into the list of free blocks - which is ordered by size of
 * the block.  Small blocks at the start of the list and large blocks at the end
 * of the list.
 */
#define prvInsertBlockIntoFreeList( pxBlockToInsert )
    {
        BlockLink_t * pxIterator;
        size_t xBlockSize;
        xBlockSize = pxBlockToInsert->xBlockSize;
        /* Iterate through the list until a block is found that has a larger size */
        /* than the block we are inserting. */
        for( pxIterator = &xStart; pxIterator->pxNextFreeBlock->xBlockSize < xBlockSize; pxIterator = pxIterator->pxNextFreeBlock )
        {
            /* There is nothing to do here - just iterate to the correct position. */
        }
        /* Update the list to include the block being inserted in the correct */
        /* position. */
        pxBlockToInsert->pxNextFreeBlock = pxIterator->pxNextFreeBlock;
        pxIterator->pxNextFreeBlock = pxBlockToInsert;
    }
/*-----------------------------------------------------------*/
```
注意：<span style="background:rgba(240, 107, 5, 0.2)">该函数使用#define这样的宏定义方式实现，而不是普通的函数</span>，这样做的原因？
主要是为了优化性能，避免函数调用开销。
函数调用时会产生帧栈开销，使用#define可以直接插入代码，避免函数调用。此外，编译器在优化时可能会不够智能，如果用static inline可能仍然会有函数调用，而#define的宏定义方式会强制展开，避免编译器做不必要的优化判断。

插入代码逻辑：free block LinkList按照块尺寸从小到大排列，向后遍历直到找到插入位置后插入

46-54行：修改一些标记
- xFreeBytesRemaining：减去pxBlock->xBlockSize
- xAllocatedBlockSize：等于pxBlock->xBlockSize
- heapALLOCATE_BLOCK( pxBlock )：把该块对应的链表节点的xBlockSize最高位修改为1，代表已分配
- pxBlock->pxNextFreeBlock = NULL
### Free

```c title:Free
void vPortFree( void * pv )
{
    uint8_t * puc = ( uint8_t * ) pv;
    BlockLink_t * pxLink;

    if( pv != NULL )
    {
        /* The memory being freed will have an BlockLink_t structure immediately
         * before it. */
        puc -= xHeapStructSize;

        /* This unexpected casting is to keep some compilers from issuing
         * byte alignment warnings. */
        pxLink = ( void * ) puc;

        configASSERT( heapBLOCK_IS_ALLOCATED( pxLink ) != 0 );
        configASSERT( pxLink->pxNextFreeBlock == NULL );

        if( heapBLOCK_IS_ALLOCATED( pxLink ) != 0 )
        {
            if( pxLink->pxNextFreeBlock == NULL )
            {
                /* The block is being returned to the heap - it is no longer
                 * allocated. */
                heapFREE_BLOCK( pxLink );
                #if ( configHEAP_CLEAR_MEMORY_ON_FREE == 1 )
                {
                    ( void ) memset( puc + xHeapStructSize, 0, pxLink->xBlockSize - xHeapStructSize );
                }
                #endif

                vTaskSuspendAll();
                {
                    /* Add this block to the list of free blocks. */
                    prvInsertBlockIntoFreeList( ( ( BlockLink_t * ) pxLink ) );
                    xFreeBytesRemaining += pxLink->xBlockSize;
                    traceFREE( pv, pxLink->xBlockSize );
                }
                ( void ) xTaskResumeAll();
            }
        }
    }
}
```
3-17行：检查该块是否合法
- 传入参数pv-链表节点尺寸=分配出的整个block的起始地址
- 断言检查：BlockSize的最高位为1，next指向NULL
19-38行：释放块（修改xBlockSize最高位为0）、插入到free block LinkList中，并且修改xFreeBytesRemaining

Heap_2()不是确定性的（可能会申请不到空间），但是比C语言的标准库malloc()和free()更快

## 2.3 Heap_3

Heap_3.c调用了C语言的标准库中的malloc()和free()，但是通过vTaskSuspendAll()和xTaskResumeAll()这种方式暂时中止了FreeRTOS的scheduler，保证了线程安全。

## 2.4 Heap_4

Heap_4和Heap_1、Heap_2一样，使用了静态数组的方式来实现堆空间，因此会看起来占用了大量的RAM。
Heap_4使用了first-fit算法（优先匹配算法）去分配空间，选择遍历时大于所需块大小的第一个块
与Heap_2不同，Heap_4能够将相邻的空闲合并为一个较大的内存块，降低了内存碎片的风险，适合反复申请并释放不同大小内存区块的场合。

结构图示：
![|375](attachments/Pasted%20image%2020250304175253.png)
代码解析：
#### Block_Manager
和Heap_2基本上一样，多了个内存保护机制：

```c title:#define
/* Keeps track of the number of calls to allocate and free memory as well as the
 * number of free bytes remaining, but says nothing about fragmentation. */
PRIVILEGED_DATA static size_t xFreeBytesRemaining = ( size_t ) 0U;
PRIVILEGED_DATA static size_t xMinimumEverFreeBytesRemaining = ( size_t ) 0U;
PRIVILEGED_DATA static size_t xNumberOfSuccessfulAllocations = ( size_t ) 0U;
PRIVILEGED_DATA static size_t xNumberOfSuccessfulFrees = ( size_t ) 0U;
```
- xFreeBytesRemaining：当前堆空间剩余空间（Malloc和Free时候修改）
- xMinimumEverFreeBytesRemaining：历史堆空间最低剩余空间（Malloc时候修改）
- xNumberOfSuccessfulAllocations：成功分配计数（Malloc时候修改）
- xNumberOfSuccessfulFrees：成功释放计数（Free时候修改）

```c title:prvHeapInit

```
#### Malloc
和Heap_2.c逻辑基本一样

#### Free
和Heap_2.c逻辑基本一样

释放块后要把Free Block插入到Free Block List中，代码如下
```c prvInsertBlockIntoFreeList
static void prvInsertBlockIntoFreeList( BlockLink_t * pxBlockToInsert ) /* PRIVILEGED_FUNCTION */
{
    BlockLink_t * pxIterator;
    uint8_t * puc;

    /* Iterate through the list until a block is found that has a higher address
     * than the block being inserted. */
    for( pxIterator = &xStart; heapPROTECT_BLOCK_POINTER( pxIterator->pxNextFreeBlock ) < pxBlockToInsert; pxIterator = heapPROTECT_BLOCK_POINTER( pxIterator->pxNextFreeBlock ) )
    {
        /* Nothing to do here, just iterate to the right position. */
    }

    if( pxIterator != &xStart )
    {
        heapVALIDATE_BLOCK_POINTER( pxIterator );
    }

    /* Do the block being inserted, and the block it is being inserted after
     * make a contiguous block of memory? */
    puc = ( uint8_t * ) pxIterator;

    if( ( puc + pxIterator->xBlockSize ) == ( uint8_t * ) pxBlockToInsert )
    {
        pxIterator->xBlockSize += pxBlockToInsert->xBlockSize;
        pxBlockToInsert = pxIterator;
    }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }

    /* Do the block being inserted, and the block it is being inserted before
     * make a contiguous block of memory? */
    puc = ( uint8_t * ) pxBlockToInsert;

    if( ( puc + pxBlockToInsert->xBlockSize ) == ( uint8_t * ) heapPROTECT_BLOCK_POINTER( pxIterator->pxNextFreeBlock ) )
    {
        if( heapPROTECT_BLOCK_POINTER( pxIterator->pxNextFreeBlock ) != pxEnd )
        {
            /* Form one big block from the two blocks. */
            pxBlockToInsert->xBlockSize += heapPROTECT_BLOCK_POINTER( pxIterator->pxNextFreeBlock )->xBlockSize;
            pxBlockToInsert->pxNextFreeBlock = heapPROTECT_BLOCK_POINTER( pxIterator->pxNextFreeBlock )->pxNextFreeBlock;
        }
        else
        {
            pxBlockToInsert->pxNextFreeBlock = heapPROTECT_BLOCK_POINTER( pxEnd );
        }
    }
    else
    {
        pxBlockToInsert->pxNextFreeBlock = pxIterator->pxNextFreeBlock;
    }

    /* If the block being inserted plugged a gap, so was merged with the block
     * before and the block after, then it's pxNextFreeBlock pointer will have
     * already been set, and should not be set here as that would make it point
     * to itself. */
    if( pxIterator != pxBlockToInsert )
    {
        pxIterator->pxNextFreeBlock = heapPROTECT_BLOCK_POINTER( pxBlockToInsert );
    }
    else
    {
        mtCOVERAGE_TEST_MARKER();
    }
}
```
与Heap_2.c不同的是，Heap_4的Free Block List是按照地址排序的，即地址越大越排在后面
然后遍历块，直到找到适合插入的位置
比较前一块->next是否等于pxBlockToInsert，若相等则与前一块合并，否则前一块指向该块
再比较后一块是否相连，若相连且不为xEnd则合并，否则指向xEnd

Heap_4是不确定的，但是比标准库的malloc()和free()更快

## 2.5 Heap_5

Heap_5和Heap_4使用相同的算法，但是Heap_4是使用了一整块静态数组实现，而Heap_5可以将多个分开的空间合并为一个堆
当FreeRTOS系统提供的RAM在内存映射中不显示为单个块时，Heap_5就会派上用场

因为算法一样，Malloc、Free、Free List的插入和删除都是和Heap_4一样的，但Free List构建的时候不同

Heap_5需要实现一个Memory Map，手动定义：
```c
static const HeapRegion_t xHeapRegions[] =
{
    { ( uint8_t * ) 0x20000000, 0x1000 }, // 第一块堆区域，起始地址 0x20000000，大小 4KB
    { ( uint8_t * ) 0x20001000, 0x2000 }, // 第二块堆区域，起始地址 0x20001000，大小 8KB
    { ( uint8_t * ) 0x20003000, 0x4000 }, // 第三块堆区域，起始地址 0x20003000，大小 16KB
    { NULL, 0 } // 终止符
};
```