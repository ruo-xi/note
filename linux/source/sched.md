# 进程调度

## 数据结构

#### 调度策略

```c
unsigned int policy;  //进程采取调度策略

#define SCHED_NORMAL		0
#define SCHED_FIFO			1
#define SCHED_RR			2
#define SCHED_BATCH			3
#define SCHED_IDLE			5
#define SCHED_DEADLINE		6
//优先级
int prio, static_prio, normal_prio;
unsigned int rt_priority;
```

#### 实时进程(rt real_time)

​	实时进程是优先级范围在0~99之间的进程

​	对于实时进程采取的调度策略有 

* SHED_FIFO   (First in First out   先进先出)

  高优先级的进程可以抢占低优先级的进程，而相同优先级的进程，我们遵循先来先得。

* SCHED_RR (round robin 轮流调度)

  采用时间片，相同优先级的任务当用完时间片会被放到队列尾部，以保证公平性，而高优先级的任务也是可以抢占低优先级的任务。

* SCHED_DEADLINE  

  当产生一个调度点的时候，DL调度器总是选择其deadline距离当前时间点最近的那个任务，并调度它执行

#### 普通进程

​	普通进程是优先级范围在100~139之间的进程	

​	调度策略

*  SCHED_NORMAL(指普通进程)
*  SCHED_BATCH (指的是后台进程)
*  SCHED_IDLE (指的是空闲时才跑得进程)

#### 执行逻辑

```c
//"include/linux/sched.h"
const struct sched_class *sched_class;
```

sched_class的几种实现

- stop_sched_class优先级最高的任务会使用这种策略，会中断所有其他线程，且不会被其他任务打断；
- dl_sched_class就对应上面的deadline调度策略；
- rt_sched_class就对应RR算法或者FIFO算法的调度策略，具体调度策略由进程的task_struct->policy指定；
- fair_sched_class就是普通进程的调度策略；
- idle_sched_class就是空闲进程的调度策略。

#### 完全公平调度算法

​	在Linux里面，实现了一个基于CFS的调度算法。CFS全称Completely Fair Scheduling，叫完全公平调度。

​	首先，你需要记录下进程的运行时间。CPU会提供一个时钟，过一段时间就触发一个时钟中断,也叫Tick。CFS会为每一个进程安排一个虚拟运行时间vruntime。如果一个进程在运行，随着时间的增长，也就是一个个tick的到来，进程的vruntime将不断增大。没有得到执行的进程vruntime不变。

​	不同优先级的进程分配不同的权重.优先级搞得权重小,优先级低的权重大.

```c
//"kernel/sched/fair.c"
/*
 * Update the current task's runtime statistics.
 */
static void update_curr(struct cfs_rq *cfs_rq)
{
	struct sched_entity *curr = cfs_rq->curr;
    //获取当前时间
	u64 now = rq_clock_task(rq_of(cfs_rq));
	u64 delta_exec;

	if (unlikely(!curr))
		return;
	//当前时间片运行时间
	delta_exec = now - curr->exec_start;
	if (unlikely((s64)delta_exec <= 0))
		return;
	
	curr->exec_start = now;

	schedstat_set(curr->statistics.exec_max,
		      max(delta_exec, curr->statistics.exec_max));

	curr->sum_exec_runtime += delta_exec;
	schedstat_add(cfs_rq->exec_clock, delta_exec);
	//vruntime += delta_exec*curr.load.weight
	curr->vruntime += calc_delta_fair(delta_exec, curr);
	update_min_vruntime(cfs_rq);

	if (entity_is_task(curr)) {
		struct task_struct *curtask = task_of(curr);

		trace_sched_stat_runtime(curtask, delta_exec, curr->vruntime);
		cgroup_account_cputime(curtask, delta_exec);
		account_group_exec_runtime(curtask, delta_exec);
	}

	account_cfs_rq_runtime(cfs_rq, delta_exec);
}

/*
 * delta /= w
 */
static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
{
	if (unlikely(se->load.weight != NICE_0_LOAD))
		delta = __calc_delta(delta, NICE_0_LOAD, &se->load);

	return delta;
}
```

#### 调度队列与调度实体

​	Linux使用红黑树来存储调度队列并且对vruntime进行排序.红黑树的节点也叫调度实体

​	task_struct中有以下成员变量

```c
//"kernel/sched/sched.h"
struct sched_entity se; //普通调度实体
struct sched_rt_entity rt;  //实时调度实体
struct sched_dl_entity dl;	//Deadline 调度实体

struct sched_entity {
	/* For load-balancing: */
	struct load_weight		load;
	struct rb_node			run_node;
	struct list_head		group_node;
	unsigned int			on_rq;
	u64				exec_start;
	u64				sum_exec_runtime;
	u64				vruntime; 
	u64				prev_sum_exec_runtime;
	u64				nr_migrations;
	struct sched_statistics		statistics;
#ifdef CONFIG_FAIR_GROUP_SCHED
	int				depth;
	struct sched_entity		*parent;
	/* rq on which this entity is (to be) queued: */
	struct cfs_rq			*cfs_rq;
	/* rq "owned" by this entity/group: */
	struct cfs_rq			*my_q;
	/* cached value of my_q->h_nr_running */
	unsigned long			runnable_weight;
#endif
    
/*
*CPU架构有
*AMP(Asymmetric Multiprocessing)  非对称多处理， 不同CPU可能运行独立的操作系统
*SMP(Symmetric Multiprocessing)  一个操作系统，管理所有CPU核
*BMP(Bound  *Multiprocessing)  是一个操作系统管理所有CPU核，但是应用锁定于某个制定核心。
*HMP(Heterogeneous Multiprocessing) 主要是ARM big.LITTLE架构在使用 HMP内部的CPU核并不完全对等。
*/

#ifdef CONFIG_SMP
	/*
	 * Per entity load average tracking.
	 *
	 * Put into separate cache line so it does not
	 * collide with read-mostly values above.
	 */
	struct sched_avg		avg;
#endif
};
```

​	每个CPU都有自己的 struct rq 结构，其用于描述在此CPU上所运行的所有进程，其包括一个实时进程队列rt_rq和一个CFS运行队列cfs_rq，在调度时，调度器首先会先去实时进程队列找是否有实时进程需要运行，如果没有才会去CFS运行队列找是否有进行需要运行。

```c
//"kernel/sched/sched.h"
struct rq {
	/* runqueue lock: */
	raw_spinlock_t		lock;
	/*
	 * nr_running and cpu_load should be in the same cacheline because
	 * remote CPUs use both these fields when doing load calculation.
	 */
	unsigned int		nr_running;
	unsigned long		nr_load_updates;
	u64			nr_switches;
.....
	struct cfs_rq		cfs;
	struct rt_rq		rt;
	struct dl_rq		dl;
.....
	struct task_struct __rcu	*curr;
	struct task_struct	*idle;
	struct task_struct	*stop;
........
};
```



```c
//"kernel/sched/sched.h"
struct cfs_rq {
	struct load_weight	load;
	unsigned int		nr_running;
	unsigned int		h_nr_running;      /* SCHED_{NORMAL,BATCH,IDLE} */
	unsigned int		idle_h_nr_running; /* SCHED_IDLE */

	u64			exec_clock;
	u64			min_vruntime;
#ifndef CONFIG_64BIT
	u64			min_vruntime_copy;
#endif

	struct rb_root_cached	tasks_timeline;

	/*
	 * 'curr' points to currently running entity on this cfs_rq.
	 * It is set to NULL otherwise (i.e when none are currently running).
	 */
	struct sched_entity	*curr;
	struct sched_entity	*next;
	struct sched_entity	*last;
	struct sched_entity	*skip;
	...............
}

struct rb_root_cached {
	struct rb_root rb_root;   //红黑树的根节点
	struct rb_node *rb_leftmost;  //红黑树的最左节点
};



```

上面这些数据结构的关系如下图所示:

![img](https://static001.geekbang.org/resource/image/ac/fd/ac043a08627b40b85e624477d937f3fd.jpeg)

#### 调度类是如何工作的

调度类的定义

```c
//"kernel/sched/sched.h"
struct sched_class {
	const struct sched_class *next;  //指向下一个调度类
    // 顺序是 stop->dl->rt->fair->idle   对应的方法会在后面加上   _****

#ifdef CONFIG_UCLAMP_TASK
	int uclamp_enabled;
#endif
	//向就绪队列中添加一个进程,当某个进程进入可运行状态时调用这个函数.
	void (*enqueue_task) (struct rq *rq, struct task_struct *p, int flags);
    //将一个进程从就绪队列中删除
	void (*dequeue_task) (struct rq *rq, struct task_struct *p, int flags);
	void (*yield_task)   (struct rq *rq);
	bool (*yield_to_task)(struct rq *rq, struct task_struct *p, bool preempt);

	void (*check_preempt_curr)(struct rq *rq, struct task_struct *p, int flags);
	//选择接下来要运行的程序
	struct task_struct *(*pick_next_task)(struct rq *rq);
	//用另一个进程替代当前运行的进程
	void (*put_prev_task)(struct rq *rq, struct task_struct *p);
	void (*set_next_task)(struct rq *rq, struct task_struct *p, bool first);

#ifdef CONFIG_SMP
	int (*balance)(struct rq *rq, struct task_struct *prev, struct rq_flags *rf);
	int  (*select_task_rq)(struct task_struct *p, int task_cpu, int sd_flag, int flags);
	void (*migrate_task_rq)(struct task_struct *p, int new_cpu);

	void (*task_woken)(struct rq *this_rq, struct task_struct *task);

	void (*set_cpus_allowed)(struct task_struct *p,
				 const struct cpumask *newmask);

	void (*rq_online)(struct rq *rq);
	void (*rq_offline)(struct rq *rq);
#endif
	//每次周期性时钟到的时候,该函数被调用,可触发调度
	void (*task_tick)(struct rq *rq, struct task_struct *p, int queued);
	void (*task_fork)(struct task_struct *p);
	void (*task_dead)(struct task_struct *p);

	/*
	 * The switched_from() call is allowed to drop rq->lock, therefore we
	 * cannot assume the switched_from/switched_to pair is serliazed by
	 * rq->lock. They are however serialized by p->pi_lock.
	 */
	void (*switched_from)(struct rq *this_rq, struct task_struct *task);
	void (*switched_to)  (struct rq *this_rq, struct task_struct *task);
	void (*prio_changed) (struct rq *this_rq, struct task_struct *task,
			      int oldprio);

	unsigned int (*get_rr_interval)(struct rq *rq,
					struct task_struct *task);
	//修改调度策略
	void (*update_curr)(struct rq *rq);

#define TASK_SET_GROUP		0
#define TASK_MOVE_GROUP		1

#ifdef CONFIG_FAIR_GROUP_SCHED
	void (*task_change_group)(struct task_struct *p, int type);
#endif
};
```

调度器的种类

```c
extern const struct sched_class stop_sched_class;
extern const struct sched_class dl_sched_class;
extern const struct sched_class rt_sched_class;
extern const struct sched_class fair_sched_class;
extern const struct sched_class idle_sched_class;
```

它们其实是放在一个链表上的。这里我们以调度最常见的操作，**取下一个任务**为例，来解析一下。可以看到，这里面有一个for_each_class循环，沿着上面的顺序，依次调用每个调度类的方法。

```c
/*
 * Pick up the highest-prio task:
 */
static inline struct task_struct *
pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
......
	for_each_class(class) {
		p = class->pick_next_task(rq);
		if (p)
			return p;
	}
......
}
```

#### 总结

![img](https://static001.geekbang.org/resource/image/10/af/10381dbafe0f78d80beb87560a9506af.jpeg)

## 调度的过程

### 主动调度

#### 调用shedule()函数

```c
//"arch/x86/include/asm/current.h"
static __always_inline struct task_struct *get_current(void)
{
	return this_cpu_read_stable(current_task);
}

#define current get_current

//"kernel/sched/core.c"
asmlinkage __visible void __sched schedule(void)
{	
	struct task_struct *tsk = current;

	sched_submit_work(tsk);
	do {
		preempt_disable();   //preempt 抢占  不允许抢占
		__schedule(false);
		sched_preempt_enable_no_resched();
	} while (need_resched());
	sched_update_worker(tsk);
}


static void __sched notrace __schedule(bool preempt)
{	
	struct task_struct *prev, *next;
	unsigned long *switch_count;
	struct rq_flags rf;
	struct rq *rq;
	int cpu;

	cpu = smp_processor_id();
	rq = cpu_rq(cpu);
    //prev  指向这个CPU的任务队列上面正在运行的那个进程curr,因为一旦将来它被切换下来，那它就成了前任了。
	prev = rq->curr;
........
    //获取下一个task_struct
	next = pick_next_task(rq, prev, &rf);
	clear_tsk_need_resched(prev);
	clear_preempt_need_resched();


if (likely(prev != next)) {
		rq->nr_switches++;

		RCU_INIT_POINTER(rq->curr, next);

		++*switch_count;

		psi_sched_switch(prev, next, !task_on_rq_queued(prev));

		trace_sched_switch(preempt, prev, next);
		/* Also unlocks the rq: */
    	//当选出的继任者和前任不同，就要进行上下文切换，继任者进程正式进入运行。
		rq = context_switch(rq, prev, next, &rf);
	} else {
		rq->clock_update_flags &= ~(RQCF_ACT_SKIP|RQCF_REQ_SKIP);
		rq_unlock_irq(rq, &rf);
	}

	balance_callback(rq);
}

```

#### 获取下一个task_struct

```c
static inline struct task_struct *
pick_next_task(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
	const struct sched_class *class;
	struct task_struct *p;
	//假如当前正在运行的进程是普通进程,那么直接从普通进程调度类中寻找下一个进程
	if (likely((prev->sched_class == &idle_sched_class ||
		    prev->sched_class == &fair_sched_class) &&
		   rq->nr_running == rq->cfs.h_nr_running)) {

		p = pick_next_task_fair(rq, prev, rf);
		if (unlikely(p == RETRY_TASK))
			goto restart;

		if (!p) {
			put_prev_task(rq, prev);
			p = pick_next_task_idle(rq);
		}

		return p;
	}

restart:
#ifdef CONFIG_SMP

	for_class_range(class, prev->sched_class, &idle_sched_class) {
		if (class->balance(rq, prev, rf))
			break;
	}
#endif

	put_prev_task(rq, prev);

	for_each_class(class) {
		p = class->pick_next_task(rq);
		if (p)
			return p;
	}

	/* The idle class should always have a runnable task: */
	BUG();
}

//"kernel/sched/fair.c"  
//以fair_sched_class.pick_next_task()为例 
struct task_struct *
pick_next_task_fair(struct rq *rq, struct task_struct *prev, struct rq_flags *rf)
{
	struct cfs_rq *cfs_rq = &rq->cfs;
	struct sched_entity *se;
	struct task_struct *p;
	int new_tasks;

again:
	if (!sched_fair_runnable(rq))
		goto idle;

#ifdef CONFIG_FAIR_GROUP_SCHED
	if (!prev || prev->sched_class != &fair_sched_class)
		goto simple;

	do {
		struct sched_entity *curr = cfs_rq->curr;
        
		if (curr) {
			if (curr->on_rq)
                //更新当前进程的vruntime，然后更新红黑树和cfs_rq -> min_vruntime以及left_most。
				update_curr(cfs_rq);
			else
				curr = NULL;
........
		}
		//获取最左边的节点
		se = pick_next_entity(cfs_rq, curr);
		cfs_rq = group_cfs_rq(se);
	} while (cfs_rq);
	//通过shce_entity 获取task_struct
	p = task_of(se);
	if (prev != p) {
		struct sched_entity *pse = &prev->se;
........
		put_prev_entity(cfs_rq, pse);
		set_next_entity(cfs_rq, se);
	}
........
return p
}
```

#### 进程上下文切换

##### 概念介绍

* pt_regs 存储的是用户态的硬件上下文

  用户态 -> 内核态后，由于使用的栈、内存地址空间、代码段等都不同，所以用户态的eip、esp、ebp等需要保存现场，内核态 -> 用户态时再将栈中的信息恢复到硬件。由于进程调度一定会在内核态的schedule函数，用户态的所有硬件信息都保存在pt_regs中了。SAVE_ALL指令就是将用户态的cpu寄存器值保存如内核栈，RESTORE_ALL就是将pt_regs中的值恢复到寄存器中，这两个指令在介绍中断的时候还会提到。

* tss(task state segment)

  这是intel为上层做进程切换提供的硬件支持，还有一个TR（task register）寄存器专门指向这个内存区域。当TR指针值变更时，intel会将当前所有寄存器值存放到当前进程的tss中，然后再讲切换进程的目标tss值加载后寄存器中，其32位结构如下：

  ![img](https://static001.geekbang.org/resource/image/df/64/dfa9762cfec16822ec74d53350db4664.png)

上下文切换主要干两件事情，一是切换进程空间，也即虚拟内存；二是切换寄存器和CPU上下文。

但是这样有个缺点。我们做进程切换的时候，没必要每个寄存器都切换，这样每个进程一个TSS，就需要全量保存，全量切换，动作太大了。

于是，Linux操作系统想了一个办法。还记得在系统初始化的时候，会调用cpu_init吗？这里面会给每一个CPU关联一个TSS，然后将TR指向这个TSS，然后在操作系统的运行过程中，TR就不切换了，永远指向这个TSS。TSS用数据结构tss_struct表示，在x86_hw_tss中可以看到和上图相应的结构。

```c
//"arch/x86/include/asm/processor.h"
struct tss_struct {
	struct x86_hw_tss	x86_tss;

	struct x86_io_bitmap	io_bitmap;
} __aligned(PAGE_SIZE);

//32位
struct x86_hw_tss {
	unsigned short		back_link, __blh;
	unsigned long		sp0;
	unsigned short		ss0, __ss0h;
	unsigned long		sp1;
	/*
	 * We don't use ring 1, so ss1 is a convenient scratch space in
	 * the same cacheline as sp0.  We use ss1 to cache the value in
	 */
	unsigned short		ss1;	/* MSR_IA32_SYSENTER_CS */

	unsigned short		__ss1h;
	unsigned long		sp2;
	unsigned short		ss2, __ss2h;
	unsigned long		__cr3;
	unsigned long		ip;
	unsigned long		flags;
	unsigned long		ax;
	unsigned long		cx;
	unsigned long		dx;
	unsigned long		bx;
	unsigned long		sp;
	unsigned long		bp;
	unsigned long		si;
	unsigned long		di;
	unsigned short		es, __esh;
	unsigned short		cs, __csh;
	unsigned short		ss, __ssh;
	unsigned short		ds, __dsh;
	unsigned short		fs, __fsh;
	unsigned short		gs, __gsh;
	unsigned short		ldt, __ldth;
	unsigned short		trace;
	unsigned short		io_bitmap_base;

} __attribute__((packed));
#else
//64位
struct x86_hw_tss {
	u32			reserved1;
	u64			sp0;
	/*
	 * We store cpu_current_top_of_stack in sp1 so it's always accessible.
	 * Linux does not use ring 1, so sp1 is not otherwise needed.
	 */
	u64			sp1;
	/*
	 * Since Linux does not use ring 2, the 'sp2' slot is unused by
	 * hardware.  entry_SYSCALL_64 uses it as scratch space to stash
	 * the user RSP value.
	 */
	u64			sp2;

	u64			reserved2;
	u64			ist[7];
	u32			reserved3;
	u32			reserved4;
	u16			reserved5;
	u16			io_bitmap_base;

} __attribute__((packed));
#endif


//"arch/x86/kernel/cpu/common.c"
void cpu_init(void)
{
	struct tss_struct *tss = this_cpu_ptr(&cpu_tss_rw);
	struct task_struct *cur = current;
	int cpu = raw_smp_processor_id();
..........................
	/* Initialize the TSS. */
	tss_setup_ist(tss);
	tss_setup_io_bitmap(tss);
	set_tss_desc(cpu, &get_cpu_entry_area(cpu)->tss.x86_tss);

	load_TR_desc();
	/*
	 * sp0 points to the entry trampoline stack regardless of what task
	 * is running.
	 */
	load_sp0((unsigned long)(cpu_entry_stack(cpu) + 1));
.........................
}
```

在Linux中，真的参与进程切换的寄存器很少，主要的就是栈顶寄存器。

于是，在task_struct里面，还有一个我们原来没有注意的成员变量thread。这里面保留了要切换进程的时候需要修改的寄存器。

```c
/* CPU-specific state of this task: */
	struct thread_struct		thread;
```

所谓的进程切换，就是将某个进程的thread_struct里面的寄存器的值，写入到CPU的TR指向的tss_struct，对于CPU来讲，这就算是完成了切换。

##### 流程介绍

```c
/*
 * context_switch - switch to the new MM and the new thread's register state.
 */
static __always_inline struct rq *
context_switch(struct rq *rq, struct task_struct *prev,
	       struct task_struct *next, struct rq_flags *rf)
{
	prepare_task_switch(rq, prev, next);

	/*
	 * For paravirt, this is coupled with an exit in switch_to to
	 * combine the page table reload and the switch backend into
	 * one hypercall.
	 */
	arch_start_context_switch(prev);

	/*
	 * kernel -> kernel   lazy + transfer active
	 *   user -> kernel   lazy + mmgrab() active
	 *
	 * kernel ->   user   switch + mmdrop() active
	 *   user ->   user   switch
	 */
	if (!next->mm) {                                // to kernel
		enter_lazy_tlb(prev->active_mm, next);

		next->active_mm = prev->active_mm;
		if (prev->mm)                           // from user
			mmgrab(prev->active_mm);
		else
			prev->active_mm = NULL;
	} else {                                        // to user
		membarrier_switch_mm(rq, prev->active_mm, next->mm);
		/*
		 * sys_membarrier() requires an smp_mb() between setting
		 * rq->curr / membarrier_switch_mm() and returning to userspace.
		 *
		 * The below provides this either through switch_mm(), or in
		 * case 'prev->active_mm == next->mm' through
		 * finish_task_switch()'s mmdrop().
		 */
		switch_mm_irqs_off(prev->active_mm, next->mm, next);

		if (!prev->mm) {                        // from kernel
			/* will mmdrop() in finish_task_switch(). */
			rq->prev_mm = prev->active_mm;
			prev->active_mm = NULL;
		}
	}

	rq->clock_update_flags &= ~(RQCF_ACT_SKIP|RQCF_REQ_SKIP);

	prepare_lock_switch(rq, next, rf);

	/* Here we just switch the register state and the stack. */
    //这里是寄存器状态和栈的切换函数
	switch_to(prev, next, prev);
	barrier();

	return finish_task_switch(prev);
}
```

* 切换进程空间

* 切换寄存器和CPU上下文

  ```c
  //"arch/x86/include/asm/switch_to.h"
  #define switch_to(prev, next, last)					\
  do {									\
  	prepare_switch_to(next);					\
  									\
  	((last) = __switch_to_asm((prev), (next)));			\
  } while (0)
  ```

  ```assembly
  /*"arch/x86/entry/entry_32.S"*/
  /*切换栈顶指针esp*/
  SYM_CODE_START(__switch_to_asm)
  .........
  	/* switch stack */
  	/*保存prev task内核栈的esp指针到thread_struct -> sp*/
  	movl	%esp, TASK_threadsp(%eax)
  	/*将next的thread_struct -> sp恢复到esp寄存器*/
  	movl	TASK_threadsp(%edx), %esp
  	/*此后所有的操作都在next task的内核栈上运行。*/
  .......
  	jmp	__switch_to
  SYM_CODE_END(__switch_to_asm)
  
  
  
  /*"arch/x86/entry/entry_64.S"*/
  /*切换栈顶指针rsp*/
  SYM_FUNC_START(__switch_to_asm)
  .........
  	movq	%rsp, TASK_threadsp(%rdi)
  	movq	TASK_threadsp(%rsi), %rsp
  .........
	jmp	__switch_to
  SYM_FUNC_END(__switch_to_asm)
  ```
  
  
  
  ```c
  //"arch/x86/kernel/process_32.c"
  __visible __notrace_funcgraph struct task_struct *
  __switch_to(struct task_struct *prev_p, struct task_struct *next_p)
  {
  	struct thread_struct *prev = &prev_p->thread,
  			     *next = &next_p->thread;
  	struct fpu *prev_fpu = &prev->fpu;
  	struct fpu *next_fpu = &next->fpu;
  	int cpu = smp_processor_id();
  
  	/* never put a printk in __switch_to... printk() calls wake_up*() indirectly */
  
  	if (!test_thread_flag(TIF_NEED_FPU_LOAD))
  		switch_fpu_prepare(prev_fpu, cpu);
  
  	/*
  	 * Save away %gs. No need to save %fs, as it was saved on the
  	 * stack on entry.  No need to save %es and %ds, as those are
  	 * always kernel segments while inside the kernel.  Doing this
  	 * before setting the new TLS descriptors avoids the situation
  	 * where we temporarily have non-reloadable segments in %fs
  	 * and %gs.  This could be an issue if the NMI handler ever
  	 * used %fs or %gs (it does not today), or if the kernel is
  	 * running inside of a hypervisor layer.
  	 */
  	lazy_save_gs(prev->gs);
  
  	/*
  	 * Load the per-thread Thread-Local Storage descriptor.
  	 */
  	load_TLS(next, cpu);
  
  	switch_to_extra(prev_p, next_p);
  
  	/*
  	 * Leave lazy mode, flushing any hypercalls made here.
  	 * This must be done before restoring TLS segments so
  	 * the GDT and LDT are properly updated.
  	 */
  	arch_end_context_switch(next_p);
  
  	/*
  	 * Reload esp0 and cpu_current_top_of_stack.  This changes
  	 * current_thread_info().  Refresh the SYSENTER configuration in
  	 * case prev or next is vm86.
  	 */
      //update_task_stack(next_p)->load_sp0(task->thread.sp0);next task的esp0加载到tss中
  	update_task_stack(next_p);
  	refresh_sysenter_cs(next);
  	this_cpu_write(cpu_current_top_of_stack,
  		       (unsigned long)task_stack_page(next_p) +
  		       THREAD_SIZE);
  
  	/*
  	 * Restore %gs if needed (which is common)
  	 */
  	if (prev->gs | next->gs)
  		lazy_load_gs(next->gs);
  
  	this_cpu_write(current_task, next_p);
  
  	switch_fpu_finish(next_fpu);
  
  	/* Load the Intel cache allocation PQR MSR. */
  	resctrl_sched_in();
  
  	return prev_p;
  }
  
  //"arch/x86/kernel/process_64.c"
  __visible __notrace_funcgraph struct task_struct *
  __switch_to(struct task_struct *prev_p, struct task_struct *next_p)
  {
  	struct thread_struct *prev = &prev_p->thread;
  	struct thread_struct *next = &next_p->thread;
  	struct fpu *prev_fpu = &prev->fpu;
  	struct fpu *next_fpu = &next->fpu;
  	int cpu = smp_processor_id();
  
  	WARN_ON_ONCE(IS_ENABLED(CONFIG_DEBUG_ENTRY) &&
  		     this_cpu_read(irq_count) != -1);
  
  	if (!test_thread_flag(TIF_NEED_FPU_LOAD))
  		switch_fpu_prepare(prev_fpu, cpu);
  
  	/* We must save %fs and %gs before load_TLS() because
  	 * %fs and %gs may be cleared by load_TLS().
  	 *
  	 * (e.g. xen_load_tls())
  	 */
  	save_fsgs(prev_p);
  
  	/*
  	 * Load TLS before restoring any segments so that segment loads
  	 * reference the correct GDT entries.
  	 */
  	load_TLS(next, cpu);
  
  	/*
  	 * Leave lazy mode, flushing any hypercalls made here.  This
  	 * must be done after loading TLS entries in the GDT but before
  	 * loading segments that might reference them.
  	 */
  	arch_end_context_switch(next_p);
  
  	/* Switch DS and ES.
  	 *
  	 * Reading them only returns the selectors, but writing them (if
  	 * nonzero) loads the full descriptor from the GDT or LDT.  The
  	 * LDT for next is loaded in switch_mm, and the GDT is loaded
  	 * above.
  	 *
  	 * We therefore need to write new values to the segment
  	 * registers on every context switch unless both the new and old
  	 * values are zero.
  	 *
  	 * Note that we don't need to do anything for CS and SS, as
  	 * those are saved and restored as part of pt_regs.
  	 */
  	savesegment(es, prev->es);
  	if (unlikely(next->es | prev->es))
  		loadsegment(es, next->es);
  
  	savesegment(ds, prev->ds);
  	if (unlikely(next->ds | prev->ds))
  		loadsegment(ds, next->ds);
  
  	x86_fsgsbase_load(prev, next);
  
  	/*
  	 * Switch the PDA and FPU contexts.
  	 */
  	this_cpu_write(current_task, next_p);
  	this_cpu_write(cpu_current_top_of_stack, task_top_of_stack(next_p));
  
  	switch_fpu_finish(next_fpu);
  
  	/* Reload sp0. */
      //update_task_stack(next_p)->load_sp0(task_top_of_stack(task))
  	update_task_stack(next_p);
  
  	switch_to_extra(prev_p, next_p);
  ..................
  	/* Load the Intel cache allocation PQR MSR. */
  	resctrl_sched_in();
  
  	return prev_p;
  }
  ```
  
  load_sp0：将next task的esp0加载到tss中。esp和esp0的区别是前者是用户态栈的esp，后者是内核栈的esp。当从用户态进入内核态（ring0优先级）时，硬件会自动将esp = tss - > esp0。

#### 指令指针的保存与恢复

你是不是觉得，这样真的就完成切换了吗？是的，不信我们来**盘点**一下。

从进程A切换到进程B，用户栈要不要切换呢？当然要，其实早就已经切换了，就在切换内存空间的时候。每个进程的用户栈都是独立的，都在内存空间里面。

那内核栈呢？已经在__switch_to里面切换了，也就是将current_task指向当前的task_struct。里面的void *stack指针，指向的就是当前的内核栈。

内核栈的栈顶指针呢？在__switch_to_asm里面已经切换了栈顶指针，并且将栈顶指针在__switch_to加载到了TSS里面。

用户栈的栈顶指针呢？如果当前在内核里面的话，它当然是在内核栈顶部的pt_regs结构里面呀。当从内核返回用户态运行的时候，pt_regs里面有所有当时在用户态的时候运行的上下文信息，就可以开始运行了。

唯一让人不容易理解的是指令指针寄存器，它应该指向下一条指令的，那它是如何切换的呢？这里有点绕，请你仔细看。

这里我先明确一点，进程的调度都最终会调用到__schedule函数。为了方便你记住，我姑且给它起个名字，就叫“**进程调度第一定律**”。后面我们会多次用到这个定律，你一定要记住。

我们用最前面的例子仔细分析这个过程。本来一个进程A在用户态是要写一个文件的，写文件的操作用户态没办法完成，就要通过系统调用到达内核态。在这个切换的过程中，用户态的指令指针寄存器是保存在pt_regs里面的，到了内核态，就开始沿着写文件的逻辑一步一步执行，结果发现需要等待，于是就调用__schedule函数。

这个时候，进程A在内核态的指令指针是指向__schedule了。这里请记住，A进程的内核栈会保存这个__schedule的调用，而且知道这是从btrfs_wait_for_no_snapshoting_writes这个函数里面进去的。

__schedule里面经过上面的层层调用，到达了context_switch的最后三行指令（其中barrier语句是一个编译器指令，用于保证switch_to和finish_task_switch的执行顺序，不会因为编译阶段优化而改变，这里咱们可以忽略它）。

```
switch_to(prev, next, prev);
barrier();
return finish_task_switch(prev);
```

当进程A在内核里面执行switch_to的时候，内核态的指令指针也是指向这一行的。但是在switch_to里面，将寄存器和栈都切换到成了进程B的，唯一没有变的就是指令指针寄存器。当switch_to返回的时候，指令指针寄存器指向了下一条语句finish_task_switch。

但这个时候的finish_task_switch已经不是进程A的finish_task_switch了，而是进程B的finish_task_switch了。

这样合理吗？你怎么知道进程B当时被切换下去的时候，执行到哪里了？恢复B进程执行的时候一定在这里呢？这时候就要用到咱的“进程调度第一定律”了。

当年B进程被别人切换走的时候，也是调用__schedule，也是调用到switch_to，被切换成为C进程的，所以，B进程当年的下一个指令也是finish_task_switch，这就说明指令指针指到这里是没有错的。

接下来，我们要从finish_task_switch完毕后，返回__schedule的调用了。返回到哪里呢？按照函数返回的原理，当然是从内核栈里面去找，是返回到btrfs_wait_for_no_snapshoting_writes吗？当然不是了，因为btrfs_wait_for_no_snapshoting_writes是在A进程的内核栈里面的，它早就被切换走了，应该从B进程的内核栈里面找。

假设，B就是最前面例子里面调用tap_do_read读网卡的进程。它当年调用__schedule的时候，是从tap_do_read这个函数调用进去的。

当然，B进程的内核栈里面放的是tap_do_read。于是，从__schedule返回之后，当然是接着tap_do_read运行，然后在内核运行完毕后，返回用户态。这个时候，B进程内核栈的pt_regs也保存了用户态的指令指针寄存器，就接着在用户态的下一条指令开始运行就可以了。

假设，我们只有一个CPU，从B切换到C，从C又切换到A。在C切换到A的时候，还是按照“进程调度第一定律”，C进程还是会调用__schedule到达switch_to，在里面切换成为A的内核栈，然后运行finish_task_switch。

这个时候运行的finish_task_switch，才是A进程的finish_task_switch。运行完毕从__schedule返回的时候，从内核栈上才知道，当年是从btrfs_wait_for_no_snapshoting_writes调用进去的，因而应该返回btrfs_wait_for_no_snapshoting_writes继续执行，最后内核执行完毕返回用户态，同样恢复pt_regs，恢复用户态的指令指针寄存器，从用户态接着运行。

到这里你是不是有点理解为什么switch_to有三个参数呢？为啥有两个prev呢？其实我们从定义就可以看到。

```
#define switch_to(prev, next, last)					\
do {									\
	prepare_switch_to(prev, next);					\
									\
	((last) = __switch_to_asm((prev), (next)));			\
} while (0)
```

在上面的例子中，A切换到B的时候，运行到__switch_to_asm这一行的时候，是在A的内核栈上运行的，prev是A，next是B。但是，A执行完__switch_to_asm之后就被切换走了，当C再次切换到A的时候，运行到__switch_to_asm，是从C的内核栈运行的。这个时候，prev是C，next是A，但是__switch_to_asm里面切换成为了A当时的内核栈。

还记得当年的场景“prev是A，next是B”，__switch_to_asm里面return prev的时候，还没return的时候，prev这个变量里面放的还是C，因而它会把C放到返回结果中。但是，一旦return，就会弹出A当时的内核栈。这个时候，prev变量就变成了A，next变量就变成了B。这就还原了当年的场景，好在返回值里面的last还是C。

通过三个变量switch_to(prev = A, next=B, last=C)，A进程就明白了，我当时被切换走的时候，是切换成B，这次切换回来，是从C回来的。

#### 总结

![img](https://static001.geekbang.org/resource/image/9f/64/9f4433e82c78ed5cd4399b4b116a9064.png)

### 抢占式调度



最常见的抢占式调度是通过时钟中断处理函数scheduler_tick().代码如下

```c
//"kernel/sched/core.c"
void scheduler_tick(void)
{
	int cpu = smp_processor_id();
	struct rq *rq = cpu_rq(cpu);
	struct task_struct *curr = rq->curr;
.........
	curr->sched_class->task_tick(rq, curr, 0);
	calc_global_load_tick(rq);
	psi_task_tick(rq);
........
}

//"kernel/sched/fair.c"
//以当前正在运行的进程是普通进程为例
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued)
{
	struct cfs_rq *cfs_rq;
	struct sched_entity *se = &curr->se;

	for_each_sched_entity(se) {
		cfs_rq = cfs_rq_of(se);
		entity_tick(cfs_rq, se, queued);
	}
........
}


static void
entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued)
{

	//Update run-time statistics of the 'current'.
	update_curr(cfs_rq);
	 //Ensure that runnable average is periodically updated.
	update_load_avg(cfs_rq, curr, UPDATE_TG);
	update_cfs_group(curr);
..........
	if (cfs_rq->nr_running > 1)
		check_preempt_tick(cfs_rq, curr); //检查是否是时候被抢占
}


static void
check_preempt_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr)
{
	unsigned long ideal_runtime, delta_exec;
	struct sched_entity *se;
	s64 delta;
	//一个调度周期中,这个进程应该运行的实际时间
	ideal_runtime = sched_slice(cfs_rq, curr);
    //本次调度中,该进程的运行时间
	delta_exec = curr->sum_exec_runtime - curr->prev_sum_exec_runtime;
    //比较后判断是否应该被抢占
	if (delta_exec > ideal_runtime) {
		resched_curr(rq_of(cfs_rq));
		/*
		 * The current task ran long enough, ensure it doesn't get
		 * re-elected due to buddy favours.
		 */
		clear_buddies(cfs_rq, curr);
		return;
	}
.......
	//从红黑树中取出最小的节点
	se = __pick_first_entity(cfs_rq);
	delta = curr->vruntime - se->vruntime;

	if (delta < 0)
		return;
	//比较后判断是否应该被抢占
	if (delta > ideal_runtime)
		resched_curr(rq_of(cfs_rq));
}

//"kernel/sched/core.c"
void resched_curr(struct rq *rq)
{
	struct task_struct *curr = rq->curr;
..........
	if (cpu == smp_processor_id()) {
        //标记当前进程需要被抢占
		set_tsk_need_resched(curr);
		set_preempt_need_resched();
		return;
	}
.........
}

//"include/linux/sched.h"
static inline void set_tsk_need_resched(struct task_struct *tsk)
{
	set_tsk_thread_flag(tsk,TIF_NEED_RESCHED);
}
```

另外一个可能抢占的场景是**当一个进程被唤醒的时候**。

我们前面说过，当一个进程在等待一个I/O的时候，会主动放弃CPU。但是当I/O到来的时候，进程往往会被唤醒。这个时候是一个时机。当被唤醒的进程优先级高于CPU上的当前进程，就会触发抢占。try_to_wake_up()调用ttwu_queue将这个唤醒的任务添加到队列当中。ttwu_queue再调用ttwu_do_activate激活这个任务。ttwu_do_activate调用ttwu_do_wakeup。这里面调用了check_preempt_curr检查是否应该发生抢占。如果应该发生抢占，也不是直接踢走当然进程，而也是将当前进程标记为应该被抢占。

#### 抢占的时机

真正的抢占还需要时机，也就是需要那么一个时刻，让正在运行中的进程有机会调用一下__schedule。

你可以想象，不可能某个进程代码运行着，突然要去调用__schedule，代码里面不可能这么写，所以一定要规划几个时机，这个时机分为用户态和内核态。

##### 用户态

对于用户态的进程来讲，从系统调用中返回的那个时刻，是一个被抢占的时机。

64位的系统调用链路do_syscall_64->syscall_return_slowpath->prepare_exit_to_usermode->exit_to_usermode_loop

```c
static void exit_to_usermode_loop(struct pt_regs *regs, u32 cached_flags)
{
	while (true) {
		/* We have work to do. */
		local_irq_enable();

		if (cached_flags & _TIF_NEED_RESCHED)
			schedule();
....................
	}
}
```

对于用户态的进程来讲，从中断中返回的那个时刻，也是一个被抢占的时机。

```assembly
/*"arch/x86/entry/entry_64.S"*/
common_interrupt:
        ASM_CLAC
        addq    $-0x80, (%rsp) 
        interrupt do_IRQ
ret_from_intr:
        popq    %rsp
        testb   $3, CS(%rsp)
        jz      retint_kernel
/* Interrupt came from user space */
GLOBAL(retint_user)
        mov     %rsp,%rdi
        call    prepare_exit_to_usermode
        TRACE_IRQS_IRETQ
        SWAPGS
        jmp     restore_regs_and_iret
/* Returning to kernel space */
retint_kernel:
#ifdef CONFIG_PREEMPT
        bt      $9, EFLAGS(%rsp)  
        jnc     1f
0:      cmpl    $0, PER_CPU_VAR(__preempt_count)
        jnz     1f
        call    preempt_schedule_irq
        jmp     0b
```

中断处理调用的是do_IRQ函数，中断完毕后分为两种情况，一个是返回用户态，一个是返回内核态。

retint_user会调用prepare_exit_to_usermode，最终调用exit_to_usermode_loop，和上面的逻辑一样，发现有标记则调用schedule()。

##### 内核态

在内核态中,被抢占的时机一般发生在preemept_enable()中.

在内核态的执行中,有的操作时不能被中断的,所以在进行这些操作之前,总是先调用preempt_disable()关闭抢占,当再次打开的时候,就是一次内核态代码被抢占的机会.

```c
//"include/linux/preempt.h"
#define preempt_enable() \
do { \
	barrier(); \
	if (unlikely(preempt_count_dec_and_test())) \
		__preempt_schedule(); \
} while (0)

#define preempt_count_dec_and_test() \
	({ preempt_count_sub(1); should_resched(0); })

static __always_inline bool should_resched(int preempt_offset)
{
	return unlikely(preempt_count() == preempt_offset &&
			tif_need_resched());
}

#define tif_need_resched() test_thread_flag(TIF_NEED_RESCHED)


//"include/asm-generic/preempt.h"
#define __preempt_schedule() preempt_schedule()
//"kernel/sched/core.c"
asmlinkage __visible void __sched notrace preempt_schedule(void)
{
	/*
	 * If there is a non-zero preempt_count or interrupts are disabled,
	 * we do not want to preempt the current task. Just return..
	 */
	if (likely(!preemptible()))
		return;

	preempt_schedule_common();
}

static void __sched notrace preempt_schedule_common(void)
{
	do {
		/*
		 * Because the function tracer can trace preempt_count_sub()
		 * and it also uses preempt_enable/disable_notrace(), if
		 * NEED_RESCHED is set, the preempt_enable_notrace() called
		 * by the function tracer will call this function again and
		 * cause infinite recursion.
		 *
		 * Preemption must be disabled here before the function
		 * tracer can trace. Break up preempt_disable() into two
		 * calls. One to disable preemption without fear of being
		 * traced. The other to still record the preemption latency,
		 * which can also be traced by the function tracer.
		 */
		preempt_disable_notrace();
		preempt_latency_start(1);
		__schedule(true);
		preempt_latency_stop(1);
		preempt_enable_no_resched_notrace();

		/*
		 * Check again in case we missed a preemption opportunity
		 * between schedule and now.
		 */
	} while (need_resched());
}

```

内核态也会遇到终端的情况,当中断返回的时候,返回的仍旧是内核态.这个时候也是一个执行抢占的时机，现在我们再来上面中断返回的代码中返回内核的那部分代码，调用的是preempt_schedule_irq。

#### 总结

好了，抢占式调度就讲到这里了。我这里画了一张脑图，将整个进程的调度体系都放在里面。

这个脑图里面第一条就是总结了进程调度第一定律的核心函数__schedule的执行过程，这是上一节讲的，因为要切换的东西比较多，需要你详细了解每一部分是如何切换的。

第二条总结了标记为可抢占的场景，第三条是所有的抢占发生的时机，这里是真正验证了进程调度第一定律的。

![img](https://static001.geekbang.org/resource/image/93/7f/93588d71abd7f007397979f0ba7def7f.png)