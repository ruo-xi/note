## 物理内存的组织方式

在linux内核中支持3中内存模型，分别是

* flat memory model(平坦)
* Discontiguous memory model(不连续)
* sparse memory model(稀疏)

```c
//"include/asm-generic/memory_model.h"
#if defined(CONFIG_FLATMEM)

#define __pfn_to_page(pfn)	(mem_map + ((pfn) - ARCH_PFN_OFFSET))
#define __page_to_pfn(page)	((unsigned long)((page) - mem_map) + \
				 ARCH_PFN_OFFSET)
#elif defined(CONFIG_DISCONTIGMEM)

#define __pfn_to_page(pfn)			\
({	unsigned long __pfn = (pfn);		\
	unsigned long __nid = arch_pfn_to_nid(__pfn);  \
	NODE_DATA(__nid)->node_mem_map + arch_local_page_offset(__pfn, __nid);\
})

#define __page_to_pfn(pg)						\
({	const struct page *__pg = (pg);					\
	struct pglist_data *__pgdat = NODE_DATA(page_to_nid(__pg));	\
	(unsigned long)(__pg - __pgdat->node_mem_map) +			\
	 __pgdat->node_start_pfn;					\
})

#elif defined(CONFIG_SPARSEMEM_VMEMMAP)

/* memmap is virtually contiguous.  */
#define __pfn_to_page(pfn)	(vmemmap + (pfn))
#define __page_to_pfn(page)	(unsigned long)((page) - vmemmap)

#elif defined(CONFIG_SPARSEMEM)
/*
 * Note: section's mem_map is encoded to reflect its start_pfn.
 * section[i].section_mem_map == mem_map's address - start_pfn;
 */
#define __page_to_pfn(pg)					\
({	const struct page *__pg = (pg);				\
	int __sec = page_to_section(__pg);			\
	(unsigned long)(__pg - __section_mem_map_addr(__nr_to_section(__sec)));	\
})

#define __pfn_to_page(pfn)				\
({	unsigned long __pfn = (pfn);			\
	struct mem_section *__sec = __pfn_to_section(__pfn);	\
	__section_mem_map_addr(__sec) + __pfn;		\
})
#endif /* CONFIG_FLATMEM/DISCONTIGMEM/SPARSEMEM */
```



### flat memory model && Discontiguous memory model

#### 节点

```c
//"include/linux/mmzone.h"
/*
 * On NUMA machines, each NUMA node would have a pg_data_t to describe
 * it's memory layout. On UMA machines there is a single pglist_data which
 * describes the whole memory.
 *
 * Memory statistics and page replacement data structures are maintained on a
 * per-zone basis.
 */
typedef struct pglist_data {
	struct zone node_zones[MAX_NR_ZONES];
	struct zonelist node_zonelists[MAX_ZONELISTS];
    //当前节点的区域的数量
	int nr_zones;
#ifdef CONFIG_FLAT_NODE_MEM_MAP	/* means !SPARSEMEM */
    //这个节点的struct page数组，用于描述这个节点里面的所有的页
	struct page *node_mem_map;
#endif
..........
    //这个节点的起始页号
	unsigned long node_start_pfn;  //pfn(page frame number 物理页号)
    //这个节点中包含不连续的物理内存地址的页面数
	unsigned long node_present_pages; /* total number of physical pages */
    //是真正可用的物理页面的数目
	unsigned long node_spanned_pages; /* total size of physical page
					     range, including holes */
	int node_id;
    wait_queue_head_t kswapd_wait;
	wait_queue_head_t pfmemalloc_wait;
	struct task_struct *kswapd;	/* Protected by
					   mem_hotplug_begin/end() */
	int kswapd_order;
	enum zone_type kswapd_classzone_idx;

	int kswapd_failures;		/* Number of 'reclaimed == 0' runs */
..................
} pg_data_t;
```

```c
//"include/linux/mmzone.h"
enum zone_type {

#ifdef CONFIG_ZONE_DMA
	ZONE_DMA,  //可做用于32位机器DMA的内存
#endif
#ifdef CONFIG_ZONE_DMA32
	ZONE_DMA32, //在64位机器中需要ZONE_DMA协同工作
#endif
	ZONE_NORMAL, //直接映射区 从物理内存到虚拟内存的内核区域，通过加上一个常量直接映射。
#ifdef CONFIG_HIGHMEM
	ZONE_HIGHMEM, //高端内存区  对于32位系统是896M以外的区域 对于64位则没必要有
#endif
	ZONE_MOVABLE, //可移动区域 通过将物理内存划分为可移动分配区域和不可移动分配区域来避免内存碎片
#ifdef CONFIG_ZONE_DEVICE
	ZONE_DEVICE,
#endif
	__MAX_NR_ZONES

};
```



既然整个内存被分成了多个节点，那pglist_data应该放在一个数组里面。每个节点一项，就像下面代码里面一样：

```c
//"arch/x86/mm/numa.c"
struct pglist_data *node_data[MAX_NUMNODES] __read_mostly;
```

#### 区域

```c
struct zone {
	unsigned long _watermark[NR_WMARK];
	unsigned long watermark_boost;

	unsigned long nr_reserved_highatomic;
	long lowmem_reserve[MAX_NR_ZONES];
    
	struct pglist_data	*zone_pgdat;
    //用于区分冷热页
    //在cpu高速缓存中的页为热页
    //每个cpu一个该数据结构
	struct per_cpu_pageset __percpu *pageset;

	/* zone_start_pfn == zone_start_paddr >> PAGE_SHIFT */
    //zone的第一个页
	unsigned long		zone_start_pfn;
	//被伙伴关系系统管理的页
	atomic_long_t		managed_pages;
    //zone最后的页号 - 最小的页号
	unsigned long		spanned_pages;
    //zone在物理内存中真实存在的page数目
	unsigned long		present_pages;

	const char		*name;


	int initialized;
    
	/* Write-intensive fields used from the page allocator */
	ZONE_PADDING(_pad1_)
        
	/* free areas of different sizes */
	struct free_area	free_area[MAX_ORDER];

	/* zone flags, see below */
	unsigned long		flags;

} ____cacheline_internodealigned_in_smp;
```

#### 页

```c
struct page {
	unsigned long flags;
	union {
		struct {	/* Page cache and anonymous pages */
			struct list_head lru; //表示这一页应该在一个链表上,例如该页被换出或在换出页的链表上
			struct address_space *mapping; //用于内存映射,匿名页最低位为1,映射文件则为0
			pgoff_t index;		 //映射区的偏移量
			unsigned long private; //
		};
		struct {	/* page_pool used by netstack */
			dma_addr_t dma_addr;
		};
		struct {	/* slab, slob and slub */
			union {
				struct list_head slab_list;
				struct {	/* Partial pages */
					struct page *next;
                    #ifdef CONFIG_64BIT
                                        int pages;	/* Nr of pages left */
                                        int pobjects;	/* Approximate count */
                    #else
                                        short int pages;
                                        short int pobjects;
                    #endif
				};
			};
			struct kmem_cache *slab_cache; /* not slob */
			/* Double-word boundary */
			void *freelist;		//空闲对象
			union {
				void *s_mem;	//已经分配乐得正在使用的slab的第一个对象
				unsigned long counters;		/* SLUB */
				struct {			/* SLUB */
					unsigned inuse:16;
					unsigned objects:15;
					unsigned frozen:1;
				};
			};
		};
		struct {	/* Tail pages of compound page 复合页尾页*/
			unsigned long compound_head;	/* Bit zero is set */
			unsigned char compound_dtor;
			unsigned char compound_order;
			atomic_t compound_mapcount;
		};
		struct {	/* Second tail page of compound page */
			unsigned long _compound_pad_1;	/* compound_head */
			atomic_t hpage_pinned_refcount;
			struct list_head deferred_list;
		};
		struct {	/* 页表页 */
			unsigned long _pt_pad_1;	/* compound_head */
			pgtable_t pmd_huge_pte; /* protected by page->ptl */
			unsigned long _pt_pad_2;	/* mapping */
			union {
				struct mm_struct *pt_mm; /* x86 pgds only */
				atomic_t pt_frag_refcount; /* powerpc */
			};
            #if ALLOC_SPLIT_PTLOCKS
                        spinlock_t *ptl;
            #else
                        spinlock_t ptl;
            #endif
		};
		struct {	/* ZONE_DEVICE pages */
	
			struct dev_pagemap *pgmap;
			void *zone_device_data;
		};

		//需要释放的列表
		struct rcu_head rcu_head;
	};

	union {		/* This union is 4 bytes in size. */
		//如果可以被映射到用户空间,则表示有多少个页表项指向了这个页
		atomic_t _mapcount;
		//see page_flags.h for more details
		unsigned int page_type;
		unsigned int active;		/* SLAB */
		int units;			/* SLOB */
	};

	/* Usage count. *DO NOT USE DIRECTLY*. See page_ref.h */
	atomic_t _refcount;
    
} _struct_page_alignment;
```

#### 页的分配

Linux中的内存管理的“页”大小为4KB。把所有的空闲页分组为11个页块链表，每个块链表分别包含很多个大小的页块，有1、2、4、8、16、32、64、128、256、512和1024个连续页的页块。最大可以申请1024个连续页，对应4MB大小的连续内存。每个页块的第一个页的物理地址是该页块大小的整数倍。

![img](https://static001.geekbang.org/resource/image/27/cf/2738c0c98d2ed31cbbe1fdcba01142cf.jpeg)

在zone中如入下定义

```c
//"include/linux/mmzone.h"
//MAX_ORDR = 11
struct free_area	free_area[MAX_ORDER];

struct free_area {
	struct list_head	free_list[MIGRATE_TYPES];
	unsigned long		nr_free;
};
```

当向内核请求分配(2^(i-1)，2^i]数目的页块时，按照2^i页块请求处理。如果对应的页块链表中没有空闲页块，那我们就在更大的页块链表中去找。当分配的页块中有多余的页时，伙伴系统会根据多余的页块大小插入到对应的空闲页块链表中。

上述过程可在以下代码中看到

```c
//"include/linux/gfp.h" //gfp get free page

static inline struct page *
alloc_pages(gfp_t gfp_mask, unsigned int order)
{
	return alloc_pages_current(gfp_mask, order);
}

/**
 * 	alloc_pages_current - Allocate pages.
 *
 *	@gfp:
 *		%GFP_USER   user allocation,
 *      	%GFP_KERNEL kernel allocation,
 *      	%GFP_HIGHMEM highmem allocation,
 *      	%GFP_FS     don't call back into a file system.
 *      	%GFP_ATOMIC don't sleep.
 *	@order: Power of two of allocation size in pages. 0 is a single page.
 *
 *	Allocate a page from the kernel page pool.  When not in
 *	interrupt context and apply the current process NUMA policy.
 *	Returns NULL when no page can be allocated.
 */
struct page *alloc_pages_current(gfp_t gfp, unsigned order)
{
	struct mempolicy *pol = &default_policy;
	struct page *page;

	if (!in_interrupt() && !(gfp & __GFP_THISNODE))
		pol = get_task_policy(current);

	/*
	 * No reference counting needed for current->mempolicy
	 * nor system default_policy
	 */
	if (pol->mode == MPOL_INTERLEAVE)
		page = alloc_page_interleave(gfp, order, interleave_nodes(pol));
	else
		page = __alloc_pages_nodemask(gfp, order,
				policy_node(gfp, pol, numa_node_id()),
				policy_nodemask(gfp, pol));

	return page;
}
```

- GFP_USER用于分配一个页映射到用户进程的虚拟地址空间，并且希望直接被内核或者硬件访问，主要用于一个用户进程希望通过内存映射的方式，访问某些硬件的缓存，例如显卡缓存；
- GFP_KERNEL用于内核中分配页，主要分配ZONE_NORMAL区域，也即直接映射区；
- GFP_HIGHMEM，顾名思义就是主要分配高端区域的内存。

接下来调用__alloc_pages_nodemask。这是伙伴系统的核心方法。它会调用get_page_from_freelist。这里面的逻辑也很容易理解，就是在一个循环中先看当前节点的zone。如果找不到空闲页，则再看备用节点的zone。

```c
tatic struct page *
get_page_from_freelist(gfp_t gfp_mask, unsigned int order, int alloc_flags,
						const struct alloc_context *ac)
{
......
	for_next_zone_zonelist_nodemask(zone, z, ac->zonelist, ac->high_zoneidx, ac->nodemask) 
	struct page *page;
......
	page = rmqueue(ac->preferred_zoneref->zone, zone, order,
				gfp_mask, alloc_flags, ac->migratetype);
......
}
```

每一个zone，都有伙伴系统维护的各种大小的队列，就像上面伙伴系统原理里讲的那样。这里调用rmqueue就很好理解了，就是找到合适大小的那个队列，把页面取下来。

接下来的调用链是rmqueue->__rmqueue->__rmqueue_smallest。在这里，我们能清楚看到伙伴系统的逻辑。

```c
static inline
struct page *__rmqueue_smallest(struct zone *zone, unsigned int order,
						int migratetype)
{
	unsigned int current_order;
	struct free_area *area;
	struct page *page;


	/* Find a page of the appropriate size in the preferred list */
	for (current_order = order; current_order < MAX_ORDER; ++current_order) {
		area = &(zone->free_area[current_order]);
		page = list_first_entry_or_null(&area->free_list[migratetype],
							struct page, lru);
		if (!page)
			continue;
		list_del(&page->lru);
		rmv_page_order(page);
		area->nr_free--;
		expand(zone, page, order, current_order, area, migratetype);
		set_pcppage_migratetype(page, migratetype);
		return page;
	}


	return NULL;
```

从当前的order，也即指数开始，在伙伴系统的free_area找2^order大小的页块。如果链表的第一个不为空，就找到了；如果为空，就到更大的order的页块链表里面去找。找到以后，除了将页块从链表中取下来，我们还要把多余的的部分放到其他页块链表里面。expand就是干这个事情的。area–就是伙伴系统那个表里面的前一项，前一项里面的页块大小是当前项的页块大小除以2，size右移一位也就是除以2，list_add就是加到链表上，nr_free++就是计数加1。

```c
//"mm/page_alloc.c"
static inline void expand(struct zone *zone, struct page *page,
	int low, int high, int migratetype)
{
	unsigned long size = 1 << high;

	while (high > low) {
		high--;
		size >>= 1;
		VM_BUG_ON_PAGE(bad_range(zone, &page[size]), &page[size]);

		/*
		 * Mark as guard pages (or page), that will allow to
		 * merge back to allocator when buddy will be freed.
		 * Corresponding page table entries will not be touched,
		 * pages will stay not present in virtual address space
		 */
		if (set_page_guard(zone, &page[size], high, migratetype))
			continue;

		add_to_free_list(&page[size], zone, high, migratetype);
		set_page_order(&page[size], high);
	}
}
```

#### 总结

![img](https://static001.geekbang.org/resource/image/3f/4f/3fa8123990e5ae2c86859f70a8351f4f.jpeg)

### sparse memory model

```c
struct mem_section {
	/*
	 * This is, logically, a pointer to an array of struct
	 * pages.  However, it is stored with some other magic.
	 * (see sparse.c::sparse_init_one_section())
	 *
	 * Additionally during early boot we encode node id of
	 * the location of the section here to guide allocation.
	 * (see sparse.c::memory_present())
	 *
	 * Making it a UL at least makes someone do a cast
	 * before using it wrong.
	 */
	unsigned long section_mem_map;

	struct mem_section_usage *usage;
#ifdef CONFIG_PAGE_EXTENSION
	/*
	 * If SPARSEMEM, pgdat doesn't have page_ext pointer. We use
	 * section. (see page_ext.h about this.)
	 */
	struct page_ext *page_ext;
	unsigned long pad;
#endif
	/*
	 * WARNING: mem_section must be a power-of-2 in size for the
	 * calculation and use of SECTION_ROOT_MASK to make sense.
	 */
};
```

## 物理内存分配

### 小内存分配

如果遇到小的对象，会使用slub分配器进行分配

创建进程的时候会调用dup_task_struct复制一个tsak_struct对象.

调用链如下dup_task_struct->alloc_task_struct_node->kmem_cache_alloc_node->kmem_cache_alloc

```c
static struct kmem_cache *task_struct_cachep;
//系統初始化的時候该变量会被kmem_cache_create_usercopy函数创建

//args: 缓冲区的名字,缓冲区中块的大小,对象的对齐方式,slab flags,用户复制区域偏移,用户复制区域大小,对象的构造方法
task_struct_cachep = kmem_cache_create_usercopy("task_struct",
			arch_task_struct_size, align,
			SLAB_PANIC|SLAB_ACCOUNT,
			useroffset, usersize, NULL);

kmem_cache_alloc_node(task_struct_cachep, GFP_KERNEL, node);
{
	return kmem_cache_alloc(s, flags);
}

struct kmem_cache *
kmem_cache_create(const char *name, unsigned int size, unsigned int align,
		slab_flags_t flags, void (*ctor)(void *))
{
	return kmem_cache_create_usercopy(name, size, align, flags, 0, 0,
					  ctor);
}

struct kmem_cache {
	struct kmem_cache_cpu __percpu *cpu_slab;
	/* Used for retriving partial slabs etc */
	unsigned long flags;
	unsigned long min_partial;
	int size;		/* The size of an object including meta data */
	int object_size;	/* The size of an object without meta data */
	int offset;		/* Free pointer offset. */
#ifdef CONFIG_SLUB_CPU_PARTIAL
	int cpu_partial;	/* Number of per cpu partial objects to keep around */
#endif
	struct kmem_cache_order_objects oo;
	/* Allocation and freeing of slabs */
	struct kmem_cache_order_objects max;
	struct kmem_cache_order_objects min;
	gfp_t allocflags;	/* gfp flags to use on each alloc */
	int refcount;		/* Refcount for slab cache destroy */
	void (*ctor)(void *);
......
	const char *name;	/* Name (only for display!) */
	struct list_head list;	/* List of slab caches */
......
	struct kmem_cache_node *node[MAX_NUMNODES];
};
```

有了这个缓存区，每次创建task_struct的时候，我们不用到内存里面去分配，先在缓存里面看看有没有直接可用的，这就是**kmem_cache_alloc_node**的作用。

当一个进程结束，task_struct也不用直接被销毁，而是放回到缓存中，这就是**kmem_cache_free**的作用。这样，新进程创建的时候，我们就可以直接用现成的缓存中的task_struct了。

对于缓存来讲，其实就是分配了连续几页的大内存块，然后根据缓存对象的大小，切成小内存块。

所以，我们这里有三个kmem_cache_order_objects类型的变量。这里面的order，就是2的order次方个页面的大内存块，objects就是能够存放的缓存对象的数量。

最终，我们将大内存块切分成小内存块，样子就像下面这样。

![img](https://static001.geekbang.org/resource/image/17/5e/172839800c8d51c49b67ec8c4d07315e.jpeg)

每一项的结构都是缓存对象后面跟一个下一个空闲对象的指针，这样非常方便将所有的空闲对象链成一个链。其实，这就相当于咱们数据结构里面学的，用数组实现一个可随机插入和删除的链表。

所以，这里面就有三个变量：size是包含这个指针的大小，object_size是纯对象的大小，offset就是把下一个空闲对象的指针存放在这一项里的偏移量。

kmem_cache_cpu和kmem_cache_node，它们都是每个NUMA节点上有一个，我们只需要看一个节点里面的情况。

![img](https://static001.geekbang.org/resource/image/45/0a/45f38a0c7bce8c98881bbe8b8b4c190a.jpeg)

在分配缓存块的时候，要分两种路径，**fast path**和**slow path**，也就是**快速通道**和**普通通道**。其中kmem_cache_cpu就是快速通道，kmem_cache_node是普通通道。每次分配的时候，要先从kmem_cache_cpu进行分配。如果kmem_cache_cpu里面没有空闲的块，那就到kmem_cache_node中进行分配；如果还是没有空闲的块，才去伙伴系统分配新的页。

```c
//"include/linux/slub_def.h"
struct kmem_cache_cpu {
	void **freelist;	/* Pointer to next available object */
	unsigned long tid;	/* Globally unique transaction id */
	struct page *page;	/* The slab from which we are allocating */
#ifdef CONFIG_SLUB_CPU_PARTIAL
    //指向大内存块的第一个页
	struct page *partial;	/* Partially allocated frozen slabs */
#endif
#ifdef CONFIG_SLUB_STATS
	unsigned stat[NR_SLUB_STAT_ITEMS];
#endif
};

//"mm/slab.h"
struct kmem_cache_node {
	spinlock_t list_lock;
...........
#ifdef CONFIG_SLUB
	unsigned long nr_partial;
	struct list_head partial;
#ifdef CONFIG_SLUB_DEBUG
	atomic_long_t nr_slabs;
	atomic_long_t total_objects;
	struct list_head full;
#endif
#endif

};
```

```c
//"mm/slub.c"
void *kmem_cache_alloc_node(struct kmem_cache *s, gfp_t gfpflags, int node)
{
	void *ret = slab_alloc_node(s, gfpflags, node, _RET_IP_);

	trace_kmem_cache_alloc_node(_RET_IP_, ret,
				    s->object_size, s->size, gfpflags, node);

	return ret;
}



/*
 * Inlined fastpath so that allocation functions (kmalloc, kmem_cache_alloc)
 * have the fastpath folded into their functions. So no function call
 * overhead for requests that can be satisfied on the fastpath.
 *
 * The fastpath works by first checking if the lockless freelist can be used.
 * If not then __slab_alloc is called for slow processing.
 *
 * Otherwise we can simply pick the next object from the lockless free list.
 */
static __always_inline void *slab_alloc_node(struct kmem_cache *s,
		gfp_t gfpflags, int node, unsigned long addr)
{
	void *object;
	struct kmem_cache_cpu *c;
	struct page *page;
	unsigned long tid;

	s = slab_pre_alloc_hook(s, gfpflags);
	if (!s)
		return NULL;
............
	tid = this_cpu_read(s->cpu_slab->tid);
	c = raw_cpu_ptr(s->cpu_slab);
.............
	object = c->freelist;
	page = c->page;
	if (unlikely(!object || !node_match(page, node))) {
		object = __slab_alloc(s, gfpflags, node, addr, c);
		stat(s, ALLOC_SLOWPATH);
	} 
...........//todo
	return object;
}

//普通通道
static void *___slab_alloc(struct kmem_cache *s, gfp_t gfpflags, int node,
			  unsigned long addr, struct kmem_cache_cpu *c)
{
	void *freelist;
	struct page *page;
	page = c->page;
.....
redo:
.......
	/* must check again c->freelist in case of cpu migration or IRQ */
	freelist = c->freelist;
	if (freelist)
		goto load_freelist;

	freelist = get_freelist(s, page);

	if (!freelist) {
		c->page = NULL;
		stat(s, DEACTIVATE_BYPASS);
		goto new_slab;
	}
load_freelist:
	c->freelist = get_freepointer(s, freelist);
	c->tid = next_tid(c->tid);
	return freelist;
new_slab:
    //替换partial 重试
	if (slub_percpu_partial(c)) {
		page = c->page = slub_percpu_partial(c);
		slub_set_percpu_partial(c, page);
		stat(s, CPU_PARTIAL_ALLOC);
		goto redo;
	}
	
 
	freelist = new_slab_objects(s, gfpflags, node, &c);

	return freelist;
}

//在这里面，get_partial会根据node id，找到相应的kmem_cache_node，然后调用get_partial_node，开始在这个节点进行分配。
static inline void *new_slab_objects(struct kmem_cache *s, gfp_t flags,
			int node, struct kmem_cache_cpu **pc)
{
	void *freelist;
	struct kmem_cache_cpu *c = *pc;
	struct page *page;

	WARN_ON_ONCE(s->ctor && (flags & __GFP_ZERO));

	freelist = get_partial(s, flags, node, c);

	if (freelist)
		return freelist;

	page = new_slab(s, flags, node);
	if (page) {
		c = raw_cpu_ptr(s->cpu_slab);
		if (c->page)
			flush_slab(s, c);

		/*
		 * No other reference to the page yet so we can
		 * muck around with it freely without cmpxchg
		 */
		freelist = page->freelist;
		page->freelist = NULL;

		stat(s, ALLOC_SLAB);
		c->page = page;
		*pc = c;
	}

	return freelist;
}

//从kmem_cache_node 中取出缓存
static void *get_partial_node(struct kmem_cache *s, struct kmem_cache_node *n,
				struct kmem_cache_cpu *c, gfp_t flags)
{
	struct page *page, *page2;
	void *object = NULL;
	unsigned int available = 0;
	int objects;
.......
	list_for_each_entry_safe(page, page2, &n->partial, slab_list) {
		void *t;

		if (!pfmemalloc_match(page, flags))
			continue;
		//从n中的partial链表中拿下一大块内存,并将freelist赋给t
		t = acquire_slab(s, n, page, object == NULL, &objects);
		if (!t)
			break;

		available += objects;
		if (!object) {
            //更新kmem_cache_cpu中的page
			c->page = page;
			stat(s, ALLOC_FROM_PARTIAL);
			object = t;
		} else {
            //更新kmem_cache_cpu中的partial
			put_cpu_partial(s, page, 0);
			stat(s, CPU_PARTIAL_NODE);
		}
		if (!kmem_cache_has_cpu_partial(s)
			|| available > slub_cpu_partial(s) / 2)
			break;

	}
	spin_unlock(&n->list_lock);
	return object;
}

//创建缓存
static struct page *allocate_slab(struct kmem_cache *s, gfp_t flags, int node)
{
	struct page *page;
	struct kmem_cache_order_objects oo = s->oo;
	gfp_t alloc_gfp;
	void *start, *p, *next;
	int idx;
	bool shuffle;

	flags &= gfp_allowed_mask;

	if (gfpflags_allow_blocking(flags))
		local_irq_enable();

	flags |= s->allocflags;
........
    //使用伙伴系统分配内存
	page = alloc_slab_page(s, alloc_gfp, node, oo);
	if (unlikely(!page)) {
        //失败的话缩小order
		oo = s->min;
		alloc_gfp = flags;
		/*
		 * Allocation may have failed due to fragmentation.
		 * Try a lower order alloc if possible
		 */
		page = alloc_slab_page(s, alloc_gfp, node, oo);
		if (unlikely(!page))
			goto out;
		stat(s, ORDER_FALLBACK);
	}
    
	return page;
}
```

## 页面换出

### 触发页面换出的条件    //todo

* direct reclaim.
* kswapd内核线程

Linux中物理内存的每个zone都有自己独立的min, low和high三个档位的watermark值,

可以通过cat /proc/zoninfo 命令查看这三个值的大小,以page为单位

进行内存分配的时候,发现当前空余内存的值低于"low"但高于"min"，说明现在内存面临一定的压力，那么在此次内存分配完成后，kswapd将被唤醒，以执行内存回收操作。





### 总结

- 物理内存分NUMA节点，分别进行管理；
- 每个NUMA节点分成多个内存区域；
- 每个内存区域分成多个物理页面；
- 伙伴系统将多个连续的页面作为一个大的内存块分配给上层；
- kswapd负责物理页面的换入换出；
- Slub Allocator将从伙伴系统申请的大内存块切成小块，分配给其他系统。

![img](https://static001.geekbang.org/resource/image/52/54/527e5c861fd06c6eb61a761e4214ba54.jpeg)