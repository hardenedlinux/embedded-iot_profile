# Google TEE环境trusty分析

[TOC]

## 概要

**trusty**是基于little kernel开发的，关于little kernel的文章请参考我之前的文章[little kernel分析](https://github.com/hardenedlinux/embedded-iot_profile/blob/master/docs/arm64/dragonboard410c/little-kernel%E5%88%86%E6%9E%90.md)。

源码获取

```
repo init -u https://android.googlesource.com/trusty/manifest
repo sync
```

编译

```
# 工具链设置
export ARCH_arm64_TOOLCHAIN_PREFIX=`pwd`/prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.8/bin/aarch64-linux-android-
export ARCH_arm_TOOLCHAIN_PREFIX=`pwd`/prebuilts/gcc/linux-x86/arm/arm-eabi-4.8/bin/arm-eabi-

# 编译
make -j24 generic-arm64
```

## LK初始化钩子

little kernel为了方便初始化过程，创造了一种初始化方法。

具体实现位于`external/lk/include/lk/init.h`、`external/lk/top/init.c`中

### 初始化过程

系统把初始化过程分步，通过初始化等级描述，值越小越早执行。初始化等级通过枚举定于

```c
enum lk_init_level {
    LK_INIT_LEVEL_EARLIEST 		 = 1,

    LK_INIT_LEVEL_ARCH_EARLY     = 0x10000,
    LK_INIT_LEVEL_PLATFORM_EARLY = 0x20000,
    LK_INIT_LEVEL_TARGET_EARLY   = 0x30000,
    LK_INIT_LEVEL_HEAP           = 0x40000,
    LK_INIT_LEVEL_VM             = 0x50000,
    LK_INIT_LEVEL_KERNEL         = 0x60000,
    LK_INIT_LEVEL_THREADING      = 0x70000,
    LK_INIT_LEVEL_ARCH           = 0x80000,
    LK_INIT_LEVEL_PLATFORM       = 0x90000,
    LK_INIT_LEVEL_TARGET         = 0xa0000,
    LK_INIT_LEVEL_APPS           = 0xb0000,

    LK_INIT_LEVEL_LAST = UINT_MAX,
};
```

### 初始化cpu

并把初始化钩子按要执行的处理器分类，类别通过枚举定义

```c
/* 处理器标示 */
enum lk_init_flags {
    LK_INIT_FLAG_PRIMARY_CPU     = 0x1,/* 主处理器 */
    LK_INIT_FLAG_SECONDARY_CPUS  = 0x2,/* 次处理器 */

    /* 主次处理器都执行 */
    LK_INIT_FLAG_ALL_CPUS        = LK_INIT_FLAG_PRIMARY_CPU | LK_INIT_FLAG_SECONDARY_CPUS,
    LK_INIT_FLAG_CPU_SUSPEND     = 0x4,/* 暂停的处理器 */
    LK_INIT_FLAG_CPU_RESUME      = 0x8,/* 恢复执行的处理器 */
};
```

### 钩子描述符

通过一个结构体记录钩子信息

```c
/* 钩子描述符 */
struct lk_init_struct {
    uint level;         /* 初始化顺序（数字越小执行越早） */
    uint flags;         /* 指定此钩子函数由哪些cpu执行 */
    lk_init_hook hook;  /* 钩子函数 */
    const char *name;   /* 名字 */
};
```

并定义如下辅助宏，用于实现一个钩子描述符

```c
#define LK_INIT_HOOK_FLAGS(_name, _hook, _level, _flags) \
    const struct lk_init_struct _init_struct_##_name __SECTION(".lk_init") = { \
        .level = _level, \
        .flags = _flags, \
        .hook = _hook, \
        .name = #_name, \
    };

#define LK_INIT_HOOK(_name, _hook, _level) \
    LK_INIT_HOOK_FLAGS(_name, _hook, _level, LK_INIT_FLAG_PRIMARY_CPU)
```

以上宏把钩子描述符放到端**.lk_init**段中，配合链接脚本定义**.lk_init**段头尾符号，便于程序中访问钩子描述符

### 初始化方法

通过函数**lk_init_level**函数执行指定的初始化

```c
/* 从链接脚本中导出的.lk_init段的头尾 */
extern const struct lk_init_struct __lk_init[];
extern const struct lk_init_struct __lk_init_end[];


void lk_init_level(enum lk_init_flags required_flag,/* 要执行钩子的cpu */
  uint start_level, uint stop_level/* 初始化等级[start_level,stop_level] */)
{
    LTRACEF("flags %#x, start_level %#x, stop_level %#x\n",
            required_flag, start_level, stop_level);

    ASSERT(start_level > 0);
    uint last_called_level = start_level - 1;/* 上一次执行钩子的等级 */
    const struct lk_init_struct *last = NULL;/* 记录上一次处理的钩子描述符 */

    for (;;) {/* 查找符号的钩子描述符并执行 */
        LTRACEF("last %p, last_called_level %#x\n", last, last_called_level);

        const struct lk_init_struct *found = NULL;/* 记录查找到的节点 */
        bool seen_last = false;/* 标记上一次处理的节点有无遍列过 */
        /* 找出一个节点 */
        for (const struct lk_init_struct *ptr = __lk_init; ptr != __lk_init_end; ptr++) {
            LTRACEF("looking at %p (%s) level %#x, flags %#x, seen_last %d\n", ptr, ptr->name, ptr->level, ptr->flags, seen_last);

            if (ptr == last)/* 同等级level hook遍历，此处标记上一次的节点 */
                seen_last = true;

            /* 确定处理器 */
            if (!(ptr->flags & required_flag))
                continue;

            /* level必须在[last_called_level,stop_level]之间 */
            if (ptr->level > stop_level)
                continue;
            if (ptr->level < last_called_level)
                continue;

            /* 优先执行低level的钩子 */
            if (found && found->level <= ptr->level)
                continue;

            /* 找出下一个level的钩子
             * 因为last_called_level初始化为start_level - 1
             * 所以此处要判断level >= start_level
             */
            if (ptr->level >= start_level && ptr->level > last_called_level) {
                found = ptr;
                continue;
            }

             /* 同等级level hook遍历
              * last记录上一次处理的节点
              * seen_last标记上一次的节点已经遍历
              */
            if (ptr->level == last_called_level && ptr != last && seen_last) {
                found = ptr;
                break;
            }
        }

        if (!found)/* 不能找出新的节点退出循环 */
            break;

#if TRACE_INIT
        if (found->level >= EARLIEST_TRACE_LEVEL) {
            printf("INIT: cpu %d, calling hook %p (%s) at level %#x, flags %#x\n",
                   arch_curr_cpu_num(), found->hook, found->name, found->level, found->flags);
        }
#endif
        found->hook(found->level);/* 执行指定的hook */

        /* 记录上一次少描的信息 */
        last_called_level = found->level;
        last = found;
    }
}

```

## Trusty App

Trusty App为TOS的应用程序，位于app目录下，一个app对应一个目录。每个app单独编译链接成一个elf，把多个elf链接在一起插入到**.data**段后。

TOS通过自己的代码加载每一个App（elf）到内存，为其创建对应的little kernel线程，此线程执行会退出异常然后执行App对应的代码。

### 数据结构

#### APP信息

通过结构体**trusty_app_t**描述app信息。

```c
typedef struct trusty_app
{
	vaddr_t end_bss;	/* bss的结束地址 */

	vaddr_t start_brk;	/* 堆的起始地址 */
	vaddr_t cur_brk;	/* 当前堆栈的结束位置 */
	vaddr_t end_brk;	/* 堆的结束地址 */
	
  	/* trusty_app属性信息（栈、堆等信息） */
	trusty_app_props_t props;

	void *app_img;		/* elf镜像的起始地址 */

	uthread_t *ut;		/* 用户线程标示符 */

	/* app local storage */
	void **als;			/* app的局部存储，用于记录一些程序相关的信息 */
} trusty_app_t;
```

#### APP属性

APP属性用于配置APP的堆栈等信息，在APP开发时指定，编译链接进app的ELF文件的**.trusty_app.manifest**段。在app加载时由加载程序进行分析处理。

APP属性结构

```c
/* 此结构体信息解析自trusty_app elf文件中的.trusty_app.manifest段
 * 由load_app_config_options函数完才解析工作
 */
typedef struct
{/* trusty_app属性信息 */
	uuid_t		uuid;
	uint32_t	min_stack_size;	/* 栈大小 */
	uint32_t	min_heap_size;	/* 堆大小 */
	uint32_t	map_io_mem_cnt;
	/* 配置信息(config_blob)有多少个字（unsigned int） */
	uint32_t	config_entry_cnt;
	/* 指向app的elf文件中的配置信息 */
	uint32_t	*config_blob;
} trusty_app_props_t;
```

为了方便APP生成**.trusty_app.manifest**段，在**lib/include/trusty_app_manifest.h**中定义了一些辅助的数据结构以及宏

```c
 /* .trusty_app.manifest段的数据结构
  * ---------------------------------------
  * trusty app 的 uuid
  * ---------------------------------------
  * 标示1（一个32bit的数）
  * ---------------------------------------
  * 配置信息1（n个32bit的数，n根具体的标示相关）
  * ---------------------------------------
  * 标示2（一个32bit的数）
  * ---------------------------------------
  * 配置信息2（n个32bit的数，n根具体的标示相关）
  * ---------------------------------------
  *			.
  *			.
  *			.
  * ---------------------------------------
  * 标示x（一个32bit的数）
  * ---------------------------------------
  * 配置信息x（n个32bit的数，n根具体的标示相关）
  * ---------------------------------------
  */
typedef struct trusty_app_manifest {
	uuid_t uuid;
	uint32_t config_options[];
} trusty_app_manifest_t;

enum {

	TRUSTY_APP_CONFIG_KEY_MIN_STACK_SIZE	= 1,/* 栈配置信息标示 */
	TRUSTY_APP_CONFIG_KEY_MIN_HEAP_SIZE	= 2,	/* 堆配置信息标示 */
	TRUSTY_APP_CONFIG_KEY_MAP_MEM		= 3,	/* 内存映射配置信息标示 */
};

/* 辅助生成栈配置信息 */
#define TRUSTY_APP_CONFIG_MIN_STACK_SIZE(sz) \
	TRUSTY_APP_CONFIG_KEY_MIN_STACK_SIZE, sz

/* 辅助生成堆配置信息 */
#define TRUSTY_APP_CONFIG_MIN_HEAP_SIZE(sz) \
	TRUSTY_APP_CONFIG_KEY_MIN_HEAP_SIZE, sz

/* 辅助生成内存映射配置信息 */
#define TRUSTY_APP_CONFIG_MAP_MEM(id,off,sz) \
	TRUSTY_APP_CONFIG_KEY_MAP_MEM, id, off, sz

/* manifest section attributes */
/* 用于把数据放到.trusty_app.manifest段 */
#define TRUSTY_APP_MANIFEST_ATTRS \
	__attribute((aligned(4))) __attribute((section(".trusty_app.manifest")))
```



#### 用户线程标示

TOS（little kernel）位于S-EL1，Trusty APP位于S-EL0。TOS需要管理APP，这里通过用户线程实现。用户线程包含了little kernel的线程描述符，线程创建启动退出等都通过little kernel实现。另外实现虚拟内存管理。

```c
typedef struct uthread
{
	vaddr_t start_stack;				/* 栈顶（向下增长）*/

	vaddr_t entry;						/* 程序入口 */
	void *stack;						/* 栈指针 */

	struct list_node map_list;			/* 虚拟内存信息 */
	mutex_t mmap_lock;					/* 锁，防止多个多个线程访问map_list */
	vaddr_t ns_va_bottom;

	/* uthread ID, unique per thread */
	uint32_t id;						/* 线程标示符 */

	void *page_table;					/* 页表 */

	thread_t *thread;					/* lk的线程标示 */
	struct list_node uthread_list_node;	/* 用户线程列表中的节点 */
	void *private_data;					/* 存放线程的trusty_app_t */

	struct arch_uthread arch;			/* 架构相关的信息 */
} uthread_t;
```

主要实现如下方法

```c
/*
 * name 		线程名字
 * entry        入口地址
 * priority     执行的优先级
 * stack_top    栈顶
 * stack_size   栈的大小
 * private_data 私有数据
 */
uthread_t *uthread_create(const char *name, vaddr_t entry, int priority,
		vaddr_t stack_top, size_t stack_size, void *private_data);

/* Start the user thread */
status_t uthread_start(uthread_t *ut);

/* Exit current uthread */
void uthread_exit(int retcode) __NO_RETURN;

/* 把一段内存映射到用户线程空间 */
/*
 * ut 用户线程结构体
 * vaddrp 要映射到用户线程空间的虚拟内存的起始地址
 * size   要映射到用户线程空间的虚拟内存的大小
 * flag   要映射到用户线程空间的虚拟内存特征
 * align  要映射到用户线程空间虚拟内存的对齐方式
 * pfn_list
 *        映射到用户线程空间虚拟内存对应的物理内存地址信息
 *		  一块虚拟内存可以对应多块物理内存
 *		  内存块数 = size/PAGE_SIZE
 */
status_t uthread_map(uthread_t *ut, vaddr_t *vaddrp, paddr_t *pfn_list,
		size_t size, u_int flags, u_int align);

/* 把一块虚拟内存从用户线程空间移除 */
status_t uthread_unmap(uthread_t *ut, vaddr_t vaddr, size_t size);

/* 检测一块内存是否属于用户线程空间 */
bool uthread_is_valid_range(uthread_t *ut, vaddr_t vaddr, size_t size);

/* 拷贝内存（用户->内核） */
static inline status_t copy_from_user(void *kdest, user_addr_t usrc, size_t len)
{
	return arch_copy_from_user(kdest, usrc, len);
}

/* 拷贝内存（内核->用户） */
static inline status_t copy_to_user(user_addr_t udest, const void *ksrc, size_t len)
{
	return arch_copy_to_user(udest, ksrc, len);
}

/* 拷贝字符串（用户->内核） */
static inline ssize_t  strncpy_from_user(char *kdest, user_addr_t usrc, size_t len)
{
	/* wrapper for now, the old strncpy_from_user was closer to strlcpy than
	 * strncpy behaviour, but could return an unterminated string */
	return arch_strlcpy_from_user(kdest, usrc, len);
}

/* 拷贝字符串（用户->内核） */
static inline ssize_t  strlcpy_from_user(char *kdest, user_addr_t usrc, size_t len)
{
	return arch_strlcpy_from_user(kdest, usrc, len);
}

static inline __ALWAYS_INLINE
/* 映射一段连续的物理空间到虚拟地址空间 */
status_t uthread_map_contig(uthread_t *ut, vaddr_t *vaddrp, paddr_t paddr,
		size_t size, u_int flags, u_int align)
{
	flags = flags | UTM_PHYS_CONTIG;
	return uthread_map(ut, vaddrp, &paddr, size, flags, align);
}

/* 获取用户线程标示（通过thread_t的局部存储实现） */
static inline uthread_t *uthread_get_current(void)
{
	return (uthread_t *)tls_get(TLS_ENTRY_UTHREAD);
}

/* 虚拟地址转实际地址 */
status_t uthread_virt_to_phys(uthread_t *ut, vaddr_t vaddr, paddr_t *paddr);

/* 释放虚拟内存对应的物理地址空间 */
status_t uthread_revoke_pages(uthread_t *ut, vaddr_t vaddr, size_t size);
```

### 链接

每个app通过链接脚本`lk/trusty/app/trusty/user-tasks.mk`编译链接成一个elf文件，并修改段名（在名字前面加.task）

```
USER_TASKS_BIN := $(BUILDDIR)/user_tasks.bin
USER_TASKS_OBJ := $(BUILDDIR)/user_tasks.o

GENERATED += $(USER_TASKS_OBJ) $(USER_TASKS_BIN)

# Create a task.o from the concatenation of tasks to be made available.
# To control placement in the final LK image, the tasks .data section is
# renamed to .task.data, so it's picked up in a page-aligned part of the
# data section by the linker script (allowing tasks to run, in-place).

$(USER_TASKS_BIN): $(ALLUSER_TASK_OBJS)
	@$(MKDIR)
	@echo combining tasks into $@: $(ALLUSER_TASK_OBJS)
	$(NOECHO)cat $(ALLUSER_TASK_OBJS) > $@

$(USER_TASKS_OBJ): $(USER_TASKS_BIN)
	@$(MKDIR)
	@echo generating $@
	$(NOECHO)$(LD) -r -b binary -o $@ $<
	$(NOECHO)$(OBJCOPY) --prefix-sections=.task $@

# Append USER_TASKS_OBJ to EXTRA_OBJ
EXTRA_OBJS += $(USER_TASKS_OBJ)
```

所有的app通过`lk/trusty/lib/trusty/trusty_apps.ld`脚本链接到**.data**段后，并以**__trusty_app_start**符号开始，以**__trusty_app_end**结束。

```
SECTIONS {
.__trusty_app : {
/* pre-built secure apps get inserted here */
. = ALIGN(0x1000);
__trusty_app_start = .;
KEEP(*(.task.*))
__trusty_app_end = .;
. = ALIGN(0x1000);
}
}
INSERT AFTER .data;
```

### 加载

为了程序的加载和执行，系统通过两个初始化钩子实现

```c
/* 主要用于加载app到内存、并为其创建对应的数据结构 */
LK_INIT_HOOK(libtrusty, trusty_init, LK_INIT_LEVEL_APPS - 1);
```

具体加载过程在函数**trusty_app_init**中实现（**trusty_init**直接调用了**trusty_app_init**）

```c
void trusty_app_init(void)
{
	trusty_app_t *trusty_app;
	u_int i;

	/* 通过链接脚本导出的符号获取app的起始结束地址，并计算大小 */
	trusty_app_image_start = (char *)&__trusty_app_start;
	trusty_app_image_end = (char *)&__trusty_app_end;
	trusty_app_image_size = (trusty_app_image_end - trusty_app_image_start);

	ASSERT(!((uintptr_t)trusty_app_image_start & PAGE_MASK));

	/* 标记程序程序已经启动，不能再申请局部存储 */
	finalize_registration();

	/* 把app对应的elf文件加载到内存，并构建trusty_app数据结构记录app信息 */
	trusty_app_bootloader();

	for (i = 0, trusty_app = trusty_app_list; i < trusty_app_count; i++, trusty_app++) {
		char name[32];
		uthread_t *uthread;
		int ret;

		snprintf(name, sizeof(name), "trusty_app_%d_%08x-%04x-%04x",
			 i,
			 trusty_app->props.uuid.time_low,
			 trusty_app->props.uuid.time_mid,
			 trusty_app->props.uuid.time_hi_and_version);

		/* entry is 0 at this point since we haven't parsed the elf hdrs
		 * yet */
		 /* 创建一个用户线程 */
		Elf32_Ehdr *elf_hdr = trusty_app->app_img;
		uthread = uthread_create(name, elf_hdr->e_entry,
					 DEFAULT_PRIORITY, TRUSTY_APP_STACK_TOP,
					 trusty_app->props.min_stack_size, trusty_app);
		if (uthread == NULL) {
			/* TODO: do better than this */
			panic("allocate user thread failed\n");
		}
		trusty_app->ut = uthread;/* 把线程绑定到trusty app */

		ret = alloc_address_map(trusty_app);/* 内存映射处理 */
		if (ret != NO_ERROR) {
			panic("failed to load address map\n");
		}

		/* attach als_cnt */
		/* 给程序创建局部存储空间 */
		trusty_app->als = calloc(1, als_slot_cnt * sizeof(void*));
		if (!trusty_app->als) {
			panic("allocate app local storage failed\n");
		}

		/* call all registered startup notifiers */
		/* 调用初始化 */
		trusty_app_notifier_t *n;
		list_for_every_entry(&app_notifier_list, n, trusty_app_notifier_t, node) {
			if (n->startup) {
				ret = n->startup(trusty_app);
				if (ret != NO_ERROR)
					panic("failed (%d) to invoke startup notifier\n", ret);
			}
		}
	}
}
```

### 执行

执行也通过初始化钩子完成

```c
static void start_apps(uint level)
{
	trusty_app_t *trusty_app;
	u_int i;
	int ret;
	/* 此时加载操作已经完才trusty_app_list、trusty_app_count初始化已经完成 */
	for (i = 0, trusty_app = trusty_app_list; i < trusty_app_count; i++, trusty_app++) {
		if (trusty_app->ut->entry) {/* 确认具有入口函数 */
			ret = uthread_start(trusty_app->ut);
			if (ret)
				panic("Cannot start Trusty app!\n");
		}
	}
}
/* 初始化钩子 用与启动app */
LK_INIT_HOOK(libtrusty_apps, start_apps, LK_INIT_LEVEL_APPS + 1);
```

执行过程与uthread实现细节相关。uthread创建时，会创建一个little kernel线程，线程入口为**arch_uthread_startup**，并把uthread_t绑定到线程的局部存储。**arch_uthread_startup**执行时会访问线程句柄存储找到uthread_t，通过uthread_t找到uthread的入口entry。在异常退出后执行对应的代码

uthread创建过程

```c
uthread_t *uthread_create(const char *name, vaddr_t entry, int priority,
		vaddr_t start_stack, size_t stack_size, void *private_data)
{
	uthread_t *ut = NULL;
	status_t err;
	vaddr_t stack_bot;
	spin_lock_saved_state_t state;

	/* 为用户线程描述符申请内存空间 */
	ut = (uthread_t *)calloc(1, sizeof(uthread_t));
	if (!ut)
		goto err_done;

	/* 初始化内存映射表（链表初始化） */
	list_initialize(&ut->map_list);
	/* 初始化内存映射表的锁，防止多线程操作内存映射表破坏数据 */
	mutex_init(&ut->mmap_lock);

	ut->id = uthread_alloc_utid();	/* 为线程分配线程id */
	ut->private_data = private_data;/* 初始化线程私有数据，一般为trusty_app_t */
	ut->entry = entry;				/* 线程入口函数 */
	/* 初始化虚拟地址空间分解
	   大于ns_va_bottom ns(Non-Secure)内存
	   小于ns_va_bottom sec(Secure)内存*/
	ut->ns_va_bottom = MAX_USR_VA;

	/* Allocate and map in a stack region */
	/* 分配栈空间 */
	ut->stack = memalign(PAGE_SIZE, stack_size);
	if(!ut->stack)
		goto err_free_ut;

	/* 把栈映射到用户线程空间 */
	stack_bot = start_stack - stack_size;
	err = uthread_map_contig(ut, &stack_bot, vaddr_to_paddr(ut->stack),
				stack_size,
				UTM_W | UTM_R | UTM_STACK | UTM_FIXED,
				UT_MAP_ALIGN_4KB);
	if (err)
		goto err_free_ut_stack;

	ut->start_stack = start_stack;

	/* arch_uthread_startup在异常退出后执行uthread->entry */
	ut->thread = thread_create(name,
			(thread_start_routine)arch_uthread_startup,
			NULL,
			priority,
			DEFAULT_STACK_SIZE);
	if (!ut->thread)
		goto err_free_ut_maps;

	/* 架构相关的线程操作 */
	err = arch_uthread_create(ut);
	if (err)
		goto err_free_ut_maps;

	/* store user thread struct into TLS slot 0 */
	/* lk线程局部存储中放入uthread标示 */
	ut->thread->tls[TLS_ENTRY_UTHREAD] = (uintptr_t) ut;

	/* Put it in global uthread list */
	/* 把uthread加入到uthread_list列表 */
	spin_lock_save(&uthread_lock, &state, SPIN_LOCK_FLAG_INTERRUPTS);
	list_add_head(&uthread_list, &ut->uthread_list_node);
	spin_unlock_restore(&uthread_lock, state, SPIN_LOCK_FLAG_INTERRUPTS);

	return ut;

err_free_ut_maps:
	uthread_free_maps(ut);

err_free_ut_stack:
	free(ut->stack);

err_free_ut:
	uthread_free_utid(ut->id);
	free(ut);

err_done:
	return NULL;
}
```

uthread执行

```c
/* 启动线程 */
status_t uthread_start(uthread_t *ut)
{
	if (!ut || !ut->thread)
		return ERR_INVALID_ARGS;

	return thread_resume(ut->thread);
}

/* 实际启动的线程 */
void arch_uthread_startup(void)
{
	/* 通过线程局部存储找到uthread标示 */
	struct uthread *ut = (struct uthread *) tls_get(TLS_ENTRY_UTHREAD);
  	/* 获取栈地址 */
	register uint64_t sp_usr asm("x2") = ROUNDDOWN(ut->start_stack, 8);
  	/* 获取程序入口地址 */
	register uint64_t entry asm("x3") = ut->entry;

	/* 配置一些执行环境，在异常退出后执行指定的代码uthread_t->entry */
	__asm__ volatile(
		"mov	x0, #0\n"
		"mov	x1, #0\n"
		"mov	x13, %[stack]\n" 	/* AArch32 SP_usr */
		"mov	x14, %[entry]\n" 	/* AArch32 LR_usr */
		"mov	x9, #0x10\n"
		"msr	spsr_el1, x9\n"		/* Mode = AArch32 User */
		"msr	elr_el1, %[entry]\n"/* 异常退出后的入口 */
		"eret\n"
		:
		: [stack]"r" (sp_usr), [entry]"r" (entry)
		: "x0", "x1", "memory"
	);
}
```

## 线程间通信

因为Trusty App为独立的ELF文件，所以TOS需要实现线程间通信。

为了实现通信，使用trusty_app_t的局部存储，绑定了一个轮询结构

### trusty app局部存储

即给app绑定一些私有信息。要使用句柄存储首先需要申请，获取一个id。然后通过这个id可以读写局部存储。

局部存储申请

```c
/* 申请一个局部存储空间，返回可用的局部存储空间编号（1-n）
 * 应为句柄存储在trysty_app启动时创建，所以申请空间操作必须在app启动之前
 */
int trusty_als_alloc_slot(void)
{
	int ret;

	mutex_acquire(&apps_lock);
	if (!apps_started)
		ret = ++als_slot_cnt;
	else
		ret = ERR_ALREADY_STARTED;
	mutex_release(&apps_lock);
	return ret;
}
```

局部存储操作

```c
static inline void *trusty_als_get(struct trusty_app *app, int slot_id)
{
	uint slot = slot_id - 1;
	ASSERT(slot < als_slot_cnt);
	return app->als[slot];
}

static inline void trusty_als_set(struct trusty_app *app, int slot_id, void *ptr)
{
	uint slot = slot_id - 1;
	ASSERT(slot < als_slot_cnt);
	app->als[slot] = ptr;
}
```

### trusty app的消息处理

这套框架构建了一个机制，在引用trusty app发生特定事件（现在只有app启动和关闭的事件）时执行相关的操作。主要通过**trusty_app_notifier_t**结构描述事件处理方法，要添加一个处理只需要在**app_notifier_list**链表中添加一个**trusty_app_notifier_t**结构体。

```c
/* app消息处理器（在app接受到特定消息时执行特定操作） */
typedef struct trusty_app_notifier
{
	struct list_node node;/* 用于构成链表 */
	status_t (*startup)(trusty_app_t *app);		/* app启动时执行 */
	status_t (*shutdown)(trusty_app_t *app);	/* app关闭时执行 */
} trusty_app_notifier_t;
```

### 轮询事件

用于实现轮询（多个用户线程调用陷入函数进入内核，内核通过轮询处理多个线程的等待）

**handle_ops**用于描述轮询方法
```c
struct handle_ops {
	uint32_t (*poll)(handle_t *handle);/* 轮询方法 */

	/* 轮询成功触发此函数、event为轮询方法返回的值 */
	void (*finalize_event)(handle_t *handle, uint32_t event);
	void (*shutdown)(handle_t *handle);	/* 关闭时触发此方法 */
	void (*destroy)(handle_t *handle);	/* 引用计算为0时触发 */
};
```

**handle_t**用于描述一个轮询对象

```c
typedef struct handle {
	refcount_t		refcnt;		/* 引用计数 */

	struct handle_ops	*ops;	/* 轮询相关操作 */

	/* pointer to a wait queue on which threads wait for events for this
	 * handle.
	 */
	event_t			*wait_event;	/* 要等待的事件 */
	spin_lock_t		slock;			/* 锁，用于保护wait_event */

	struct list_node	hlist_node;	/* 用于构成链表 */

	void			*cookie;
} handle_t;
```

**handle_list_t**用于描述一组轮询对象（用于等待多个轮询事件）

```c
typedef struct handle_list {
	struct list_node	handles;		/* handle的链表 */
	mutex_t				lock;		 	/* 锁，保护handle */
	event_t				*wait_event;	/* 要等待的事件 */
} handle_list_t;
```

等待操作**handle_wait** 用于等待一个轮询事件

```c
/* handle要轮询的对象 handle_event轮询对象返回值 timeout等待时间 */
int handle_wait(handle_t *handle, uint32_t *handle_event, lk_time_t timeout)
{
	uint32_t event;
	event_t ev;
	int ret = 0;

	if (!handle || !handle_event)
		return ERR_INVALID_ARGS;

	/* 初始化event 用于等待时让出cpu资源 */
	event_init(&ev, false, EVENT_FLAG_AUTOUNSIGNAL);

	/* 给handle绑定event资源 */
	ret = _prepare_wait_handle(&ev, handle);
	if (ret) {/* handle已经绑定event，说明已经有等待操作了 */
		LTRACEF("someone is already waiting on handle %p\n", handle);
		goto err_prepare_wait;
	}

	while (true) {
		event = handle->ops->poll(handle);/* 轮询 */
		if (event)/* 轮询成功退出 */
			break;
		ret = __do_wait(&ev, timeout);/* 等待 */
		if (ret < 0)
			goto finish_wait;/* 超时 */
		/* 等待成功 */
	}

	if (handle->ops->finalize_event)/* handle有事件处理函数，调用 */
		handle->ops->finalize_event(handle, event);
	*handle_event = event;/* 记录等待到的事件 */
	ret = NO_ERROR;

finish_wait:
	_finish_wait_handle(handle);/* 取消handle与event的绑定 */
err_prepare_wait:
	event_destroy(&ev);/* 销毁event事件 */
	return ret;
}
```

等待操作**handle_list_wait** 用于等待一组轮询事件

```c
 /* 等待一组轮询事件
  * hlist 要等待的handle list对象
  * handle_ptr 返回值，轮询成功的对象
  * event_ptr  返回值，轮询方法的返回值
  * timeout	   等待时间
  */
int handle_list_wait(handle_list_t *hlist, handle_t **handle_ptr,
                     uint32_t *event_ptr, lk_time_t timeout)
{
	int ret;
	event_t ev;

	DEBUG_ASSERT(hlist);
	DEBUG_ASSERT(handle_ptr);
	DEBUG_ASSERT(event_ptr);

	/* 初始化一个event 用于等待时让出cpu资源 */
	event_init(&ev, false, EVENT_FLAG_AUTOUNSIGNAL);

	*event_ptr = 0;
	*handle_ptr = 0;

	/* 防止多个线程同时操作handle list，破坏数据 */
	mutex_acquire(&hlist->lock);

	DEBUG_ASSERT(hlist->wait_event == NULL);

	hlist->wait_event = &ev;
	/* 对handle list中的对象轮询一次 */
	ret = _hlist_do_poll_locked(hlist, handle_ptr, event_ptr, true);
	if (ret < 0)
		goto err_do_poll;/* handle list中有对象被等待 */

	if (ret == 0) {/* 轮询不成功 */
		do {
			/* 等待 */
			mutex_release(&hlist->lock);
			ret = __do_wait(&ev, timeout);
			mutex_acquire(&hlist->lock);

			if (ret < 0)
				break;/* 超时 */
			/* event等待成功 */

			/* 重新轮询一次 */
			ret = _hlist_do_poll_locked(hlist, handle_ptr,
						    event_ptr, false);
		} while (!ret);
		/* handle list中的对象解邦event */
		_hlist_finish_wait_locked(hlist, NULL);
	}

	if (ret == 1) {/* 轮询成功 */
		handle_t *handle = *handle_ptr;/* 记录轮询成功的句柄 */

		/* 轮询成功调用对应的消息处理函数 */
		if (handle->ops->finalize_event)
			handle->ops->finalize_event(handle, *event_ptr);

		handle_incref(handle);

		/* move list head after item we just found */
		/* 把轮询成功的handle移动到链表的尾部，下次优先轮询其他事件 */
		list_delete(&hlist->handles);
		list_add_head(&handle->hlist_node, &hlist->handles);

		ret = NO_ERROR;
	}

err_do_poll:
	hlist->wait_event = NULL;/* 解邦event */
	mutex_release(&hlist->lock);
	event_destroy(&ev);/* 销毁event事件 */
	return ret;
}

```

### 轮询事件绑定到trusty app

为了把轮询事件绑定到用户线程，在应用程序创建之前为每个应用创建局部存储，并通过消息处理设置局部存储**uctx**

```c
/* trusty_app启动时初始化uctx */
static status_t _uctx_startup(trusty_app_t *app);

static uint _uctx_slot_id;/* 申请到的trusty_app句柄存储的编号 */

/* 给trusty_app注册的消息处理器 */
static struct trusty_app_notifier _uctx_notifier = {
	.startup = _uctx_startup,
};

static status_t _uctx_startup(trusty_app_t *app)
{
	uctx_t *uctx;

	int err = uctx_create(app, &uctx);/* 创建uctx */
	if (err)
		return err;

	trusty_als_set(app, _uctx_slot_id, uctx);/* 绑定uctx到app的局部存储 */
	return NO_ERROR;
}

static void uctx_init(uint level)
{
	int res;

	/* allocate als slot */
	/* 为app申请句柄存储 用于绑定uctx */
	res = trusty_als_alloc_slot();
	if (res < 0)
		panic("failed (%d) to alloc als slot\n", res);
	_uctx_slot_id = res;

	/* register notifier */
	/* 注册消息处理方法 初始化uctx并绑定到局部存储 */
	res = trusty_register_app_notifier(&_uctx_notifier);
	if (res < 0)
		panic("failed (%d) to register uctx notifier\n", res);
}

/* 初始化钩子 */
LK_INIT_HOOK(uctx, uctx_init, LK_INIT_LEVEL_APPS - 2);
```

### uctx

**uctx**用于给app绑定轮询事件，数据结构如下

```c
/* 一个app最多轮询的事件数 */
#define IPC_MAX_HANDLES		64

struct uctx {
	/* 位图 标记handles中那些指针被使用了 */
	unsigned long	inuse[BITMAP_NUM_WORDS(IPC_MAX_HANDLES)];
	handle_t		*handles[IPC_MAX_HANDLES];/* 通过id速查handle */
	void			*priv;/* 指向对应的trusty_app */
	handle_list_t	handle_list;/* 记录下被等待的handle */
};
```

**uctx**创建了一些函数用于绑定**handle** 、解除绑定**handle**、通过id获取**handle**

```c
 /* 在uctx中添加一个handle
  * handle要添加的handle
  * id返回变量，添加进uctx的handle的id
  */
int uctx_handle_install(uctx_t *ctx, handle_t *handle, handle_id_t *id);

 /* 根据id获取handle
  * handle_id 要查找的handle的id
  * handle_ptr 返回值，查找到的handle
  */
int uctx_handle_get(uctx_t *ctx, handle_id_t handle_id, handle_t **handle_ptr)；
 
 /* 从uctx中移除一个handle
  * handle_id要移除的handle的id
  * handle_ptr返回变量，移除的handle
  */
int uctx_handle_remove(uctx_t *ctx, handle_id_t handle_id, handle_t **handle_ptr)；
```

### IPC

线程通信主要实现类似与socket的通信框架。通信分为服务器和客户端，服务器创建端口**ipc_port_t**，然后等待客户端链接，客户端链接后，就可以相互发送消息。

**IPC**主要由端口（**ipc_port**）和链接（**ipc_chan**）组成。**ipc_port**、**ipc_chan**都是内核轮询对象，通过包含**handle**实现。

#### 通信端口

服务器，用于维护所有链接的数据结构。

```c
/* 端口，服务器创建，可以由多个客户端链接 */
typedef struct ipc_port {
	/* e.g. /service/sys/crypto, /service/usr/drm/widevine */
	char				path[IPC_PORT_PATH_MAX];/* 端口路径（名字） */
	const struct uuid	*uuid;

	uint32_t			state;	/* 端口状态，用于确定通道可用 */
	uint32_t			flags;	/* 端口权限，用于确定那些客户端可以连接（connect） */

	uint				num_recv_bufs;
	size_t				recv_buf_size;

	handle_t			handle;

	/* 每次connect创建一对链接ipc_chan，此链表保存这些链接的server */
	struct list_node	pending_list;

	struct list_node	node;/* 链表ipc_port_list节点  */
} ipc_port_t;
```

端口有状态，记录端口是否可用

```c
/* 标记一个IPC port状态 */
enum {
	IPC_PORT_STATE_INVALID		= 0,	/* 创建时的状态，创建中不可用 */
	IPC_PORT_STATE_LISTENING	= 1,	/* 创建完成，等待连接 */
	IPC_PORT_STATE_CLOSING		= 2,	/* 销毁中，标记正在关闭通道，通道不可用 */
};
```

端口有权限控制

```c
/* 通道的权限 */
enum {
	IPC_PORT_ALLOW_TA_CONNECT	= 0x1,	/* 允许trusty App连接此通道 */
	IPC_PORT_ALLOW_NS_CONNECT	= 0x2,	/* 允许Non-Secure链接此通道 */
};
```


#### 连接

一个连接对应一对**ipc_chan**，**ipc_chan**通过**peer**指向自己的通信对象（server指向client，client指向server）。

```c
typedef struct ipc_chan {
	obj_t				refobj;
	obj_ref_t			peer_ref;

	/* 一对链接 ipc_chan指向ipc_chan，一个server一个client*/
	struct ipc_chan		*peer;
	const struct uuid	*uuid;		/* 对应程序的uuid */

	uint32_t			state;		/* 连接的状态 */
	uint32_t			flags;		/* 标记链接是server或client */
	uint32_t			aux_state;	/* 附加状态，主要用在通知peer改变状态时 */

	/* handle_ref is a self reference when there are
	 * outstanding handles out there. It is removed
	 * when last handle ref goes away.
	 */
	obj_ref_t			handle_ref;
	handle_t			handle;		/* 对应的轮询句柄 */

	/* used for port's pending list. node_ref field is a
	 * self reference when node field is inserted in the list.
	 *
	 * TODO: consider creating generic solution by grouping
	 * together list_node and obj_ref_t into single struct.
	 */
	obj_ref_t			node_ref;
	struct list_node	node; 		/* 在port->pending_list构成链表 */

	ipc_msg_queue_t		*msg_queue;	/* 用于存放消息 */

	/*
	 * TODO: consider changing async connect to preallocate
	 *       not-yet-existing port object then we can get rid
	 *       of this field.
	 */
	/* client链接server时如果server不存在就把要连接的server的路径保存在这儿 */
	const char			*path;
} ipc_chan_t;
```

标记连接是服务器还是客户端

```c
/* 标记一个ipc_chans是服务器还是客户端  */
enum {
	IPC_CHAN_FLAG_SERVER		= 0x1,
};
```

连接状态通过枚举实现

```c
/* 连接的状态 */
enum {
	IPC_CHAN_STATE_INVALID		= 0,		/* 创建中不能用于轮询 */
	IPC_CHAN_STATE_ACCEPTING	= 1,		/* connect执行后server的状态 */
	IPC_CHAN_STATE_CONNECTING	= 2,		/* connect执行后client的状态 */
	IPC_CHAN_STATE_CONNECTED	= 3,		/* accept执行后的状态 */
	IPC_CHAN_STATE_DISCONNECTING	= 4,	/* 端口关闭时的状态 */
	/* client发起链接时如果server不存在，client进入此状态 */
	IPC_CHAN_STATE_WAITING_FOR_PORT	= 5,
};

/* 连接的附加状态，用在peer执行指定操作后，当前ipc_chan改变状态 */
enum {
	/* 发送消息时ip_chan中没有空闲的消息队列可用时 进入此状态 */
	IPC_CHAN_AUX_STATE_SEND_BLOCKED = 0x1,
	/* 接受消息后
	 * 如果peer对象是IPC_CHAN_AUX_STATE_SEND_BLOCKED状态
	 * 就标记为IPC_CHAN_AUX_STATE_SEND_UNBLOCKED */
	IPC_CHAN_AUX_STATE_SEND_UNBLOCKED = 0x2,
	/* 用于通知client server接受了链接（执行了accept） */
	IPC_CHAN_AUX_STATE_CONNECTED = 0x4,
};
```

#### 消息队列

消息队列由**ipc_msg_queue_t**描述

**ipc_msg_queue_t**包含三个链表（**free_list**、**filled_list**、**read_list**）用于记录哪些空间是空闲的、哪些空间是有数据的、哪些空间是可读的

实际的存储空间在**ipc_msg_queue_t**的一个**buf**对象中

**msg_item_t**用于记录存储空间信息（状态、用了多少空间）

```c
/* item的状态 */
enum {
	MSG_ITEM_STATE_FREE	= 0,		/* 空闲的 */
	MSG_ITEM_STATE_FILLED	= 1,	/* 填充过 */
	MSG_ITEM_STATE_READ	= 2,		/* 可读 */
};

typedef struct msg_item {
	uint8_t			id;		/* 编号，对应器在ipc_msg_queue->items中的数组下标 */
	uint8_t			state;	/* 状态信息 */
	uint			num_handles;				/* 未用 */
	handle_id_t		handles[MAX_MSG_HANDLES];	/* 未用 */
	size_t			len;						/* 使用的大小 */
	struct list_node	node;
} msg_item_t;

typedef struct ipc_msg_queue {
	struct list_node	free_list;	/* 链表，记录空闲的item */
	struct list_node	filled_list;/* 链表，记录有数据的item */
	struct list_node	read_list;	/* 链表，记录可以读的item */

	uint			num_items;		/* item的个数 */
	size_t			item_sz;		/* item空间大小 */

	/* 实际的存储空间
	 * 大小 num_items * item_sz
	 * 第n个item的存储起始地址 buf + n * item_sz
	 */
	uint8_t			*buf;

	/* store the message descriptors in the queue,
	 * and the buffer separately. The buffer could
	 * eventually move to a separate area that can
	 * be mapped into the process directly.
	 */
	msg_item_t		items[0];
} ipc_msg_queue_t;
```

#### 主要方法

```c
/* 创建一个端口，由服务器调用
 * sid 服务器程序的uuid
 * path 路径，用于标示一个端口。客户段通过此路径连接此服务
 * num_recv_bufs 消息队列大小
 * recv_buf_size 消息队列中一个单元的大小
 * flags 端口的权限
 * phandle_ptr 返回值，返回端口的轮询句柄
 */
int ipc_port_create(const uuid_t *sid, const char *path,
                    uint num_recv_bufs, size_t recv_buf_size,
                    uint32_t flags,  handle_t **phandle_ptr);

/* 服务器端发布端口（建立端口和链接的关系） */
int ipc_port_publish(handle_t *phandle);

/* 接受一个链接
 * phandle 为port的轮询handle
 * chandle_ptr 返回值，server的轮询handle
 * uuid_ptr 返回值，client的uuid
 */
int ipc_port_accept(handle_t *phandle, handle_t **chandle_ptr,
                    const uuid_t **uuid_ptr);

/* 客户端链接到服务器
 * cid 客户段的uuid
 * path 要连接的服务器的路径
 * max_path 用于限制path的长度
 * flags 用于标记是否可以在port创建之前调用此函数，如果可以
 *       此创建的客户端连接ipc_chan将会加入一个空闲队列waiting_for_port_chan_list
 * chandle_ptr 返回值，返回创建的链接的轮询句柄
 */
int ipc_port_connect_async(const uuid_t *cid, const char *path, size_t max_path,
			   uint flags, handle_t **chandle_ptr);

/* 把读过的消息从filled_list删除插入到read_list */
int ipc_get_msg(handle_t *chandle, ipc_msg_info_t *msg_info);
/* 读取一条消息 */
int ipc_read_msg(handle_t *chandle, uint32_t msg_id, uint32_t offset,
		 ipc_msg_kern_t *msg);
/* 把读过的消息从read_list删除插入到free_list */
int ipc_put_msg(handle_t *chandle, uint32_t msg_id);
/* 发送一条消息 */
int ipc_send_msg(handle_t *chandle, ipc_msg_kern_t *msg);
```

## 系统调用

### 异常框架

分析系统调用需要分析异常处理代码

以下是异常入口代码，可以看出系统调用在app和OS中都可以执行（**regsave_long**保存现场、**arm64_sync_exc_lower_el_32**恢复现场并退出异常）。**arm64_sync_exception**为同步异常处理函数，接受现场保护的堆栈地址作为参数。

```assembly
/* exceptions from current EL, using SPx */
.org 0x200
LOCAL_FUNCTION(arm64_sync_exc_current_el_SPx)
    regsave_long
    mov x0, sp
    bl  arm64_sync_exception
    b  arm64_exc_shared_restore_long

/* exceptions from lower EL, running arm32 */
.org 0x600
LOCAL_FUNCTION(arm64_sync_exc_lower_el_32)
    regsave_long
    mov x0, sp
    bl  arm64_sync_exception
    b  arm64_exc_shared_restore_long
```

同步异常处理函数。主要处理系统调用和浮点异常

```c
void arm64_sync_exception(struct arm64_iframe_long *iframe)
{
    struct fault_handler_table_entry *fault_handler;
    uint32_t esr = ARM64_READ_SYSREG(esr_el1);/* 获取esr_el1的值 */
    /* 获取异常原因 */
    uint32_t ec = esr >> 26;
    uint32_t il = (esr >> 25) & 0x1;
    uint32_t iss = esr & ((1<<24) - 1);

#ifdef WITH_LIB_SYSCALL
    if (ec == 0x15 || ec == 0x11) { // syscall 64/32
        /* 判断是系统调用 */
        void arm64_syscall(struct arm64_iframe_long *iframe);
        arch_enable_fiqs();/* 使能fiq */
        arm64_syscall(iframe);/* 系统调用实际函数 */
        arch_disable_fiqs();/* 失能fiq */
        return;
    }
#endif

    /* floating point */
    if (ec == 0x07) {/* 浮点异常 */
        arm64_fpu_exception(iframe);
        return;
    }

    for (fault_handler = __fault_handler_table_start; fault_handler < __fault_handler_table_end; fault_handler++) {
        if (fault_handler->pc == iframe->elr) {
            iframe->elr = fault_handler->fault_handler;
            return;
        }
    }

    printf("sync_exception\n");
    dump_iframe(iframe);

    printf("ESR 0x%x: ec 0x%x, il 0x%x, iss 0x%x\n", esr, ec, il, iss);

    if (ec == 0x15) { // syscall
        printf("syscall\n");
        return;
    }
    /* 出错 */
    panic("die\n");
}
```

系统调用实际函数。通过分析，**W12指定异常调用的编号，X0-X3为系统调用参数，X0为返回值，nr_syscalls为系统调用的个数，syscall_table是一个数组存放系统调用的编号。**

```assembly
FUNCTION (arm64_syscall)
	push	x0, x30            /* 保存x0 x30 */
	ldr	w12, [x0, #(12 << 3)]  /* 从堆栈中获取w12(系统调用号) */

	msr	daifclr, #DAIF_MASK_IAF/* 禁止中断 */

	ldr	x14, =nr_syscalls
	ldr	x14, [x14]             /* 获取系统调用个数 */
	cmp	x12, x14
	b.hs	.Lundefined        /* 系统调用号必须小于系统调用个数 */
	ldr	x14, =syscall_table    /* 获取系统调用函数的起始地址 */
	ldr	x14, [x14, x12, lsl#3] /* 获取系统调用句柄 */
	cbnz	x14, .Ldefined
.Lundefined:
	ldr	x14,=sys_undefined     /* 系统调用句柄为0，执行sys_undefined */
.Ldefined:
	ldp	x2, x3, [x0, #16]      /* 从堆栈中获取系统调用参数 x2 x3 */
	ldp	x0, x1, [x0]           /* 从堆栈中获取系统调用参数 x0 x1 */
	blr	x14                    /* 执行系统调用 */

	msr	daifset, #DAIF_MASK_IAF/* 使能中断 */

	pop	x1, x30
	str	x0, [x1, 0]            /* 返回值写入堆栈 */

	ret
```

### 系统调用声明

声明通过**lk/trusty/lib/trusty/include/syscall_table.h**文件声明

生成变量**nr_syscalls**、**syscall_table**通过文件**lk/trusty/lib/syscall/syscall.c**实现

```c
/* 用于处理不存在的系统调用 */
long sys_undefined(int num)
{
	dprintf(SPEW, "%p invalid syscall %d requested\n", get_current_thread(), num);
	return ERR_NOT_SUPPORTED;
}

#ifdef WITH_SYSCALL_TABLE

/* Generate fake function prototypes */
/* 函数声明 */
#define DEF_SYSCALL(nr, fn, rtype, nr_args, ...) rtype sys_##fn (void);
#include <syscall_table.h>
#undef DEF_SYSCALL

#endif

/* 系统调用函数入口地址表（数组 下标为系统调用编号，值为函数入口地址） */
#define DEF_SYSCALL(nr, fn, rtype, nr_args, ...) [(nr)] = (unsigned long) (sys_##fn),
const unsigned long syscall_table [] = {

#ifdef WITH_SYSCALL_TABLE
#include <syscall_table.h>
#endif

};
#undef DEF_SYSCALL

/* 系统调用的个数 */
unsigned long nr_syscalls = countof(syscall_table);
```

### 用户空间处理

因为有明确的系统调用规范，用户空间处理比较简单

声明位于**lib/include/trusty_syscalls.h**，声明系统调用的编号以及函数声明

```c
/* 系统调用编号声明 */
#define __NR_write		0x1
#define __NR_brk		0x2
#define __NR_exit_group		0x3
#define __NR_read		0x4
#define __NR_ioctl		0x5
#define __NR_nanosleep		0x6
#define __NR_gettime		0x7
#define __NR_mmap		0x8
#define __NR_munmap		0x9
#define __NR_prepare_dma		0xa
#define __NR_finish_dma		0xb
#define __NR_port_create		0x10
#define __NR_connect		0x11
#define __NR_accept		0x12
#define __NR_close		0x13
#define __NR_set_cookie		0x14
#define __NR_wait		0x18
#define __NR_wait_any		0x19
#define __NR_get_msg		0x20
#define __NR_read_msg		0x21
#define __NR_put_msg		0x22
#define __NR_send_msg		0x23

#ifndef ASSEMBLY

__BEGIN_CDECLS

/* 函数声明 */
long write (uint32_t fd, void* msg, uint32_t size);
long brk (uint32_t brk);
long exit_group (void);
long read (uint32_t fd, void* msg, uint32_t size);
long ioctl (uint32_t fd, uint32_t req, void* buf);
long nanosleep (uint32_t clock_id, uint32_t flags, uint64_t sleep_time);
long gettime (uint32_t clock_id, uint32_t flags, int64_t *time);
long mmap (void* uaddr, uint32_t size, uint32_t flags, uint32_t handle);
long munmap (void* uaddr, uint32_t size);
long prepare_dma (void* uaddr, uint32_t size, uint32_t flags, void* pmem);
long finish_dma (void* uaddr, uint32_t size, uint32_t flags);
long port_create (const char *path, uint num_recv_bufs, size_t recv_buf_size, uint32_t flags);
long connect (const char *path, uint flags);
long accept (uint32_t handle_id, uuid_t *peer_uuid);
long close (uint32_t handle_id);
long set_cookie (uint32_t handle, void *cookie);
long wait (uint32_t handle_id, uevent_t *event, unsigned long timeout_msecs);
long wait_any (uevent_t *event, unsigned long timeout_msecs);
long get_msg (uint32_t handle, ipc_msg_info_t *msg_info);
long read_msg (uint32_t handle, uint32_t msg_id, uint32_t offset, ipc_msg_t *msg);
long put_msg (uint32_t handle, uint32_t msg_id);
long send_msg (uint32_t handle, ipc_msg_t *msg);

__END_CDECLS
```

实现位于**lib/lib/libc-trusty/arch/arm/trusty_syscall.S**

```assembly
.section .text.write
FUNCTION(write)
    ldr     r12, =__NR_write	/* 系统调用编号 */
    swi     #0					/* 触发系统调用 */
    bx      lr					/* 函数返回 */

.section .text.brk
FUNCTION(brk)
    ldr     r12, =__NR_brk
    swi     #0
    bx      lr

.section .text.exit_group
FUNCTION(exit_group)
    ldr     r12, =__NR_exit_group
    swi     #0
    bx      lr

.section .text.read
FUNCTION(read)
    ldr     r12, =__NR_read
    swi     #0
    bx      lr

.section .text.ioctl
FUNCTION(ioctl)
    ldr     r12, =__NR_ioctl
    swi     #0
    bx      lr

.section .text.nanosleep
FUNCTION(nanosleep)
    ldr     r12, =__NR_nanosleep
    swi     #0
    bx      lr

.section .text.gettime
FUNCTION(gettime)
    ldr     r12, =__NR_gettime
    swi     #0
    bx      lr

.section .text.mmap
FUNCTION(mmap)
    ldr     r12, =__NR_mmap
    swi     #0
    bx      lr

.section .text.munmap
FUNCTION(munmap)
    ldr     r12, =__NR_munmap
    swi     #0
    bx      lr

.section .text.prepare_dma
FUNCTION(prepare_dma)
    ldr     r12, =__NR_prepare_dma
    swi     #0
    bx      lr

.section .text.finish_dma
FUNCTION(finish_dma)
    ldr     r12, =__NR_finish_dma
    swi     #0
    bx      lr

.section .text.port_create
FUNCTION(port_create)
    ldr     r12, =__NR_port_create
    swi     #0
    bx      lr

.section .text.connect
FUNCTION(connect)
    ldr     r12, =__NR_connect
    swi     #0
    bx      lr

.section .text.accept
FUNCTION(accept)
    ldr     r12, =__NR_accept
    swi     #0
    bx      lr

.section .text.close
FUNCTION(close)
    ldr     r12, =__NR_close
    swi     #0
    bx      lr

.section .text.set_cookie
FUNCTION(set_cookie)
    ldr     r12, =__NR_set_cookie
    swi     #0
    bx      lr

.section .text.wait
FUNCTION(wait)
    ldr     r12, =__NR_wait
    swi     #0
    bx      lr

.section .text.wait_any
FUNCTION(wait_any)
    ldr     r12, =__NR_wait_any
    swi     #0
    bx      lr

.section .text.get_msg
FUNCTION(get_msg)
    ldr     r12, =__NR_get_msg
    swi     #0
    bx      lr

.section .text.read_msg
FUNCTION(read_msg)
    ldr     r12, =__NR_read_msg
    swi     #0
    bx      lr

.section .text.put_msg
FUNCTION(put_msg)
    ldr     r12, =__NR_put_msg
    swi     #0
    bx      lr

.section .text.send_msg
FUNCTION(send_msg)
    ldr     r12, =__NR_send_msg
    swi     #0
    bx      lr
```

## 与Monitor通信

**trusty**是32位程序，所以只能处理**monitor**交给它的32位的SMC调用（访问不了64为寄存器）。

根据[SMC调用规范](http://infocenter.arm.com/help/topic/com.arm.doc.den0028b/ARM_DEN0028B_SMC_Calling_Convention.pdf)可知：

>W0用于传递SMC调用标示（即功能）
>W1-W3用于传递参数
>W0用于返回调用结果

调用标示具体描述如下

```
b31    调用类型 1->Fast Call 0->Yielding Call
b30    标示32bit还是64bit调用
b29:24 服务类型
        0x0        ARM架构相关
        0x1        CPU服务
        0x2        SiP服务
        0x3        OEM服务
        0x4        标准安全服务
        0x5        标准Hypervisor服务
        0x6        厂商定制的Hypervisor服务
        0x07-0x2f  预留
        0x30-0x31  Trusted Application Calls
        0x32-0x3f  Trusted OS Calls
b23:16 在Fast Call时这些比特位必须为0，其他值保留未用，ARMv7传统的Trusted OS在Fast Call时必须为1
b15:0  函数号
```

### SMC服务类型描述

此不分主要用来声明常量（调用标示相关）

```c
#define SMC_NUM_ENTITIES	64	/* 单个entry下的函数个数 */
#define SMC_NUM_ARGS		4	/* SMC调用参数个数 */
#define SMC_NUM_PARAMS		(SMC_NUM_ARGS - 1)/* 去除功能描述w0后的参数个数 */

#define SMC_IS_FASTCALL(smc_nr)	((smc_nr) & 0x80000000)/* 判断一个功能号是fastcall */
#define SMC_IS_SMC64(smc_nr) ((smc_nr) & 0x40000000)/* 判断一个功能号是64位调用 */
#define SMC_ENTITY(smc_nr) (((smc_nr) & 0x3F000000) >> 24)/* 获取一个功能号的entry */
#define SMC_FUNCTION(smc_nr) ((smc_nr) & 0x0000FFFF)/* 获取一个功能号的函数编号 */

/* 宏，辅助生成一个功能号
 * entry 	服务类型
 * fn		功能号
 * fastcall 标示调用类型 1->fastcall 0->stdcall
 * smc64	标示调用为64位调用
 */
#define SMC_NR(entity, fn, fastcall, smc64) ((((fastcall) & 0x1) << 31) | \
					     (((smc64) & 0x1) << 30) | \
					     (((entity) & 0x3F) << 24) | \
					     ((fn) & 0xFFFF) \
					    )

/* 辅助生成一个32位fastcall的功能号 */
#define SMC_FASTCALL_NR(entity, fn)	SMC_NR((entity), (fn), 1, 0)

/* 辅助生成一个32位stdcall的功能号 */
#define SMC_STDCALL_NR(entity, fn)	SMC_NR((entity), (fn), 0, 0)

/* 辅助生成一个64位fastcall的功能号 */
#define SMC_FASTCALL64_NR(entity, fn)	SMC_NR((entity), (fn), 1, 1)

/* 辅助生成一个64位stdcall的功能号 */
#define SMC_STDCALL64_NR(entity, fn)	SMC_NR((entity), (fn), 0, 1)

/* 服务类型（entry）声明 */
#define	SMC_ENTITY_ARCH			0	/* ARM Architecture calls */
#define	SMC_ENTITY_CPU			1	/* CPU Service calls */
#define	SMC_ENTITY_SIP			2	/* SIP Service calls */
#define	SMC_ENTITY_OEM			3	/* OEM Service calls */
#define	SMC_ENTITY_STD			4	/* Standard Service calls */
#define	SMC_ENTITY_RESERVED		5	/* Reserved for future use */
#define	SMC_ENTITY_TRUSTED_APP		48	/* Trusted Application calls */
#define	SMC_ENTITY_TRUSTED_OS		50	/* Trusted OS calls */
#define	SMC_ENTITY_LOGGING		51	/* Used for secure -> nonsecure logging */
#define	SMC_ENTITY_SECURE_MONITOR	60	/* Trusted OS calls internal to secure monitor */


/* 以下是具体的系统调用号 */

/* FC = Fast call, SC = Standard call */
#define SMC_SC_RESTART_LAST	SMC_STDCALL_NR  (SMC_ENTITY_SECURE_MONITOR, 0)
#define SMC_SC_LOCKED_NOP	SMC_STDCALL_NR  (SMC_ENTITY_SECURE_MONITOR, 1)

/**
 * SMC_SC_RESTART_FIQ - Re-enter trusty after it was interrupted by an fiq
 *
 * No arguments, no return value.
 *
 * Re-enter trusty after returning to ns to process an fiq. Must be called iff
 * trusty returns SM_ERR_FIQ_INTERRUPTED.
 *
 * Enable by selecting api version TRUSTY_API_VERSION_RESTART_FIQ (1) or later.
 */
#define SMC_SC_RESTART_FIQ	SMC_STDCALL_NR  (SMC_ENTITY_SECURE_MONITOR, 2)

/**
 * SMC_SC_NOP - Enter trusty to run pending work.
 *
 * No arguments.
 *
 * Returns SM_ERR_NOP_INTERRUPTED or SM_ERR_NOP_DONE.
 * If SM_ERR_NOP_INTERRUPTED is returned, the call must be repeated.
 *
 * Enable by selecting api version TRUSTY_API_VERSION_SMP (2) or later.
 */
#define SMC_SC_NOP		SMC_STDCALL_NR  (SMC_ENTITY_SECURE_MONITOR, 3)

/*
 * Return from secure os to non-secure os with return value in r1
 */
#define SMC_SC_NS_RETURN	SMC_STDCALL_NR  (SMC_ENTITY_SECURE_MONITOR, 0)

#define SMC_FC_RESERVED		SMC_FASTCALL_NR (SMC_ENTITY_SECURE_MONITOR, 0)
#define SMC_FC_FIQ_EXIT		SMC_FASTCALL_NR (SMC_ENTITY_SECURE_MONITOR, 1)
#define SMC_FC_REQUEST_FIQ	SMC_FASTCALL_NR (SMC_ENTITY_SECURE_MONITOR, 2)
#define SMC_FC_GET_NEXT_IRQ	SMC_FASTCALL_NR (SMC_ENTITY_SECURE_MONITOR, 3)
#define SMC_FC_FIQ_ENTER	SMC_FASTCALL_NR (SMC_ENTITY_SECURE_MONITOR, 4)

#define SMC_FC64_SET_FIQ_HANDLER SMC_FASTCALL64_NR(SMC_ENTITY_SECURE_MONITOR, 5)
#define SMC_FC64_GET_FIQ_REGS	SMC_FASTCALL64_NR (SMC_ENTITY_SECURE_MONITOR, 6)

#define SMC_FC_CPU_SUSPEND	SMC_FASTCALL_NR (SMC_ENTITY_SECURE_MONITOR, 7)
#define SMC_FC_CPU_RESUME	SMC_FASTCALL_NR (SMC_ENTITY_SECURE_MONITOR, 8)

#define SMC_FC_AARCH_SWITCH	SMC_FASTCALL_NR (SMC_ENTITY_SECURE_MONITOR, 9)
#define SMC_FC_GET_VERSION_STR	SMC_FASTCALL_NR (SMC_ENTITY_SECURE_MONITOR, 10)

/**
 * SMC_FC_API_VERSION - Find and select supported API version.
 *
 * @r1: Version supported by client.
 *
 * Returns version supported by trusty.
 *
 * If multiple versions are supported, the client should start by calling
 * SMC_FC_API_VERSION with the largest version it supports. Trusty will then
 * return a version it supports. If the client does not support the version
 * returned by trusty and the version returned is less than the version
 * requested, repeat the call with the largest supported version less than the
 * last returned version.
 *
 * This call must be made before any calls that are affected by the api version.
 */
#define TRUSTY_API_VERSION_RESTART_FIQ	(1)
#define TRUSTY_API_VERSION_SMP		(2)
#define TRUSTY_API_VERSION_SMP_NOP	(3)
#define TRUSTY_API_VERSION_CURRENT	(3)
#define SMC_FC_API_VERSION	SMC_FASTCALL_NR (SMC_ENTITY_SECURE_MONITOR, 11)

/* TRUSTED_OS entity calls */
#define SMC_SC_VIRTIO_GET_DESCR	SMC_STDCALL_NR(SMC_ENTITY_TRUSTED_OS, 20)
#define SMC_SC_VIRTIO_START	SMC_STDCALL_NR(SMC_ENTITY_TRUSTED_OS, 21)
#define SMC_SC_VIRTIO_STOP	SMC_STDCALL_NR(SMC_ENTITY_TRUSTED_OS, 22)

#define SMC_SC_VDEV_RESET	SMC_STDCALL_NR(SMC_ENTITY_TRUSTED_OS, 23)
#define SMC_SC_VDEV_KICK_VQ	SMC_STDCALL_NR(SMC_ENTITY_TRUSTED_OS, 24)
#define SMC_NC_VDEV_KICK_VQ	SMC_STDCALL_NR(SMC_ENTITY_TRUSTED_OS, 25)
```

### 服务处理

服务处理句柄定义如下

```c
/* SMC调用的参数 */
typedef struct smc32_args {
	uint32_t smc_nr;/* 服务号W0 */
	uint32_t params[SMC_NUM_PARAMS];/* 参数W1-W3 */
} smc32_args_t;

/* 服务处理句柄 */
typedef long (*smc32_handler_t)(smc32_args_t *args);
```

因为**trusty**不处理64位调用，所有要确定一个调用只需要关心三个域（调用类型、服务类型、函数号）

把调用类型和服务类型关联到一起定义两个表

```c
/* fast调用的entry表 表中的函数需要分析function number执行相关操作 */
smc32_handler_t sm_fastcall_table[SMC_NUM_ENTITIES] = {
	[0 ... SMC_ENTITY_SECURE_MONITOR - 1] = smc_undefined,
	[SMC_ENTITY_SECURE_MONITOR] = smc_fastcall_secure_monitor,
	[SMC_ENTITY_SECURE_MONITOR + 1 ... SMC_NUM_ENTITIES - 1] = smc_undefined
};

/* std调用的entry表 表中的函数需要分析function number执行相关操作 */
smc32_handler_t sm_stdcall_table[SMC_NUM_ENTITIES] = {
	[0 ... SMC_ENTITY_SECURE_MONITOR - 1] = smc_undefined,
	[SMC_ENTITY_SECURE_MONITOR] = smc_stdcall_secure_monitor,
	[SMC_ENTITY_SECURE_MONITOR + 1 ... SMC_NUM_ENTITIES - 1] = smc_undefined
};
```

stdcall中有一个特别的调用**SMC_SC_NOP**，会根据第一个参数（W0）的值选择不同的功能。所以这里特别定义了如下的表

```c
smc32_handler_t sm_nopcall_table[SMC_NUM_ENTITIES] = {
	[0] = smc_nop_secure_monitor,
	[1 ... SMC_NUM_ENTITIES - 1] = smc_undefined
};
```

表中的函数需要获取调用类型的功能号，进行相关操作。这里选取**smc_stdcall_secure_monitor**为列介绍

```c
/* stdcall 函数表（通过函数编号找到对应的函数） */
static smc32_handler_t sm_stdcall_function_table[] = {
	[SMC_FUNCTION(SMC_SC_RESTART_LAST)] = smc_restart_stdcall,
	[SMC_FUNCTION(SMC_SC_LOCKED_NOP)] = smc_nop_stdcall,
	[SMC_FUNCTION(SMC_SC_RESTART_FIQ)] = smc_restart_stdcall,
	[SMC_FUNCTION(SMC_SC_NOP)] = smc_undefined, /* reserve slot in table, not called */
};

static long smc_stdcall_secure_monitor(smc32_args_t *args)
{
	u_int function = SMC_FUNCTION(args->smc_nr);/* 获取函数编号 */
	smc32_handler_t handler_fn = NULL;

	if (function < countof(sm_stdcall_function_table))/* 防止越界访问数组 */
		handler_fn = sm_stdcall_function_table[function];/* 获取函数 */

	if (!handler_fn)
		handler_fn = smc_undefined;/* 函数指针为空即不支持 */

	return handler_fn(args);/* 执行对应的函数 */
}
```

### 服务注册

服务注册，用于添加一个服务类型的处理

```c
status_t sm_register_entity(uint entity_nr, smc32_entity_t *entity)
{
	status_t err = NO_ERROR;

	if (entity_nr >= SMC_NUM_ENTITIES)
		return ERR_INVALID_ARGS;/* 服务类似是否过大 */

	if (entity_nr >= SMC_ENTITY_RESERVED && entity_nr < SMC_ENTITY_TRUSTED_APP)
		return ERR_NOT_ALLOWED;/* 检测服务类型可用（有一段保留）*/

	if (!entity)/* 用于保存函数句柄不能为空 */
		return ERR_INVALID_ARGS;

	if (!entity->fastcall_handler && !entity->stdcall_handler)
		return ERR_NOT_VALID;/* 必须初始化一种服务 */

	mutex_acquire(&smc_table_lock);

	/* Check if entity is already claimed */
	/* 防止重复初始乎一个服务 */
	if (sm_fastcall_table[entity_nr] != smc_undefined ||
		sm_nopcall_table[entity_nr] != smc_undefined ||
		sm_stdcall_table[entity_nr] != smc_undefined) {
		err = ERR_ALREADY_EXISTS;
		goto unlock;
	}

	/* 登记服务 */
	if (entity->fastcall_handler)
		sm_fastcall_table[entity_nr] = entity->fastcall_handler;

	if (entity->nopcall_handler)
		sm_nopcall_table[entity_nr] = entity->nopcall_handler;

	if (entity->stdcall_handler)
		sm_stdcall_table[entity_nr] = entity->stdcall_handler;
unlock:
	mutex_release(&smc_table_lock);
	return err;
}
```

### TOS与服务接口

通过初始化钩子在系统初始化完才后等待服务请求

```c
/* 所有处理器初始化钩子，开启线程处理SMC调用 */
LK_INIT_HOOK_FLAGS(libsm_cpu, sm_secondary_init, LK_INIT_LEVEL_PLATFORM - 2, 					LK_INIT_FLAG_ALL_CPUS);

/* 主处理器初始化钩子，为stdcall服务 */
LK_INIT_HOOK(libsm, sm_init, LK_INIT_LEVEL_PLATFORM - 1);
```

### fastcall处理

fastcall在函数**sm_sched_nonsecure**中处理

**libsm_cpu**初始化钩子为每个CPU创建了一个线程**sm_wait_for_smcall**，此线程最终会调用**sm_sched_nonsecure**。调用过程如下sm_wait_for_smcall -> sm_return_and_wait_for_next_stdcall -> sm_sched_nonsecure。

**sm_sched_nonsecure**实现如下。参数**retval**用于返回stdcall的执行结果，参数**args**用于获取stdcall调用的参数（W0-W3）

**sm_sched_nonsecure**通过SMC指令进入**monitor**，X0用于标记功能**SMC_SC_NS_RETURN**（stdcall return to Non-Secure），X1用于记录返回的值。**monitor**在通过退出异常进入**Non-Secure**。在下一次**Non-Secure**异常调用时，通过**monitor**中转返回SMC指令下一条地址。此时判定是不是32位fastcall。如果是32位fastcall，就获取系统调用功能号的Entry（服务类型），通过Entry查表**sm_fastcall_table**获取异常处理函数，执行函数后再次通过SMC进入**Non-Secure**。如果是32位stdcall，就把系统调用参数写入**args**，然后函数返回让调用**sm_sched_nonsecure**的函数处理32位stdcall。

```assembly
/* void sm_sched_nonsecure(long retval, smc32_args_t *args); */
FUNCTION(sm_sched_nonsecure)
	push	x1, x30
.Lfastcall_complete:
	mov	x1, x0
.Lreturn_sm_err:
	ldr	x0, =SMC_SC_NS_RETURN
	mov	x2, xzr
	mov	x3, xzr
	smc	#0 /* 返回non-secure X0为传递给monitor的标示，X1为返回给non-secure的值 */

	tbnz	x0, #30, .Lsm_err_not_supported /* 第30比特用于标记64位调用 */
	tbz	x0, #31, .Lnot_fast_call/* 第31比特用于标记fastcall */

	/* fastcall */
	sub	sp, sp, #(4 * SMC_NUM_ARGS) /* 在堆栈上开辟空间 存放smc32_args_t */
	stp	w0, w1, [sp]                /* 保存系统调用参数到零时变量中 */
	stp	w2, w3, [sp, #4 * 2]

	ubfx	x0, x0, #24, #6		/* x0 = entity */    /* 获取entry的值 */
	ldr	x9, =sm_fastcall_table /* 加载起始地址 */
	ldr	x9, [x9, x0, lsl #3]   /* 获取函数地址 */

	mov	x0, sp			/* x0 = smc_args_t* args */
	blr	x9              /* 调用处理函数 */
	add	sp, sp, #(4 * SMC_NUM_ARGS)/* 恢复栈 */
	b	.Lfastcall_complete        /* 退出到用户空间 */

    /* stdcall */
.Lnot_fast_call:
	pop	x9, x30
	stp	w0, w1, [x9], #8           /* 保存系统调用参数到指针args中 */
	stp	w2, w3, [x9], #8
	ret

.Lsm_err_not_supported: /* 不支持 退出到用户空间 */
	mov	x1, #SM_ERR_NOT_SUPPORTED
	b	.Lreturn_sm_err

.Lsm_err_busy:
	mov	x1, #SM_ERR_BUSY
	b	.Lreturn_sm_err
```

### stdcall处理

通过代码分析，一般的stdcall在时间上不可用并发（**SMC_SC_NOP**可以在多个cpu上同时执行）。必须等待一个stdcall处理完后才能执行下一个stdcall。

为了实现时间上不可并发，主处理器初始化钩子**libsm**，创建一个线程**sm_stdcall_loop**用于处理实际的stdcall，所有处理器初始化钩子**libsm_cpu**创建一个线程**sm_wait_for_smcall**，用于向主处理器发送处理请求。

线程**sm_stdcall_loop**通过**stdcallstate**变量与**sm_wait_for_smcall**通信

**stdcallstate**数据类型为**sm_std_call_state**，定义如下

```c
struct sm_std_call_state {
	spin_lock_t lock;	/* 锁用于保护此结构体不被多个线程同时访问破坏数据 */
	event_t event;		/* 事件，用于通知主处理器有可以处理的stdcall请求 */
	smc32_args_t args;	/* stdcall请求的参数 */
	long ret;			/* stdcall处理的结果 */
	bool done;			/* 标记处理完成 */
	int active_cpu; 	/* 记录活动的cpu */
	int initial_cpu; 	/* 记录当前请求stdcall的处理器编号 */
	int last_cpu; 		/* 记录当前请求stdcall的处理器编号 */
	int restart_count;
};
```

**sm_stdcall_loop**用于处理stdcall请求，代码如下。一般此线程处于空闲状态等待stdcall请求（等待事件**stdcallstate.event**），在接受到请求后通过系统调用功能号查表**sm_stdcall_table**找到处理函数并执行，把结果写入**stdcallstate**中然后继续等待新的请求。

```c
static int __NO_RETURN sm_stdcall_loop(void *arg)
{
	long ret;
	spin_lock_saved_state_t state;

	while (true) {
		LTRACEF("cpu %d, wait for stdcall\n", arch_curr_cpu_num());
		event_wait(&stdcallstate.event);/* 等待信号处理stdcall */

		/* Dispatch 'standard call' handler */
		LTRACEF("cpu %d, got stdcall: 0x%x, 0x%x, 0x%x, 0x%x\n",
			arch_curr_cpu_num(),
			stdcallstate.args.smc_nr, stdcallstate.args.params[0],
			stdcallstate.args.params[1], stdcallstate.args.params[2]);
		/* 查表调用stdcall处理函数 */
		ret = sm_stdcall_table[SMC_ENTITY(stdcallstate.args.smc_nr)](&stdcallstate.args);
		LTRACEF("cpu %d, stdcall(0x%x, 0x%x, 0x%x, 0x%x) returned 0x%lx (%ld)\n",
			arch_curr_cpu_num(),
			stdcallstate.args.smc_nr, stdcallstate.args.params[0],
			stdcallstate.args.params[1], stdcallstate.args.params[2], ret, ret);

		/* 防止多个线程写stdcallstate */
		spin_lock_save(&stdcallstate.lock, &state, SPIN_LOCK_FLAG_IRQ);
		stdcallstate.ret = ret;/* 记录stdcall的处理结果 */
		stdcallstate.done = true;/* 标记stdcall处理完成 */
		event_unsignal(&stdcallstate.event);
		spin_unlock_restore(&stdcallstate.lock, state, SPIN_LOCK_FLAG_IRQ);
	}
}
```

**sm_wait_for_smcall**用于等待新的stdcall请求并转法给**sm_stdcall_loop**处理，代码如下。

```c
static void sm_wait_for_smcall(void)
{
	int cpu;
	long ret = 0;

	LTRACEF("wait for stdcalls, on cpu %d\n", arch_curr_cpu_num());

	while (true) {
		/*
		 * Disable interrupts so stdcallstate.active_cpu does not
		 * change to or from this cpu after checking it below.
		 */
		arch_disable_ints();

		/* Switch to sm-stdcall if sm_queue_stdcall woke it up */
		thread_yield();/* 让出cpu资源，给主线程处理stdcall */

		cpu = arch_curr_cpu_num();
		if (cpu == stdcallstate.active_cpu)
			ret = sm_get_stdcall_ret();/* 获取stdcall处理结果 */
		else
			ret = SM_ERR_NOP_DONE;

		/* 等待下一个stdcall */
		sm_return_and_wait_for_next_stdcall(ret, cpu);

		/* Re-enable interrupts (needed for SMC_SC_NOP) */
		arch_enable_ints();
	}
}

static void sm_return_and_wait_for_next_stdcall(long ret, int cpu)
{
	smc32_args_t args = {0};

	do {
		arch_disable_fiqs();
		sm_sched_nonsecure(ret, &args);/* 退出到Non-secure，等待下一次stdcall请求 */
		arch_enable_fiqs();

		/* Allow concurrent SMC_SC_NOP calls on multiple cpus */
		if (args.smc_nr == SMC_SC_NOP) {
			/* 特别的stdcall可以在多个cpu上并发 */
			LTRACEF_LEVEL(3, "cpu %d, got nop\n", cpu);
			ret = sm_nopcall_table[SMC_ENTITY(args.params[0])](&args);
		} else {
			ret = sm_queue_stdcall(&args);/* 请求sm_stdcall_loop处理 */
		}
	} while (ret);
}

static long sm_queue_stdcall(smc32_args_t *args)
{
	long ret;
	uint cpu = arch_curr_cpu_num();

	spin_lock(&stdcallstate.lock);

	if (stdcallstate.event.signalled || stdcallstate.done) {
	/* 主线程正在处理（stdcallstate.event.signalled）
	*  或其他数据还没有把数据拿走（stdcallstate.done） */
		if (args->smc_nr == SMC_SC_RESTART_LAST && stdcallstate.active_cpu == -1) {
			stdcallstate.restart_count++;
			LTRACEF_LEVEL(3, "cpu %d, restart std call, restart_count %d\n",
				      cpu, stdcallstate.restart_count);
			goto restart_stdcall;
		}
		dprintf(CRITICAL, "%s: cpu %d, std call busy\n", __func__, cpu);
		ret = SM_ERR_BUSY;
		goto err;
	} else {
		if (args->smc_nr == SMC_SC_RESTART_LAST) {
			dprintf(CRITICAL, "%s: cpu %d, unexpected restart, no std call active\n",
				__func__, arch_curr_cpu_num());
			ret = SM_ERR_UNEXPECTED_RESTART;
			goto err;
		}
	}

	LTRACEF("cpu %d, queue std call 0x%x\n", cpu, args->smc_nr);
	stdcallstate.initial_cpu = cpu;				/* 记录当前主线程为那个cpu工作 */
	stdcallstate.ret = SM_ERR_INTERNAL_FAILURE;
	stdcallstate.args = *args;					/* 把参数传递给主线程处理 */
	stdcallstate.restart_count = 0;
	event_signal(&stdcallstate.event, false);	/* 通知主线程工作 */

restart_stdcall:
	stdcallstate.active_cpu = cpu;				/* 记录活动cpu */
	ret = 0;

err:
	spin_unlock(&stdcallstate.lock);

	return ret;
}
```

