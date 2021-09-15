# 进程的创建

fork是一个系统调用，根据系统调用的流程，流程的最后会在sys_call_table中找到相应的系统调用sys_fork。

sys_fork是根据SYSCALL_DEFINE0这个宏的定义，下面这段代码就定义了sys_fork。

```c
//"kernel/fork.c"
SYSCALL_DEFINE0(fork)
{
#ifdef CONFIG_MMU  //MMU 内存管理单元
	struct kernel_clone_args args = {
		.exit_signal = SIGCHLD,
	};
cc
	return _do_fork(&args);
#else
	/* can not support in nommu mode */
	return -EINVAL;
#endif
}

long _do_fork(struct kernel_clone_args *args)
{
	u64 clone_flags = args->flags;
	struct completion vfork;
	struct pid *pid;
	struct task_struct *p;
	int trace = 0;
	long nr;
..........
    //复制结构
	p = copy_process(NULL, trace, NUMA_NO_NODE, args);
	if (IS_ERR(p))
		return PTR_ERR(p);
	/*
	 * Do this prior waking up the new thread - the thread pointer
	 * might get invalid after that point, if the thread exits quickly.
	 */
	trace_sched_process_fork(current, p);

	pid = get_task_pid(p, PIDTYPE_PID);
	nr = pid_vnr(pid);

	if (clone_flags & CLONE_PARENT_SETTID)
		put_user(nr, args->parent_tid);
..............
    //唤醒新进程
	wake_up_new_task(p);

	/* forking complete and child started to run, tell ptracer */
.........

	put_pid(pid);
	return nr;
}
```

## 复制结构

```c
static __latent_entropy struct task_struct *copy_process(
					struct pid *pid,
					int trace,
					int node,
					struct kernel_clone_args *args)
{
	int pidfd = -1, retval;
	struct task_struct *p;
	struct multiprocess_signals delayed;
	struct file *pidfile = NULL;
	u64 clone_flags = args->flags;
	struct nsproxy *nsp = current->nsproxy;

	//错误处理
    ......
    //延迟处理信号
	......
	p = dup_task_struct(current, node);
	......
    //权限相关
	retval = copy_creds(p, clone_flags);
	//调度相关
	retval = sched_fork(clone_flags, p);
	
    retval = perf_event_init_task(p);
    retval = audit_alloc(p);
    retval = security_task_alloc(p, clone_flags);
    retval = copy_semundo(clone_flags, p);
	//文件相关
	retval = copy_files(clone_flags, p);
	//文件系统相关
	retval = copy_fs(clone_flags, p);
	//信号量相关
	retval = copy_sighand(clone_flags, p);
	retval = copy_signal(clone_flags, p);
	//内存相关
	retval = copy_mm(clone_flags, p);
	//命名空间
	retval = copy_namespaces(clone_flags, p);
    
	retval = copy_io(clone_flags, p);
	retval = copy_thread_tls(clone_flags, args->stack, args->stack_size, p,
				 args->tls);
	
	stackleak_task_init(p);
    ............
	//分配pid 设置tid.group_leader并且建立进程之间的亲缘关系
	p->pid = pid_nr(pid);
	if (clone_flags & CLONE_THREAD) {
		p->exit_signal = -1;
		p->group_leader = current->group_leader;
		p->tgid = current->tgid;
	} else {
		if (clone_flags & CLONE_PARENT)
			p->exit_signal = current->group_leader->exit_signal;
		else
			p->exit_signal = args->exit_signal;
		p->group_leader = p;
		p->tgid = p->pid;
	}

	retval = cgroup_can_fork(p, args);

	p->start_time = ktime_get_ns();
	p->start_boottime = ktime_get_boottime_ns();

	if (clone_flags & (CLONE_PARENT|CLONE_THREAD)) {
		p->real_parent = current->real_parent;
		p->parent_exec_id = current->parent_exec_id;
	} else {
		p->real_parent = current;
		p->parent_exec_id = current->self_exec_id;
	}
}
```

### dup_task_struct

```c
static struct task_struct *dup_task_struct(struct task_struct *orig, int node)
{
	struct task_struct *tsk;
	unsigned long *stack;
......
    //为task_struct分配内存空间
	tsk = alloc_task_struct_node(node);
	//分配内核栈空间
	stack = alloc_thread_stack_node(tsk, node);

	if (memcg_charge_kernel_stack(tsk))
		goto free_stack;

	stack_vm_area = task_stack_vm_area(tsk);
	//进行复制 调用memcpy()
	err = arch_dup_task_struct(tsk, orig);

	tsk->stack = stack;
........
	//设置thread_info
	setup_thread_stack(tsk, orig);
    
	clear_user_return_notifier(tsk);
	clear_tsk_need_resched(tsk);
	set_task_stack_end_magic(tsk);

	if (orig->cpus_ptr == &orig->cpus_mask)
		tsk->cpus_ptr = &tsk->cpus_mask;

	return tsk;
}
```

### copy_cerd

```c
int copy_creds(struct task_struct *p, unsigned long clone_flags)
{
	struct cred *new;
	int ret;
........
	//准备一个新的struct *cerd *new 即从内村中分配一个新的,然后调用memcpy复制父进程的
	new = prepare_creds();
..........

	atomic_inc(&new->user->processes);
    //设置cred和real_cred
	p->cred = p->real_cred = get_cred(new);
.........
	return 0;

error_put:
	put_cred(new);
	return ret;
}

```

### sched_fork

```c
int sched_fork(unsigned long clone_flags, struct task_struct *p)
{
	unsigned long flags;
	//将on_rq设为0,初始化sched_entity,将里面的exec_start、sum_exec_runtime、				prev_sum_exec_runtime、vruntime都设为0。
	__sched_fork(clone_flags, p);
	//设置进程的状态
	p->state = TASK_NEW;
	//初始化进程的优先级
	p->prio = current->normal_prio;
	
	uclamp_fork(p);
	//根据sched_reset_on_fork判断是否需要重置调度策略和优先级
	if (unlikely(p->sched_reset_on_fork)) {
		if (task_has_dl_policy(p) || task_has_rt_policy(p)) {
			p->policy = SCHED_NORMAL;
			p->static_prio = NICE_TO_PRIO(0);
			p->rt_priority = 0;
		} else if (PRIO_TO_NICE(p->static_prio) < 0)
			p->static_prio = NICE_TO_PRIO(0);

		p->prio = p->normal_prio = __normal_prio(p);
		set_load_weight(p, false);

		p->sched_reset_on_fork = 0;
	}
	//设置调度类
	if (dl_prio(p->prio))
		return -EAGAIN;
	else if (rt_prio(p->prio))
		p->sched_class = &rt_sched_class;
	else
		p->sched_class = &fair_sched_class;
	
	init_entity_runnable_average(&p->se);

	/*
	 * The child is not yet in the pid-hash so no cgroup attach races,
	 * and the cgroup is pinned to this child due to cgroup_fork()
	 * is ran before sched_fork().
	 *
	 * Silence PROVE_RCU.
	 */
	raw_spin_lock_irqsave(&p->pi_lock, flags);
	rseq_migrate(p);
	/*
	 * We're setting the CPU for the first time, we don't migrate,
	 * so use __set_task_cpu().
	 */
	__set_task_cpu(p, smp_processor_id());
    //调用task_forf_fair()其中调用update_curr 对当前进程进行统计量更新,调用place_entity,初始化sched_entity的vruntime.其中可以根据sysctl_sched_child_runs_first设置父进程和子进程谁先运行.
    //如果设置了子进程先运行，即便两个进程的vruntime一样，也要把子进程的sched_entity放在前面，然后调用resched_curr，标记当前运行的进程TIF_NEED_RESCHED，也就是说，把父进程设置为应该被调度，这样下次调度的时候，父进程会被子进程抢占。
	if (p->sched_class->task_fork)
		p->sched_class->task_fork(p);
	raw_spin_unlock_irqrestore(&p->pi_lock, flags);

#ifdef CONFIG_SCHED_INFO
	if (likely(sched_info_on()))
		memset(&p->sched_info, 0, sizeof(p->sched_info));
#endif
#if defined(CONFIG_SMP)
	p->on_cpu = 0;
#endif
	init_task_preempt_count(p);
#ifdef CONFIG_SMP
	plist_node_init(&p->pushable_tasks, MAX_PRIO);
	RB_CLEAR_NODE(&p->pushable_dl_tasks);
#endif
	return 0;
}
```

### copy_files&copy_fs

* copy_files 

  主要用于复制一个进程打开的文件信息.这些信息用files_struct维护,每个打开的文件都有一个文件描述符.

  ```c
  static int copy_files(unsigned long clone_flags, struct task_struct *tsk)
  {
  	struct files_struct *oldf, *newf;
  	int error = 0;
  
  	/*
  	 * A background process may not have any files ...
  	 */
  	oldf = current->files;
  	if (!oldf)
  		goto out;
  
  	if (clone_flags & CLONE_FILES) {
  		atomic_inc(&oldf->count);
  		goto out;
  	}
  
  	newf = dup_fd(oldf, &error);
  	if (!newf)
  		goto out;
  
  	tsk->files = newf;
  	error = 0;
  out:
  	return error;
  }
  ```

* copy_fs

  主要用于复制一个进程的目录信息.这些信息通过fs_struct结构体来维护,一个进程有自己的根目录和根文件系统root,也有当前目录pwd和当前目录的文件系统,都在fs_struct中维护.copy_fs函数里面调用copy_fs_struct,创建一个新的fs_struct,并复制原来的fs_struct.

  ```c
  static int copy_fs(unsigned long clone_flags, struct task_struct *tsk)
  {
  	struct fs_struct *fs = current->fs;
  	if (clone_flags & CLONE_FS) {
  		/* tsk->fs is already what we want */
  		spin_lock(&fs->lock);
  		if (fs->in_exec) {
  			spin_unlock(&fs->lock);
  			return -EAGAIN;
  		}
  		fs->users++;
  		spin_unlock(&fs->lock);
  		return 0;
  	}
  	tsk->fs = copy_fs_struct(fs);
  	if (!tsk->fs)
  		return -ENOMEM;
  	return 0;
  }
  
  struct fs_struct *copy_fs_struct(struct fs_struct *old)
  {
  	struct fs_struct *fs = kmem_cache_alloc(fs_cachep, GFP_KERNEL);
  	/* We don't need to lock fs - think why ;-) */
  	if (fs) {
  		fs->users = 1;
  		fs->in_exec = 0;
  		spin_lock_init(&fs->lock);
  		seqcount_init(&fs->seq);
  		fs->umask = old->umask;
  
  		spin_lock(&old->lock);
  		fs->root = old->root;
  		path_get(&fs->root);
  		fs->pwd = old->pwd;
  		path_get(&fs->pwd);
  		spin_unlock(&old->lock);
  	}
  	return fs;
  }
  ```

  

### copy_sighand&copy_signal

* copy_sighand

  copy_sighand会分配一个新的sighand_struct。这里最主要的是维护信号处理函数，在copy_sighand里面会调用memcpy，将信号处理函数sighand->action从父进程复制到子进程。

* copy_signal

  copy_signal函数会分配一个新的signal_struct，并进行初始化。

### copy_mm

进程都自己的内存空间，用mm_struct结构来表示。copy_mm函数中调用dup_mm，分配一个新的mm_struct结构，调用memcpy复制这个结构。dup_mmap用于复制内存空间中内存映射的部分。

## 唤醒新进程

```c
//"kernel/sched/core.c"
void wake_up_new_task(struct task_struct *p)
{
	struct rq_flags rf;
	struct rq *rq;

	raw_spin_lock_irqsave(&p->pi_lock, rf.flags);
    //设置进程状态为可运行
	p->state = TASK_RUNNING;
.......
	activate_task(rq, p, ENQUEUE_NOCLOCK);
    trace_sched_wakeup_new(p);
    //查看能否抢占当前进程
	check_preempt_curr(rq, p, WF_FORK);
........
	task_rq_unlock(rq, p, &rf);
}
```

* activate_task

  ```c
  void activate_task(struct rq *rq, struct task_struct *p, int flags)
  {
  	if (task_contributes_to_load(p))
  		rq->nr_uninterruptible--;
  	
  	enqueue_task(rq, p, flags);
  
  	p->on_rq = TASK_ON_RQ_QUEUED;
  }
  
  static inline void enqueue_task(struct rq *rq, struct task_struct *p, int flags)
  {
  	if (!(flags & ENQUEUE_NOCLOCK))
  		update_rq_clock(rq);
  
  	if (!(flags & ENQUEUE_RESTORE)) {
  		sched_info_queued(rq, p);
  		psi_enqueue(p, flags & ENQUEUE_WAKEUP);
  	}
  
  	uclamp_rq_inc(rq, p);
  	p->sched_class->enqueue_task(rq, p, flags);
  }
  
  static void
  enqueue_task_fair(struct rq *rq, struct task_struct *p, int flags)
  {
  	struct cfs_rq *cfs_rq;
  	struct sched_entity *se = &p->se;
  	int idle_h_nr_running = task_has_idle_policy(p);
  ..........
  	for_each_sched_entity(se) {
  		if (se->on_rq)
  			break;
  		cfs_rq = cfs_rq_of(se);
      	//在enqueue_entity函数里面，会调用update_curr，更新运行的统计量，然后调用__enqueue_entity，将sched_entity加入到红黑树里面，然后将se->on_rq = 1设置在队列上。
  		enqueue_entity(cfs_rq, se, flags);
  		//表示队列中的进程数
  		cfs_rq->h_nr_running++;
  		cfs_rq->idle_h_nr_running += idle_h_nr_running;
      
  		flags = ENQUEUE_WAKEUP;
  	}
  ...........
  	assert_list_leaf_cfs_rq(rq);
  
  	hrtick_update(rq);
  }
  ```

* check_preempt_curr

  ```c
  void check_preempt_curr(struct rq *rq, struct task_struct *p, int flags)
  {
  	const struct sched_class *class;
  
  	if (p->sched_class == rq->curr->sched_class) {
  		rq->curr->sched_class->check_preempt_curr(rq, p, flags);
  	} else {
  		for_each_class(class) {
  			if (class == rq->curr->sched_class)
  				break;
  			if (class == p->sched_class) {
  				resched_curr(rq);
  				break;
  			}
  		}
  	}
  ........
  }
  
  static void check_preempt_wakeup(struct rq *rq, struct task_struct *p, int wake_flags)
  {
  	struct task_struct *curr = rq->curr;
  	struct sched_entity *se = &curr->se, *pse = &p->se;
  	struct cfs_rq *cfs_rq = task_cfs_rq(curr);
  ...........
  	//前面调用task_fork_fair的时候，如果设置sysctl_sched_child_runs_first了，那么当前父进程的TIF_NEED_RESCHED也会被设置，则直接返回。
  	if (test_tsk_need_resched(curr))
  		return;
  
  	/* Idle tasks are by definition preempted by non-idle tasks. */
  	if (unlikely(task_has_idle_policy(curr)) &&
  	    likely(!task_has_idle_policy(p)))
  		goto preempt;
  
  	/*
  	 * Batch and idle tasks do not preempt non-idle tasks (their preemption
  	 * is driven by the tick):
  	 */
  	if (unlikely(p->policy != SCHED_NORMAL) || !sched_feat(WAKEUP_PREEMPTION))
  		return;
  
  	find_matching_se(&se, &pse);
      //更新当前的统计量
  	update_curr(cfs_rq_of(se));
  	BUG_ON(!pse);
      //通过比较判断是头需要抢占
  	if (wakeup_preempt_entity(se, pse) == 1) {
  		if (!next_buddy_marked)
  			set_next_buddy(pse);
  		goto preempt;
  	}
  
  	return;
  
  preempt:
      //设置为可被抢占
  	resched_curr(rq);
  ..........
  }
  
  static inline int test_tsk_need_resched(struct task_struct *tsk)
  {
  	return unlikely(test_tsk_thread_flag(tsk,TIF_NEED_RESCHED));
  }
  ```

  如果新创建的进程应该抢占父进程,fork是一个系统调用，从系统调用返回的时候，是抢占的一个好时机，如果父进程判断自己已经被设置为TIF_NEED_RESCHED，就让子进程先跑，抢占自己。



## 总结

![img](https://static001.geekbang.org/resource/image/9d/58/9d9c5779436da40cabf8e8599eb85558.jpeg)