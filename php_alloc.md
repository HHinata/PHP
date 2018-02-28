#### 整体架构



![](https://images0.cnblogs.com/blog/481281/201402/081618571154399.jpg)

#### 接口层

```c
/* Standard wrapper macros */
#define emalloc(size)							    _emalloc((size)...)
#define emalloc_large(size) 						_emalloc_large((size)...)
#define emalloc_huge(size)						    _emalloc_huge((size)...)
#define safe_emalloc(nmemb, size, offset)	        _safe_emalloc((nmemb)...)
#define efree(ptr)								    _efree((ptr)...)
#define efree_large(ptr)						    _efree_large((ptr)...)
#define efree_huge(ptr)							    _efree_huge((ptr)...)
#define ecalloc(nmemb, size)				  	    _ecalloc((nmemb), (size)...)
#define erealloc(ptr, size)						    _erealloc((ptr), (size)...)
#define erealloc2(ptr, size, copy_size)	            _erealloc2((ptr)...)
#define safe_erealloc(ptr, nmemb, size, offset)	    _safe_erealloc((ptr),(nmemb)...)
#define erealloc_recoverable(ptr, size)		        _erealloc((ptr), (size)...)
#define erealloc2_recoverable(ptr, size, copy_size) _erealloc2((ptr), (size),...
#define estrdup(s)									_estrdup((s)...)
#define estrndup(s, length)							_estrndup((s), (length)...)
#define zend_mem_block_size(ptr)					_zend_mem_block_size((ptr)...)
```





#### 数据结构

##### *AG*

```c
typedef struct _zend_alloc_globals {
	zend_mm_heap *mm_heap;
} zend_alloc_globals;
```

##### *zend_mm_heap*

```c
struct _zend_mm_heap {
#if ZEND_MM_CUSTOM
	int                use_custom_heap;
#endif
#if ZEND_MM_STORAGE
	zend_mm_storage   *storage;
#endif
#if ZEND_MM_STAT
	size_t             size;    //当前已经使用内存数(php从内存池中申请的大小,即实际使用的数量)
	size_t             peak;    //内存单次申请的峰值
#endif
	zend_mm_free_slot *free_slot[ZEND_MM_BINS]; //小内存空闲内存链表
#if ZEND_MM_STAT || ZEND_MM_LIMIT          //实际内存申请大小(内存池实际从OS申请的大小)
	size_t             real_size;              /* current size of allocated pages */
#endif
#if ZEND_MM_STAT							//实际内存申请峰值
	size_t             real_peak;               /* peak size of allocated pages */
#endif
#if ZEND_MM_LIMIT
	size_t             limit;                   /* memory limit */
	int                overflow;                /* memory overflow flag */
#endif

	zend_mm_huge_list *huge_list;       //大内存链表

	zend_mm_chunk     *main_chunk;      //指向chunk链表头部
	zend_mm_chunk     *cached_chunks;			/* list of unused chunks */
	int                chunks_count;			//chunk的数量
	int                peak_chunks_count;		//当前请求中chunk块申请数量峰值
	int                cached_chunks_count;		/* number of cached chunks */
	double             avg_chunks_count;		//每次请求平均chunk块申请数量
#if ZEND_MM_CUSTOM  //自定义分配方法
	union {
		struct {
			void      *(*_malloc)(size_t);
			void       (*_free)(void*);
			void      *(*_realloc)(void*, size_t);
		} std;
		struct {
			void      *(*_malloc)(size_t ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC);
			void       (*_free)(void*  ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC);
			void      *(*_realloc)(void*, size_t  ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC);
		} debug;
	} custom_heap;
#endif
};
```

##### *zend_mm_huge_list*

```c
struct _zend_mm_huge_list {
	void              *ptr; 
	size_t             size;
	zend_mm_huge_list *next;
	...
};
```

##### *zend_mm_chunk*

```c
struct _zend_mm_chunk {
	zend_mm_heap      *heap;  //指向heap
	zend_mm_chunk     *next;  //指向下一个chunk
	zend_mm_chunk     *prev;  //指向上一个chunk
	int                free_pages;	//空闲page数
	int                free_tail;   //记录空闲后缀的起始,即从当前位置往后的page都是可用的
	int                num;
	char               reserve[64 - (sizeof(void*) * 3 + sizeof(int) * 3)];
	zend_mm_heap       heap_slot;          //heap结构,只有主chunk使用了
	zend_mm_page_map   free_map;          //包含16/8个uint_32类型的数组,共512bit,记录page的状态
	zend_mm_page_info  map[ZEND_MM_PAGES];      /* 2 KB = 512 * 4 */ //page信息
};
```

##### *zend_mm_free_slot*

```c
struct _zend_mm_free_slot {
	zend_mm_free_slot *next_free_slot;
};
```

##### *zend_mm_page_map*

```c
typedef zend_mm_bitset zend_mm_page_map[ZEND_MM_PAGE_MAP_LEN]; //数组
```

##### *zend_mm_bitset*

```c
typedef zend_ulong zend_mm_bitset;
typedef uint32_t zend_ulong;  //或者uint64_t
```

##### *zend_mm_page_info*

```c
typedef uint32_t   zend_mm_page_info; /* 4-byte integer */
```



#### 分配策略

- ***Huge(chunk)*:** 申请内存大于*2M*，直接调用系统分配，分配若干个*chunk*
- ***Large(page)*:** 申请内存大于*3092B(3/4 page_size)*，小于*2044KB(511 page_size)*，分配若干个*page*
- ***Small(slot)*:** 申请内存小于等于*3092B(3/4 page_size)*，内存池提前定义好了*30*种同等大小的内存*(8,16,24,32，...3072)*，他们分配在不同的*page*上(不同大小的内存可能会分配在多个连续的*page*)，申请内存时直接在对应*page*上查找可用位置


#### 执行流程

##### 内存池初始化



*  *php_module_startup* 阶段 *zend_startup* 方法中调用*start_memory_manager* 初始化内存池
*  *start_memory_manager* 调用*alloc_globals_ctor* 方法初始化*AG*结构体
*  *alloc_globals_ctor*调用*zend_mm_init*方法申请*chunk*结构体并初始化得到其中的*heap*



##### 内存池销毁



* *zend_mm_shutdown*方法销毁内存池
* 遍历*huge*链表调用*zend_mm_chunk_free*方法释放所有*huge*内存块
* 遍历*chunk*链表把除了主*chunk*外的所有*chunk*内存块加入*cache*链表
* 若需要释放内存池本体结构*heap*则调用*zend_mm_chunk_free*方法释放所有*cache*的*chunk*和主*chunk*
* 若不需要释放本体则检查*cache*数量,释放多出的*chunk*内存块,后清理所有*cache*内容,重置*heap*结构体



##### 内存池*GC*



* *zend_mm_gc*方法实现内存池清理回收闲置内存块
* 遍历*30*种*small*内存空闲链表,对于每种*small*内存块,记录所在*page*块未使用的个数
* 若某次分配用于*small*块使用的*page*块全部未使用时,把这些*page*块中的*small*块全部从空闲链表移除
* 遍历所有的*chunk*块,每个*chunk*遍历*page*块,回收在上步标记的*page*块,更新*chunk*块的*page*块分配记录
* 若某个*chunk*块所有的*page*都在未使用状态,调用*zend_mm_delete_chunk*释放内存,从*chunk*链表移除



##### 分配内存*_emalloc*

![](https://github.com/pangudashu/php7-internal/raw/master/img/alloc_all.png)

* *_emalloc* 调用 *zend_mm_alloc_heap* 申请内存
* *zend_mm_alloc_heap* 中若申请的内存小于 *ZEND_MM_MAX_SMALL_SIZE* 则调用 *zend_mm_alloc_small*
* 若申请的内存小于 *ZEND_MM_MAX_LARGE_SIZE* 则调用 *zend_mm_alloc_large*申请*large*内存块
* 否则调用 *zend_mm_alloc_huge*申请大内存块

##### 释放内存*_efree*



* *_efree*调用*zend_mm_free_heap*释放内存
* *zend_mm_free_heap*中若地址相对*chunk*块首地址偏移量为*0*则调用*zend_mm_free_huge*释放*huge*块
* 若不为*0*则找到所在*page*块,获取*map*中记录的使用信息,若为*small*使用则调用*zend_mm_free_small*释放
* 否则,计算需要释放的*page*块数量,后调用*zend_mm_free_large*释放*page*块



##### *chunk*内存块分配和释放



* *zend_mm_chunk_alloc*方法调用*zend_mm_chunk_alloc_int*申请*chunk*结构体所需要的内存
* *zend_mm_chunk_alloc_int*方法调用系统函数申请内存并保持地址对齐
* *zend_mm_chunk_init*方法完成对*chunk*结构体的初始化,并把指针放入*chunk*链表中,更新*heap*记录
* *zend_mm_delete_chunk*方法释放*chunk*块,先把*chunk*块从链表里移除
* 再判断*cache*的*chunk*块是否到上限,若没到上限加入cache链表,暂不释放
* 若超过上限,(寻找*cache*链表最适合释放的释放掉,暂未做),调用*zend_mm_chunk_free*释放
* *zend_mm_chunk_free*方法直接调用系统函数释放*chunk*结构的内存



##### *huge*内存块分配和释放

##### *large*内存块分配和释放

##### *small*内存块分配和释放

#### 函数

##### *start_memory_manager* 

 初始化内存池

```php
ZEND_API void start_memory_manager(void)
{
#ifdef ZTS //...多线程操作
	alloc_globals_ctor(&alloc_globals);
#ifndef _WIN32//...
}
```

##### *alloc_globals_ctor*

初始化*AG*结构体

```c
static void alloc_globals_ctor(zend_alloc_globals *alloc_globals)
{
#if ZEND_MM_CUSTOM  
	...
#ifdef MAP_HUGETLB 
	...
	alloc_globals->mm_heap = zend_mm_init(); //调用初始化方法,申请一个chunk,初始化其中的heap
}
```

##### *zend_mm_shutdown*

```c
void zend_mm_shutdown(zend_mm_heap *heap, int full, int silent)
{
	zend_mm_chunk *p;
	zend_mm_huge_list *list;

#if ZEND_MM_CUSTOM
	...
#endif
#if ZEND_DEBUG
    ...
#endif
	/* free huge blocks */
	list = heap->huge_list;
	heap->huge_list = NULL; 
	while (list) {  //释放huge内存块
		zend_mm_huge_list *q = list;
		list = list->next;
		zend_mm_chunk_free(heap, q->ptr, q->size);
	}
	/* move all chunks except of the first one into the cache */
	p = heap->main_chunk->next;
	while (p != heap->main_chunk) { //除了第一块其他的放入cache链表中
		zend_mm_chunk *q = p->next;
		p->next = heap->cached_chunks;
		heap->cached_chunks = p;
		p = q;
		heap->chunks_count--;
		heap->cached_chunks_count++;
	}
	if (full) {
		/* free all cached chunks */
		while (heap->cached_chunks) {  //释放所有cache的chunk块
			p = heap->cached_chunks;
			heap->cached_chunks = p->next;
			zend_mm_chunk_free(heap, p, ZEND_MM_CHUNK_SIZE);
		}
		/* free the first chunk */ //释放第一块chunk块
		zend_mm_chunk_free(heap, heap->main_chunk, ZEND_MM_CHUNK_SIZE);
	} else {
		zend_mm_heap old_heap;

		/* free some cached chunks to keep average count */
		heap->avg_chunks_count = (heap->avg_chunks_count + (double)heap->peak_chunks_count) / 2.0;
		while ((double)heap->cached_chunks_count + 0.9 > heap->avg_chunks_count &&
		       heap->cached_chunks) { //释放cache的chunk块到一定数量
			p = heap->cached_chunks;
			heap->cached_chunks = p->next;
			zend_mm_chunk_free(heap, p, ZEND_MM_CHUNK_SIZE);
			heap->cached_chunks_count--;
		}
		/* clear cached chunks */
		p = heap->cached_chunks;
		while (p != NULL) {  //清空cache的chunk块内存的内容
			zend_mm_chunk *q = p->next;
			memset(p, 0, sizeof(zend_mm_chunk));
			p->next = q;
			p = q;
		}
		/* reinitialize the first chunk and heap */
      	//重置heap结构
		old_heap = *heap;
		p = heap->main_chunk;
		memset(p, 0, ZEND_MM_FIRST_PAGE * ZEND_MM_PAGE_SIZE);
		*heap = old_heap;
		memset(heap->free_slot, 0, sizeof(heap->free_slot));
		heap->main_chunk = p;
		p->heap = &p->heap_slot;
		p->next = p;
		p->prev = p;
		p->free_pages = ZEND_MM_PAGES - ZEND_MM_FIRST_PAGE;
		p->free_tail = ZEND_MM_FIRST_PAGE;
		p->free_map[0] = (1L << ZEND_MM_FIRST_PAGE) - 1;
		p->map[0] = ZEND_MM_LRUN(ZEND_MM_FIRST_PAGE);
		heap->chunks_count = 1;
		heap->peak_chunks_count = 1;
#if ZEND_MM_STAT || ZEND_MM_LIMIT
		heap->real_size = ZEND_MM_CHUNK_SIZE;
#endif
#if ZEND_MM_STAT
		heap->real_peak = ZEND_MM_CHUNK_SIZE;
		heap->size = heap->peak = 0;
#endif
	}
}
```



##### *zend_mm_init*

 申请*chunk*结构体并初始化,返回其中的*heap*

```c
static zend_mm_heap *zend_mm_init(void)
{
  	//申请大小为2M的内存块,内存对齐为2M,chunk内存块
	zend_mm_chunk *chunk = (zend_mm_chunk*)zend_mm_chunk_alloc_int(ZEND_MM_CHUNK_SIZE, ZEND_MM_CHUNK_SIZE);
	zend_mm_heap *heap;
	if (UNEXPECTED(chunk == NULL)) {
		...
		return NULL;
	}
  	//初始化chunk结构体
	heap = &chunk->heap_slot;
	chunk->heap = heap;
	chunk->next = chunk;
	chunk->prev = chunk;
	chunk->free_pages = ZEND_MM_PAGES - ZEND_MM_FIRST_PAGE;
	chunk->free_tail = ZEND_MM_FIRST_PAGE;
	chunk->num = 0;
	chunk->free_map[0] = (Z_L(1) << ZEND_MM_FIRST_PAGE) - 1;
	chunk->map[0] = ZEND_MM_LRUN(ZEND_MM_FIRST_PAGE);
	heap->main_chunk = chunk;
	heap->cached_chunks = NULL;
	heap->chunks_count = 1;
	heap->peak_chunks_count = 1;
	heap->cached_chunks_count = 0;
	heap->avg_chunks_count = 1.0;
#if ZEND_MM_STAT || ZEND_MM_LIMIT
	heap->real_size = ZEND_MM_CHUNK_SIZE;
#endif
#if ZEND_MM_STAT
	heap->real_peak = ZEND_MM_CHUNK_SIZE;
	heap->size = 0;
	heap->peak = 0;
#endif
#if ZEND_MM_LIMIT
	heap->limit = (Z_L(-1) >> Z_L(1));
	heap->overflow = 0;
#endif
#if ZEND_MM_CUSTOM
	heap->use_custom_heap = ZEND_MM_CUSTOM_HEAP_NONE;
#endif
#if ZEND_MM_STORAGE
	heap->storage = NULL;
#endif
	heap->huge_list = NULL;
	return heap;
}
```



##### *zend_mm_mmap*

调用*mmap*系统函数申请一块大小为*size*的内存

```c
static void *zend_mm_mmap(size_t size)
{
#ifdef _WIN32
	...
#else
	void *ptr;
#ifdef MAP_HUGETLB
	...
#endif
	ptr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_PRIVATE | MAP_ANON, -1, 0);
	if (ptr == MAP_FAILED) {
      	...
		return NULL;
	}
	return ptr;
#endif
}
```

##### *zend_mm_munmap*

调用系统方法*munmap*释放首地址为*addr*大小为*size*的内存块

```c
static void zend_mm_munmap(void *addr, size_t size)
{
#ifdef _WIN32
	...
#else
	if (munmap(addr, size) != 0) {
#if ZEND_MM_ERROR
		fprintf(stderr, "\nmunmap() failed: [%d] %s\n", errno, strerror(errno));
#endif
	}
#endif
}
```



##### *zend_mm_gc*



```c
ZEND_API size_t zend_mm_gc(zend_mm_heap *heap)
{
	zend_mm_free_slot *p, **q;
	zend_mm_chunk *chunk;
	size_t page_offset;
	int page_num;
	zend_mm_page_info info;
	int i, has_free_pages, free_counter;
	size_t collected = 0;
#if ZEND_MM_CUSTOM
	...
#endif
	for (i = 0; i < ZEND_MM_BINS; i++) { //遍历30种small内存空闲链表
		has_free_pages = 0;
		p = heap->free_slot[i];
		while (p != NULL) { //遍历链表
			chunk = (zend_mm_chunk*)ZEND_MM_ALIGNED_BASE(p, ZEND_MM_CHUNK_SIZE);//获取chunk
			ZEND_MM_CHECK(chunk->heap == heap, "zend_mm_heap corrupted");//检查是否正确
			page_offset = ZEND_MM_ALIGNED_OFFSET(p, ZEND_MM_CHUNK_SIZE);//得到在chunk的位置
			ZEND_ASSERT(page_offset != 0);
			page_num = (int)(page_offset / ZEND_MM_PAGE_SIZE); //得到是第几个
			info = chunk->map[page_num]; //获取使用信息
			ZEND_ASSERT(info & ZEND_MM_IS_SRUN);
			if (info & ZEND_MM_IS_LRUN) { //如果不是分配时的第一块用作small内存
				page_num -= ZEND_MM_NRUN_OFFSET(info); //找到第一块
				info = chunk->map[page_num]; //获取第一块的使用信息
				ZEND_ASSERT(info & ZEND_MM_IS_SRUN); //检查
				ZEND_ASSERT(!(info & ZEND_MM_IS_LRUN));
			}
			ZEND_ASSERT(ZEND_MM_SRUN_BIN_NUM(info) == i);
			free_counter = ZEND_MM_SRUN_FREE_COUNTER(info) + 1; //未使用small块的个数加1
			if (free_counter == bin_elements[i]) { //如果一次分配的全部未使用
				has_free_pages = 1; //标记
			}
			chunk->map[page_num] = ZEND_MM_SRUN_EX(i, free_counter);//更新未使用块数
			p = p->next_free_slot; //下一块
		}

		if (!has_free_pages) { //如果一次分配的全部未使用往下走
			continue;
		}

		q = &heap->free_slot[i];
		p = *q;
		while (p != NULL) {
			chunk = (zend_mm_chunk*)ZEND_MM_ALIGNED_BASE(p, ZEND_MM_CHUNK_SIZE);//获取chunk
			ZEND_MM_CHECK(chunk->heap == heap, "zend_mm_heap corrupted");//检查
			page_offset = ZEND_MM_ALIGNED_OFFSET(p, ZEND_MM_CHUNK_SIZE);//找到page的位置
			ZEND_ASSERT(page_offset != 0);//检查不是第一块
			page_num = (int)(page_offset / ZEND_MM_PAGE_SIZE); //找到page的编号
			info = chunk->map[page_num]; //获取使用信息
			ZEND_ASSERT(info & ZEND_MM_IS_SRUN); //检查用于small块
			if (info & ZEND_MM_IS_LRUN) { //如果不是第一块
				page_num -= ZEND_MM_NRUN_OFFSET(info); //找到第一块的编号
				info = chunk->map[page_num];  //获取使用信息
				ZEND_ASSERT(info & ZEND_MM_IS_SRUN);  //检查
				ZEND_ASSERT(!(info & ZEND_MM_IS_LRUN));
			}
			ZEND_ASSERT(ZEND_MM_SRUN_BIN_NUM(info) == i);
			if (ZEND_MM_SRUN_FREE_COUNTER(info) == bin_elements[i]) {//如果分配的全部未使用
				/* remove from cache */ //从空闲链表里把这些small块全部移除
				p = p->next_free_slot;;
				*q = p;
			} else {
				q = &p->next_free_slot;
				p = *q;
			}
		}
	}

	chunk = heap->main_chunk;
	do { //遍历chunk链表
		i = ZEND_MM_FIRST_PAGE; //从第一块page开始
		while (i < chunk->free_tail) { //遍历page块
			if (zend_mm_bitset_is_set(chunk->free_map, i)) { //如果已经使用
				info = chunk->map[i]; //获取使用信息
				if (info & ZEND_MM_IS_SRUN) { //是用作small块使用
					int bin_num = ZEND_MM_SRUN_BIN_NUM(info); //获取是哪种small块
					int pages_count = bin_pages[bin_num]; //当前small块需要的page数量
					if (ZEND_MM_SRUN_FREE_COUNTER(info) == bin_elements[bin_num]) {
						/* all elemens are free */ //如果page都是未使用的
						zend_mm_free_pages_ex(heap, chunk, i, pages_count, 0);//释放page块
						collected += pages_count;//记录释放个数
					} else {
						/* reset counter */ //恢复map记录的使用信息,即清空未使用个数
						chunk->map[i] = ZEND_MM_SRUN(bin_num);
					}
					i += bin_pages[bin_num]; //跳过释放的page块
				} else /* if (info & ZEND_MM_IS_LRUN) */ { //跳过用于large块分配的page块
					i += ZEND_MM_LRUN_PAGES(info);
				}
			} else {
				i++;
			}
		}
		if (chunk->free_pages == ZEND_MM_PAGES - ZEND_MM_FIRST_PAGE) { //如果整个chunk未使用
			zend_mm_chunk *next_chunk = chunk->next;
			zend_mm_delete_chunk(heap, chunk); //释放了这个chunk块
			chunk = next_chunk;
		} else {
			chunk = chunk->next;
		}
	} while (chunk != heap->main_chunk);

	return collected * ZEND_MM_PAGE_SIZE; //返回释放的内存大小
}
```





##### *_emalloc*

申请大小为*size*的内存

```c
ZEND_API void* ZEND_FASTCALL _emalloc(size_t size ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC)
{

#if ZEND_MM_CUSTOM
	...
	return zend_mm_alloc_heap(AG(mm_heap), size ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC);
}
```

##### *zend_mm_alloc_heap*

```c
static zend_always_inline void *zend_mm_alloc_heap(zend_mm_heap *heap, size_t size ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC)
{
	void *ptr;
#if ZEND_DEBUG
	...
#endif
	if (size <= ZEND_MM_MAX_SMALL_SIZE) { //申请小内存块
		ptr = zend_mm_alloc_small(heap, size, ZEND_MM_SMALL_SIZE_TO_BIN(size) ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC);
#if ZEND_DEBUG
	...
#endif
		return ptr;
	} else if (size <= ZEND_MM_MAX_LARGE_SIZE) { //申请large内存块
		ptr = zend_mm_alloc_large(heap, size ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC);
#if ZEND_DEBUG
	...
#endif
		return ptr;
	} else {
#if ZEND_DEBUG
		size = real_size;
#endif
      	//申请huge内存块
		return zend_mm_alloc_huge(heap, size ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC);
	}
}
```

##### *zend_mm_alloc_small*

申请小内存

```c
static zend_always_inline void *zend_mm_alloc_small(zend_mm_heap *heap, size_t size, int bin_num ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC)
{
#if ZEND_MM_STAT
	do {//记录当前内存使用数和申请峰值
		size_t size = heap->size + bin_data_size[bin_num]; //加上本次申请内存大小
		size_t peak = MAX(heap->peak, size);
		heap->size = size;
		heap->peak = peak;
	} while (0);
#endif

	if (EXPECTED(heap->free_slot[bin_num] != NULL)) { //如果当前大小的链表不是空
		zend_mm_free_slot *p = heap->free_slot[bin_num]; //取到对应大小的小内存链表第一个结点
		heap->free_slot[bin_num] = p->next_free_slot; //链表头部后移一个结点
		return (void*)p;
	} else {  //如果当前大小的链表为空
          return zend_mm_alloc_small_slow(heap, bin_num ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC);
	}
}
```

##### *zend_mm_alloc_small_slow*

```c
static zend_never_inline void *zend_mm_alloc_small_slow(zend_mm_heap *heap, int bin_num ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC)
{
    zend_mm_chunk *chunk;
    int page_num;
	zend_mm_bin *bin;
	zend_mm_free_slot *p, *end;
#if ZEND_DEBUG
	...
#else
	bin = (zend_mm_bin*)zend_mm_alloc_pages(heap, bin_pages[bin_num] ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC);
#endif
	if (UNEXPECTED(bin == NULL)) {//分配失败
		/* insufficient memory */
		return NULL;
	}
	chunk = (zend_mm_chunk*)ZEND_MM_ALIGNED_BASE(bin, ZEND_MM_CHUNK_SIZE);//所在chunk
	page_num = ZEND_MM_ALIGNED_OFFSET(bin, ZEND_MM_CHUNK_SIZE) / ZEND_MM_PAGE_SIZE;//编号
	chunk->map[page_num] = ZEND_MM_SRUN(bin_num); //标识当前块用于第bin_num种small内存使用
	if (bin_pages[bin_num] > 1) {
		int i = 1;
		do { //把申请的所有page块标识用于第bin_num种small内存使用
			chunk->map[page_num+i] = ZEND_MM_NRUN(bin_num, i);
			i++;
		} while (i < bin_pages[bin_num]);
	}
	/* create a linked list of elements from 1 to last *///把申请的结点串成链表
	end = (zend_mm_free_slot*)((char*)bin + (bin_data_size[bin_num] * (bin_elements[bin_num] - 1))); //最后一块small内存的地址
	heap->free_slot[bin_num] = p = (zend_mm_free_slot*)((char*)bin + bin_data_size[bin_num]); //第一块用于分配出去,从第二块开始放入链表中
	do {
		p->next_free_slot = (zend_mm_free_slot*)((char*)p + bin_data_size[bin_num]);;
#if ZEND_DEBUG
		...
#endif
		p = (zend_mm_free_slot*)((char*)p + bin_data_size[bin_num]);
	} while (p != end); //生成空闲链表
	/* terminate list using NULL */
	p->next_free_slot = NULL; //链表尾部
#if ZEND_DEBUG
		...
#endif
	/* return first element */
	return (char*)bin; //返回第一块用于本次分配
}
```



##### *zend_mm_alloc_large*

申请*large*内存块

```c
static zend_always_inline void *zend_mm_alloc_large(zend_mm_heap *heap, size_t size ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC)
{
	int pages_count = (int)ZEND_MM_SIZE_TO_NUM(size, ZEND_MM_PAGE_SIZE);
#if ZEND_DEBUG
	void *ptr = zend_mm_alloc_pages(heap, pages_count, size ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC);
#else
	void *ptr = zend_mm_alloc_pages(heap, pages_count ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC); //申请内存
#endif
#if ZEND_MM_STAT
	do { //更新记录值
		size_t size = heap->size + pages_count * ZEND_MM_PAGE_SIZE;
		size_t peak = MAX(heap->peak, size);
		heap->size = size;   
		heap->peak = peak;
	} while (0);
#endif
	return ptr;
}
```



##### *zend_mm_alloc_pages*

分配*pages_count*个*page*块

```c
static void *zend_mm_alloc_pages(zend_mm_heap *heap, int pages_count ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC)
#endif
{
	zend_mm_chunk *chunk = heap->main_chunk;
	int page_num, len;

	while (1) {
		if (UNEXPECTED(chunk->free_pages < pages_count)) {//空闲Page数不够分配
			goto not_found;
#if 0
	...
#endif
		} else {
			/* Best-Fit Search */
			int best = -1;   //记录可以分配的位置
			int best_len = ZEND_MM_PAGES;  //记录可以分配的位置page的个数
			int free_tail = chunk->free_tail; //此位置之后的Page都是未分配的
			zend_mm_bitset *bitset = chunk->free_map;
			zend_mm_bitset tmp = *(bitset++);
			int i = 0;

			while (1) {
				/* skip allocated blocks */
				while (tmp == (zend_mm_bitset)-1) {//跳过全部已经分配的块
					i += ZEND_MM_BITSET_LEN;
					if (i == ZEND_MM_PAGES) { //如果已经到了最后一块
						if (best > 0) {
							page_num = best;
							goto found;
						} else {
							goto not_found;
						}
					}
					tmp = *(bitset++); //下一块
				}
				//找到第一个0位,此处先找到后缀连续1位的个数,即注意bitset最后一位标识第一个page块
				page_num = i + zend_mm_bitset_nts(tmp);
				/* reset bits from 0 to "bit" */
				tmp &= tmp + 1; //把后缀的1位全部取反置为0位
				/* skip free blocks */
				while (tmp == 0) { //如果剩下的全部都未分配
					i += ZEND_MM_BITSET_LEN;
					if (i >= free_tail || i == ZEND_MM_PAGES) { //最后一块,或者后面都未分配
						len = ZEND_MM_PAGES - page_num;
						if (len >= pages_count && len < best_len) {
							chunk->free_tail = page_num + pages_count;//未分配后缀起始
							goto found;
						} else {
							/* set accurate value */
							chunk->free_tail = page_num;//未分配后缀起始
							if (best > 0) {
								page_num = best;
								goto found;
							} else {
								goto not_found;
							}
						}
					}
					tmp = *(bitset++);
				}
				//寻找第一个1位的位置,即找出未分配的长度
				len = i + zend_mm_bitset_ntz(tmp) - page_num;
				if (len >= pages_count) { //如果足够分配,则记录最优
					if (len == pages_count) {
						goto found;
					} else if (len < best_len) {
						best_len = len;
						best = page_num;
					}
				}
				/* set bits from 0 to "bit" */
				tmp |= tmp - 1;//把后缀的0位全部置为1位
			}
		}

not_found:
		if (chunk->next == heap->main_chunk) { //如果是最后个chunk块
get_chunk:
			if (heap->cached_chunks) { //cached的chunk块中取出一块
				heap->cached_chunks_count--;
				chunk = heap->cached_chunks;
				heap->cached_chunks = chunk->next;
			} else {
#if ZEND_MM_LIMIT
              	//如果有申请的内存大于内存申请限制了
				if (UNEXPECTED(heap->real_size + ZEND_MM_CHUNK_SIZE > heap->limit)) {
					if (zend_mm_gc(heap)) {//调用GC后再次尝试申请chunk块
						goto get_chunk;
					} else if (heap->overflow == 0) { 分配失败
#if ZEND_DEBUG
...
#endif
						return NULL;
					}
				}
#endif
				chunk = (zend_mm_chunk*)zend_mm_chunk_alloc(heap, ZEND_MM_CHUNK_SIZE, ZEND_MM_CHUNK_SIZE);
				if (UNEXPECTED(chunk == NULL)) { //分配失败
					/* insufficient memory */
					if (zend_mm_gc(heap) &&
					    (chunk = (zend_mm_chunk*)zend_mm_chunk_alloc(heap, ZEND_MM_CHUNK_SIZE, ZEND_MM_CHUNK_SIZE)) != NULL) {//尝试调用GC后再次申请一次
						/* pass */
					} else { //再次分配失败
#if !ZEND_MM_LIMIT
...
#endif
						return NULL;
					}
				}
#if ZEND_MM_STAT
				do {
					size_t size = heap->real_size + ZEND_MM_CHUNK_SIZE;
					size_t peak = MAX(heap->real_peak, size);
					heap->real_size = size; //记录内存分配数
					heap->real_peak = peak; //记录内存分配峰值
				} while (0);
#elif ZEND_MM_LIMIT
				heap->real_size += ZEND_MM_CHUNK_SIZE;

#endif
			}
			heap->chunks_count++; //记录chunk块数量
			if (heap->chunks_count > heap->peak_chunks_count) {//chunk申请峰值
				heap->peak_chunks_count = heap->chunks_count;
			}
			zend_mm_chunk_init(heap, chunk); //把申请的chunk块放入内存池中
			page_num = ZEND_MM_FIRST_PAGE;
			len = ZEND_MM_PAGES - ZEND_MM_FIRST_PAGE;
			goto found;
		} else {
			chunk = chunk->next;
		}
	}

found:
	/* mark run as allocated */
	chunk->free_pages -= pages_count; //更新未分配page个数
	zend_mm_bitset_set_range(chunk->free_map, page_num, pages_count); //标识已分配
	chunk->map[page_num] = ZEND_MM_LRUN(pages_count); //标识为large内存使用
	if (page_num == chunk->free_tail) { //更新未分配后缀头部
		chunk->free_tail = page_num + pages_count;
	}
	return ZEND_MM_PAGE_ADDR(chunk, page_num); //返回首地址
}
```

##### *zend_mm_chunk_alloc*

```c
static void *zend_mm_chunk_alloc(zend_mm_heap *heap, size_t size, size_t alignment)
{
#if ZEND_MM_STORAGE
	...
#endif
	return zend_mm_chunk_alloc_int(size, alignment);
}
```



##### *zend_mm_chunk_alloc_int*

申请大小为*size*,内存对齐*alignment*的内存块

```c
static void *zend_mm_chunk_alloc_int(size_t size, size_t alignment)
{
	void *ptr = zend_mm_mmap(size); //申请大小为size的内存

	if (ptr == NULL) {
		return NULL;
	} else if (ZEND_MM_ALIGNED_OFFSET(ptr, alignment) == 0) { //内存对齐的话直接返回
      	...
		return ptr; 
	} else {
		size_t offset;
		/* chunk has to be aligned */
		zend_mm_munmap(ptr, size); //释放申请的内存块
		ptr = zend_mm_mmap(size + alignment - REAL_PAGE_SIZE); //重新申请一块更大的内存
#ifdef _WIN32
		...
#else
        //对齐内存后返回
		offset = ZEND_MM_ALIGNED_OFFSET(ptr, alignment); //获取ptr对alignment取模的值
		if (offset != 0) {
			offset = alignment - offset; //得到首地址需要后移的大小
			zend_mm_munmap(ptr, offset); //释放这部分内存
			ptr = (char*)ptr + offset; //后移指针
			alignment -= offset;
		}
		if (alignment > REAL_PAGE_SIZE) { //释放多申请的内存
			zend_mm_munmap((char*)ptr + size, alignment - REAL_PAGE_SIZE);
		}
        ...
#endif
		return ptr;
	}
}
```



##### *zend_mm_chunk_init*

*chunk*结构初始化,插入内存池*chunk*链表中

```c
static zend_always_inline void zend_mm_chunk_init(zend_mm_heap *heap, zend_mm_chunk *chunk)
{
	chunk->heap = heap;   //指向内存池结构
    //插入chunk链表尾部
	chunk->next = heap->main_chunk;  
	chunk->prev = heap->main_chunk->prev; 
	chunk->prev->next = chunk;
	chunk->next->prev = chunk;
	chunk->free_pages = ZEND_MM_PAGES - ZEND_MM_FIRST_PAGE;
	chunk->free_tail = ZEND_MM_FIRST_PAGE;
	/* the younger chunks have bigger number */
	chunk->num = chunk->prev->num + 1;
	/* mark first pages as allocated */
	chunk->free_map[0] = (1L << ZEND_MM_FIRST_PAGE) - 1;
	chunk->map[0] = ZEND_MM_LRUN(ZEND_MM_FIRST_PAGE);
}
```



##### *zend_mm_alloc_huge*

申请huge内存块

```c
static void *zend_mm_alloc_huge(zend_mm_heap *heap, size_t size ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC)
{
#ifdef ZEND_WIN32
	...
#else
	size_t new_size = ZEND_MM_ALIGNED_SIZE_EX(size, REAL_PAGE_SIZE); //需要申请的大小
#endif
	void *ptr;
#if ZEND_MM_LIMIT
	if (UNEXPECTED(heap->real_size + new_size > heap->limit)) {//如果超过大小限制
		if (zend_mm_gc(heap) && heap->real_size + new_size <= heap->limit) { //调用gc后重试
			/* pass */
		} else if (heap->overflow == 0) {
          	...
			return NULL;
		}
	}
#endif
	ptr = zend_mm_chunk_alloc(heap, new_size, ZEND_MM_CHUNK_SIZE); //申请内存
	if (UNEXPECTED(ptr == NULL)) { //申请失败
		if (zend_mm_gc(heap) &&
		    (ptr = zend_mm_chunk_alloc(heap, new_size, ZEND_MM_CHUNK_SIZE)) != NULL) {
			//调用gc后重试,成功则继续
		} else { //失败返回
			...
			return NULL;
		}
	}
#if ZEND_DEBUG
	zend_mm_add_huge_block(heap, ptr, new_size, size ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC);
#else
	zend_mm_add_huge_block(heap, ptr, new_size ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC); //申请huge链表结点,把申请的内存放入huge链表
#endif
#if ZEND_MM_STAT
	do {
		size_t size = heap->real_size + new_size; 
		size_t peak = MAX(heap->real_peak, size);
		heap->real_size = size; 
		heap->real_peak = peak;
	} while (0);
	do {
		size_t size = heap->size + new_size; 
		size_t peak = MAX(heap->peak, size); 
		heap->size = size;//记录内存使用数
		heap->peak = peak;//更新内存使用峰值
	} while (0);
#elif ZEND_MM_LIMIT
	heap->real_size += new_size;
#endif
	return ptr;
}
```



##### *zend_mm_add_huge_block*

申请*huge*链表结点,保存下*ptr*指针

```c
static void zend_mm_add_huge_block(zend_mm_heap *heap, void *ptr, size_t size ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC)
#endif
{
  	//申请链表结点
	zend_mm_huge_list *list = (zend_mm_huge_list*)zend_mm_alloc_heap(heap, sizeof(zend_mm_huge_list) ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC);
	list->ptr = ptr;
	list->size = size;
	list->next = heap->huge_list; 
#if ZEND_DEBUG
	...
#endif
	heap->huge_list = list; //插入链表头部
}
```



##### *_efree*

```c
ZEND_API void ZEND_FASTCALL _efree(void *ptr ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC)
{

#if ZEND_MM_CUSTOM
	...
#endif
	zend_mm_free_heap(AG(mm_heap), ptr ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC);
}
```



##### *zend_mm_free_heap*

```c
static zend_always_inline void zend_mm_free_heap(zend_mm_heap *heap, void *ptr ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC)
{
 	//page_offset就是ptr距离当前chunk起始位置的偏移量
	size_t page_offset = ZEND_MM_ALIGNED_OFFSET(ptr, ZEND_MM_CHUNK_SIZE);
	if (UNEXPECTED(page_offset == 0)) { //如果偏移量为0,那肯定是huge内存块,因为large,small的内存第一个page一定被chunk结构体占用,所以偏移量不会是0
		if (ptr != NULL) {
			zend_mm_free_huge(heap, ptr ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC); //释放huge内存
		}
	} else {
      	//根据ptr获取chunk的起始位置
		zend_mm_chunk *chunk = (zend_mm_chunk*)ZEND_MM_ALIGNED_BASE(ptr, ZEND_MM_CHUNK_SIZE);
		int page_num = (int)(page_offset / ZEND_MM_PAGE_SIZE); //page编号
		zend_mm_page_info info = chunk->map[page_num]; //page块信息
		ZEND_MM_CHECK(chunk->heap == heap, "zend_mm_heap corrupted"); //检查chunk块
		if (EXPECTED(info & ZEND_MM_IS_SRUN)) { //如果是small内存块
			zend_mm_free_small(heap, ptr, ZEND_MM_SRUN_BIN_NUM(info));
		} else /* if (info & ZEND_MM_IS_LRUN) */ { //如果是large内存块
			int pages_count = ZEND_MM_LRUN_PAGES(info);//计算需要释放的块数
			ZEND_MM_CHECK(ZEND_MM_ALIGNED_OFFSET(page_offset, ZEND_MM_PAGE_SIZE) == 0, "zend_mm_heap corrupted"); //再次检查地址是否正确
			zend_mm_free_large(heap, chunk, page_num, pages_count); //释放内存
		}
	}
}
```



##### *zend_mm_free_huge*

```c
static void zend_mm_free_huge(zend_mm_heap *heap, void *ptr ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC)
{
	size_t size;
	ZEND_MM_CHECK(ZEND_MM_ALIGNED_OFFSET(ptr, ZEND_MM_CHUNK_SIZE) == 0, "zend_mm_heap corrupted"); //再次检查
	size = zend_mm_del_huge_block(heap, ptr ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC); //释放对应huge链表的结点
	zend_mm_chunk_free(heap, ptr, size); //释放对应内存
#if ZEND_MM_STAT || ZEND_MM_LIMIT
	heap->real_size -= size;   //更新记录
#endif
#if ZEND_MM_STAT
	heap->size -= size;
#endif
}
```



##### *zend_mm_chunk_free*

释放*chunk*块内存

```C
static void zend_mm_chunk_free(zend_mm_heap *heap, void *addr, size_t size)
{
#if ZEND_MM_STORAGE
	...
#endif
	zend_mm_munmap(addr, size); //使用系统调用释放内存
}
```



##### *zend_mm_free_large*

释放*large*内存块

```c
static zend_always_inline void zend_mm_free_large(zend_mm_heap *heap, zend_mm_chunk *chunk, int page_num, int pages_count)
{
#if ZEND_MM_STAT
	heap->size -= pages_count * ZEND_MM_PAGE_SIZE; //更新记录值
#endif
	zend_mm_free_pages(heap, chunk, page_num, pages_count); //释放内存
}
```



##### *zend_mm_free_pages*

释放*pages*内存

```c
static void zend_mm_free_pages(zend_mm_heap *heap, zend_mm_chunk *chunk, int page_num, int pages_count)
{
	zend_mm_free_pages_ex(heap, chunk, page_num, pages_count, 1);
}
```



##### *zend_mm_free_pages_ex*

释放*pages*内存

```c
static zend_always_inline void zend_mm_free_pages_ex(zend_mm_heap *heap, zend_mm_chunk *chunk, int page_num, int pages_count, int free_chunk)
{
	chunk->free_pages += pages_count; //更新空闲pages数
	zend_mm_bitset_reset_range(chunk->free_map, page_num, pages_count); //更新标识位
	chunk->map[page_num] = 0; //更新使用信息
	if (chunk->free_tail == page_num + pages_count) { //更新未分配后缀头部
		/* this setting may be not accurate */
		chunk->free_tail = page_num;
	}
	if (free_chunk && chunk->free_pages == ZEND_MM_PAGES - ZEND_MM_FIRST_PAGE) {
		zend_mm_delete_chunk(heap, chunk);//在需要释放chunk且当前chunk块为空闲时释放chunk块
	}
}
```



##### *zend_mm_delete_chunk*

释放*chunk*内存块,更新*chunk*链表

```c
static zend_always_inline void zend_mm_delete_chunk(zend_mm_heap *heap, zend_mm_chunk *chunk)
{
	chunk->next->prev = chunk->prev; //从chunk链表移除
	chunk->prev->next = chunk->next;
	heap->chunks_count--;   //更新chunk数
	if (heap->chunks_count + heap->cached_chunks_count < heap->avg_chunks_count + 0.1) {
		/* delay deletion */ //延迟删除,加入cached链表
		heap->cached_chunks_count++;
		chunk->next = heap->cached_chunks;
		heap->cached_chunks = chunk;
	} else {
#if ZEND_MM_STAT || ZEND_MM_LIMIT
		heap->real_size -= ZEND_MM_CHUNK_SIZE; //更新记录
#endif
		if (!heap->cached_chunks || chunk->num > heap->cached_chunks->num) {
			zend_mm_chunk_free(heap, chunk, ZEND_MM_CHUNK_SIZE); //直接释放内存
		} else {
//TODO: select the best chunk to delete??? 还没做？？？找到最优删除的chunk块
			chunk->next = heap->cached_chunks->next; //加入cached链表
			zend_mm_chunk_free(heap, heap->cached_chunks, ZEND_MM_CHUNK_SIZE); //释放内存
			heap->cached_chunks = chunk; //把当前块加入cached链表
		}
	}
}
```





##### *zend_mm_free_small*

释放*small*内存块

```c
static zend_always_inline void zend_mm_free_small(zend_mm_heap *heap, void *ptr, int bin_num)
{
	zend_mm_free_slot *p;
#if ZEND_MM_STAT
	heap->size -= bin_data_size[bin_num]; //更新记录值
#endif
#if ZEND_DEBUG
	...
#endif
    p = (zend_mm_free_slot*)ptr; //把当前内存地址插入空闲small内存链表头部
    p->next_free_slot = heap->free_slot[bin_num];
    heap->free_slot[bin_num] = p;
}
```





##### *zend_mm_del_huge_block*

释放*huge*链表结点结构体内存

```c
static size_t zend_mm_del_huge_block(zend_mm_heap *heap, void *ptr ZEND_FILE_LINE_DC ZEND_FILE_LINE_ORIG_DC)
{
	zend_mm_huge_list *prev = NULL;
	zend_mm_huge_list *list = heap->huge_list;
	while (list != NULL) { //遍历整个huge内存块链表
		if (list->ptr == ptr) { //找到需要释放的内存块
			size_t size; 
          	//从huge链表把对应结点移除
			if (prev) {  
				prev->next = list->next;
			} else {
				heap->huge_list = list->next;
			}
			size = list->size;
			zend_mm_free_heap(heap, list ZEND_FILE_LINE_RELAY_CC ZEND_FILE_LINE_ORIG_RELAY_CC); //释放了链表结点的内存
			return size;
		}
		prev = list;
		list = list->next;
	}
	ZEND_MM_CHECK(0, "zend_mm_heap corrupted");
	return 0;
}
```



##### *zend_mm_bitset_nts*

获取*bitset*后缀有多少个连续的*1*位

```c
static zend_always_inline int zend_mm_bitset_nts(zend_mm_bitset bitset)
{
#if ...
#else
	int n;
	if (bitset == (zend_mm_bitset)-1) return ZEND_MM_BITSET_LEN;
	n = 0;
#if SIZEOF_ZEND_LONG == 8
	if (sizeof(zend_mm_bitset) == 8) {
		if ((bitset & 0xffffffff) == 0xffffffff) {n += 32; bitset = bitset >> Z_UL(32);}
	}
#endif
  	//利用二分,找出bitset后缀有多少个1位
	if ((bitset & 0x0000ffff) == 0x0000ffff) {n += 16; bitset = bitset >> 16;}
	if ((bitset & 0x000000ff) == 0x000000ff) {n +=  8; bitset = bitset >>  8;}
	if ((bitset & 0x0000000f) == 0x0000000f) {n +=  4; bitset = bitset >>  4;}
	if ((bitset & 0x00000003) == 0x00000003) {n +=  2; bitset = bitset >>  2;}
	return n + (bitset & 1);
#endif
}
```



##### *zend_mm_bitset_set_range*



```c
static zend_always_inline void zend_mm_bitset_set_range(zend_mm_bitset *bitset, int start, int len)
{
	if (len == 1) {
		zend_mm_bitset_set_bit(bitset, start);
	} else {
		int pos = start / ZEND_MM_BITSET_LEN;
		int end = (start + len - 1) / ZEND_MM_BITSET_LEN;
		int bit = start & (ZEND_MM_BITSET_LEN - 1);
		zend_mm_bitset tmp;

		if (pos != end) {
			/* set bits from "bit" to ZEND_MM_BITSET_LEN-1 */
			tmp = (zend_mm_bitset)-1 << bit;
			bitset[pos++] |= tmp;
			while (pos != end) {
				/* set all bits */
				bitset[pos++] = (zend_mm_bitset)-1;
			}
			end = (start + len - 1) & (ZEND_MM_BITSET_LEN - 1);
			/* set bits from "0" to "end" */
			tmp = (zend_mm_bitset)-1 >> ((ZEND_MM_BITSET_LEN - 1) - end);
			bitset[pos] |= tmp;
		} else {
			end = (start + len - 1) & (ZEND_MM_BITSET_LEN - 1);
			/* set bits from "bit" to "end" */
			tmp = (zend_mm_bitset)-1 << bit;
			tmp &= (zend_mm_bitset)-1 >> ((ZEND_MM_BITSET_LEN - 1) - end);
			bitset[pos] |= tmp;
		}
	}
}
```





#### 宏定义

##### *ZEND_MM_CUSTOM*

​	支持自定义内存分配方法

##### *ZEND_MM_LRUN(count)*    

​         *(ZEND_MM_IS_LRUN | ((count) << ZEND_MM_LRUN_PAGES_OFFSET))*

​	标识从当前*page*开始的*count*块都是用作*large*内存分配

##### *ZEND_MM_SRUN(bin_num)*           

​	 *(ZEND_MM_IS_SRUN | ((bin_num) << ZEND_MM_SRUN_BIN_NUM_OFFSET))*

​	标识当前*page*块是用作第*bin_num*种*small*块分配的

##### *ZEND_MM_NRUN(bin_num, offset)*    

​	*(ZEND_MM_IS_SRUN | ZEND_MM_IS_LRUN | ((bin_num) << ZEND_MM_SRUN_BIN_NUM_OFFSET) | ((offset) << ZEND_MM_NRUN_OFFSET_OFFSET))*

​	标识当前*page*块是申请来用作第*bin_num*种*small*块分配的,是申请的第*offset*块



##### *ZEND_MM_NRUN_OFFSET(info)*

​	*(((info) & ZEND_MM_NRUN_OFFSET_MASK) >> ZEND_MM_NRUN_OFFSET_OFFSET)*

​	获取当前*page*块分配用作*small*内存使用时是第几块



##### *ZEND_MM_SRUN_FREE_COUNTER(info)* 

​	*(((info) & ZEND_MM_SRUN_FREE_COUNTER_MASK) >> ZEND_MM_SRUN_FREE_COUNTER_OFFSET)*

​	获取未使用*page*块个数

##### *ZEND_MM_SRUN_EX(bin_num, count)*

​	*(ZEND_MM_IS_SRUN | ((bin_num) << ZEND_MM_SRUN_BIN_NUM_OFFSET) | ((count) << ZEND_MM_SRUN_FREE_COUNTER_OFFSET))*

​	设置未使用*page*块个数

##### *ZEND_MM_SRUN_BIN_NUM(info)*       

​	*(((info) & ZEND_MM_SRUN_BIN_NUM_MASK) >> ZEND_MM_SRUN_BIN_NUM_OFFSET)*

​	获取当前*page*块是用作哪种*small*块使用的	



*0x00000000*  *0x40000000*(标识用作*large*内存块分配)  *0x80000000*(标识用作*small*内存块分配) 一共*32*位

用作*large*内存分配时高*4*位标识位为*0100*,低*16*位存放当次分配*page*的个数(只有当次分配时第一块*page*有标识)

用作*small*内存分配时的*page*块,第一块高*4*位标识位为*1000*,低*16*位存放是哪种*small*块使用,中间*12*位存放当前分配的*small*块未使用的个数,剩下的*page*块,高*4*位标识位为*1100*,低*16*位存放是哪种*small*块使用,中间*12*位存放是当次分配的第几个*page*块






> 相关链接
>
> [php7-internal](https://github.com/pangudashu/php7-internal/blob/master/5/zend_alloc.md)
>
> [mmap函数详解](https://www.cnblogs.com/huxiao-tee/p/4660352.html)
>
> [php的内存分配](https://www.cnblogs.com/taek/p/4229869.html)
>
> 





> 相关问题
>
> **为什么使用mmap**
>
> **PEAL_PAGE_SIZE宏**
>
> **ZEND_MM_ALIGNED_SIZE_EX**











