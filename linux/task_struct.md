

# Thread

## 数据结构介绍

![img](https://static001.geekbang.org/resource/image/1c/bc/1c91956b52574b62a4418a7c6993d8bc.jpeg)

```c
//   task_truct结构体定义位置 "include/linux/sched.h"
```



#### 任务ID

```c
pid_t pid;   //线程id
pid_t tgid;  //主线程id
struct task_struct *group_leader;   //主线程指针
//可以通过tgid 和 group_leader 判断task_struct代表的是什么
```

#### 信号处理

```c
struct signal_struct			*signal;      
struct sighand_struct			*sighand;  //正在处理的信号
sigset_t						blocked;  //阻塞暂不处理的信号
sigset_t						real_blocked;  
sigset_t						saved_sigmask;
struct sigpending				pending;    //等待处理的信号
unsigned long					sas_ss_sp;
size_t							sas_ss_size;
unsigned int					sas_ss_flags;
//task_struct里面有一个struct sigpending pending。如果我们进入struct signal_struct *signal去看的话，还有一个struct sigpending shared_pending。它们一个是本任务的，一个是线程组共享的。
```

#### 任务状态

```c
volatile long state; /* -1 unrunnable 0 runnable >0 stopped */
int exit_state;
unsigned int flags;
```

state（状态）可以取的值定义在include/linux/sched.h头文件中。

```c
/* Used in tsk->state: */
#define TASK_RUNNING                    0
#define TASK_INTERRUPTIBLE              1
#define TASK_UNINTERRUPTIBLE            2
#define __TASK_STOPPED                  4
#define __TASK_TRACED                   8
/* Used in tsk->exit_state: */
#define EXIT_DEAD                       16
#define EXIT_ZOMBIE                     32
#define EXIT_TRACE                      (EXIT_ZOMBIE | EXIT_DEAD)
/* Used in tsk->state again: */
#define TASK_DEAD                       64
#define TASK_WAKEKILL                   128
#define TASK_WAKING                     256
#define TASK_PARKED                     512
#define TASK_NOLOAD                     1024
#define TASK_NEW                        2048
#define TASK_STATE_MAX                  4096
```

![img](https://static001.geekbang.org/resource/image/e2/88/e2fa348c67ce41ef730048ff9ca4c988.jpeg)

* **TASK_RUNNING** 表示程序可运行

* **TASK_INTERRUPTIBLE** 可中断的睡眠状态,可被信号唤醒

* **TASK_UNINTERRUPTIBLE**  不可中断的睡眠状态,不可被信号唤醒(无法接收kill信号,程序中断后无法关闭)

* **TASK_LILLABLE**  可终止的睡眠状态,只可以接收kill信号

  ```c
  #define TASK_KILLABLE           (TASK_WAKEKILL | TASK_UNINTERRUPTIBLE)
  ```

* **TASK_STOPPED**  进程接收到SIGSTOP、SIGTTIN、SIGTSTP或者SIGTTOU信号之后进入该状态。

* **TASK_TRACED**表示进程被debugger等进程监视，进程执行被调试程序所停止。当一个进程被另外的进程所监视，每一个信号都会让进程进入该状态。

  一旦一个进程要结束，先进入的是EXIT_ZOMBIE状态，但是这个时候它的父进程还没有使用wait()等系统调用来获知它的终止信息，此时进程就成了僵尸进程。

* **EXIT_DEAD**是进程的最终状态。

* **EXIT_ZOMBIE**  进程结束时收件进入该状态,若父进程没有使用wait()等系统叫用获取它的终止信息,则该进程成为僵尸进程.              和EXIT_DEAD也可以用于exit_state。





上面的进程状态和进程的运行、调度有关系，还有其他的一些状态，我们称为**标志**。放在flags字段中，这些字段都被定义称为**宏**，以PF开头。我这里举几个例子。

```
#define PF_EXITING		0x00000004
#define PF_VCPU			0x00000010
#define PF_FORKNOEXEC		0x00000040
```

**PF_EXITING**表示正在退出。当有这个flag的时候，在函数find_alive_thread中，找活着的线程，遇到有这个flag的，就直接跳过。

**PF_VCPU**表示进程运行在虚拟CPU上。在函数account_system_time中，统计进程的系统运行时间，如果有这个flag，就调用account_guest_time，按照客户机的时间进行统计。

**PF_FORKNOEXEC**表示fork完了，还没有exec。在_do_fork函数里面调用copy_process，这个时候把flag设置为PF_FORKNOEXEC。当exec中调用了load_elf_binary的时候，又把这个flag去掉。	

#### 进程调度

```c
//是否在运行队列上
int				on_rq;
//优先级
int				prio;
int				static_prio;
int				normal_prio;
unsigned int			rt_priority;
//调度器类
const struct sched_class	*sched_class;
//调度实体
struct sched_entity		se;
struct sched_rt_entity		rt;
struct sched_dl_entity		dl;
//调度策略
unsigned int			policy;
//可以使用哪些CPU
int				nr_cpus_allowed;
cpumask_t			cpus_allowed;
struct sched_info		sched_info;
```



#### 运行统计信息

```c
u64				utime;//用户态消耗的CPU时间
u64				stime;//内核态消耗的CPU时间
unsigned long			nvcsw;//自愿(voluntary)上下文切换计数
unsigned long			nivcsw;//非自愿(involuntary)上下文切换计数
u64				start_time;//进程启动时间，不包含睡眠时间
u64				real_start_time;//进程启动时间，包含睡眠时间
```

#### 进程亲缘关系

```c
struct task_struct __rcu *real_parent; /* real parent process */
struct task_struct __rcu *parent; /* recipient of SIGCHLD, wait4() reports */
struct list_head children;      /* list of my children */
struct list_head sibling;       /* linkage in my parent's children list */
```

- parent指向其父进程。当它终止时，必须向它的父进程发送信号。

- children表示链表的头部。链表中的所有元素都是它的子进程。

- sibling用于把当前进程插入到兄弟链表中。

   	通常情况下，real_parent和parent是一样的，但是也会有另外的情况存在。例如，bash创建一个进程，那进程的parent和real_parent就都是bash。如果在bash上使用GDB来debug一个进程，这个时候GDB是real_parent，bash是这个进程的parent。

#### 进程权限

```c
/* Objective and real subjective task credentials (COW): */
const struct cred __rcu         *real_cred;
/* Effective (overridable) subjective task credentials (COW): */
const struct cred __rcu         *cred;
```

* Objective  被操作的对象
* Subjective 操作对象
* cred 我能操作谁
* real_cred 谁能操作我

```c
struct cred {
......
        kuid_t          uid;            /* real UID of the task */
        kgid_t          gid;            /* real GID of the task */
        kuid_t          suid;           /* saved UID of the task */
        kgid_t          sgid;           /* saved GID of the task */
        kuid_t          euid;           /* effective UID of the task */
        kgid_t          egid;           /* effective GID of the task */
        kuid_t          fsuid;          /* UID for VFS ops */
        kgid_t          fsgid;          /* GID for VFS ops */
......
        kernel_cap_t    cap_inheritable; /* caps our children can inherit */
        kernel_cap_t    cap_permitted;  /* caps we're permitted */
        kernel_cap_t    cap_effective;  /* caps we can actually use */
        kernel_cap_t    cap_bset;       /* capability bounding set */
        kernel_cap_t    cap_ambient;    /* Ambient capability set */
......
} __randomize_layout;
```

第一个是uid和gid，注释是real user/group id。一般情况下，谁启动的进程，就是谁的ID。但是权限审核的时候，往往不比较这两个，也就是说不大起作用。

第二个是suid和sgid, 保存该程序所述的用户和用户组

第三个是euid和egid，注释是effective user/group id。一看这个名字，就知道这个是起“作用”的。当这个进程要操作消息队列、共享内存、信号量等对象的时候，其实就是在比较这个用户和组是否有权限。

第四个是fsuid和fsgid，也就是filesystem user/group id。这个是对文件操作会审核的权限。

通过chmod u+s program命令，给这个游戏程序设置set-user-ID的标识位,程序启动时会将e和f修改为 s

cap_permitted表示进程能够使用的权限。但是真正起作用的是cap_effective。cap_permitted中可以包含cap_effective中没有的权限。一个进程可以在必要的时候，放弃自己的某些权限，这样更加安全。假设自己因为代码漏洞被攻破了，但是如果啥也干不了，就没办法进一步突破。

cap_inheritable表示当可执行文件的扩展属性设置了inheritable位时，调用exec执行该程序会继承调用者的inheritable集合，并将其加入到permitted集合。但在非root用户下执行exec时，通常不会保留inheritable集合，但是往往又是非root用户，才想保留权限，所以非常鸡肋。

cap_bset，也就是capability bounding set，是系统中所有进程允许保留的权限。如果这个集合中不存在某个权限，那么系统中的所有进程都没有这个权限。即使以超级用户权限执行的进程，也是一样的。

这样有很多好处。例如，系统启动以后，将加载内核模块的权限去掉，那所有进程都不能加载内核模块。这样，即便这台机器被攻破，也做不了太多有害的事情。

cap_ambient是比较新加入内核的，就是为了解决cap_inheritable鸡肋的状况，也就是，非root用户进程使用exec执行一个程序的时候，如何保留权限的问题。当执行exec的时候，cap_ambient会被添加到cap_permitted中，同时设置到cap_effective中。

#### 内存管理

```c
struct mm_struct                *mm;
struct mm_struct                *active_mm;
```

#### 文件与文件系统

```c
/* Filesystem information: */
struct fs_struct                *fs;
/* Open file information: */
struct files_struct             *files;
```

#### 内核栈

```c
struct thread_info		thread_info;
void  *stack;
```

## 函数栈

![img](https://static001.geekbang.org/resource/image/82/5c/82ba663aad4f6bd946d48424196e515c.jpeg)

#### 用户态函数栈

​	在进程的内存空间里面，栈是一个从高地址到低地址，往下增长的结构，也就是上面是栈底，下面是栈顶，入栈和出栈的操作都是从下面的栈顶开始的。

* 32bit

  ​	A调用B，A的栈里面包含A函数的局部变量，然后是调用B的时候要传给它的参数，然后返回A的地址，这个地址也应该入栈，这就形成了A的栈帧。接下来就是B的栈帧部分了，先保存的是A栈帧的栈底位置，也就是EBP。因为在B函数里面获取A传进来的参数，就是通过这个指针获取的，接下来保存的是B的局部变量等等。

  ​	当B返回的时候，返回值会保存在EAX寄存器中，从栈中弹出返回地址，将指令跳转回去，参数也从栈中弹出，然后继续执行A。

* 64bit

  ​	rax用于保存函数调用的返回结果。栈顶指针寄存器变成了rsp，指向栈顶位置。堆栈的Pop和Push操作会自动调整rsp，栈基指针寄存器变成了rbp，指向当前栈帧的起始位置。

  ​	比较多的是参数传递。rdi、rsi、rdx、rcx、r8、r9这6个寄存器，用于传递存储函数调用时的6个参数。如果超过6的时候，还是需要放到栈里面。

  ​	然而，前6个参数有时候需要进行寻址，但是如果在寄存器里面，是没有地址的，因而还是会放到栈里面，只不过放到栈里面的操作是被调用函数做的。

#### 内核态函数栈

```c
union thread_union {
#ifndef CONFIG_ARCH_TASK_STRUCT_ON_STACK
	struct task_struct task;
#endif
#ifndef CONFIG_THREAD_INFO_IN_TASK
	struct thread_info thread_info;
#endif
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```



* 32bit

  ```c
  #define THREAD_SIZE_ORDER	1
  #define THREAD_SIZE		(PAGE_SIZE << THREAD_SIZE_ORDER)
  ```

* 64bit

  ```c
  #ifdef CONFIG_KASAN
  #define KASAN_STACK_ORDER 1
  #else
  #define KASAN_STACK_ORDER 0
  #endif
  
  #define THREAD_SIZE_ORDER	(2 + KASAN_STACK_ORDER)
  #define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
  ```

#### 通过task_struct 寻找内核栈

通过task_struct的stack指针找到这个线程内核栈

```c
static inline void *task_stack_page(const struct task_struct *task)
{
	return task->stack;
}
```

从task_struct如何得到相应的pt_regs

```c
#define task_pt_regs(task) 
({									
	unsigned long __ptr = (unsigned long)task_stack_page(task);	
	__ptr += THREAD_SIZE - TOP_OF_KERNEL_STACK_PADDING;		
	((struct pt_regs *)__ptr) - 1;					
})

#ifdef CONFIG_X86_32
# ifdef CONFIG_VM86
#  define TOP_OF_KERNEL_STACK_PADDING 16
# else
#  define TOP_OF_KERNEL_STACK_PADDING 8
# endif
#else
# define TOP_OF_KERNEL_STACK_PADDING 0
#endif
```

​	因为压栈pt_regs有两种情况。我们知道，CPU用ring来区分权限，从而Linux可以区分内核态和用户态。

​	因此，第一种情况，我们拿涉及从用户态到内核态的变化的系统调用来说。因为涉及权限的改变，会压栈保存SS、ESP寄存器的，这两个寄存器共占用8个byte。

​	另一种情况是，不涉及权限的变化，就不会压栈这8个byte。这样就会使得两种情况不兼容。如果没有压栈还访问，就会报错，所以还不如预留在这里，保证安全。在64位上，修改了这个问题，变成了定长的。

#### 通过内核栈找task_struct

```c
//"include/linux/thread_info.h"
#define current_thread_info() ((struct thread_info *)current)
```

```c
//"arch/x86/include/asm/thread_info.h"
struct thread_info {
	unsigned long		flags;		/* low level flags */
	u32			status;		/* thread synchronous flags */
};


//"arch/x86/include/asm/current.h"
//要使用Per CPU变量，首先要声明这个变量
DECLARE_PER_CPU(struct task_struct *, current_task);

static __always_inline struct task_struct *get_current(void)
{
	return this_cpu_read_stable(current_task);
}

#define current get_current()

//"arch/x86/kernel/cpu/common.c"
//系统刚刚初始化的时候，current_task都指向init_task。
DEFINE_PER_CPU(struct task_struct *, current_task) = &init_task;

//"arch/x86/kernel/process_32.c"/"arch/x86/kernel/process_64.c"
//当某个CPU上的进程进行切换的时候，current_task被修改为将要切换到的目标进程。例如，进程切换函数__switch_to就会改变current_task。
__visible __notrace_funcgraph struct task_struct *
__switch_to(struct task_struct *prev_p, struct task_struct *next_p)
{
......
this_cpu_write(current_task, next_p);
......
return prev_p;
}
```

