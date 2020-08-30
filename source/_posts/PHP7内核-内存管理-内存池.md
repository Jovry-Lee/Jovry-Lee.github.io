---
title: PHP7内核-内存管理-内存池
date: 2020-08-25 14:29:45
tags: ["PHP"]
categories: ["PHP"]
---

#### 1 简介
在C语言中，通常直接使用malloc进行内存的分配，而频繁的分配、释放内存无疑会产生内存碎片，降低系统性能。PHP自己实现了一套内存池（`ZendMM`：`Zend Memery Manager`）用于替换`glibc`的`malloc`、`free`，以解决内存频繁分配、释放的问题。

<!--more-->

内存池技术主要作用：
- ①、减少内存分配及释放的次数
- ②、有效控制内存碎片的产生

PHP的内存池的实现参考了`tcmalloc`的设计，`tcmalloc`是`Google`开源的一个非常优秀的内存分配器。

内存池是PHP内核中最底层的内存操作，他是非常独立的一个模块，可以移植到其他C语言应用中去。  



**内存池定义了三种粒度的内存块**：

- **chunk**：每个chunk的大小为2M
- **page**：page的大小为4KB，每个chunk被切割为512个page（2048/4）
- **slot**：一个或若干个page被切割为多个slot



申请内存时按照不同的申请大小决定具体的分配策略：

- **Huge(chunk)**:申请内存大于2MB，<u>直接调用系统分配</u>，分配若干个chunk.(>2MB)
- **Large(page)**：申请内~~存大于3092B~~（即page大小的3/4, 4 * 1024 * 3/4 = 3072B）,小于2044KB（即511个page的大小），分配若干个page。(3072B~2044KB)
- **Small(slot)**:申请内存小于等于3092B(即page大小的3/4)，<u>内存池提前定义好了30种同等大小的内存（8、16、24、32...3072）,它们分配在不同的page上（不同大小的内存可能会分配在多个连续的page），申请内存时直接在对应的page上查找可用的slot</u>。(<3092B)

内存池通过**zend_mm_heap**结构存储内存池的主要信息，比如`大内存链表`、`chunk链表`、`slot各大小内存链表`等。
```C
// file:zend_alloc.c
# define AG(v) (alloc_globals.v)
static zend_alloc_globals alloc_globals;

// file:zend_alloc.h
typedef struct _zend_mm_heap zend_mm_heap;
struct _zend_mm_heap {
#if ZEND_MM_CUSTOM
	int                use_custom_heap;
#endif
#if ZEND_MM_STORAGE
	zend_mm_storage   *storage;
#endif
#if ZEND_MM_STAT
    // 当前已使用的内存数
	size_t             size;                    /* current memory usage */
	// 内存单次申请的峰值
	size_t             peak;                    /* peak memory usage */
#endif
    // 小内存分配的可用位置链表，ZEND_MM_BINS等于30，即此数组表示的是各种大小内存对应的链表头部
	zend_mm_free_slot *free_slot[ZEND_MM_BINS]; /* free lists for small sizes */

    ...

    // 大内存链表
	zend_mm_huge_list *huge_list;               /* list of huge allocated blocks */
    // 指向chunk链表头部
	zend_mm_chunk     *main_chunk;
	// 缓存的chunk链表
	zend_mm_chunk     *cached_chunks;			/* list of unused chunks */
	// 已分配的chunk数
	int                chunks_count;			/* number of alocated chunks */
	// 当前request使用chunk峰值
	int                peak_chunks_count;		/* peak number of allocated chunks for current request */
	// 缓存的chunk数
	int                cached_chunks_count;		/* number of cached chunks */
	// chunk使用均值，每次请求结束后会根据peak_chunk_count重新计算：(avg_chunks_count + peak_chunks_count)/2.0
	double             avg_chunks_count;		/* average number of chunks allocated per request */
	int                last_chunks_delete_boundary; /* numer of chunks after last deletion */
	int                last_chunks_delete_count;    /* number of deletion over the last boundary */
    ...
};
```
大内存分配的是若干个chunk，然后通过一个`zend_mm_huge_list结构`进行管理，**大内存之间构成一个单向链表**。
```c
// file:zend_alloc.h
typedef struct  _zend_mm_huge_list zend_mm_huge_list;

struct _zend_mm_huge_list {
	void              *ptr;
	size_t             size;
	zend_mm_huge_list *next;
#if ZEND_DEBUG
	zend_mm_debug_info dbg;
#endif
};
```
![huge_list链表](https://note.youdao.com/yws/api/personal/file/48872B5FBB994E018B73A4AF38B466CE?method=download&shareKey=b96499c0e19a3b8b9803bcfe2f50d9eb)

<u>chunk是内存池向系统申请、释放内存的最小粒度。</u>`chunk`之间构成双向链表，第一个chunk的地址保存于`zend_mm_heap->main_chunk`。

每个chunk的大小为2MB，被切割为512个page，所以每个page的大小为4KB，其中第一个page的内存用于chunk自己的结构体成员，主要记录chunk的一些信息，比如前后chunk的指针，当前chunk上各个page的使用情况等。

chunk的定义结构如下：
```c
// file:zend_alloc.c
struct _zend_mm_chunk {
	zend_mm_heap      *heap;
	// 指向下一个chunk
	zend_mm_chunk     *next;
	// 指向上一个chunk
	zend_mm_chunk     *prev;
	// 当前chunk的剩余可用page数
	int                free_pages;				/* number of free pages */
	int                free_tail;               /* number of free pages at the end of chunk */
	int                num;
	char               reserve[64 - (sizeof(void*) * 3 + sizeof(int) * 3)];
	// heap结构，只有主chunk会用到（即第一个chunk）
	zend_mm_heap       heap_slot;               /* used only in main chunk */
	// 标识各page是否已使用的bitmap，总大小为512bit，对应page总数，每个page占一个bit位。
	zend_mm_page_map   free_map;                /* 512 bits or 64 bytes */
	// 各page的信息：当前page使用类型（用于large分配还是small）、占用的page数等
	zend_mm_page_info  map[ZEND_MM_PAGES];      /* 2 KB = 512 * 4 */
};
```



slot内存是把若干个page按照固定大小分割好的内存块。

**内存池定义了30中大小的slot内存**：8、16、24、32...1792、2048、3072.这些slot的大小是有规律的。
- ①、最小的slot大小为8byte
- ②、前8个slot一次递增8byte（0~7递增 8byte）
- ③、后面每隔4个递增值乘以2（8~11递增16byte、12~15递增32byte、16~19递增64byte、20~23递增128byte、24~27递增256byte、28~30递增512byte）



每种大小的slot占用的page数不相同的：
- ①、slot0~15各占1个page

- ②、slot16~29分别占5、3、1、1、5、3、2、2、5、3、7、4、5、3个page；  
    注：这个值实际也是有规律可循的，其算法为，目的是为了减少内存的碎片。
    ```
    page的个数 = 最小公倍数(slot对应的内存大小, page的大小即4096) / 4096
    ```

    
    分配各个规格的slot时会按照各这个配置申请对应的数量的page，然后进行分割组成链表。<u>相同大小的slot之间构成单链表</u>。
```
struct _zend_mm_free_slot {
	zend_mm_free_slot *next_free_slot;
};
```

heap、huge、chunk、page、slot之间的关系如下图所示：
![heap、huge、chunk、page、slot结构关系](https://note.youdao.com/yws/api/personal/file/A8793900579C4D84A8CCE6E131F70B04?method=download&shareKey=7c1b42213c6357aae7855f4bcc43c78c)

#### 2 内存池的初始化
初始化过程主要是`分配heap结构`，<u>如果是多线程环境，则会为每一个线程分配一个内存池，线程之间互不影响</u>。

注：<u>`zend_mm_heap`这个结构不是单独分配的，它嵌在`chunk`结构体中（即`heap_slot`成员）</u>。也就是说内存池初始化时是分配了一个chunk结构，`zend_mm_chunk->heap_slot`作为内存池的heap结构，这个chunk也是第一个chunk，即main_chunk,如下图所示：
![zend_mm_heap结构的分配](https://note.youdao.com/yws/api/personal/file/1598F914D6D043A2928403E3B65280FE?method=download&shareKey=fa722c4892712fb681baea9ac6c93cf8)

**问题：为什么内存时的heap结构要嵌在chunk中而不是单独分配呢？** 
因为每个chunk的第一个page始终是给chunk结构体自己使用的，剩下的511个page才会做内存分配，但是chunk结构体并不需要一个page那么大的内存。也就是说被占用的page会有剩余的空间，因此，为了尽可能利用空间，就将heap结构嵌在了chunk中。

具体的分配过成在zend_mm_init()中实现：
```c
// file:zend_alloc.c
static zend_mm_heap *zend_mm_init(void)
{
    // 向系统申请2MB大小的chunk
	zend_mm_chunk *chunk = (zend_mm_chunk*)zend_mm_chunk_alloc_int(ZEND_MM_CHUNK_SIZE, ZEND_MM_CHUNK_SIZE);
	zend_mm_heap *heap;

	if (UNEXPECTED(chunk == NULL)) {
#if ZEND_MM_ERROR
#ifdef _WIN32
		stderr_last_error("Can't initialize heap");
#else
		fprintf(stderr, "\nCan't initialize heap: [%d] %s\n", errno, strerror(errno));
#endif
#endif
		return NULL;
	}
	// heap结构实际数主chunk嵌入的一个结构，后面再分配的chunk的heap_slot不再使用
	heap = &chunk->heap_slot;
	chunk->heap = heap;
	chunk->next = chunk;
	chunk->prev = chunk;
	// 剩余可用page数
	chunk->free_pages = ZEND_MM_PAGES - ZEND_MM_FIRST_PAGE;
	chunk->free_tail = ZEND_MM_FIRST_PAGE;
	chunk->num = 0
	// 将第一个page的bit分配标识位设置为1，表示已经被分配、占用
	chunk->free_map[0] = (Z_L(1) << ZEND_MM_FIRST_PAGE) - 1;
	// 第一个page的类型为ZEND_MM_IS_LRUN,即large内存
	chunk->map[0] = ZEND_MM_LRUN(ZEND_MM_FIRST_PAGE);
	// 指向主chunk
	heap->main_chunk = chunk;
	// 初始化剩下的一些成员
	...
	// huge内存链表
	heap->huge_list = NULL;
	return heap;
}

```

#### 3 内存的分配
Huge大内存的分配过程比较简单、而Large与Small内存分配涉及到page的查找操作，过程稍显复杂。

使用`emalloc`申请时，内存池会按照申请内存的大小自动选择那种格内存进行分配，如下图所示：
![内存的分配流程](https://note.youdao.com/yws/api/personal/file/1C273A1B04A44C379427B94CBE2A42FB?method=download&shareKey=d5059f5c0dafd673a661edf8efe31120)

##### 3.1 Huge分配
Huge是指超多2MB大小内存的分配，<u>实际分配时将对齐到n个chunk</u>，分配完还会分配一个zend_mm_huge_list结构，用于管理所有的Huge内存。
```c
static void *zend_mm_alloc_huge(zend_mm_heap *heap, size_t size ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC)
{
#ifdef ZEND_WIN32
	/* On Windows we don't have ability to extend huge blocks in-place.
	 * We allocate them with 2MB size granularity, to avoid many
	 * reallocations when they are extended by small pieces
	 */
	 // 按页大小重置实际要分配的内存
	size_t new_size = ZEND_MM_ALIGNED_SIZE_EX(size, MAX(REAL_PAGE_SIZE, ZEND_MM_CHUNK_SIZE));
#else
	size_t new_size = ZEND_MM_ALIGNED_SIZE_EX(size, REAL_PAGE_SIZE);
#endif
	void *ptr;

#if ZEND_MM_LIMIT
    // 如果有内存使用限制，则检查是否已达上限，达到的话进行zend_mm_gc清理后再检查
	if (UNEXPECTED(heap->real_size + new_size > heap->limit)) {
		if (zend_mm_gc(heap) && heap->real_size + new_size <= heap->limit) {
			/* pass */
		} else if (heap->overflow == 0) {
#if ZEND_DEBUG
			zend_mm_safe_error(heap, "Allowed memory size of %zu bytes exhausted at %s:%d (tried to allocate %zu bytes)", heap->limit, __zend_filename, __zend_lineno, size);
#else
			zend_mm_safe_error(heap, "Allowed memory size of %zu bytes exhausted (tried to allocate %zu bytes)", heap->limit, size);
#endif
			return NULL;
		}
	}
#endif
    // 分配chunk()，系统返回的地址是随机的，并不一定是ZEND_MM_CHUNK_ZIZE的整数倍，内存池需要自己移动到对齐的位置，比如：返回地址ptr是2000，而最近的一个对齐地址是2048，内存池会把ptr移动到2048，从这个位置使用。
	ptr = zend_mm_chunk_alloc(heap, new_size, ZEND_MM_CHUNK_SIZE);
   
    ... 
    
#if ZEND_DEBUG
    // 将申请的内存通过zend_mm_huge_list插入到链表中
	zend_mm_add_huge_block(heap, ptr, new_size, size ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC);
#else
	zend_mm_add_huge_block(heap, ptr, new_size ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC);
#endif
    ...
	return ptr;
}
```
除了Huge分配以外，分配chunk内存也是Large、Small内存分配的基础，<u>它是ZendMM向系统申请内存的唯一粒度</u>。

注：*分配chunk时，会将内存地址对齐到chunk的大小2MB（`ZEND_MM_THUNK_SIZE`）。也就是说，分配的chunk地址都是`ZEND_MM_THUNK_SIZE`的整数倍。（**实际上这个对齐并不是由系统简单的完成，而是需要内存池在申请内存后自己进行调整！**）*



`ZendMM`具体处理对齐的方法是：

- ①、按实际要申请的内存大小申请一次
    - 若系统分配的地址恰好是ZEND_MM_CHUNK_SIZE的整数倍，则不需要调整，直接返回。
    - 若系统分配的地址不是ZEND_MM_CHUNK_SIZE的整数倍，则需要使用第②步进行调整。
- ②、调整方式：
    - 首先ZendMM会将这块内存释放掉；
    - 按照“`实际要申请的内存大小 + ZEND_MM_CHUNK_SIZE`”的大小重新申请一块内存，多申请的ZEND_MM_CHUNK大小的内存是用来调整的`，ZendMM`会从系统分配的地址向后偏移到最近一个`ZEND_MM_CHUNK_SIZE的整数倍`位置，调整完之后再把剩余的内存释放掉。

```c
static void *zend_mm_chunk_alloc_int(size_t size, size_t alignment)
{
    // 向系统申请size大小的内存
	void *ptr = zend_mm_mmap(size);

	if (ptr == NULL) {
		return NULL;
	} else if (ZEND_MM_ALIGNED_OFFSET(ptr, alignment) == 0) { // 判断申请的内存是都为alignment的整数倍，是的话直接返回
#ifdef MADV_HUGEPAGE
	    madvise(ptr, size, MADV_HUGEPAGE);
#endif
		return ptr;
	} else {
	    // 申请的内存不是按照alignment对齐的
		size_t offset;

		// 将申请的内存释放掉重新申请
		zend_mm_munmap(ptr, size);
		// 重新申请一块内存，这里会多申请一块内存，用于截取到alignment的整数倍，可以忽略REAL_PAGE_SIZE
		ptr = zend_mm_mmap(size + alignment - REAL_PAGE_SIZE);
#ifdef _WIN32
		offset = ZEND_MM_ALIGNED_OFFSET(ptr, alignment);
		zend_mm_munmap(ptr, size + alignment - REAL_PAGE_SIZE);
		ptr = zend_mm_mmap_fixed((void*)((char*)ptr + (alignment - offset)), size);
		offset = ZEND_MM_ALIGNED_OFFSET(ptr, alignment);
		if (offset != 0) {
			zend_mm_munmap(ptr, size);
			return NULL;
		}
		return ptr;
#else
        // offset为ptr距离上一个alignment对齐内存位置的大小，注意不能往前移，因为前面的内存都是分配了的。
		offset = ZEND_MM_ALIGNED_OFFSET(ptr, alignment);
		if (offset != 0) {
			offset = alignment - offset;
			zend_mm_munmap(ptr, offset);
			// 偏移ptr，对齐到alignment
			ptr = (char*)ptr + offset;
			alignment -= offset;
		}
		if (alignment > REAL_PAGE_SIZE) {
			zend_mm_munmap((char*)ptr + size, alignment - REAL_PAGE_SIZE);
		}
# ifdef MADV_HUGEPAGE
	    madvise(ptr, size, MADV_HUGEPAGE);
# endif
#endif
		return ptr;
	}
}
```

其中用到了`ZEND_MM_ALIGNED_OFFSET宏`，这个宏的作用是计算按alignment对齐的内存地址距离上一个alignment整数倍内存地址的大小，也就是offset偏移量。

**alignment必须是2的n次方**，比如一段`n*alignment`大小的内存，ptr为其中一个位置，那么就可以通过位运算计算得到ptr在所属alignment内存块中的offset，如下图所示：

```
#define ZEND_MM_ALIGNED_OFFSET(size, alignment) \
	(((size_t)(size)) & ((alignment) - 1))
```
![相对对齐值的内存offset](https://note.youdao.com/yws/api/personal/file/9940B8821F1E4B19B2783B68381DFD5C?method=download&shareKey=8e13fe6980c3aa04d6cf3c97f1522b9d)

这个位运算是因为`alignment`为`2^n`（用二进制表示即为第n个位上为1，其余位为0，当alignment-1相当于将除了第n位为0，其余低位全部为1），所以通过alignment取到最低的位置，也就是相对上一个整数倍的alignment的offset，非位运算算式如下，但效率没有位运算高。
```
offset = ptr - (ptr/alignment取整 * alignment)
```

##### 3.2 Large分配
当申请的内存大于3072B、小于2044KB时，内存池会选择在chunk上查找对应数量的page返回。<u>Large内存申请的粒度是page，也就是分配n页连续的page</u>，所以Large分配的过程就转化为在chunk上查找n页连续可用的page的过程。

```c
static zend_always_inline void *zend_mm_alloc_large(zend_mm_heap *heap, size_t size ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC)
{
    // 根据size大小计算需要分配多少个page
	int pages_count = (int)ZEND_MM_SIZE_TO_NUM(size, ZEND_MM_PAGE_SIZE);
#if ZEND_DEBUG
    // 分配page_count个page
	void *ptr = zend_mm_alloc_pages(heap, pages_count, size ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC);
#else
	void *ptr = zend_mm_alloc_pages(heap, pages_count ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC);
#endif
#if ZEND_MM_STAT
	do {
		size_t size = heap->size + pages_count * ZEND_MM_PAGE_SIZE;
		size_t peak = MAX(heap->peak, size);
		heap->size = size;
		heap->peak = peak;
	} while (0);
#endif
	return ptr;
}
```

chunk结构中有两个成员用于记录page分配信息：
- **free_map**：类型为`zend_mm_page_map`，实际就是`zend_ulong_free_map[16/8]`，这是一个bitmap，总大小为64byte，也就是512bit，用于记录用当前chunk上512个page是否分配，512个page对应512bit，1表示已分配，0表示未分配。
![free_map](https://note.youdao.com/yws/api/personal/file/C879457EC2154FAEA678D562C217773B?method=download&shareKey=29d90ba60d479fdeccd33ac88325813a)
- **map**：这个是一个可容纳512个类型为uint32_t元素的数组，<u>该数组用于记录各page的分配类型及分配的page页数</u>，每个page对应一个数组成员。Large内存、Small内存都会占用page，正是通过这个数组标识改page属于哪个类型（最高两位用于标识page的分配类型）：
    
    - Large：01（0x40000000）
    
- Small：10（0x80000000）
    
      
    
      示例：申请12KB的内存，即3个page，内存池分配了page1，2，3，则map[1] = 0x400000000|3,如下图所示：
      ![page的分配类型及页数](https://note.youdao.com/yws/api/personal/file/474A05B6CB2D401785FA0FAB75FB3E97?method=download&shareKey=a1ce9cdd22022755fffe79243bd1db0e)

page分配时从第一个chunk开始遍历，依次查找各chunk是否有满足要求的page，如果当前chunk没有合适的，则进入下一chunk，如果直到最后都没有找到，则新分配一个chunk。
<u>分配准则为：申请的page页数要尽可能地填满chunk的空隙</u>，也就是说尽可能的与分配了的page连在一起，避免中间出现page空隙。减少后续分配时的茶之后按次数，提高内存利用率。

最优page的检索过程如下：
- step1：  
  首先从第一个page分组（page0~63）开始检查，如果当前分组无空闲page（即free_map[x]=-1）则进入下一分组，知道当前分组有空闲page，然后进入step2.
- step2：  
  当前分组有可用page，首先检查当前page分组的bit位，找到第一个空闲page的位置，记做page_num，接着继续向下查找空闲page，知道遇到第一个已经分配的page位置，将最后一个空闲page位置记做`end_page_num`。（*注：查找end_page_num时并不局限在当前page分组内，会向下查找，直到最后一页。其查找做成主要依据free_map*），page_num至end_page_num为找到的可用page，接着判断找到的page页数是否够用：
  
    - 不够的情况：将page_num至end_page_num这些page的bit位标为1，也就是已分配，然后回到step1继续检索其他page分组。
    - 刚好是要申请的页数：直接使用，中断检索。
    - page页数比申请的页数大，则表示可用，但不一定是最优的，将page_num暂存起来，接着回到step1继续向后找别的空闲page，最后比较选择best_len最小的，即能够最大程度填满page间隔
	  ```c
				/* find first 0 bit */
				 // tmp为当前page分组的bit位，i为当前分组第1个page的页码。
				page_num = i + zend_mm_bitset_nts(tmp);
				/* reset bits from 0 to "bit" */
				tmp &= tmp + 1;
				/* skip free blocks */
				// 快速跳过剩余page全部可用的分组
				while (tmp == 0) {
					i += ZEND_MM_BITSET_LEN;
					if (i >= free_tail || i == ZEND_MM_PAGES) {
						len = ZEND_MM_PAGES - page_num;
						if (len >= pages_count && len < best_len) {
							chunk->free_tail = page_num + pages_count;
							goto found;
						} else {
							/* set accurate value */
							chunk->free_tail = page_num;
							if (best > 0) {
								page_num = best;
								goto found;
							} else {
								goto not_found;
							}
						}
					}
					// 当前分组剩下的page都是可用的，直接跳到下一分组
					tmp = *(bitset++);
				}
				/* find first 1 bit */
				// 找到第一个已分配page
				len = i + zend_mm_bitset_ntz(tmp) - page_num;
				if (len >= pages_count) {
					if (len == pages_count) {
						goto found;
					} else if (len < best_len) {
						best_len = len;
						best = page_num;
					}
				}
				/* set bits from 0 to "bit" */
				// 把找到的这些page标为已分配，注：此时tmp已经经过tmp &= tmp + 1处理
  			tmp |= tmp - 1;
  ```
示例：当前某个chunk的page分配情况如下图中的A所示，page：0，1，2，6，9，10已经分配占用，接下来要申请2页的page。
  ![page的查找过程](https://note.youdao.com/yws/api/personal/file/335A8945237E483EAD280F5A26DED219?method=download&shareKey=29b8b43bfeb58653a22ace5a4562f538)
- step3：
   最后没找到合适的page页后设置对应page的分配信息，即free_map、map,然后返回找到第一页page的地址
    ```c
    found:
	if (steps > 2 && pages_count < 8) {
		/* move chunk into the head of the linked-list */
		chunk->prev->next = chunk->next;
		chunk->next->prev = chunk->prev;
		chunk->next = heap->main_chunk->next;
		chunk->prev = heap->main_chunk;
		chunk->prev->next = chunk;
		chunk->next->prev = chunk;
	}
	/* mark run as allocated */
	chunk->free_pages -= pages_count;
	zend_mm_bitset_set_range(chunk->free_map, page_num, pages_count);
	chunk->map[page_num] = ZEND_MM_LRUN(pages_count);
	if (page_num == chunk->free_tail) {
		chunk->free_tail = page_num + pages_count;
	}
	return ZEND_MM_PAGE_ADDR(chunk, page_num);
    ```
   
##### 3.3 Small分配
Small内存在分配时，首先检查申请规格的内存是否已经分配，如果没有分配或者分配的已经用完了，则申请相应页数的page，page的分配过成与Larg分配完全一致，申请到page以后按固定大小将page切割为slot，slot之间构成单链表，链表头部保存至`AG(mm_heap)->free_slot`；如果对应的slot已经分配，则直接返回`AG(mm_heap)->free_slot`。  

示例：16byte、3072byte大小的slot，将分别申请1个、3个page、然后切割为256个16byte的slot，以及4个3072byte的slot，如下图所示：
![slot[1]与slot[29]链表](https://note.youdao.com/yws/api/personal/file/4345829756E149EEAB1C4DE3928C76AC?method=download&shareKey=c358be1e7b73c1ff04ccac1eb772c992)

#### 4 系统内存分配
<u>内存池向系统申请内存的最小粒度是chunk</u>，通过mmap()来申请。

#### 5 内存释放
内存释放主要通过`efree()`来完成，内存池会根据释放的内存地址自动判断属于哪种粒度的内存，从而执行不同的释放逻辑。



**问：内存池是如何只根据一个地址就判断出改地址属于哪种内存类型的呢？**  
因为chunk分配时是按照ZEND_MM_CHUNK_SIZE（即2MB）对齐的，也就是chunk的起始内存地址一定是ZEND_MM_CHUNK_SIZE的整数倍，所以可以根据chunk上的任意位置知道chunk的起始位置与所在page。

##### 5.1 Huge内存的释放
首先，根据释放地址ptr计算该机制相对chunk起始位置的内存偏移量，这个值通过宏ZEND_MM_ALIGNED_OFFSET()的到，通过位运算计算的到。  

示例：ptr = 0x7ffff7c01000，计算的到offset = 4096

```
offet = ptr & (alignment - 1) = 0x7ffff7c01000 & 0x1fffff = 0x1000 = 4096
```

<u>Huge内存能够完全使用chunk，也就是Huge内存地址相对chunk的offset一定等于0，而Large、Small内存因为chunk的第1个page被占用了，所以这两种内存的offset不可能为0.</u>

内存池根据offset值判断出释放的内存是否为`Huge类型`，如果是则将占用的chunk释放，同时从AG(mm_heap)->huge_list链表中删除。

##### 5.2 Large内存的释放
若计算得到的offset不等于0，则表示该地址是Large内存或者Small内存，然后根据offset值进一步计算出属于第几个page   
<u>计算方法：根据offset除page的大小取整，的到`page_num`，的到`page`页码后就可以从`chunk->map`中获取该`page`的分配类型，知道是何种粒度的内存了。</u>

Large内存，<u>并不会直接释放物理内存</u>，只是将对应的page的分配信息重新设置为未分配。若释放page后，<u>当前chunk下所有的page都是未分配的，则会释放chunk</u>，释放时优先选择把chunk移到`AG(mm_heap)->cached_chunks`缓存队列中，缓存数达到一定值后就不在继续缓存新加入的chunk，将内存归还系统，便面占用过多的资源。
（分配chunk时，如果发现cached_chunks中有缓存的chunk,就直接取出使用，不再向系统申请。）

##### 5.3 Small内存的释放
若待释放的地址为Small内存，则会将释放的slot插入到该规格slot可用链表的头部，如下图所示：
![释放slot](https://note.youdao.com/yws/api/personal/file/9141601D64BF4D76AF0A211F55A479A2?method=download&shareKey=7df69cd6f617df511be8d5f87c7a58f4)