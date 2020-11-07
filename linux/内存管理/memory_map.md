

# 内存映射

内存映射包括:

* 物理内存映射到虚拟内存
* 文件中的内容映射到虚拟内存

## 用户态内存映射

### mmap原理

```c
struct mm_struct {
	struct vm_area_struct *mmap;		/* list of VMAs */
......
}
```

```c
//"arch/x86/kernel/sys_x86_64.c"
//若映射文件则传入一个文件描述符.
SYSCALL_DEFINE6(mmap, unsigned long, addr, unsigned long, len,
		unsigned long, prot, unsigned long, flags,
		unsigned long, fd, unsigned long, off)
{
	long error;
	error = -EINVAL;
	if (off & ~PAGE_MASK)
		goto out;

	error = ksys_mmap_pgoff(addr, len, prot, flags, fd, off >> PAGE_SHIFT);
out:
	return error;
}
//"mm/mmap.c"
SYSCALL_DEFINE6(mmap_pgoff, unsigned long, addr, unsigned long, len,
		unsigned long, prot, unsigned long, flags,
		unsigned long, fd, unsigned long, pgoff)
{
	return ksys_mmap_pgoff(addr, len, prot, flags, fd, pgoff);
}

```

接下来的调用链是vm_mmap_pgoff->do_mmap_pgoff->do_mmap。

```c
//"mm/util.c"
unsigned long vm_mmap_pgoff(struct file *file, unsigned long addr,
	unsigned long len, unsigned long prot,
	unsigned long flag, unsigned long pgoff)
{
.........
		ret = do_mmap_pgoff(file, addr, len, prot, flag, pgoff,
				    &populate, &uf);
.........
	return ret;
}

//"include/linux/mm.h"
static inline unsigned long
do_mmap_pgoff(struct file *file, unsigned long addr,
	unsigned long len, unsigned long prot, unsigned long flags,
	unsigned long pgoff, unsigned long *populate,
	struct list_head *uf)
{
	return do_mmap(file, addr, len, prot, flags, 0, pgoff, populate, uf);
}

//"mm/mmap.c"
/*
 * The caller must hold down_write(&current->mm->mmap_sem).
 */
unsigned long do_mmap(struct file *file, unsigned long addr,
			unsigned long len, unsigned long prot,
			unsigned long flags, vm_flags_t vm_flags,
			unsigned long pgoff, unsigned long *populate,
			struct list_head *uf)
{
	struct mm_struct *mm = current->mm;
	int pkey = 0;

...........
	addr = get_unmapped_area(file, addr, len, pgoff, flags);
............
	addr = mmap_region(file, addr, len, vm_flags, pgoff, uf);
...........
	return addr;
}

/*
这里面如果是匿名映射，则调用mm_struct里面的get_unmapped_area函数。这个函数其实是arch_get_unmapped_area。它会调用find_vma_prev，在表示虚拟内存区域的vm_area_struct红黑树上找到相应的位置。之所以叫prev，是说这个时候虚拟内存区域还没有建立，找到前一个vm_area_struct。

如果不是匿名映射，而是映射到一个文件，这样在Linux里面，每个打开的文件都有一个struct file结构，里面有一个file_operations，用来表示和这个文件相关的操作。如果是我们熟知的ext4文件系统，调用的是thp_get_unmapped_area。如果我们仔细看这个函数，最终还是调用mm_struct里面的get_unmapped_area函数。殊途同归。
*/
unsigned long
get_unmapped_area(struct file *file, unsigned long addr, unsigned long len,
		unsigned long pgoff, unsigned long flags)
{
	unsigned long (*get_area)(struct file *, unsigned long,
				  unsigned long, unsigned long, unsigned long);
........
	get_area = current->mm->get_unmapped_area;
	if (file) {
		if (file->f_op->get_unmapped_area)
			get_area = file->f_op->get_unmapped_area;
	} else if (flags & MAP_SHARED) {
		pgoff = 0;
		get_area = shmem_get_unmapped_area;
	}

	addr = get_area(file, addr, len, pgoff, flags);
.....
	return error ? error : addr;
}


unsigned long mmap_region(struct file *file, unsigned long addr,
		unsigned long len, vm_flags_t vm_flags, unsigned long pgoff,
		struct list_head *uf)
{
	struct mm_struct *mm = current->mm;
	struct vm_area_struct *vma, *prev;
	struct rb_node **rb_link, *rb_parent;
	/* Check against address space limit. */
........
	/* Clear old maps */
...........
	vma = vma_merge(mm, prev, addr, addr + len, vm_flags,
			NULL, file, pgoff, NULL, NULL_VM_UFFD_CTX);
	if (vma)
		goto out;
	vma = vm_area_alloc(mm);
	if (!vma) {
		error = -ENOMEM;
		goto unacct_error;
	}

	vma->vm_start = addr;
	vma->vm_end = addr + len;
	vma->vm_flags = vm_flags;
	vma->vm_page_prot = vm_get_page_prot(vm_flags);
	vma->vm_pgoff = pgoff;

	if (file) {
..........
		vma->vm_file = get_file(file);
		error = call_mmap(file, vma); 
     //file->f_op->mmap(file, vma)   ->   vma->vm_ops = &ext4_file_vm_ops;
     //在这里我们将vm_area_struct的内存操作设置为文件系统操作，也就是说，读写内存其实就是读写文件系统。
	}
..........
	//将vm_area_struct 挂载到mm_struct的红黑树上面
    //通过_vma_link_file 建立文件到内存的映射
	vma_link(mm, vma, prev, rb_link, rb_parent);
...............
}

//对于打开的文件，会有一个结构struct file来表示。它有个成员指向struct address_space结构，这里面有棵变量名为i_mmap的红黑树，vm_area_struct就挂在这棵树上。
struct address_space {
	struct inode		*host;		/* owner: inode, block_device */
......
	struct rb_root		i_mmap;		/* tree of private and shared mappings */
......
	const struct address_space_operations *a_ops;	/* methods */
......
}


static void __vma_link_file(struct vm_area_struct *vma)
{
	struct file *file;


	file = vma->vm_file;
	if (file) {
		struct address_space *mapping = file->f_mapping;
		vma_interval_tree_insert(vma, &mapping->i_mmap);
	}
}
```

**以上并没有与物理内存产生关联,因为内存管理并不直接分配物理内存,只有用到的时候才会开始分配**

### 用户态却页异常

**一旦开始访问虚拟内存的某个地址，如果我们发现，并没有对应的物理页，那就触发缺页中断，调用do_page_fault。**

## 内核态内存映射