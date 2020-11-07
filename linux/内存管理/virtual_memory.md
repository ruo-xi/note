# 内存管理

* 物理内存管理

* 虚拟地址管理

* 虚拟地址和物理地址间的映射

```c
#include <stdio.h>
#include <stdlib.h>

int max_length = 128;

char * generate(int length){
  int i;
  char * buffer = (char*) malloc (length+1);
  if (buffer == NULL)
    return NULL;
  for (i=0; i<length; i++){
    buffer[i]=rand()%26+'a';
  }
  buffer[length]='\0';
  return buffer;
}

int main(int argc, char *argv[])
{
  int num;
  char * buffer;

  printf ("Input the string length : ");
  scanf ("%d", &num);

  if(num > max_length){
    num = max_length;
  }

  buffer = generate(num);

  printf ("Random string is: %s\n",buffer);
  free (buffer);

  return 0;
}
```

分析下这段代码的内存使用情况

用户态:

* 代码
* 全局变量
* 常量字符串
* 函数栈
* 堆
* glibc的调用

malloc也会调用系统调用,进入内核运行时也需要分配内存.

内核态:

* 内核的代码
* 内核中的全局变量
* task_struc
* 内核栈
* 动态分配的内存
* 虚拟地址到物理地址的映射表

以上全部都是使用虚拟地址.

32位系统有4G的内存空间

64位系统在x86_64中只使用了48位,对应了256TB的地址空间.

## 虚拟内存空间

首先虚拟空间分为两个部分

* 内核空间
* 用户空间

用户空间占用低地址,内核空间占用高地址

![img](https://static001.geekbang.org/resource/image/af/83/afa4beefd380effefb0e54a8d9345c83.jpeg)

**Memory Mapping Segment**可以用来把文件映射进内存用的，如果二进制的执行文件依赖于某个动态链接库，就是在这个区域里面将so文件映射到了内存中。

但是到了内核里面，无论是从哪个进程进来的，看到的都是同一个内核空间，看到的都是同一个进程列表。虽然内核栈是各用个的，但是如果想知道的话，还是能够知道每个进程的内核栈在哪里的。所以，如果要访问一些公共的数据结构，需要进行锁保护。

![img](https://static001.geekbang.org/resource/image/4e/9d/4ed91c744220d8b4298237d2ab2eda9d.jpeg)

### 分段映射

​	![img](https://static001.geekbang.org/resource/image/96/eb/9697ae17b9f561e78514890f9d58d4eb.jpg)

​	分段机制下的虚拟地址由两部分组成，**段选择子**和**段内偏移量**。段选择子就保存在咱们前面讲过的段寄存器里面。段选择子里面最重要的是**段号**，用作段表的索引。段表里面保存的是这个段的**基地址**、**段的界限**和**特权等级**等。虚拟地址中的段内偏移量应该位于0和段界限之间。如果段内偏移量是合法的，就将段基地址加上段内偏移量得到物理内存地址。

​	在Linux里面，段表全称**段描述符表**（segment descriptors），放在**全局描述符表GDT**（Global Descriptor Table）里面，会有下面的宏来初始化段描述符表里面的表项。

```c
struct desc_struct {
	u16	limit0;
	u16	base0;
	u16	base1: 8, type: 4, s: 1, dpl: 2, p: 1;
	u16	limit1: 4, avl: 1, l: 1, d: 1, g: 1, base2: 8;
} __attribute__((packed));

#define GDT_ENTRY_INIT(flags, base, limit)			\
	{							\
		.limit0		= (u16) (limit),		\
		.limit1		= ((limit) >> 16) & 0x0F,	\
		.base0		= (u16) (base),			\
		.base1		= ((base) >> 16) & 0xFF,	\
		.base2		= ((base) >> 24) & 0xFF,	\
		.type		= (flags & 0x0f),		\ //segment type
		.s		= (flags >> 4) & 0x01,		\ //descruotir type(0=system 1=code or data)
		.dpl		= (flags >> 5) & 0x03,		\ //descriptor privilege(特权) level
		.p		= (flags >> 7) & 0x01,		\ //segment present
		.avl		= (flags >> 12) & 0x01,		\ //avaliable for use bt system software
		.l		= (flags >> 13) & 0x01,		\ //64-bit cide segment(IA-32e mode only)
		.d		= (flags >> 14) & 0x01,		\ //default opration size
		.g		= (flags >> 15) & 0x01,		\//granularity(粒度)
	}
```

```c
DEFINE_PER_CPU_PAGE_ALIGNED(struct gdt_page, gdt_page) = { .gdt = {
#ifdef CONFIG_X86_64
	/*
	 * We need valid kernel segments for data and code in long mode too
	 * IRET will check the segment types  kkeil 2000/10/28
	 * Also sysret mandates a special GDT layout
	 *
	 * TLS descriptors are currently at a different place compared to i386.
	 * Hopefully nobody expects them at a fixed place (Wine?)
	 */
	[GDT_ENTRY_KERNEL32_CS]		= GDT_ENTRY_INIT(0xc09b, 0, 0xfffff),
	[GDT_ENTRY_KERNEL_CS]		= GDT_ENTRY_INIT(0xa09b, 0, 0xfffff),
	[GDT_ENTRY_KERNEL_DS]		= GDT_ENTRY_INIT(0xc093, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER32_CS]	= GDT_ENTRY_INIT(0xc0fb, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_DS]	= GDT_ENTRY_INIT(0xc0f3, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_CS]	= GDT_ENTRY_INIT(0xa0fb, 0, 0xfffff),
#else
	[GDT_ENTRY_KERNEL_CS]		= GDT_ENTRY_INIT(0xc09a, 0, 0xfffff),
	[GDT_ENTRY_KERNEL_DS]		= GDT_ENTRY_INIT(0xc092, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_CS]	= GDT_ENTRY_INIT(0xc0fa, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_DS]	= GDT_ENTRY_INIT(0xc0f2, 0, 0xfffff),
	/*
	 * Segments used for calling PnP BIOS have byte granularity.
	 * They code segments and data segments have fixed 64k limits,
	 * the transfer segment sizes are set at run time.
	 */
	/* 32-bit code */
	[GDT_ENTRY_PNPBIOS_CS32]	= GDT_ENTRY_INIT(0x409a, 0, 0xffff),
	/* 16-bit code */
	[GDT_ENTRY_PNPBIOS_CS16]	= GDT_ENTRY_INIT(0x009a, 0, 0xffff),
	/* 16-bit data */
	[GDT_ENTRY_PNPBIOS_DS]		= GDT_ENTRY_INIT(0x0092, 0, 0xffff),
	/* 16-bit data */
	[GDT_ENTRY_PNPBIOS_TS1]		= GDT_ENTRY_INIT(0x0092, 0, 0),
	/* 16-bit data */
	[GDT_ENTRY_PNPBIOS_TS2]		= GDT_ENTRY_INIT(0x0092, 0, 0),
	/*
	 * The APM segments have byte granularity and their bases
	 * are set at run time.  All have 64k limits.
	 */
	/* 32-bit code */
	[GDT_ENTRY_APMBIOS_BASE]	= GDT_ENTRY_INIT(0x409a, 0, 0xffff),
	/* 16-bit code */
	[GDT_ENTRY_APMBIOS_BASE+1]	= GDT_ENTRY_INIT(0x009a, 0, 0xffff),
	/* data */
	[GDT_ENTRY_APMBIOS_BASE+2]	= GDT_ENTRY_INIT(0x4092, 0, 0xffff),

	[GDT_ENTRY_ESPFIX_SS]		= GDT_ENTRY_INIT(0xc092, 0, 0xfffff),
	[GDT_ENTRY_PERCPU]		= GDT_ENTRY_INIT(0xc092, 0, 0xfffff),
	GDT_STACK_CANARY_INIT
#endif
} };
EXPORT_PER_CPU_SYMBOL_GPL(gdt_page);
```

另外，还会定义下面四个段选择子，指向上面的段描述符表项。这四个段选择子看着是不是有点眼熟？咱们讲内核初始化的时候，启动第一个用户态的进程，就是将这四个值赋值给段寄存器。

```c
#define __KERNEL_CS			(GDT_ENTRY_KERNEL_CS*8)
#define __KERNEL_DS			(GDT_ENTRY_KERNEL_DS*8)
#define __USER_DS			(GDT_ENTRY_DEFAULT_USER_DS*8 + 3)
#define __USER_CS			(GDT_ENTRY_DEFAULT_USER_CS*8 + 3)
```

通过分析，我们发现，所有的段的起始地址都是一样的，都是0。这算哪门子分段嘛！所以，在Linux操作系统中，并没有使用到全部的分段功能。那分段是不是完全没有用处呢？分段可以做权限审核，例如用户态DPL是3，内核态DPL是0。当用户态试图访问内核态的时候，会因为权限不足而报错。

### 分页映射

对于物理内存，操作系统把它分成一块一块大小相同的页，这样更方便管理，例如有的内存页面长时间不用了，可以暂时写到硬盘上，称为**换出**。一旦需要的时候，再加载进来，叫作**换入**。这样可以扩大可用物理内存的大小，提高物理内存的利用率。

```c
//在x86系统中页的大小的定义
#define PAGE_SIZE		(_AC(1,UL) << PAGE_SHIFT)
```

这个换入和换出都是以页为单位的。页面的大小一般为4KB。为了能够定位和访问每个页，需要有个页表，保存每个页的起始地址，再加上在页内的偏移量，组成线性地址，就能对于内存中的每个位置进行访问了。

![img](https://static001.geekbang.org/resource/image/ab/40/abbcafe962d93fac976aa26b7fcb7440.jpg)

虚拟地址分为两部分，**页号**和**页内偏移**。页号作为页表的索引，页表包含物理页每页所在物理内存的基地址。这个基地址与页内偏移的组合就形成了物理内存地址。

下面的图，举了一个简单的页表的例子，虚拟内存中的页通过页表映射为了物理内存中的页。

![img](https://static001.geekbang.org/resource/image/84/eb/8495dfcbaed235f7500c7e11149b2feb.jpg)

* 32位
  ![img](https://static001.geekbang.org/resource/image/b6/b8/b6960eb0a7eea008d33f8e0c4facc8b8.jpg)

* 64位

![img](https://static001.geekbang.org/resource/image/42/0b/42eff3e7574ac8ce2501210e25cd2c0b.jpg)

### 总结

- 第一，虚拟内存空间的管理，将虚拟内存分成大小相等的页；
- 第二，物理内存的管理，将物理内存分成大小相等的页；
- 第三，内存映射，将虚拟内存也和物理内存也映射起来，并且在内存紧张的时候可以换出到硬盘中。

![img](https://static001.geekbang.org/resource/image/7d/91/7dd9039e4ad2f6433aa09c14ede92991.jpg)

## 管理虚拟内存空间

### 用户态和内核态的划分

```c
struct mm_struct {
	struct {
		struct vm_area_struct *mmap;		/* list of VMAs */
		struct rb_root mm_rb;
		u64 vmacache_seqnum;                   /* per-thread vmacache */
#ifdef CONFIG_MMU
		unsigned long (*get_unmapped_area) (struct file *filp,
				unsigned long addr, unsigned long len,
				unsigned long pgoff, unsigned long flags);
#endif
		unsigned long mmap_base;	/* base of mmap area */
		unsigned long mmap_legacy_base;	/* base of mmap area in bottom-up allocations */
............
		unsigned long task_size;	/* size of task vm space */
		unsigned long highest_vm_end;	/* highest vma end address */
		pgd_t * pgd;
........
		unsigned long hiwater_rss; /* High-watermark of RSS usage */
		unsigned long hiwater_vm;  /* High-water virtual memory usage */

		unsigned long total_vm;	   /* Total pages mapped */
		unsigned long locked_vm;   /* Pages that have PG_mlocked set */
		atomic64_t    pinned_vm;   /* Refcount permanently increased */
		unsigned long data_vm;	   /* VM_WRITE & ~VM_SHARED & ~VM_STACK */
		unsigned long exec_vm;	   /* VM_EXEC & ~VM_WRITE & ~VM_STACK */
		unsigned long stack_vm;	   /* VM_STACK */
		unsigned long def_flags;

		spinlock_t arg_lock; /* protect the below fields */
		unsigned long start_code, end_code, start_data, end_data;
		unsigned long start_brk, brk, start_stack;
		unsigned long arg_start, arg_end, env_start, env_end;

		unsigned long saved_auxv[AT_VECTOR_SIZE]; /* for /proc/PID/auxv */

		/*
		 * Special counters, in some configurations protected by the
		 * page_table_lock, in other configurations by being atomic.
		 */
		struct mm_rss_stat rss_stat;

		struct linux_binfmt *binfmt;

		/* Architecture-specific MM context */
		mm_context_t context;

		unsigned long flags; /* Must use atomic bitops to access */

		struct core_state *core_state; /* coredumping support */

		struct user_namespace *user_ns;

		/* store ref to file /proc/<pid>/exe symlink points to */
		struct file __rcu *exe_file;

		struct uprobes_state uprobes_state;
        
		struct work_struct async_put_work;
	} __randomize_layout;

	/*
	 * The mm_cpumask needs to be at the end of mm_struct, because it
	 * is dynamically sized based on nr_cpu_ids.
	 */
	unsigned long cpu_bitmap[];
};
```

```c
#ifdef CONFIG_X86_32
/*
 * User space process size: 3GB (default).
 */
#define IA32_PAGE_OFFSET	PAGE_OFFSET
#define TASK_SIZE		PAGE_OFFSET
#define TASK_SIZE_LOW		TASK_SIZE
#define TASK_SIZE_MAX		TASK_SIZE
#define DEFAULT_MAP_WINDOW	TASK_SIZE
#define STACK_TOP		TASK_SIZE
#define STACK_TOP_MAX		STACK_TOP

/**
"arch/x86/Kconfig"
config PAGE_OFFSET
	hex
	default 0xB0000000 if VMSPLIT_3G_OPT
	default 0x80000000 if VMSPLIT_2G
	default 0x78000000 if VMSPLIT_2G_OPT
	default 0x40000000 if VMSPLIT_1G
	default 0xC0000000
	depends on X86_32
*/

#else
#define TASK_SIZE_MAX	((1UL << __VIRTUAL_MASK_SHIFT) - PAGE_SIZE)

#define DEFAULT_MAP_WINDOW	((1UL << 47) - PAGE_SIZE)

/* This decides where the kernel will search for a free chunk of vm
 * space during mmap's.
 */
#define IA32_PAGE_OFFSET	((current->personality & ADDR_LIMIT_3GB) ? \
					0xc0000000 : 0xFFFFe000)

#define TASK_SIZE_LOW		(test_thread_flag(TIF_ADDR32) ? \
					IA32_PAGE_OFFSET : DEFAULT_MAP_WINDOW)
#define TASK_SIZE		(test_thread_flag(TIF_ADDR32) ? \
					IA32_PAGE_OFFSET : TASK_SIZE_MAX)
#define TASK_SIZE_OF(child)	((test_tsk_thread_flag(child, TIF_ADDR32)) ? \
					IA32_PAGE_OFFSET : TASK_SIZE_MAX)

#define STACK_TOP		TASK_SIZE_LOW
#define STACK_TOP_MAX		TASK_SIZE_MAX

#define INIT_THREAD  {						\
	.addr_limit		= KERNEL_DS,			\
}

extern unsigned long KSTK_ESP(struct task_struct *task);

#endif /* CONFIG_X86_64 */
```

### 用户态布局(32位)

#### 位置

```
unsigned long mmap_base;	/* base of mmap area */
unsigned long total_vm;		/* Total pages mapped */
unsigned long locked_vm;	/* Pages that have PG_mlocked set */
unsigned long pinned_vm;	/* Refcount permanently increased  */
unsigned long data_vm;		/* VM_WRITE & ~VM_SHARED & ~VM_STACK */
unsigned long exec_vm;		/* VM_EXEC & ~VM_WRITE & ~VM_STACK */
unsigned long stack_vm;		/* VM_STACK */
unsigned long start_code, end_code, start_data, end_data;
unsigned long start_brk, brk, start_stack;
unsigned long arg_start, arg_end, env_start, env_end;
```

* mmap_base 表示虚拟地址空间中用于内存映射的起始地址 该空间从高地址到低地址增长.malloc申请大块地址时通过mmap在这里映射一块区域到物理内存.
* total_vm 总共映射的页的数目
* locked_vm 锁定不能换出的页的数目
* pinned_vm 不能换出和移动的页的数目
* data_vm 存放数据的页的数目
* exec_vm 存放可执行文件的页的数目
* stack_vm 栈所占的页的数目
* start_brk 堆的起始位置
* brk 堆当前的结束位置 malloc申请小块内存时通过改变brk位置
* start_stack 栈的起始位置  栈的结束位置在寄存器的栈顶指针中
* arg_start & arg_end 参数列表的位置
* env_start & env_end 环境变量的位置

![img](https://static001.geekbang.org/resource/image/f8/b1/f83b8d49b4e74c0e255b5735044c1eb1.jpg)

#### 区域信息

```c
struct vm_area_struct *mmap;		/* list of VMAs */
struct rb_root mm_rb; //红黑树
```

这里面一个将这些区域串起来的单链表,还有一个红黑树用来快速查找一个内存区域.

```c
//"include/linux/mm_types.h"
struct vm_area_struct {
	/* The first cache line has the info for VMA tree walking. */
	unsigned long vm_start;		/* Our start address within vm_mm. */
	unsigned long vm_end;		/* The first byte after our end address
					   within vm_mm. */

	/* linked list of VM areas per task, sorted by address */
	struct vm_area_struct *vm_next, *vm_prev;

	struct rb_node vm_rb;
............
	struct mm_struct *vm_mm;	/* The address space we belong to. */
..............
    //虚拟内存区域可以映射到物理内存，也可以映射到文件，映射到物理内存的时候称为匿名映射
    //(anonymous 匿名)
	struct list_head anon_vma_chain; /* Serialized by mmap_sem &
					  * page_table_lock */
	struct anon_vma *anon_vma;	/* Serialized by page_table_lock */

	//vm_ops里面是对这个内存区域可以做的操作的定义
	const struct vm_operations_struct *vm_ops;
.........
    //映射到文件就需要有vm_file指定被映射的文件
	struct file * vm_file;		/* File we map to (can be NULL). */
	void * vm_private_data;		/* was vm_pte (shared mem) */
.........
} __randomize_layout;
```

该结构通过load_elf_binary和上面的内存区域产生联系

```c
static int load_elf_binary(struct linux_binprm *bprm)
{
	//设置内存调用区mmap_base
	setup_new_exec(bprm);
	//设置栈的vm_area_struct  mm->arg_start 指向 栈底
	retval = setup_arg_pages(bprm, randomize_stack_top(STACK_TOP),
				 executable_stack);
	//将elf文件中的代码部分映射到内存中来
	error = elf_map(bprm->file, load_bias + vaddr, elf_ppnt,
			elf_prot, elf_flags, total_size);
    //设置了堆的vm_area_struct current->mm->start_brk = current->mm->brk
    retval = set_brk(elf_bss, elf_brk, bss_prot);
	//将依赖的so映射到内存中的内存映射区域
    elf_entry = load_elf_interp(interp_elf_ex,
					    interpreter,
					    load_bias, interp_elf_phdata);
		
	mm = current->mm;
	mm->end_code = end_code;
	mm->start_code = start_code;
	mm->start_data = start_data;
	mm->end_data = end_data;
	mm->start_stack = bprm->p;
}
```

![img](https://static001.geekbang.org/resource/image/7a/4c/7af58012466c7d006511a7e16143314c.jpeg)

映射完毕后,在以下情况下会发生变化

* 函数的调用   设计函数栈的变化,主要改变栈顶指针
* 通过malloc申请一个堆的空间,要么执行brk 要么执行mmap.

brk系统调用

```c
//"mm/mmap.c"
SYSCALL_DEFINE1(brk, unsigned long, brk)
{
	unsigned long retval;
	unsigned long newbrk, oldbrk, origbrk;
	struct mm_struct *mm = current->mm;
	struct vm_area_struct *next;
..........
    
    // 页对齐
    // (typeof(brk))(page_size) - 1 => mask
    // (((brk) + (mask)) & ~(mask)) => PAGE_ALIGN
	newbrk = PAGE_ALIGN(brk);
	oldbrk = PAGE_ALIGN(mm->brk);
    //若相等则说明增加的堆的量很小,还在一个页里面不需要另行分配页
	if (oldbrk == newbrk) {
		mm->brk = brk;
		goto success;
	}
	//若小于 则说明至少释放了一页
	if (brk <= mm->brk) {
		int ret;
        
		mm->brk = brk;
        //释放内存映射
		ret = __do_munmap(mm, newbrk, oldbrk-newbrk, &uf, true);
		if (ret < 0) {
			mm->brk = origbrk;
			goto out;
		} else if (ret == 1) {
			downgraded = true;
		}
		goto success;
	}

	/* Check against existing mmap mappings. */
	//查找堆所在的vm_area_struct的下一个vm_area_struct  并且判断是否越界
	next = find_vma(mm, oldbrk); //   //该函数的作用是找到mm中第一个vm_end>addr的vma
	if (next && newbrk + PAGE_SIZE > vm_start_gap(next))
		goto out;

	/* Ok, looks good - let it rip. */
	if (do_brk_flags(oldbrk, newbrk-oldbrk, 0, &uf) < 0)
		goto out;
	mm->brk = brk;

success:
	populate = newbrk > oldbrk && (mm->def_flags & VM_LOCKED) != 0;
	if (downgraded)
		up_read(&mm->mmap_sem);
	else
		up_write(&mm->mmap_sem);
	userfaultfd_unmap_complete(mm, &uf);
	if (populate)
		mm_populate(oldbrk, newbrk - oldbrk);
	return brk;

}

static int do_brk_flags(unsigned long addr, unsigned long len, unsigned long flags, struct list_head *uf)
{
	struct mm_struct *mm = current->mm;
	struct vm_area_struct *vma, *prev;
	struct rb_node **rb_link, *rb_parent;
	pgoff_t pgoff = addr >> PAGE_SHIFT;
	int error;
    // Clear old maps.  this also does some error checking for us
	while (find_vma_links(mm, addr, addr + len, &prev, &rb_link,
			      &rb_parent)) {
		if (do_munmap(mm, addr, len, uf))
			return -ENOMEM;
	}
	/* Can we just expand an old private anonymous mapping? */
	vma = vma_merge(mm, prev, addr, addr + len, flags,
			NULL, NULL, pgoff, NULL, NULL_VM_UFFD_CTX);
	if (vma)
		goto out;

	/*
	 * create a vma struct for an anonymous mapping
	 */
	vma = vm_area_alloc(mm);
	if (!vma) {
		vm_unacct_memory(len >> PAGE_SHIFT);
		return -ENOMEM;
	}

	vma_set_anonymous(vma);
	vma->vm_start = addr;
	vma->vm_end = addr + len;
	vma->vm_pgoff = pgoff;
	vma->vm_flags = flags;
	vma->vm_page_prot = vm_get_page_prot(flags);
	vma_link(mm, vma, prev, rb_link, rb_parent);
out:
	perf_event_mmap(vma);
	mm->total_vm += len >> PAGE_SHIFT;
	mm->data_vm += len >> PAGE_SHIFT;
	if (flags & VM_LOCKED)
		mm->locked_vm += (len >> PAGE_SHIFT);
	vma->vm_flags |= VM_SOFTDIRTY;
	return 0;
}
```

在do_brk中，调用find_vma_links找到将来的vm_area_struct节点在红黑树的位置，找到它的父节点、前序节点。接下来调用vma_merge，看这个新节点是否能够和现有树中的节点合并。如果地址是连着的，能够合并，则不用创建新的vm_area_struct了，直接跳到out，更新统计值即可；如果不能合并，则创建新的vm_area_struct，既加到anon_vma_chain链表中，也加到红黑树中。

### 内核态的布局

#### 32位

![img](https://static001.geekbang.org/resource/image/83/04/83a6511faf802014fbc2c02afc397a04.jpg)

32位的内核态虚拟地址空间一共就1G，占绝大部分的前896M，我们称为**直接映射区**。

所谓的直接映射区，就是这一块空间是连续的，和物理内存是非常简单的映射关系，其实就是虚拟内存地址减去3G，就得到物理内存的位置。

在内核里面，有两个宏：

- __pa(vaddr) 返回与虚拟地址 vaddr 相关的物理地址；
- __va(paddr) 则计算出对应于物理地址 paddr 的虚拟地址。

```c
    #define __va(x)			((void *)((unsigned long)(x)+PAGE_OFFSET))
    #define __pa(x)		__phys_addr((unsigned long)(x))
    #define __phys_addr(x)		__phys_addr_nodebug(x)
    #define __phys_addr_nodebug(x)	((x) - PAGE_OFFSET)
```



但是你要注意，这里虚拟地址和物理地址发生了关联关系，在物理内存的开始的896M的空间，会被直接映射到3G至3G+896M的虚拟地址，这样容易给你一种感觉，是这些内存访问起来和物理内存差不多，别这样想，在大部分情况下，对于这一段内存的访问，在内核中，还是会使用虚拟地址的，并且将来也会为这一段空间建设页表，对这段地址的访问也会走上一节我们讲的分页地址的流程，只不过页表里面比较简单，是直接的一一对应而已。

这896M还需要仔细分解。在系统启动的时候，物理内存的前1M已经被占用了，从1M开始加载内核代码段，然后就是内核的全局变量、BSS等，也是ELF里面涵盖的。这样内核的代码段，全局变量，BSS也就会被映射到3G后的虚拟地址空间里面。具体的物理内存布局可以查看/proc/iomem。

在内核运行的过程中，如果碰到系统调用创建进程，会创建task_struct这样的实例，内核的进程管理代码会将实例创建在3G至3G+896M的虚拟空间中，当然也会被放在物理内存里面的前896M里面，相应的页表也会被创建。

在内核运行的过程中，会涉及内核栈的分配，内核的进程管理的代码会将内核栈创建在3G至3G+896M的虚拟空间中，当然也就会被放在物理内存里面的前896M里面，相应的页表也会被创建。

896M这个值在内核中被定义为high_memory，在此之上常称为“高端内存”。这是个很笼统的说法，到底是虚拟内存的3G+896M以上的是高端内存，还是物理内存896M以上的是高端内存呢？

这里仍然需要辨析一下，高端内存是物理内存的概念。它仅仅是内核中的内存管理模块看待物理内存的时候的概念。前面我们也说过，在内核中，除了内存管理模块直接操作物理地址之外，内核的其他模块，仍然要操作虚拟地址，而虚拟地址是需要内存管理模块分配和映射好的。

假设咱们的电脑有2G内存，现在如果内核的其他模块想要访问物理内存1.5G的地方，应该怎么办呢？如果你觉得，我有32位的总线，访问个2G还不小菜一碟，这就错了。

首先，你不能使用物理地址。你需要使用内存管理模块给你分配的虚拟地址，但是虚拟地址的0到3G已经被用户态进程占用去了，你作为内核不能使用。因为你写1.5G的虚拟内存位置，一方面你不知道应该根据哪个进程的页表进行映射；另一方面，就算映射了也不是你真正想访问的物理内存的地方，所以你发现你作为内核，能够使用的虚拟内存地址，只剩下1G减去896M的空间了。

于是，我们可以将剩下的虚拟内存地址分成下面这几个部分。

- 在896M到VMALLOC_START之间有8M的空间。
- VMALLOC_START到VMALLOC_END之间称为内核动态映射空间，也即内核想像用户态进程一样malloc申请内存，在内核里面可以使用vmalloc。假设物理内存里面，896M到1.5G之间已经被用户态进程占用了，并且映射关系放在了进程的页表中，内核vmalloc的时候，只能从分配物理内存1.5G开始，就需要使用这一段的虚拟地址进行映射，映射关系放在专门给内核自己用的页表里面。
- PKMAP_BASE到FIXADDR_START的空间称为持久内核映射。使用alloc_pages()函数的时候，在物理内存的高端内存得到struct page结构，可以调用kmap将其在映射到这个区域。
- FIXADDR_START到FIXADDR_TOP(0xFFFF F000)的空间，称为固定映射区域，主要用于满足特殊需求。
- 在最后一个区域可以通过kmap_atomic实现临时内核映射。假设用户态的进程要映射一个文件到内存中，先要映射用户态进程空间的一段虚拟地址到物理内存，然后将文件内容写入这个物理内存供用户态进程访问。给用户态进程分配物理内存页可以通过alloc_pages()，分配完毕后，按说将用户态进程虚拟地址和物理内存的映射关系放在用户态进程的页表中，就完事大吉了。这个时候，用户态进程可以通过用户态的虚拟地址，也即0至3G的部分，经过页表映射后访问物理内存，并不需要内核态的虚拟地址里面也划出一块来，映射到这个物理内存页。但是如果要把文件内容写入物理内存，这件事情要内核来干了，这就只好通过kmap_atomic做一个临时映射，写入物理内存完毕后，再kunmap_atomic来解映射即可。

#### 64位

![img](https://static001.geekbang.org/resource/image/7e/f6/7eaf620768c62ff53e5ea2b11b4940f6.jpg)

64位的内核主要包含以下几个部分。

从0xffff800000000000开始就是内核的部分，只不过一开始有8T的空档区域。

从__PAGE_OFFSET_BASE(0xffff880000000000)开始的64T的虚拟地址空间是直接映射区域，也就是减去PAGE_OFFSET就是物理地址。虚拟地址和物理地址之间的映射在大部分情况下还是会通过建立页表的方式进行映射。

从VMALLOC_START（0xffffc90000000000）开始到VMALLOC_END（0xffffe90000000000）的32T的空间是给vmalloc的。

从VMEMMAP_START（0xffffea0000000000）开始的1T空间用于存放物理页面的描述结构struct page的。

从__START_KERNEL_map（0xffffffff80000000）开始的512M用于存放内核代码段、全局变量、BSS等。这里对应到物理内存开始的位置，减去__START_KERNEL_map就能得到物理内存的地址。这里和直接映射区有点像，但是不矛盾，因为直接映射区之前有8T的空当区域，早就过了内核代码在物理内存中加载的位置。

### 总结

用户态：

- 代码段、全局变量、BSS
- 函数栈
- 堆
- 内存映射区

内核态：

- 内核的代码、全局变量、BSS
- 内核数据结构例如task_struct
- 内核栈
- 内核中动态分配的内存

![img](https://static001.geekbang.org/resource/image/28/e8/2861968d1907bc314b82c34c221aace8.jpeg)

![img](https://static001.geekbang.org/resource/image/2a/ce/2ad275ff8fdf6aafced4a7aeea4ca0ce.jpeg)