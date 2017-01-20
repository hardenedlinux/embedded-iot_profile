# 1 简介

little kernel是一个基于线程的操作系统，是运行在AARCH32状态下的操作系统，跟uCOS类似程序不可以动态加载，程序需要在编译操作系统时一起编译，little kernel提供event、mutex、timer以及thread的支持。

little kernel现在用于安卓的bootloader，bootloader作为其一个应用程序（aboot）实现

本文分析的little kernel是dragonboard410c的源码

# 2 源码获取
```
git clone git://codeaurora.org/kernel/lk.git
git checkout -b mylk remotes/origin/master
```

# 3 编译
安装编译器 sudo apt install gcc-arm-linux-gnueabi

设置工具链 export TOOLCHAIN_PREFIX=arm-linux-gnueabi-

编译 make msm8916 EMMC_BOOT=1

# 4 运行流程
通过链接脚本arch/arm/system-onesegment.ld第4行ENTRY(\_start)，可以知道程序从\_start开始执行，\_start位于arch/arm/crt0.S中，此文件实现了异常向量表、堆栈初始化、数据段初始化（data、BSS）并把自己移动到合适的地址，最后跳转到kmain

```c
#define DSB .byte 0x4f, 0xf0, 0x7f, 0xf5	/* ARM64的数据屏障指令 */
#define ISB .byte 0x6f, 0xf0, 0x7f, 0xf5	/* ARM64的指令屏障指令 */

.section ".text.boot"
.globl _start
/*
 * 异常向量
 */
_start:
	b	reset				//复位异常
	b	arm_undefined		//未定义指令异常
	b	arm_syscall			//软件中断，系统调用
	b	arm_prefetch_abort	//指令预取异常
	b	arm_data_abort		//数据访问异常
	b	arm_reserved		//保留未用
	b	arm_irq				//interrupt
	b	arm_fiq				//fast interrupt

reset:

#ifdef ENABLE_TRUSTZONE
	/*Add reference to TZ symbol so linker includes it in final image */
	ldr r7, =_binary_tzbsp_tzbsp_bin_start
#endif
	/* do some cpu setup */

/*
 * 以下代码用于初始化SCTLR(System Control Register)
 * 以下代码 清除SCTLR b15/b13/b12/b2/b1/b0
 *			如果是ARMv8 置位SCTLR b5
 * b15保留
 * b13 0 异常基地址remap到VBAR、HVBAR(Hyp mode)、MVBAR(Monitor mode)
 *     1 异常固定 地址0xFFFF0000
 * b12 1 使能指令缓存
 * b5  1 使能barrier指令
 * b2  1 使能数据缓存
 * b1  1 使能对齐检测异常
 * b0  1 MMU使能
 *
 * 关闭指令数据缓存，关闭MMU，使能异常地在remap，使能barrier指令
 */
#if ARM_WITH_CP15
        /* Read SCTLR */
	mrc		p15, 0, r0, c1, c0, 0
		/* XXX this is currently for arm926, revist with armv6 cores */
		/* new thumb behavior, low exception vectors, i/d cache disable, mmu disabled */
	bic		r0, r0, #(1<<15| 1<<13 | 1<<12)
	bic		r0, r0, #(1<<2 | 1<<0)
		/* disable alignment faults */
	bic		r0, r0, #(1<<1)
	/* Enable CP15 barriers by default */
#ifdef ARM_CORE_V8
	orr		r0, r0, #(1<<5)
#endif
        /* Write SCTLR */
	mcr		p15, 0, r0, c1, c0, 0
#ifdef ENABLE_TRUSTZONE
  /*nkazi: not needed ? Setting VBAR to location of new vector table : 0x80000      */
/*
 * 此处设置异常向量表到0x8000
 */
 ldr             r0, =0x00080000
 mcr             p15, 0, r0, c12, c0, 0	
#endif
#endif

#if WITH_CPU_EARLY_INIT
	/* call platform/arch/etc specific init code */
#ifndef ENABLE_TRUSTZONE
	/* Not needed when TrustZone is the first bootloader that runs.*/
	bl __cpu_early_init
#endif
	/* declare return address as global to avoid using stack */
.globl _cpu_early_init_complete
	_cpu_early_init_complete:

#endif

/*
 * 检测是否是热启动，如果是热启动复位
 * 使用冷启动时，内存中warm_boot_tag为0
 * 如果热启动，内存中的值不变，warm_boot_tag中的值为1，促发重启
 */
#if (!ENABLE_NANDWRITE)
#if WITH_CPU_WARM_BOOT
	ldr 	r0, warm_boot_tag
	cmp 	r0, #1

	/* if set, warm boot */
	ldreq 	pc, =BASE_ADDR		/* 复位芯片,BASE_ADDR是芯片启示地在 */

	mov 	r0, #1
	str	r0, warm_boot_tag		/* 设置warm_boot_tag地在中的值为1 */
#endif
#endif



	/* see if we need to relocate */
	mov		r0, pc
	sub		r0, r0, #(.Laddr - _start)//计算_start的绝对地址，即本段代码的加载位置
.Laddr:
	ldr		r1, =_start	//加载复位地址
	cmp		r0, r1		//对比_start与复位是否一致，即本段代码是否加载到复位地址	
	beq		.Lstack_setup	//加载到复位地址跳转到.Lstack_setup

/* 以下代码用于将本段程序移动到复位地址 */
	/* we need to relocate ourselves to the proper spot */
	ldr		r2, =__data_end	
.Lrelocate_loop:
	ldr		r3, [r0], #4
	str		r3, [r1], #4
	cmp		r1, r2
	bne		.Lrelocate_loop

/* 此处使用绝对跳转，跳转到正确的地址 */
	/* we're relocated, jump to the right address */
	ldr		r0, =.Lstack_setup
	bx		r0

/* 检测热启动的全局变量 */
.ltorg
#if WITH_CPU_WARM_BOOT
warm_boot_tag:
	.word 0
#endif

/*
 * 以下代码对32位模式下的SP指针初始化
 *
 * CPSR
 *    b4 0 AARCH64
 *       1 AARCH32
 *    b4 = 1时
 *		b3-b0 0000 User
 *            0001 FIQ
 *            0010 IRQ
 *            0011 Supervisor
 *            0110 Monitor
 *            0111 Abort
 *            1010 Hyp
 *            1011 Undefined
 *            1111 System
 *
 */
.Lstack_setup:
	/* set up the stack for irq, fiq, abort, undefined, system/user, and lastly supervisor mode */
/* 把cpsr的b0-b4清零保存到r0，便于之后切换异常模式 */
	mrs     r0, cpsr
	bic     r0, r0, #0x1f

/* 加载内存地址，堆栈向低地址增长 */
	ldr		r2, =abort_stack_top

	orr     r1, r0, #0x12 // irq
	msr     cpsr_c, r1
	ldr		r13, =irq_save_spot		/* save a pointer to a temporary dumping spot used during irq delivery */
	    
	orr     r1, r0, #0x11 // fiq
	msr     cpsr_c, r1
	mov		sp, r2
	            
	orr     r1, r0, #0x17 // abort
	msr     cpsr_c, r1
	mov		sp, r2
	    
	orr     r1, r0, #0x1b // undefined
	msr     cpsr_c, r1
	mov		sp, r2
	    
	orr     r1, r0, #0x1f // system
	msr     cpsr_c, r1
	mov		sp, r2

	orr		r1, r0, #0x13 // supervisor
	msr		cpsr_c, r1
	mov		sp, r2

	/* copy the initialized data segment out of rom if necessary */
	ldr		r0, =__data_start_rom
	ldr		r1, =__data_start
	ldr		r2, =__data_end

/* 如果没有静态数据跳过初始化过程.L__copy_loop */
	cmp		r0, r1
	beq		.L__do_bss

.L__copy_loop:/* 静态数据初始化 */
	cmp		r1, r2
	ldrlt	r3, [r0], #4
	strlt	r3, [r1], #4
	blt		.L__copy_loop

/* bss数据段初始化为0 */
.L__do_bss:
	/* clear out the bss */
	ldr		r0, =__bss_start
	ldr		r1, =_end
	mov		r2, #0
.L__bss_loop:
	cmp		r0, r1
	strlt	r2, [r0], #4
	blt		.L__bss_loop

#ifdef ARM_CPU_CORTEX_A8
	DSB	/* 数据屏障 */
	ISB /* 指令屏障 */
#endif

	bl		kmain	/* 调用lk的main函数 */
	b		.		/* 死循环 */

.ltorg

/* 堆栈 */
.bss
.align 2
	/* the abort stack is for unrecoverable errors.
	 * also note the initial working stack is set to here.
	 * when the threading system starts up it'll switch to a new 
	 * dynamically allocated stack, so we don't need it for very long
	 */
abort_stack:
	.skip 1024
abort_stack_top:
```

kmain是little kernel的主函数，操作系统初始化后创建一个线程bootstrap2

```c
void kmain(void)
{
	/* 
	 * 系统进程相关的初始化
	 * 初始化全局线程链表thread_list
	 * 初始化运行队列run_queue
	 * 为当前创建创建进程标示 
	 */
	thread_init_early();

	/* 
	 * 架构（ARM、x86、MIPS）相关初始化
	 * 设置异常向量基地址
	 * 使能MMU
	 * 使能cp10 cp11，这两个协处理器与FPU相关) 
	 * 使能周期计数器(cycle count)
	 */
	arch_early_init();

	/*
	 * Soc相关初始化
	 * msm8916执行了以下操作
	 * 获取platform相关信息，保存到borad全局变量中
	 * 子系统时钟控制句柄初始化，保存在msm_clk_list结构体中  
	 * 中断控制其初始化
	 * 初始化ticks_per_sec
	 * 通过SCM调用，判断系统是否支持SCM（scm_arm_support）
	 */
	platform_early_init();

	/*
	 * 目标板相关初始化
	 * dragonboard 410c执行了以下操作
	 * 串口初始化
	 */
	target_early_init();

	dprintf(INFO, "welcome to lk\n\n");
	bs_set_timestamp(BS_BL_START);/* 获取系统开始时钟 */

	// deal with any static constructors
	dprintf(SPEW, "calling constructors\n");
	call_constructors();/* 编译器相关的构造，链接器输出.ctors段 */

	// bring up the kernel heap
	dprintf(SPEW, "initializing heap\n");
	heap_init();/* 初始化栈 */

	__stack_chk_guard_setup();

	// initialize the threading system
	dprintf(SPEW, "initializing threads\n");
	thread_init();/* 定时器timer相关初始化 */

	// initialize the dpc system
	dprintf(SPEW, "initializing dpc\n");
	/* dpc为一个系统服务
	 * 用于执行一些比较重要的任务
	 * 优先级仅次与highest
	 */
	dpc_init();

	// initialize kernel timers
	dprintf(SPEW, "initializing timers\n");
	/*
	 * 启动定时任务
	 */
	timer_init();

#if (!ENABLE_NANDWRITE)
	
	dprintf(SPEW, "creating bootstrap completion thread\n");
	thread_resume(thread_create("bootstrap2", &bootstrap2, NULL, DEFAULT_PRIORITY, DEFAULT_STACK_SIZE));

	// 使能中断
	exit_critical_section();

	/* 标记当前线程为空闲线程 */
	thread_become_idle();
#else
    bootstrap_nandwrite();
#endif
}
```

bootstrap2还有一些与设备相关的初始化工作之后启动apps_init，apps_init实现了一个应用框架，具体实现常见第6章

```c
static int bootstrap2(void *arg)
{
    dprintf(SPEW, "top of bootstrap2()\n");
    arch_init();
    // XXX put this somewhere else
#if WITH_LIB_BIO
    bio_init();
#endif
#if WITH_LIB_FS
    fs_init();
#endif
  
    // initialize the rest of the platform
    dprintf(SPEW, "initializing platform\n");
    platform_init();

    // initialize the target
    dprintf(SPEW, "initializing target\n");
    target_init();

    dprintf(SPEW, "calling apps_init()\n");
    apps_init();

    return 0;
}

```



# 5 little kernel OS



## 5.1 概要

little kernel是一个基于线程（`thread`）的操作系统

little kernel根据线程优先级进行调度，优先调度优先级高的线程。同一个优先级下可以有多个线程，这些线程通过时间片轮询调度。

little kernel提供定时器(`timer`)功能，可以在多长时间后执行一个任务，也可以设定为周期性执行。

little kernel为线程通信提供了基本的服务`event`、`mutex`

`event`用于向其他程序发送消息，通知其他线程执行某个动作

`mutex`用于多线程之间保护共享资源，防止共享资源被多个线程同时访问

## 5.2 线程

### 5.2.1 数据结构

little kernel每个线程为一个函数，函数格式如下

```c
typedef int (*thread_start_routine)(void *arg);
```

little kernel每个线程对应一个线程标示符，标示符定义如下：

```c
typedef struct thread {
    int magic;/* 一个常数，用于检测结构体是否合法 */
    struct list_node thread_list_node;/* 在thread_list中的节点 */
    struct list_node queue_node;/* 在运行队列或者等待队列中的节点 */
    int priority;/* 线程的优先级 */
    enum thread_state state; /* 线程状态 */
    int saved_critical_section_count;
    int remaining_quantum;/* 线程剩余的时间片 */
  	/*
  	 * 线程进入BLOCK状态时
  	 * 这个指针指向等待队列 
  	 * 在带超时等待的event、mutex时有效
  	 * 用于超时后从队列中删除后把队列中的计数器减1
  	 */
    struct wait_queue *blocking_wait_queue;
  	/* 记录带超时等待的状态，超时或等到 */
    status_t wait_queue_block_ret;
  	/* 架构与线程相关的数据 */
    struct arch_thread arch;
	
  	/* 线程的栈空间 */
    void *stack;
    size_t stack_size;

    thread_start_routine entry;/* 线程入口函数 */
    void *arg;/* 线程入口函数的参数 */

    int retcode;/* 线程结束的返回值 */

    /*
     * 线程的局部存储
     * 线程创建时从父线程继承
     * static inline __ALWAYS_INLINE uint32_t tls_get(uint entry)
     * static inline __ALWAYS_INLINE uint32_t tls_set(uint entry, uint32_t val)
     * 以上两个函数分别用来修改获取局部存储
     * 在调试时，会输出线程的局部存储
     * 在当前代码中此功能没有使用
     */
    uint32_t tls[MAX_TLS_ENTRY];

    char name[32];/* 线程名 */
} thread_t;
```

little kernel的线程有如下状态

```c
enum thread_state {
	THREAD_SUSPENDED = 0,/* 线程创建时的状态 */
	THREAD_READY,		/* 准备就绪的线程，处于run_queue中 */
	THREAD_RUNNING,		/* 运行中的线程，不在run_queue中 */
	THREAD_BLOCKED,		/* 阻塞中的线程（等待event、mutex），处于某个wait_queue中 */
	THREAD_SLEEPING,	/* 线程调用thread_sleep后的状态 */
	THREAD_DEATH,		/* 线程退出后的状态，等待dpc服务回收资源 */
};
```

相关的全局变量

```c
static struct list_node thread_list;	/* 线程列表 */
static thread_t bootstrap_thread;		/* 系统上电的线程 */
thread_t *current_thread;				/* 当前正在执行的线程 */
thread_t *idle_thread;					/* 空闲线程 */
```

little kernel通过优先级调度程序，准备就绪的程序会被加入一个运行队列。little kernel具有多少个优先级就有多少个运行队列。运行队列是一个列表，多个运行队列组成一个数组。线程通过thread_t.priority找到对应的线程队列。其中优先级值越大等级越高。

```c
static struct list_node run_queue[NUM_PRIORITIES];/* 多个运行队列组成的数组 */
static uint32_t run_queue_bitmap;/* 标记run_queue数组中，哪些非空 */
```

run_queue_bitmap用于标记那个队列非空，使用run_queue_bitmap.bit[n]== 1标示run_queue[n]非空。因为run_queue_bitmap是uint32_t类型，只有32个比特位，所以little kernel最多有32个优先级。

### 5.2.2 相关操作

#### 5.2.2.1 thread_init_early

初始化线程相关的全局变量（thread_list、run_queue、current_thread），标记当前线程为开机线程`bootstrap_thread`，注意开机线程具有最高的优先级`HIGHEST_PRIORITY`

```
void thread_init_early(void)
```

#### 5.2.2.2 thread_init

初始化定时器preempt_timer，此定时器将用于添加周期任务`thread_timer_tick`，用于处理线程的时间片。任务的维护由线程调度函数`thread_resched`维护

```c
void thread_init(void);
```

#### 5.2.2.3 thread_create

此操作用于创建一个线程标示符，进行一些基本的初始化，创建线程的堆栈空间，并添加到全局线程列表thread_list。注意此时的线程处于`THREAD_SUSPENDED`状态，不会立即执行，将等待`thread_resume`调用把线程加入到运行队列。
```c
thread_t *thread_create(
  const char *name, 			/* 要创建的线程的名字 */
  thread_start_routine entry, 	/* 是线程的入口函数 */
  void *arg, 					/* 线程入口函数的参数 */
  int priority, 				/* 线程的优先级*/
  size_t stack_size)			/* 线程的堆栈的大小 */
```

#### 5.2.2.4 thread_resume

把`thread_create`创建的线程加入到运行队列的开始，并触发一次线程调度。

```c
status_t thread_resume(thread_t *t)/* t是thread_create创建的线程标示符 */
```

#### 5.2.2.5 thread_exit

线程退出，会向系统`dpc`复位注册一个任务，用于线程资源回收（堆栈、标示符、从全局线程列表删除任务）

`dpc`是一个优先级较高的线程，作为系统复位，可以接受其他线程的任务

```c
void thread_exit(int retcode)/* retcode为线程返回值 */
```

#### 5.2.2.6 thread_resched

触发一次线程调度（执行就绪队列中优先级最高的线程），统计空闲进程的执行时间，并且根据情况初始化一个周期定时任务`thread_timer_tick`。thread_timer_tick用于处理线程的时间片。

```c
void thread_resched(void)
```

#### 5.2.2.7 thread_yield

让出CPU资源，触发线程调度，优先调度同等优先级的其他线程

```c
void thread_yield(void)
```

#### 5.2.2.8 thread_preempt

处理线程的时间片，时间片用完时，调度相同优先级的下一个线程

```c
void thread_preempt(void)
```

#### 5.2.2.9 thread_block

使进程进入等待状态，在调用此函数前需要处理好进程标示符，把进程标示符加入合适的等待队列并标记进程为`THREAD_BLOCK`

```c
void thread_block(void)
```

#### 5.2.2.10 thread_sleep

延时函数，把当前线程标记为`THREAD_SLEEPING`，添加一个定时任务（在指定时间后唤醒自己），触发线程调度

```c
void thread_sleep(time_t delay)/* delay为要延时的时间 */
```

#### 5.2.2.11 thread_set_name

修改当前线程的名字

```c
void thread_set_name(const char *name);
```

#### 5.2.2.12 thread_set_priority

```c
void thread_set_priority(int priority);
```

修改当前线程的优先级

#### 5.2.2.13 thread_become_idle

把当前进程标记为空闲进程

```c
void thread_become_idle(void) __NO_RETURN;
```



## 5.3 定时器

### 5.3.1 数据结构

```c
/* timer_queue为定时器列表
 * 按时间顺序排序任务 
 * 所有的定时任务都将添加到这个列表中
 */
static struct list_node timer_queue;
```

```c
typedef struct timer {
	int magic;				/* 常数用于检查结构体为合法的定时器 */
	struct list_node node;	/* 用于构成链表 */
	time_t scheduled_time;	/* 促发时间的时间 */
	time_t periodic_time;	/* 标记为周期时间 */
	timer_callback callback;/* 要执行的任务，函数回调句柄 */
	void *arg;				/* 回调函数的参数 */
} timer_t;
```

```c
/* 定时任务，函数类型定义 */
typedef enum handler_return (*timer_callback)(struct timer *, time_t now, void *arg);
```

### 5.3.2 相关操作

#### 5.3.2.1 timer_init

启动定时任务，创建一个周期定时任务此任务会维护定时器队列`timer_queue`

```c
void timer_init(void); 
```

#### 5.3.2.2 timer_initialize

初始化一个timer_t结构

```c
void timer_initialize(timer_t *);
```

#### 5.3.2.3 timer_set_oneshot
添加一个一次性定时任务，指定延时delay，要执行的任务timer_callback，以及任务的参数arg
```c
void timer_set_oneshot(timer_t *, time_t delay, timer_callback, void *arg);
```
#### 5.3.2.4 timer_set_periodic
添加一个周期性定时任务，指定延时delay，要执行的任务timer_callback，以及任务的参数arg
```c
void timer_set_periodic(timer_t *, time_t period, timer_callback, void *arg);
```
#### 5.3.2.5 timer_cancel
取消一个定时器，将从timer_queue中删除
```c
void timer_cancel(timer_t *);
```
## 5.4 等待队列

### 5.4.1 数据结构

等待队列，是处于阻塞`THREAD_BLOCK`状态的线程的列表。为`enent`和`mutex`的属性，用于记录被阻塞的线程。

```c
typedef struct wait_queue {
	int magic;				/* 一个常数标记当前结构体为一个合法的wait_queue_t */
	struct list_node list;	/* 链表，被堵塞的线程标示符列表 */
	int count;				/* 等待队列中线程的个数 */
} wait_queue_t;
```

### 5.4.2 相关操作

#### 5.4.2.1 wait_queue_init

初始化等待队列，此时列表初始化为空

```c
void wait_queue_init(wait_queue_t *);
```

#### 5.4.2.2 wait_queue_destroy

释放等待队列的资源

```c
void wait_queue_destroy(wait_queue_t *, bool reschedule);
```

所有被阻塞的线程会被添加到运行队列。如果reschedule为真，触发线程调度，当前线程将让出CPU资源，优先调度优级大于等于当前线程的线程。

#### 5.4.2.3 wait_queue_block

把当前线程添加到等待队列，`timeout`不等于`INFINITE_TIME`时添加一个定时任务。此任务用于把当前线程唤醒（添加到运行队列并触发任务调度），并把当前任务从等待队列中删除。

```
status_t wait_queue_block(wait_queue_t *, time_t timeout);
```

#### 5.4.2.4 wait_queue_wake_one

唤醒等待队列中的第一个线程，reschedule标记要不要触发一次调度，wait_queue_error是唤醒的原因

```c
int wait_queue_wake_one(wait_queue_t *wait, bool reschedule, status_t wait_queue_error)
```

#### 5.4.2.5 wait_queue_wake_all

唤醒等待队列中的所有线程，reschedule标记要不要触发一次调度，wait_queue_error是唤醒的原因

```c
int wait_queue_wake_all(wait_queue_t *, bool reschedule, status_t wait_queue_error);
```

#### 5.4.2.6 thread_unblock_from_wait_queue

把线程从等待队列中删除，添加到运行队列。reschedule标记要不要触发一次调度，wait_queue_error是唤醒的原因

```c
status_t thread_unblock_from_wait_queue(thread_t *t, bool reschedule, status_t wait_queue_error)
```

## 5.5 event

### 5.5.1 数据结构

event有两种，一种需要调用event_unsignal，一种不需要调用event_unsignal。

```c
typedef struct event {
	int magic;			/* 一个常数用于标记当前结构体为合法的event_t */
	bool signalled;		/* 标记envent是否被标记 */
	uint flags;			/* 标记event的种类，0不会自动复位，EVENT_FLAG_AUTOUNSIGNAL会自动复位 */
	wait_queue_t wait;	/* 被event阻塞的线程的等待队列 */
} event_t;
```

### 5.5.2 相关操作

#### 5.5.2.1 event_init

初始化一个`event`，指定其初始状态为`initial`，并指定`event`的复位类型

```c
void event_init(event_t *, bool initial, uint flags);
```
####5.5.2.2 event_destroy
销毁一个`event`对象，并把等待event的线程添加到运行队列，并触发线程调度
```c
void event_destroy(event_t *);
```
####5.5.2.3 event_wait
等待`event`
```c
status_t event_wait(event_t *);
```
####5.5.2.4 event_wait_timeout
带超时等待`event`
status_t event_wait_timeout(event_t *, time_t); 
####5.5.2.5 event_signal
标记一个`event`，reschedule指定要不要立马调度
```c
status_t event_signal(event_t *, bool reschedule);
```
####5.5.2.6 event_unsignal
清除`event`标记
```c
status_t event_unsignal(event_t *);
```
## 5.6 mutex

### 5.6.1 数据结构
```c
typedef struct mutex {
	/* 常数用于检测一个结构体为合法的mutex_t */
	int magic;	
	/* 
	 * 每获取一次mutex count加1
	 * 如果count大于1进入等待状态
 	 * count不大于1获取mutex成功，继续执行
 	 */
	int count; 		
    /* 获取到互斥锁的进程标志 */
	thread_t *holder;	
	/* 没有获取到互斥锁的进程的等待队列 */
	wait_queue_t wait;	
} mutex_t;
```
### 5.6.2 相关操作

####5.6.2.1 mutex_init
初始化一个mutex_init
```c
void mutex_init(mutex_t *);
```

####5.6.2.2 mutex_destroy
销毁一个mutex，把等待队列中所有的进程添加到运行队列并触发线程调度
```c
void mutex_destroy(mutex_t *);
```
####5.6.2.3 mutex_acquire
等待一个mutex
```c
status_t mutex_acquire(mutex_t *);
```

####5.6.2.4 mutex_acquire_timeout
带超时等待一个mutex
```c
status_t mutex_acquire_timeout(mutex_t *, time_t); 
```

####5.6.2.5 mutex_release
释放mutex_release，唤醒等待队列中的一个线程
```c
status_t mutex_release(mutex_t *);
```


# 5 little kernel APP架构
app_descriptor结构体用于记录应用程序信息，定义与include/app.h中
```c
struct app_descriptor {
    const char *name;       //应用名
    app_init  init;         //初始化函数
    app_entry entry;        //线程函数
    unsigned int flags;     //标记，现在只有两个值，1在apps_init时不启动，0在apps_init时启动
};
```

另外在include/app.h中定义了两个宏
```c
#define APP_START(appname) \
    struct app_descriptor _app_##appname \
    __SECTION(".apps") = { .name = #appname,
#define APP_END };
```
通过这两个宏可以实现一个app，主要把一个struct app_descriptor变量放到.apps段
然后结合链接脚本arch/arm/system-onesegment.ld以及app\app.c中的apps_init实现app的启动
arch/arm/system-onesegment.ld把所有的.apps段放到了一起，起始符号\_\_apps\_start，结束符号\_\_apps\_end
apps\_init通过\_\_apps\_start、\_\_apps\_end遍列所有的app\_descriptor建立线程并执行（执行完所有线程的init，然后使用entry创建线程）

# 6 little kernel实现了一个命令行接口
命令行接口是其中一个线程，在app/shell/shell.c中实现
命令行接口实现位于lib/console.c中
重要的数据结构在include/lib/console.h中声明
其中命令是一个函数 typedef int (*console\_cmd)(int argc, const cmd\_args *argv);
命令的参数值通过cmd\_args结构体传递，结构体保存了字符串和数值形式
```c
typedef struct {
    const char *str;    //字符串格式
    unsigned int u;     //无符号整数格式
    int i;              //有符号整数格式
} cmd_args;
```
命令信息记录在结构体cmd中
```c
typedef struct {
    const char *cmd_str;                //命令名
    const char *help_str;               //命令的帮助信息
    const console_cmd cmd_callback;     //命令的回调函数
} cmd;
```
多条命令信息通过结构体cmd_block保存
```c
typedef struct _cmd_block {
    struct _cmd_block *next;            //指向下一组命令信息
    size_t count;                       //命令个数
    const cmd *list;	                //指针，指向cmd数组
} cmd_block;
```
同时定义了两个宏用于辅助生成命令块
```c
#define STATIC_COMMAND_START static const cmd _cmd_list[] = {
#define STATIC_COMMAND_END(name) }; \
	const cmd_block _cmd_block_##name \
	__SECTION(".commands")= \
	{ NULL, sizeof(_cmd_list) / sizeof(_cmd_list[0]), _cmd_list }
```
这里命令数组是固定的名子\_cmd\_list，所以一个文件只能出现一次STATIC\_COMMAND\_START、STATIC\_COMMAND\_END宏
命令信息块被放到.commands段
链接脚本arch/arm/system-onesegment.ld把所有的.commands段放到了一起，起始符号\_\_commands\_start，结束符号\_\_commands\_cmd
console\_init函数通过\_\_commands\_start、\_\_commands\_cmd遍列所有的结构体完成命令的注册（组成cmd\_block的列表），注册过程在shell\_init中完成
然后通过console\_start函数，等待命令行输入并调用相应的处理函数，console\_start在shell\_entry中被调用
​ 
经过测试little kernel命令行接口并不能正常工作，因为aboot程序在init过程中实现，所有的app线程都没有创建（包含命令行app shell）

# 7 aboot

一个为android设计的bootloader，作为一个little kernel的一个非标准程序（在init中完成所有的操作，没有线程入口entry）存在，aboot入口位于app/aboot/aboot.c aboot\_init函数中，以下是aboot的主要流程：

```flow
st=>start: 开始
e=>end: 结束
op1=>operation: 设置存储器信息（页大小）
op2=>operation: 获取设备信息(加锁、校验等信息)位于devinfo第一个扇区、aboot最后一个扇区或者emmc的RPMB分区
op3=>operation: 获取oem解锁信息，位于config、frp最后一个扇区最后一个字节的LSB比特位
op4=>operation: 显示相关初始化（可以配置是否闹钟唤醒跳过显示初始化）
op5=>operation: 用户强制重启直接启动boot分区
op6=>operation: 根据按键设置启动模式
op7=>operation: 读取一些重启信息（寄存器）设置启动模式
op8=>operation: 启动系统（boot或recovery，此处校验整个启动镜像）或者进入fastboot
st->op1->op2->op3->op4->op5->op6->op7->op8->e
```
其中，启动系统（boot、recovery）时，对系统进行签名校验。其中与签名校验相关的部分

- 签名有两个不同的方式，根据VERIFIED_BOOT配置决定
  - VERIFIED_BOOT为真
    - 公钥保存在platform/msm_shared/include/oem_keystore.h或者keystone分区中,格式未知（d2i_KEYSTORE解码出keystone）
    - 签名信息比较复杂，有被签名对象的名字以及数据长度
    - hash算法固定SHA256
  - VERIFIED_BOOT为假
    - 公钥保存在platform/msm_shared/certificate.c certBuffer[]中，x509格式
    - 签名格式比较简单，即用RAS私钥加密的hash值
    - hash算法可配置SHA1/SHA256
- 设备信息device_info（位于devinfo第一个扇区，或aboot最后一个扇区，或emmc的RPMB分区中）,其中is_unlocked域为真可以跳过签名检查
- OEM解锁信息（位于config、frp最后一个扇区最后一个字节的LSB比特位），此位如果为1将跳过签名校验

启动分区放的镜像文件为一种raw，没有文件系统，格式参见bootimg.h

镜像文件包含3部分，镜像、initrd、second（作用未知），device tree可以放在两个位置：

- dt_size = 0时，device tree在内核中的后半部分，地址用tags_addr指定
- dt_size > 0时，device tree在second之后，地址紧接着second







