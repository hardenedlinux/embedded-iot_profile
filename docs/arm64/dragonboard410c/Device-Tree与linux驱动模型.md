# 0 概述

本文旨在分析Device Tree在linux驱动模型中的作用，以及系统内核启动后Device Tree的处理过程。此文的分析基于dragonboard410c的官方内核源代码[下载](https://git.linaro.org/landing-teams/working/qualcomm/kernel.git/snapshot/kernel-debian-qcom-dragonboard410c-16.09.tar.gz)。

# 1 内核实现的类似面向对象的方法

linux中驱动模型有很多层对象结构，内核使用一种特别的方式实现。

继承：通过在子类中嵌入父类的结构体实现（可以实现C++多重继承，有多个父类）

虚函数：子类可以在初始化是重写虚函数（给父类的函数指针赋值），这样实现多态（框架）

通过父类指针访问子类（虚函数传进的是父类指针，要做具体类的操作需要类型转换），通过宏container_of实现。container_of具体实现如下

```c
#define container_of(ptr,type,member)   \
        (type*)((char*)ptr - (size_t)(&(((type*)0)->member)))
//ptr为父类的指针
//type为子类的类型
//member为父类在子类中的成员名
```

# 2 内核扩展方式与内核模块

内核通过一些结构体，以及一些框架代码来实现内核功能扩展。要扩展内核的功能，只需要在内核初始化过程中调用一些函数（bus_register、device_register、driver_register...）。在这些函数中把新定义的结构体加到系统结构中（数组、链表等）。系统有两种方式扩展功能，编译进内核，编译成模块。

## 2.1 编译进内核

系统定义了一些列的宏，这些宏把函数的指针放到指定的段,便于程序访问

```c
#define early_initcall(fn)      __define_initcall(fn, early)
#define pure_initcall(fn)		__define_initcall(fn, 0)
#define core_initcall(fn)		__define_initcall(fn, 1)
#define core_initcall_sync(fn)		__define_initcall(fn, 1s)
#define postcore_initcall(fn)		__define_initcall(fn, 2)
#define postcore_initcall_sync(fn)	__define_initcall(fn, 2s)
#define arch_initcall(fn)		__define_initcall(fn, 3)
#define arch_initcall_sync(fn)		__define_initcall(fn, 3s)
#define subsys_initcall(fn)		__define_initcall(fn, 4)
#define subsys_initcall_sync(fn)	__define_initcall(fn, 4s)
#define fs_initcall(fn)			__define_initcall(fn, 5)
#define fs_initcall_sync(fn)		__define_initcall(fn, 5s)
#define rootfs_initcall(fn)		__define_initcall(fn, rootfs)
#define device_initcall(fn)		__define_initcall(fn, 6)
#define device_initcall_sync(fn)	__define_initcall(fn, 6s)
#define late_initcall(fn)		__define_initcall(fn, 7)
#define late_initcall_sync(fn)		__define_initcall(fn, 7s)
```

这些宏都通过__define_initcall宏实现

```c
#define __define_initcall(fn, id) \
	static initcall_t __initcall_##fn##id __used \
	__attribute__((__section__(".initcall" #id ".init"))) = fn; \
	LTO_REFERENCE_INITCALL(__initcall_##fn##id)
```

此宏定义了一个函数指针放在固定名字的段中，配合链接脚本，可以访问到对应的函数指针。

其中initcall_t是函数指针类型，`typedef int (*initcall_t)(void)`

__used是防止链接器警告

```c
#if GCC_VERSION < 30300
        #define __used __attribute__((__unused__))
    #else
        #define __used __attribute__((__used__))
    #endif
```

LTO_REFERENCE_INITCALL，用于防止链接器优化，生成一个静态函数并使用指针

```c
#ifdef CONFIG_LTO
	#define LTO_REFERENCE_INITCALL(x) \
	; /* yes this is needed */			\
	static __used __exit void *reference_##x(void)	\
	{						\
		return &x;				\
	}
#else
    #define LTO_REFERENCE_INITCALL(x)
#endif
```

链接脚本处理与C程序访问，为了便于链接脚本生成，在include/asm-generic/vmlinux.lds.h头文件中定义了大量的宏（默认对齐、段保留丢弃宏各种段）。其中定义了INIT_CALLS，用于存放上面声明的函数指针

```c
#define INIT_CALLS							\
		VMLINUX_SYMBOL(__initcall_start) = .;			\
		*(.initcallearly.init)					\
		INIT_CALLS_LEVEL(0)					\
		INIT_CALLS_LEVEL(1)					\
		INIT_CALLS_LEVEL(2)					\
		INIT_CALLS_LEVEL(3)					\
		INIT_CALLS_LEVEL(4)					\
		INIT_CALLS_LEVEL(5)					\
		INIT_CALLS_LEVEL(rootfs)				\
		INIT_CALLS_LEVEL(6)					\
		INIT_CALLS_LEVEL(7)					\
		VMLINUX_SYMBOL(__initcall_end) = .;
#define INIT_CALLS_LEVEL(level)						\
		VMLINUX_SYMBOL(__initcall##level##_start) = .;		\
		*(.initcall##level##.init)				\
		*(.initcall##level##s.init)				\
```

上面的两个宏生成了一个段，以符号\_\_initcall\_start开头，以符号\_\_initcall\_end结尾。中间嵌插符号\_\_initcall0\_start \_\_initcall1\_start \_\_initcall2\_start \.\.\. \_\_initcall7\_start。并且\_\_initcall\_start为\.initcallearly\.init段的开始符号，\_\_initcall0\_start为\.initcall0\.init \.initcall0s\.init段的开始符号，\_\_initcall1\_start为\.initcall1\.init \.initcall1s\.init段的开始符号，\_\_initcall2\_start为\.initcall2\.init \.initcall2s\.init段的开始符号\.\.\.\_\_initcall7\_start为\.initcall7\.init \.initcall7s\.init段的开始符号。

在init/main.c中声明

```c
extern initcall_t __initcall_start[];
extern initcall_t __initcall0_start[];
extern initcall_t __initcall1_start[];
extern initcall_t __initcall2_start[];
extern initcall_t __initcall3_start[];
extern initcall_t __initcall4_start[];
extern initcall_t __initcall5_start[];
extern initcall_t __initcall6_start[];
extern initcall_t __initcall7_start[];
extern initcall_t __initcall_end[];
static initcall_t *initcall_levels[] __initdata = {
    __initcall0_start,
    __initcall1_start,
    __initcall2_start,
    __initcall3_start,
    __initcall4_start,
    __initcall5_start,
    __initcall6_start,
    __initcall7_start,
    __initcall_end,
};
```
通过do\_pre\_smp\_initcalls、do\_initcalls函数完才初始化工作。

```c
static void __init do_pre_smp_initcalls(void);
static void __init do_initcalls(void);
```

## 2.2 编译成模块

一个模块需要两个宏

module\_init(fn) 把函数指针放到段\.initcall7\.init中

module\_exit(fn) 把函数指针放到段\.exitcall\.exit中

模块在加载时调用段.initcall7.init中的函数

模块在移除时调用段.exitcall.exit中的函数

其他的驱动也定义了一些宏，处理这些问题比如platform

```c
#define module_platform_driver(__platform_driver) \
        module_driver(__platform_driver, platform_driver_register, \
            platform_driver_unregister)
#define module_driver(__driver, __register, __unregister, ...) \
        static int __init __driver##_init(void) \
        { \
            return __register(&(__driver) , ##__VA_ARGS__); \
        } \
        module_init(__driver##_init); \
        static void __exit __driver##_exit(void) \
        { \
            __unregister(&(__driver) , ##__VA_ARGS__); \
        } \
        module_exit(__driver##_exit);
```

__platform_driver是要注册的驱动对象（结构体）

platform_driver_register platform驱动注册函数

platform_driver_unregister platform反驱动注册函数

这样就可以在驱动加载时，把结构体添加到系统的链表中，便于框架处理

# 3 sysfs

sysfs是linux内核模块在用户空间的映射，sysfs通过ramfs实现，与kobject密切相关。kobject与kset助成树形结构，对应目录结构。kobj_type对应目录中的文件。

## 3.1 kobject

kobject是驱动模型的关键，提供了以下功能

- 实现一个树形结构
- 对象上锁
- 引用计数（用于生存周期计算，用于释放内存）
- 在用户空间的表示

以下是kobject结构体

```c
struct kobject {
        const  char         *name;      //名字对应在/sys目录中的名字
        struct list_head    entry;      //一个双向链表，用于链接对象
        struct kobject      *parent;    //指向父指针，对应所在的目录
        struct kset         *kset;      //集合，当前对象所在的集合
        //包含一些属性，以及属性操作相关的函数，用于释放kobject对象的函数
        struct kobj_type    *ktype;
        struct kernfs_node  *sd;        //内核文件系统相关
        struct kref         kref;       //引用计数
        #ifdef CONFIG_DEBUG_KOBJECT_RELEASE
        struct delayed_work release;
        #endif
        unsigned int        state_initialized:1;    //初始化标记
        unsigned int        state_in_sysfs:1;       //指示为sysfs对象
        unsigned int        state_add_uevent_sent:1;
        unsigned int        state_remove_uevent_sent:1;
        unsigned int        uevent_suppress:1;
    };
```

系统为kobject对象提供了，设置名字、初始化、添加、移除、获取路径相关的方法

## 3.2 kset

kset结构体如下

```c
struct kset{
        struct list_head    list;//与集合元素构成链表用于访问集合中的元素
        spinlock_t          list_lock;//锁
        struct              kobject kobj;//kobject中parent指向此处
        const struct        kset_uevent_ops *uevent_ops;//消息处理回调函数
    };
```

通过关联分析kobject、kset，可以知道kobject和kset一起组成了一个树形结构

## 3.3 kobj_type

kobj_type结构如下

```c
struct kobj_type{
        //用于释放kobject内存
        void (*release)(struct kobject *kobj);
        //sysfs_ops主要包含两个函数分别用于属性显示和修改
        const struct sysfs_ops *sysfs_ops;
        //属性对应sysfs中的文件，这个结构体中有文件名和权限
        struct attribute **default_attrs;
        const struct kobj_ns_type_operations *
                            (*child_ns_type)(struct kobject *kobj);
        const void *(*namespace)(struct kobject *kobj);
    };
struct attribute {
        const char *name;   //属性名，对应sysfs中的文件名
        umode_t     mode;   //访问权限
    };
struct sysfs_ops {
        ssize_t (*show)(struct kobject *,
                        struct attribute *,
                        char *);
        ssize_t (*store)(struct kobject *,
                        struct attribute *,
                        const char *, size_t);
    };
```

show/store用于显示属性和写属性

## 3.4 kref

内核中的引用计算器，用于监视对象的生命周期，在适当的时候释放内存销毁对象。结构如下：
```c
struct kref {
        atomic_t refcount;
    };
```

atomic_t是内核维护的原子量，即一个整数，对多线程是安全的

它主要操作有init(refcount=1)

它主要操作有put(refcount--)

它主要操作有gut(refcount++)

还可以在put时，查看refcount==0时调用释放内存的函数

# 4 驱动架构

驱动框架实现主要位于drivers/base中

框架有关的结构体

- bus_type

- device

- device_driver


## 4.1 bus_type

定义如下

```c
struct bus_type {
	    const char		*name;//总线的名字
	    const char		*dev_name;//对应设备的名字
	    struct device		*dev_root;//总线设备
	    struct device_attribute	*dev_attrs;	//总线的设备的属性
	    const struct attribute_group **bus_groups;//链接到总线的总线属性
	    const struct attribute_group **dev_groups;//链接到总线的设备属性
	    const struct attribute_group **drv_groups;//链接到总线的驱动属性
        //用于发现设备或者注册新驱动时，匹配设备和驱动
	    int (*match)(struct device *dev, struct device_driver *drv);
	    //在发送热插拔事件消息到用户空间之前添加环境变量
	    int (*uevent)(struct device *dev, struct kobj_uevent_env *env);
	    //用于确认设备可以链接到总线并进行初始化
	    int (*probe)(struct device *dev);
	    //移除设备相关操作
	    int (*remove)(struct device *dev);
	    //断点需要的操作
	    void (*shutdown)(struct device *dev);

        //可热插拔相关
	    int (*online)(struct device *dev);
	    int (*offline)(struct device *dev);

        //系统挂起恢复时调用
	    int (*suspend)(struct device *dev, pm_message_t state);
	    int (*resume)(struct device *dev);

	    const struct dev_pm_ops *pm;//电源相关操作
	    const struct iommu_ops *iommu_ops;//mmu相关操作

        //总线私有数据，里面包含devices_kset/drivers_kset
	    struct subsys_private *p;
	    struct lock_class_key lock_key;
    };
```

在有新的设备注册时，系统会调用match函数遍历所有连接到当前总线的驱动。

在有新的驱动注册时，系统会调用match函数遍历所有连接到当前总线的设备。

match只执行和总线相关的匹配，probe执行设备相关的匹配，并进行初始化操作。match时会设置dev的driver指针，所以probe不需要传入drv指针。

一般不对bus_type进行扩展，在系统中bus_type一般以静态变量形式存在

在定义好bus_type结构体后需要调用bus_register，把驱动注册到驱动核心。bus_register负责维护对应的sys，以及把bus_type添加到bus_kset。对应的bus_unregister负责从系统反注册bus_type。

## 4.2 device

定义如下

```c
struct device {
	    struct device *parent;//父设备（总线设备）
	    struct device_private *p;//私有数据
	    struct kobject kobj;//用于处理sysfs以及引用计数
	    const char *init_name;//设备名
	    const struct device_type *type;//设备类型
	    struct mutex mutex;	//互斥锁
	    struct bus_type	*bus;//设备所在的总线
	    struct device_driver *driver; //驱动
	    void *platform_data; //platform总线相关的数据
	    void *driver_data; //驱动相关的数据
	    struct dev_pm_info	power;
	    struct dev_pm_domain *pm_domain;

    #ifdef CONFIG_GENERIC_MSI_IRQ_DOMAIN
	    struct irq_domain	*msi_domain;
    #endif
    #ifdef CONFIG_PINCTRL
	    struct dev_pin_info	*pins;
    #endif
    #ifdef CONFIG_GENERIC_MSI_IRQ
	    struct list_head	msi_list;
    #endif

    #ifdef CONFIG_NUMA
	    int		numa_node;	/* NUMA node this device is close to */
    #endif
	    u64		*dma_mask;	/* dma mask (if dma'able device) */
	    u64		coherent_dma_mask;/* Like dma_mask, but for
					         alloc_coherent mappings as
					         not all hardware supports
					         64 bit addresses for consistent
					         allocations such descriptors. */
	    unsigned long	dma_pfn_offset;

	    struct device_dma_parameters *dma_parms;

	    struct list_head	dma_pools;	/* dma pools (if dma'ble) */

	    struct dma_coherent_mem	*dma_mem; /* internal for coherent mem
					         override */
    #ifdef CONFIG_DMA_CMA
	    struct cma *cma_area;		/* contiguous memory area for dma
					       allocations */
    #endif
	    /* arch specific additions */
	    struct dev_archdata	archdata;

	    struct device_node	*of_node; /* associated device tree node */
	    struct fwnode_handle	*fwnode; /* firmware device node */

	    dev_t			devt;	/* dev_t, creates the sysfs "dev" */
	    u32			id;	/* device instance */

	    spinlock_t		devres_lock;
	    struct list_head	devres_head;

	    struct klist_node	knode_class;
	    struct class		*class;
	    const struct attribute_group **groups;	/* optional groups */

	    void	(*release)(struct device *dev);
	    struct iommu_group	*iommu_group;

	    bool			offline_disabled:1;
	    bool			offline:1;
    };
```

device中保存了各自驱动操作相关的资源，一般需要对device进行扩展

device_register注册设备，把对应的结构添加到全局变量devices_kset中，并且把device加到对应的总线并进行驱动匹配

一般需要对device进行继承，并重写device_register。在其在加入设备特定操作，并调用父类的device_register进行通用操作。

## 4.3 device_driver

定义如下

```c
struct device_driver {
	    const char *name;//驱动名
	    struct bus_type *bus;//要挂载的总线
	    struct module *owner;//模块
	    const char *mod_name;/* used for built-in modules */
	    bool suppress_bind_attrs;/* disables bind/unbind via sysfs */
	    enum probe_type probe_type;

	    const struct of_device_id	*of_match_table;//用于匹配设备的表
	    const struct acpi_device_id	*acpi_match_table;//用于acpi匹配的表

	    int (*probe) (struct device *dev);//确认设备是否匹配并初始化
	    int (*remove) (struct device *dev);//移除设备时调用
	    void (*shutdown) (struct device *dev);//设备断电时调用
	    int (*suspend) (struct device *dev, pm_message_t state);//设备挂起时调用
	    int (*resume) (struct device *dev);//设备从挂起恢复时调用
	    const struct attribute_group **groups;//驱动属性

	    const struct dev_pm_ops *pm;//电源相关操作

	    struct driver_private *p;//私有数据，其在有驱动的所有的设备列表
    };
```

驱动程序中保存了

- 驱动设备相关的总线、模块、驱动名字

- 用于匹配设备的信息（of_match_table与device-tree相关）

- 以及驱动相关的基本操作（匹配初始化、移除、挂起、恢复）

一般具体的驱动需要继承device_driver，并实现自己的注册函数，在注册函数中调用driver_register函数

driver_register会维护sysfs以及负责把设备挂载到驱动等基础操作

# 5 驱动与Device-Tree

linux通过三个基本的类（bus_type/device/device_driver）来描述驱动相关信息

并在注册阶段完成sysfs的维护，以及建立bus_type/device/device_driver关联

建立关联时涉及三个匹配
```c++
bus_type.match(struct device *dev, struct device_driver *drv);
bus_type.probe(struct device *dev);
device_driver.probe(struct device *dev);
```
bus_type.match在设备或驱动注册到系统时被调用，完成设备和驱动的匹配，并修改dev的device_driver指针。

bus_type.match没有实现，默认认为设备和驱动是匹配的，并调用bus_type.probe进行设备匹配和初始化。

如果bus_type.probe没有实现，就调用device_driver.probe进行设备匹配和初始化。

匹配操作要使用device_driver中of_match_table。of_match_table保存了Device-Tree的compatible字段。具体定义如下：

```c
struct of_device_id {
        char    name[32];//要匹配的设备的名字
        char    type[32];//要匹配的设备的类型
        char    compatible[128];//要匹配的设备的device-tree的compatible字段
        const void *data;//
    };
```

具体匹配过程和具体设备相关，使用不使用Device-Tree和具体驱动相关。

在device_driver中还有一个值得关注的对象acpi_match_table，此对象和of_device_id类似，用于acpi总线的匹配。

# 6 ARMv8总线自举过程 

根据ARMv8启动协议（内核代码位置Documentation/arm64/booting.txt），可知内核通过x0传递Device-Tree的首地在给内核，x1-x3保留并且必须为0。

arch/arm64/kernel/head.S为内核入口代码，其中主要关注Device-Tree保存到__fdt_pointer相关的代码

```asm
/*
 * Kernel startup entry point.
 * ---------------------------
 *
 * The requirements are:
 *   MMU = off, D-cache = off, I-cache = on or off,
 *   x0 = physical address to the FDT blob.
 *
 * This code is mostly position independent so you call this at
 * __pa(PAGE_OFFSET + TEXT_OFFSET).
 *
 * Note that the callee-saved registers are used for storing variables
 * that are useful before the MMU is enabled. The allocations are described
 * in the entry routines.
 */
	__HEAD  //在linux/init.h中定义#define  __HEAD  .section  ".head.text","ax"，结合链接脚本，这个段就是输出段的头部

	/*
	 * DO NOT MODIFY. Image header expected by Linux boot-loaders.
	 */
#ifdef CONFIG_EFI
efi_head:
	/*
	 * This add instruction has no meaningful effect except that
	 * its opcode forms the magic "MZ" signature required by UEFI.
	 */
	add	x13, x18, #0x16// 此条指令无效，用于伪装成PE的DOS头的signature “MZ”
	b	stext// 跳转到主程序
	/*
	……此处主要用于模拟PE文件
	*/
	ENTRY(stext)
	bl	preserve_boot_args// 保存bootloader传递过来的参数(X0 ... X3)到boot_args
	//根据Documentation/arm64/booting.txt描述X0保存dtb地址，其他几个寄存器保留未用
	//x0被暂时保存到x21
	bl	el2_setup// Drop to EL1, w20=cpu_boot_mode 
	//异常等级降到EL1
	//如果在原来在EL2需要开放一些寄存器给EL1访问，记录原本的异常等级到W20
	adrp	x24, __PHYS_OFFSET
	bl	set_cpu_boot_mode_flag
	bl	__create_page_tables		// x25=TTBR0, x26=TTBR1
	/*
	 * The following calls CPU setup code, see arch/arm64/mm/proc.S for
	 * details.
	 * On return, the CPU will be ready for the MMU to be turned on and
	 * the TCR will have been set.
	 */
	ldr	x27, =__mmap_switched// address to jump to after
	// MMU has been enabled
	adr_l	lr, __enable_mmu// return (PIC) address
	b	__cpu_setup// initialise processor
		//函数__cpu_setup返回后进入__enable_mmu
		//__enable_mmu出口调用__mmap_switched
ENDPROC(stext)
/*
 * 省略部分不相关代码
 */
	.set	initial_sp, init_thread_union + THREAD_START_SP
__mmap_switched:
	adr_l	x6, __bss_start
	adr_l	x7, __bss_stop

1:	cmp	x6, x7
	b.hs	2f
	str	xzr, [x6], #8			// Clear BSS
	b	1b
2:
	adr_l	sp, initial_sp, x4
	str_l	x21, __fdt_pointer, x5// Save FDT pointer 
		//这里保存x21到__fdt_pointer
		//x21在preserve_boot_args保存了x0的值
		//x0的值为Device-Tree的地址
	str_l	x24, memstart_addr, x6		// Save PHYS_OFFSET
	mov	x29, #0
#ifdef CONFIG_KASAN
	bl	kasan_early_init
#endif
	b	start_kernel
ENDPROC(__mmap_switched)
```

init/main.c中的start_kernel为C语言部分的入口，其中调用了setup_arch。在setup_arch中对Device-Tree进行校验并展开为树的形式。

```c
void __init setup_arch(char **cmdline_p)
{
	//……

	setup_machine_fdt(__fdt_pointer);
    //__fdt_pointer在head.S中被设置，此处设initial_boot_parmas(指向fdt)，校验fdt并初始化
  
	//……
	if (acpi_disabled) {
		unflatten_device_tree();//把fdt(flat device tree)转换为树(device_node)结构
		psci_dt_init();
	} else {
		psci_acpi_init();
	}
	//……
  	//此处用于校验bootloader有没有按启动协议传参数
	if (boot_args[1] || boot_args[2] || boot_args[3]) {
		pr_err("WARNING: x1-x3 nonzero in violation of boot protocol:\n"
			"\tx1: %016llx\n\tx2: %016llx\n\tx3: %016llx\n"
			"This indicates a broken bootloader or old kernel\n",
			boot_args[1], boot_args[2], boot_args[3]);
	}
}
```

设备注册通过arm64_device\_init完成。这里和前面第2章相关，最终arm64\_device\_init会在init/main.c中被调用。

```c
static int __init arm64_device_init(void)
{
	if (of_have_populated_dt()) {//判断DT有无展开(__fdt_pointer -> of_root)
		of_iommu_init();
		//处理DT，创建Device并注册到对应的总线
		of_platform_populate(NULL, of_default_bus_match_table,
				NULL, NULL);
	} else if (acpi_disabled) {
		pr_crit("Device tree not populated\n");
	}
	return 0;
}
arch_initcall_sync(arm64_device_init);
```

设备注册主要通过of_platform_populate完才。of_platform_populate会遍历Device-Tree注册设备，注册设备的设备会挂载到platform总线。如果注册的设备是总线设备，会触发bus_register。根据的类型会自动匹配（可自举总线，如USB等）。不可自举总线会在遍历过程中进行设备的注册挂载。