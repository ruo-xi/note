# 线程的创建

线程不是一个完全由内核实现的机制，它是由内核态和用户态合作完成的.

pthread_create不是一个系统调用，是Glibc库的一个函数，所以我们还要去Glibc里面去找线索。

glibc源码下载方法

* `git clone https://mirrors.tuna.tsinghua.edu.cn/git/glibc.git`
* [glibc源码下载地址](http://ftp.gnu.org/gnu/libc/)

```c
int
__pthread_create_2_1 (pthread_t *newthread, const pthread_attr_t *attr,
		      void *(*start_routine) (void *), void *arg)
{
  STACK_VARIABLES; //void *stackaddr = NULL

  /* Avoid a data race in the multi-threaded case.  */
  if (__libc_single_threaded)
    __libc_single_threaded = 0;
  //如何传入的attr为空则采用默认的attr
  const struct pthread_attr *iattr = (struct pthread_attr *) attr;
  union pthread_attr_transparent default_attr;
  bool destroy_default_attr = false;
  bool c11 = (attr == ATTR_C11_THREAD);
  if (iattr == NULL || c11)
    {
      int ret = __pthread_getattr_default_np (&default_attr.external);
      if (ret != 0)
	return ret;
      destroy_default_attr = true;
      iattr = &default_attr.internal;
    }
  //线程的结构
  struct pthread *pd = NULL;
  //创建线程的栈结构
  int err = ALLOCATE_STACK (iattr, &pd);
  int retval = 0;

  if (__glibc_unlikely (err != 0))
    /* Something went wrong.  Maybe a parameter of the attributes is
       invalid or we could not allocate memory.  Note we have to
       translate error codes.  */
    {
      retval = err == ENOMEM ? EAGAIN : err;
      goto out;
    }


  /* Initialize the TCB.  All initializations with zero should be
     performed in 'get_cached_stack'.  This way we avoid doing this if
     the stack freshly allocated with 'mmap'.  */

#if TLS_TCB_AT_TP
  /* Reference to the TCB itself.  */
  pd->header.self = pd;

  /* Self-reference for TLS.  */
  pd->header.tcb = pd;
#endif
    //线程的函数,参数
  pd->start_routine = start_routine;
  pd->arg = arg;
    //是否是c11thread
  pd->c11 = c11;

  /* Copy the thread attribute flags.  */
  struct pthread *self = THREAD_SELF;
  pd->flags = ((iattr->flags & ~(ATTR_FLAG_SCHED_SET | ATTR_FLAG_POLICY_SET))
	       | (self->flags & (ATTR_FLAG_SCHED_SET | ATTR_FLAG_POLICY_SET)));

  /* Initialize the field for the ID of the thread which is waiting
     for us.  This is a self-reference in case the thread is created
     detached.  */
  pd->joinid = iattr->flags & ATTR_FLAG_DETACHSTATE ? pd : NULL;

  /* The debug events are inherited from the parent.  */
  pd->eventbuf = self->eventbuf;


	//线程的调度策略和调度参数
  pd->schedpolicy = self->schedpolicy;
  pd->schedparam = self->schedparam;

  //设置tcb head
  tls_setup_tcbhead (pd);

  /* Determine scheduling parameters for the thread.  */
  if (__builtin_expect ((iattr->flags & ATTR_FLAG_NOTINHERITSCHED) != 0, 0)
      && (iattr->flags & (ATTR_FLAG_SCHED_SET | ATTR_FLAG_POLICY_SET)) != 0)
    {
      /* Use the scheduling parameters the user provided.  */
      if (iattr->flags & ATTR_FLAG_POLICY_SET)
        {
          pd->schedpolicy = iattr->schedpolicy;
          pd->flags |= ATTR_FLAG_POLICY_SET;
        }
      if (iattr->flags & ATTR_FLAG_SCHED_SET)
        {
          /* The values were validated in pthread_attr_setschedparam.  */
          pd->schedparam = iattr->schedparam;
          pd->flags |= ATTR_FLAG_SCHED_SET;
        }

      if ((pd->flags & (ATTR_FLAG_SCHED_SET | ATTR_FLAG_POLICY_SET))
          != (ATTR_FLAG_SCHED_SET | ATTR_FLAG_POLICY_SET))
        collect_default_sched (pd);
    }

  if (__glibc_unlikely (__nptl_nthreads == 1))
    _IO_enable_locks ();

  /* Pass the descriptor to the caller.  */
  *newthread = (pthread_t) pd;

  LIBC_PROBE (pthread_create, 4, newthread, attr, start_routine, arg);
    
//线程数加一
  atomic_increment (&__nptl_nthreads);
..........
	//创建线程
    retval = create_thread (pd, iattr, &stopped_start,
			    STACK_VARIABLES_ARGS, &thread_ran);

  /* Return to the previous signal mask, after creating the new
     thread.  */
  __libc_signal_restore_set (&original_sigmask);

  if (__glibc_unlikely (retval != 0))
    {
      if (thread_ran)
	/* State (c) or (d) and we may not have PD ownership (see
	   CONCURRENCY NOTES above).  We can assert that STOPPED_START
	   must have been true because thread creation didn't fail, but
	   thread attribute setting did.  */
	/* See bug 19511 which explains why doing nothing here is a
	   resource leak for a joinable thread.  */
	assert (stopped_start);
      else
	{
	  /* State (e) and we have ownership of PD (see CONCURRENCY
	     NOTES above).  */

	  /* Oops, we lied for a second.  */
	  atomic_decrement (&__nptl_nthreads);

	  /* Perhaps a thread wants to change the IDs and is waiting for this
	     stillborn thread.  */
	  if (__glibc_unlikely (atomic_exchange_acq (&pd->setxid_futex, 0)
				== -2))
	    futex_wake (&pd->setxid_futex, 1, FUTEX_PRIVATE);

	  /* Free the resources.  */
	  __deallocate_stack (pd);
	}

      /* We have to translate error codes.  */
      if (retval == ENOMEM)
	retval = EAGAIN;
    }
  else
    {
      /* We don't know if we have PD ownership.  Once we check the local
         stopped_start we'll know if we're in state (a) or (b) (see
	 CONCURRENCY NOTES above).  */
      if (stopped_start)
	/* State (a), we own PD. The thread blocked on this lock either
	   because we're doing TD_CREATE event reporting, or for some
	   other reason that create_thread chose.  Now let it run
	   free.  */
	lll_unlock (pd->lock, LLL_PRIVATE);

      /* We now have for sure more than one thread.  The main thread might
	 not yet have the flag set.  No need to set the global variable
	 again if this is what we use.  */
      THREAD_SETMEM (THREAD_SELF, header.multiple_threads, 1);
    }

 out:
  if (destroy_default_attr)
    __pthread_attr_destroy (&default_attr.external);

  return retval;
}

versioned_symbol (libpthread, __pthread_create_2_1, pthread_create, GLIBC_2_1);
```

### ALLOCATE_STACK

```c
//"nptl/allocatestack.c"
# define ALLOCATE_STACK(attr, pd) allocate_stack (attr, pd, &stackaddr)

static int
allocate_stack (const struct pthread_attr *attr, struct pthread **pdp,
		ALLOCATE_STACK_PARMS)
{
  struct pthread *pd;
  size_t size;
  size_t pagesize_m1 = __getpagesize () - 1;
......
  //根据attr设置栈的大小,若没有取默认值
  if (attr->stacksize != 0)
    size = attr->stacksize;
  else
    {
      lll_lock (__default_pthread_attr_lock, LLL_PRIVATE);
      size = __default_pthread_attr.internal.stacksize;
      lll_unlock (__default_pthread_attr_lock, LLL_PRIVATE);
    }
    
  //如果用户制定了栈的地址,根据attr获取栈的地址,并且确保栈生长的空间size	
  //线程栈也是自顶向下生长的，还记得每个线程要有一个pthread结构，这个结构也是放在栈的空间里面的。在栈底的位置，其实是地址最高位；
  if (__glibc_unlikely (attr->flags & ATTR_FLAG_STACKADDR))
    {
      uintptr_t adj;
      char *stackaddr = (char *) attr->stackaddr;
      if (_STACK_GROWS_UP)
	stackaddr += attr->stacksize;
        //假如用户指定了栈的大小,要确保足够大 >__static_tls_size + MINIMAL_REST_STACK(2048)
      if (attr->stacksize != 0
	  && attr->stacksize < (__static_tls_size + MINIMAL_REST_STACK))
	return EINVAL;
  		//调整栈的大小来校准TLS block
#if TLS_TCB_AT_TP
      adj = ((uintptr_t) stackaddr - TLS_TCB_SIZE)
	    & __static_tls_align_m1;
      assert (size > adj + TLS_TCB_SIZE);
#elif TLS_DTV_AT_TP
      adj = ((uintptr_t) stackaddr - __static_tls_size)
	    & __static_tls_align_m1;
      assert (size > adj);
#endif
//用户需要自行指定guard pages
#if TLS_TCB_AT_TP
      pd = (struct pthread *) ((uintptr_t) stackaddr
			       - TLS_TCB_SIZE - adj);
#elif TLS_DTV_AT_TP
      pd = (struct pthread *) (((uintptr_t) stackaddr
				- __static_tls_size - adj)
			       - TLS_PRE_TCB_SIZE);
#endif

      //格式化用户提供的内存
      memset (pd, '\0', sizeof (struct pthread));

      /* The first TSD block is included in the TCB.  */
      pd->specific[0] = pd->specific_1stblock;

      /* Remember the stack-related values.  */
      pd->stackblock = (char *) stackaddr - size;
      pd->stackblock_size = size;

		//这是用户提供的堆栈。它不会在堆栈缓存中排队，也不会释放内存（TLS内存除外）
      pd->user_stack = true;
.......//todo
    }
  else
    {
      /* Allocate some anonymous memory.  If possible use the cache.  */
      //为了防止栈的访问越界，在栈的末尾会有一块空间guardsize，一旦访问到这里就错误了
      size_t guardsize;
      size_t reqsize;
      void *mem;
      const int prot = (PROT_READ | PROT_WRITE
			| ((GL(dl_stack_flags) & PF_X) ? PROT_EXEC : 0));

      /* Adjust the stack size for alignment.  */
      size &= ~__static_tls_align_m1;
      assert (size != 0);

      /* Make sure the size of the stack is enough for the guard and
	 eventually the thread descriptor.  */
      guardsize = (attr->guardsize + pagesize_m1) & ~pagesize_m1;
      if (guardsize < attr->guardsize || size + guardsize < guardsize)
	/* Arithmetic overflow.  */
	return EINVAL;
      size += guardsize;
      if (__builtin_expect (size < ((guardsize + __static_tls_size
				     + MINIMAL_REST_STACK + pagesize_m1)
				    & ~pagesize_m1),
			    0))
	/* The stack is too small (or the guard too large).  */
	return EINVAL;
		
      //其实线程栈是在进程的堆里面创建的。如果一个进程不断地创建和删除线程，我们不可能不断地去申请和清除线程栈使用的内存块，这样就需要有一个缓存。get_cached_stack就是根据计算出来的size大小，看一看已经有的缓存中，有没有已经能够满足条件的；
      reqsize = size;
      pd = get_cached_stack (&size, &mem);
      //如果缓存里面没有，就需要调用__mmap创建一块新的
      if (pd == NULL)
	{
	  /* To avoid aliasing effects on a larger scale than pages we
	     adjust the allocated stack size if necessary.  This way
	     allocations directly following each other will not have
	     aliasing problems.  */
#if MULTI_PAGE_ALIASING != 0
	  if ((size % MULTI_PAGE_ALIASING) == 0)
	    size += pagesize_m1 + 1;
#endif

	  /* If a guard page is required, avoid committing memory by first
	     allocate with PROT_NONE and then reserve with required permission
	     excluding the guard page.  */
	  mem = __mmap (NULL, size, (guardsize == 0) ? prot : PROT_NONE,
			MAP_PRIVATE | MAP_ANONYMOUS | MAP_STACK, -1, 0);

	  if (__glibc_unlikely (mem == MAP_FAILED))
	    return errno;

	  /* SIZE is guaranteed to be greater than zero.
	     So we can never get a null pointer back from mmap.  */
	  assert (mem != NULL);

	  /* Place the thread descriptor at the end of the stack.  */
#if TLS_TCB_AT_TP
	  pd = (struct pthread *) ((((uintptr_t) mem + size)
				    - TLS_TCB_SIZE)
				   & ~__static_tls_align_m1);
#elif TLS_DTV_AT_TP
	  pd = (struct pthread *) ((((uintptr_t) mem + size
				    - __static_tls_size)
				    & ~__static_tls_align_m1)
				   - TLS_PRE_TCB_SIZE);
#endif

	  //计算出guard内存的位置，调用setup_stack_prot设置这块内存的是受保护的；
	  if (__glibc_likely (guardsize > 0))
	    {
	      char *guard = guard_position (mem, size, guardsize, pd,
					    pagesize_m1);
	      if (setup_stack_prot (mem, size, guard, guardsize, prot) != 0)
		{
		  __munmap (mem, size);
		  return errno;
		}
	    }


	  pd->stackblock = mem;
	  pd->stackblock_size = size;
	  pd->guardsize = guardsize;
	  pd->specific[0] = pd->specific_1stblock;

	  /* This is at least the second thread.  */
	  pd->header.multiple_threads = 1;
#ifndef TLS_MULTIPLE_THREADS_IN_TCB
	  __pthread_multiple_threads = *__libc_multiple_threads_ptr = 1;
#endif

#ifdef NEED_DL_SYSINFO
	  SETUP_THREAD_SYSINFO (pd);
#endif

	  /* Don't allow setxid until cloned.  */
	  pd->setxid_futex = -1;

	  /* Allocate the DTV for this thread.  */
	  if (_dl_allocate_tls (TLS_TPADJ (pd)) == NULL)
	    {
	      /* Something went wrong.  */
	      assert (errno == ENOMEM);

	      /* Free the stack memory we just allocated.  */
	      (void) __munmap (mem, size);

	      return errno;
	    }


	  /* Prepare to modify global data.  */
	  lll_lock (stack_cache_lock, LLL_PRIVATE);

	  //将这个线程栈放到stack_used链表中，其实管理线程栈总共有两个链表，一个是stack_used，也就是这个栈正被使用；另一个是stack_cache，就是上面说的，一旦线程结束，先缓存起来，不释放，等有其他的线程创建的时候，给其他的线程用
	  stack_list_add (&pd->list, &stack_used);

	  lll_unlock (stack_cache_lock, LLL_PRIVATE);


	  /* There might have been a race.  Another thread might have
	     caused the stacks to get exec permission while this new
	     stack was prepared.  Detect if this was possible and
	     change the permission if necessary.  */
	  if (__builtin_expect ((GL(dl_stack_flags) & PF_X) != 0
				&& (prot & PROT_EXEC) == 0, 0))
	    {
	      int err = change_stack_perm (pd
#ifdef NEED_SEPARATE_REGISTER_STACK
					   , ~pagesize_m1
#endif
					   );
	      if (err != 0)
		{
		  /* Free the stack memory we just allocated.  */
		  (void) __munmap (mem, size);

		  return err;
		}
	    }


	  /* Note that all of the stack and the thread descriptor is
	     zeroed.  This means we do not have to initialize fields
	     with initial value zero.  This is specifically true for
	     the 'tid' field which is always set back to zero once the
	     stack is not used anymore and for the 'guardsize' field
	     which will be read next.  */
	}

      /* Create or resize the guard area if necessary.  */
      if (__glibc_unlikely (guardsize > pd->guardsize))
	{
	  char *guard = guard_position (mem, size, guardsize, pd,
					pagesize_m1);
	  if (__mprotect (guard, guardsize, PROT_NONE) != 0)
	    {
	    mprot_error:
	      lll_lock (stack_cache_lock, LLL_PRIVATE);

	      /* Remove the thread from the list.  */
	      stack_list_del (&pd->list);

	      lll_unlock (stack_cache_lock, LLL_PRIVATE);

	      /* Get rid of the TLS block we allocated.  */
	      _dl_deallocate_tls (TLS_TPADJ (pd), false);

	      /* Free the stack memory regardless of whether the size
		 of the cache is over the limit or not.  If this piece
		 of memory caused problems we better do not use it
		 anymore.  Uh, and we ignore possible errors.  There
		 is nothing we could do.  */
	      (void) __munmap (mem, size);

	      return errno;
	    }

	  pd->guardsize = guardsize;
	}
      else if (__builtin_expect (pd->guardsize - guardsize > size - reqsize,
				 0))
	{
	  /* The old guard area is too large.  */

#ifdef NEED_SEPARATE_REGISTER_STACK
	  char *guard = mem + (((size - guardsize) / 2) & ~pagesize_m1);
	  char *oldguard = mem + (((size - pd->guardsize) / 2) & ~pagesize_m1);

	  if (oldguard < guard
	      && __mprotect (oldguard, guard - oldguard, prot) != 0)
	    goto mprot_error;

	  if (__mprotect (guard + guardsize,
			oldguard + pd->guardsize - guard - guardsize,
			prot) != 0)
	    goto mprot_error;
#elif _STACK_GROWS_DOWN
	  if (__mprotect ((char *) mem + guardsize, pd->guardsize - guardsize,
			prot) != 0)
	    goto mprot_error;
#elif _STACK_GROWS_UP
         char *new_guard = (char *)(((uintptr_t) pd - guardsize)
                                    & ~pagesize_m1);
         char *old_guard = (char *)(((uintptr_t) pd - pd->guardsize)
                                    & ~pagesize_m1);
         /* The guard size difference might be > 0, but once rounded
            to the nearest page the size difference might be zero.  */
         if (new_guard > old_guard
             && __mprotect (old_guard, new_guard - old_guard, prot) != 0)
	    goto mprot_error;
#endif

	  pd->guardsize = guardsize;
	}
      /* The pthread_getattr_np() calls need to get passed the size
	 requested in the attribute, regardless of how large the
	 actually used guardsize is.  */
      pd->reported_guardsize = guardsize;
    }

  /* Initialize the lock.  We have to do this unconditionally since the
     stillborn thread could be canceled while the lock is taken.  */
  pd->lock = LLL_LOCK_INITIALIZER;

  /* The robust mutex lists also need to be initialized
     unconditionally because the cleanup for the previous stack owner
     might have happened in the kernel.  */
  pd->robust_head.futex_offset = (offsetof (pthread_mutex_t, __data.__lock)
				  - offsetof (pthread_mutex_t,
					      __data.__list.__next));
  pd->robust_head.list_op_pending = NULL;
#if __PTHREAD_MUTEX_HAVE_PREV
  pd->robust_prev = &pd->robust_head;
#endif
  pd->robust_head.list = &pd->robust_head;

  /* We place the thread descriptor at the end of the stack.  */
  *pdp = pd;

#if _STACK_GROWS_DOWN
  void *stacktop;

# if TLS_TCB_AT_TP
  /* The stack begins before the TCB and the static TLS block.  */
  stacktop = ((char *) (pd + 1) - __static_tls_size);
# elif TLS_DTV_AT_TP
  stacktop = (char *) (pd - 1);
# endif

# ifdef NEED_SEPARATE_REGISTER_STACK
  *stack = pd->stackblock;
  *stacksize = stacktop - *stack;
# else
  *stack = stacktop;
# endif
#else
  *stack = pd->stackblock;
#endif

  return 0;
}
```

### create_thread



```c
//"sysdeps/unix/sysv/linux/createthread.c"
static int start_thread (void *arg) __attribute__ ((noreturn));

static int
create_thread (struct pthread *pd, const struct pthread_attr *attr,
	       bool *stopped_start, STACK_VARIABLES_PARMS, bool *thread_ran)
{
.........
  /* We rely heavily on various flags the CLONE function understands:

     CLONE_VM, CLONE_FS, CLONE_FILES
	These flags select semantics with shared address space and
	file descriptors according to what POSIX requires.

     CLONE_SIGHAND, CLONE_THREAD
	This flag selects the POSIX signal semantics and various
	other kinds of sharing (itimers, POSIX timers, etc.).

     CLONE_SETTLS
	The sixth parameter to CLONE determines the TLS area for the
	new thread.

     CLONE_PARENT_SETTID
	The kernels writes the thread ID of the newly created thread
	into the location pointed to by the fifth parameters to CLONE.

	Note that it would be semantically equivalent to use
	CLONE_CHILD_SETTID but it is be more expensive in the kernel.

     CLONE_CHILD_CLEARTID
	The kernels clears the thread ID of a thread that has called
	sys_exit() in the location pointed to by the seventh parameter
	to CLONE.

     The termination signal is chosen to be zero which means no signal
     is sent.  */
  const int clone_flags = (CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SYSVSEM
			   | CLONE_SIGHAND | CLONE_THREAD
			   | CLONE_SETTLS | CLONE_PARENT_SETTID
			   | CLONE_CHILD_CLEARTID
			   | 0);

  TLS_DEFINE_INIT_TP (tp, pd);

  if (__glibc_unlikely (ARCH_CLONE (&start_thread, STACK_VARIABLES_ARGS,
				    clone_flags, pd, &pd->tid, tp, &pd->tid)
			== -1))
    return errno;

  /* It's started now, so if we fail below, we'll have to cancel it
     and let it clean itself up.  */
  *thread_ran = true;

  /* Now we have the possibility to set scheduling parameters etc.  */
  if (attr != NULL)
    {
      int res;

      /* Set the affinity mask if necessary.  */
      if (need_setaffinity)
	{
	  assert (*stopped_start);

	  res = INTERNAL_SYSCALL_CALL (sched_setaffinity, pd->tid,
				       attr->extension->cpusetsize,
				       attr->extension->cpuset);

	  if (__glibc_unlikely (INTERNAL_SYSCALL_ERROR_P (res)))
	  err_out:
	    {
	      /* The operation failed.  We have to kill the thread.
		 We let the normal cancellation mechanism do the work.  */

	      pid_t pid = __getpid ();
	      INTERNAL_SYSCALL_CALL (tgkill, pid, pd->tid, SIGCANCEL);

	      return INTERNAL_SYSCALL_ERRNO (res);
	    }
	}

      /* Set the scheduling parameters.  */
      if ((attr->flags & ATTR_FLAG_NOTINHERITSCHED) != 0)
	{
	  assert (*stopped_start);

	  res = INTERNAL_SYSCALL_CALL (sched_setscheduler, pd->tid,
				       pd->schedpolicy, &pd->schedparam);

	  if (__glibc_unlikely (INTERNAL_SYSCALL_ERROR_P (res)))
	    goto err_out;
	}
    }

  return 0;
}
```

```assembly
# define ARCH_CLONE __clone
    
/*"sysdeps/unix/sysv/linux/x86_64/clone.S"*/

/* The userland implementation is:
   int clone (int (*fn)(void *arg), void *child_stack, int flags, void *arg),
   the kernel entry is:
   int clone (long flags, void *child_stack).

   The parameters are passed in register and on the stack from userland:
   rdi: fn
   rsi: child_stack
   rdx:	flags
   rcx: arg
   r8d:	TID field in parent
   r9d: thread pointer
%esp+8:	TID field in child

   The kernel expects:
   rax: system call number
   rdi: flags
   rsi: child_stack
   rdx: TID field in parent
   r10: TID field in child
   r8:	thread pointer  */


        .text
ENTRY (__clone)
	/* Sanity check arguments.  */
	movq	$-EINVAL,%rax
	testq	%rdi,%rdi		/* no NULL function pointers */
	jz	SYSCALL_ERROR_LABEL
	testq	%rsi,%rsi		/* no NULL stack pointers */
	jz	SYSCALL_ERROR_LABEL

	/* Insert the argument onto the new stack.  */
	subq	$16,%rsi
	movq	%rcx,8(%rsi)

	/* Save the function pointer.  It will be popped off in the
	   child in the ebx frobbing below.  */
	movq	%rdi,0(%rsi)

	/* Do the system call.  */
	movq	%rdx, %rdi
	movq	%r8, %rdx
	movq	%r9, %r8
	mov	8(%rsp), %R10_LP
	movl	$SYS_ify(clone),%eax

	/* End FDE now, because in the child the unwind info will be
	   wrong.  */
	cfi_endproc;
	syscall

	testq	%rax,%rax
	jl	SYSCALL_ERROR_LABEL
	jz	L(thread_start)

	ret

L(thread_start):
	cfi_startproc;
	/* Clearing frame pointer is insufficient, use CFI.  */
	cfi_undefined (rip);
	/* Clear the frame pointer.  The ABI suggests this be done, to mark
	   the outermost frame obviously.  */
	xorl	%ebp, %ebp

	/* Set up arguments for the function call.  */
	popq	%rax		/* Function to call.  */
	popq	%rdi		/* Argument.  */
	call	*%rax
	/* Call exit with return value from function call. */
	movq	%rax, %rdi
	movl	$SYS_ify(exit), %eax
	syscall
	cfi_endproc;

	cfi_startproc;
PSEUDO_END (__clone)

libc_hidden_def (__clone)
weak_alias (__clone, clone)
```

```c
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
		 int __user *, parent_tidptr,
		 int __user *, child_tidptr,
		 unsigned long, tls)
#endif
{
	struct kernel_clone_args args = {
		.flags		= (lower_32_bits(clone_flags) & ~CSIGNAL),
		.pidfd		= parent_tidptr,
		.child_tid	= child_tidptr,
		.parent_tid	= parent_tidptr,
		.exit_signal	= (lower_32_bits(clone_flags) & CSIGNAL),
		.stack		= newsp,
		.tls		= tls,
	};

	if (!legacy_clone_args_valid(&args))
		return -EINVAL;

	return _do_fork(&args);`
}
```

因为上面对flags的设置产生影响如下:

* CLONE_VM---set if VM shared between processes

  ```c
  //"kernel/fork.c"
  static int copy_mm(unsigned long clone_flags, struct task_struct *tsk)
  {
  	struct mm_struct *mm, *oldmm;
  	int retval;
  .........
  	oldmm = current->mm;
  	if (!oldmm)
  		return 0;
  ...........
  	if (clone_flags & CLONE_VM) {
  		mmget(oldmm);
  		mm = oldmm;
  		goto good_mm;
  	}
  	retval = -ENOMEM;
  	mm = dup_mm(tsk, current->mm);
  ...........
  good_mm:
  	tsk->mm = mm;
  	tsk->active_mm = mm;
  	return 0;
  
  fail_nomem:
  	return retval;
  }
  
  ```

* CLONE_FS---set if fs info shared between processes

  ```c
  //"kernel/fork.c"
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
  ```

* CLONE_FILES---set if open files shared between processes

  ```c
  //"kernel/fork.c"
  static int copy_files(unsigned long clone_flags, struct task_struct *tsk)
  {
  	struct files_struct *oldf, *newf;
  	int error = 0;
  	oldf = current->files;
  	if (clone_flags & CLONE_FILES) {
  		atomic_inc(&oldf->count);
  		goto out;
  	}
  	newf = dup_fd(oldf, &error);
  	tsk->files = newf;
  	error = 0;
  out:
  	return error;
  }
  ```

* CLONE_SYSVSEM---share system V SEM_UNDO semantics

  ```c
  //"ipc/sem.c"
  int copy_semundo(unsigned long clone_flags, struct task_struct *tsk)
  {
  	struct sem_undo_list *undo_list;
  	int error;
  
  	if (clone_flags & CLONE_SYSVSEM) {
  		error = get_undo_list(&undo_list);
  		if (error)
  			return error;
  		refcount_inc(&undo_list->refcnt);
  		tsk->sysvsem.undo_list = undo_list;
  	} else
  		tsk->sysvsem.undo_list = NULL;
  
  	return 0;
  }
  ```

* CLONE_SIGHAND---set if signal handlers and blocked signals shared

  ```c
  //"kernel/fork.c"
  static int copy_sighand(unsigned long clone_flags, struct task_struct *tsk)
  {
  	struct sighand_struct *sig;
  
  	if (clone_flags & CLONE_SIGHAND) {
  		refcount_inc(&current->sighand->count);
  		return 0;
  	}
  .........
  	return 0;
  }
  ```

  

* CLONE_THREAD---Same thread group?

  ```c
  //"kernel/fork.c"
  static int copy_signal(unsigned long clone_flags, struct task_struct *tsk)
  {
  	struct signal_struct *sig;
  
  	if (clone_flags & CLONE_THREAD)
  		return 0;
      sig = kmem_cache_zalloc(signal_cachep, GFP_KERNEL);
  	tsk->signal = sig;
      init_sigpending(&sig->shared_pending);
      .............
  }
  /*在copy_process的主流程里面，无论是创建进程还是线程，都会初始化struct sigpending pending，也就是每个task_struct，都会有这样一个成员变量。这就是一个信号列表。如果这个task_struct是一个线程，这里面的信号就是发给这个线程的；如果这个task_struct是一个进程，这里面的信号是发给主线程的。
  另外，上面copy_signal的时候，我们可以看到，在创建进程的过程中，会初始化signal_struct里面的struct sigpending shared_pending。但是，在创建线程的过程中，连signal_struct都共享了。也就是说，整个进程里的所有线程共享一个shared_pending，这也是一个信号列表，是发给整个进程的，哪个线程处理都一样。*/
  
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
  ```

* CLONE_SETTLS----create a new TLS for the child

  ```c
  //"arch/x86/kernel/process.c"
  int copy_thread_tls(unsigned long clone_flags, unsigned long sp,
  		    unsigned long arg, struct task_struct *p, unsigned long tls)
  {
  ....................
  	if (clone_flags & CLONE_SETTLS)
  		ret = set_new_tls(p, tls);
  ....................
  	return ret;
  }
  ```

* CLONE_PARENT_SETTID---set the TID in the parent

  ```c
  //"kernel/fork.c"
  long _do_fork(struct kernel_clone_args *args)
  {
  .............
  	if (clone_flags & CLONE_PARENT_SETTID)
  		put_user(nr, args->parent_tid);
  .................
  }
  ```

* CLONE_CHILD_CLEARTID---clear the TID in the child

  ```C
  //"kernel/fork.c"
  static __latent_entropy struct task_struct *copy_process(
  					struct pid *pid,
  					int trace,
  					int node,
  					struct kernel_clone_args *args)
  {
  p->clear_child_tid = (clone_flags & CLONE_CHILD_CLEARTID) ? args->child_tid : NULL;
  }
  ```

### start_thread

```c
#define START_THREAD_DEFN \
  static int __attribute__ ((noreturn)) start_thread (void *arg)
#define START_THREAD_SELF arg

/* pthread_create.c defines this using START_THREAD_DEFN
   We need a forward declaration here so we can take its address.  */
static int start_thread (void *arg) __attribute__ ((noreturn));
```

START_THREAD_DEFN

```c
//"nptl/pthread_create.c"
START_THREAD_DEFN
{
  struct pthread *pd = START_THREAD_SELF;

  .............
      /* Run the code the user provided.  */
      void *ret;
      if (pd->c11)
	{
	  /* The function pointer of the c11 thread start is cast to an incorrect
	     type on __pthread_create_2_1 call, however it is casted back to correct
	     one so the call behavior is well-defined (it is assumed that pointers
	     to void are able to represent all values of int.  */
	  int (*start)(void*) = (int (*) (void*)) pd->start_routine;
	  ret = (void*) (uintptr_t) start (pd->arg);
	}
      else
	ret = pd->start_routine (pd->arg);
      THREAD_SETMEM (pd, result, ret);
    }

  /* Call destructors for the thread_local TLS variables.  */
#ifndef SHARED
  if (&__call_tls_dtors != NULL)
#endif
    __call_tls_dtors ();

  /* Run the destructor for the thread-local data.  */
  __nptl_deallocate_tsd ();

  /* Clean up any state libc stored in thread-local variables.  */
  __libc_thread_freeres ();

  /* If this is the last thread we terminate the process now.  We
     do not notify the debugger, it might just irritate it if there
     is no thread left.  */
  if (__glibc_unlikely (atomic_decrement_and_test (&__nptl_nthreads)))
    /* This was the last thread.  */
    exit (0);
.........................

  /* If the thread is detached free the TCB.  */
  if (IS_DETACHED (pd))
    /* Free the TCB.  */
    __free_tcb (pd);

  /* We cannot call '_exit' here.  '_exit' will terminate the process.

     The 'exit' implementation in the kernel will signal when the
     process is really dead since 'clone' got passed the CLONE_CHILD_CLEARTID
     flag.  The 'tid' field in the TCB will be set to zero.

     The exit code is zero since in case all threads exit by calling
     'pthread_exit' the exit status must be 0 (zero).  */
  __exit_thread ();

  /* NOTREACHED */
}

```

在用户的函数执行完毕之后，会释放这个线程相关的数据。例如，线程本地数据thread_local variables，线程数目也减一。如果这是最后一个线程了，就直接退出进程，另外__free_tcb用于释放pthread。

free_tcb会调用deallocate_stack来释放整个线程栈，这个线程栈要从当前使用线程栈的列表stack_used中拿下来，放到缓存的线程栈列表stack_cache中。

整个线程的生命周期到这里就结束了。

```c
# define THREAD_SETMEM(descr, member, value) \
  ({									      \
     _Static_assert (sizeof (descr->member) == 1			      \
		     || sizeof (descr->member) == 4			      \
		     || sizeof (descr->member) == 8,			      \
		     "size of per-thread data");			      \
     if (sizeof (descr->member) == 1)					      \
       asm volatile ("movb %b0,%%fs:%P1" :				      \
		     : "iq" (value),					      \
		       "i" (offsetof (struct pthread, member)));	      \
     else if (sizeof (descr->member) == 4)				      \
       asm volatile ("movl %0,%%fs:%P1" :				      \
		     : IMM_MODE (value),				      \
		       "i" (offsetof (struct pthread, member)));	      \
     else /* 8 */							      \
       {								      \
	 asm volatile ("movq %q0,%%fs:%P1" :				      \
		       : IMM_MODE ((uint64_t) cast_to_integer (value)),	      \
			 "i" (offsetof (struct pthread, member)));	      \
       }})
```

## 总结

创建进程的话，调用的系统调用是fork，在copy_process函数里面，会将五大结构files_struct、fs_struct、sighand_struct、signal_struct、mm_struct都复制一遍，从此父进程和子进程各用各的数据结构。而创建线程的话，调用的是系统调用clone，在copy_process函数里面， 五大结构仅仅是引用计数加一，也即线程共享进程的数据结构。

![img](https://static001.geekbang.org/resource/image/14/4b/14635b1613d04df9f217c3508ae8524b.jpeg)