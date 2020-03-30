# opensbi代码分析

riscv约定了一个操作系统二进制接口，被称为[SBI(RISC-V Supervisor Binary Interface Specification)](https://github.com/riscv/riscv-sbi-doc/blob/master/riscv-sbi.adoc)。opensbi是SBI的一个开源实现。本文用于分析opensbi的源码结构。

## 固件型态

opensbi，接受前一阶段传递过来的参数：

- a0：hartid，核心的id编号  
- a1：next_arg1，opensbi执行完成后，需要把这个参数传递给下一个阶段  
- a2：opensbi dynamic型态固件额外的信息  

opensbi，下一阶段可以获得的参数：

- a0：hartid，核心的id编号  
- a1：arg1，opensbi接受到的参数a1  

opensbi可以以三种型态工作：

- jump，opensbi在执行完成后跳转到一个固定的地址  
- payload，opensbi下一阶段要执行的程序被作为payload链接到opensbi  
- dynamic，下一阶段的信息由额外的寄存器传递给opensbi  

## 程序入口 firmware/fw_base.S

此文件主要执行以下操着

```
                  +------------------+
                  | select boot hart |
                  +------------------+
                           |
          +----------------+--------------------+
          ↓  boot hart           Non-boot hart  |
   +------------------------------+             |
   | move opensbi to link address |             ↓
   +------------------------------+  +--------------------+
          |                          | waitting move done |
          ↓                          +--------------------+
   +---------------+                            |
   | init scratch  |                            |
   +---------------+                            |
          ↓                                     |
   +-----------+                                |
   | init bss  |                                |
   +-----------+                                |
          ↓                                     |
   +----------------------------+               |
   | move fdt to target address |               |
   +----------------------------+     +------------------------+
          |                           | watting boot hart done |
          |                           +------------------------+
          |                                     |
          +----------------+--------------------+
                           ↓
                  +------------------+
                  | init sp/mscratch |
                  +------------------+
                           ↓
                     +-----------+     
                     | init trap |
                     +-----------+
                           ↓
                   +---------------+     
                   | call sbi_init |
                   +---------------+
```

`sbi_init`是opensbi的c语言程序入口

opensbi的固件型态，是通过在入口程序中添加的一些hook来实现的，hook方法如下

```c
/* 用于选择boot_hart
 * 放回-1，表示没有指定boot_hart，fw_base.S通过原子操着选择boot_hart
 */
uintptr_t fw_boot_hart();

/* 用于保存前一阶段传递过来的信息 */
void fw_save_info(uintptr_t a0,uintptr_t a1,uintptr_t a2);

/* 前一阶段传递过来的参数a1 */
uintptr_t fw_prev_arg1();

/* 传递给下一阶段的参数a1 */
uintptr_t fw_next_arg1(); 

/* 下一阶段程序的地址 */
uintptr_t fw_next_addr(); 

/* 下一阶段程序的运行模式 */
uintptr_t fw_next_mode();

/* opensbi的选项 */
uintptr_t fw_options();
```

以上描述使用了伪代码，这些函数使用汇编实现，上下文保护请参考源码

## hart id 不连续问题

根据riscv spec规定，必须具备hart id为0的hart用于早期初始化，但hart id不需要连续。为了获得一个连续的索引，opensbi添加了一个表`platform->hart_index2id`，用于记录id和索引的关系

## 内存布局

内存布局，下图中的fireware为可执行程序的镜像，其中的hart n（n为hart index，不是hart id）

```
+------+-----------------------------
|      |              ↑
|      |              |
|      |              |
|      |           firmware
|      |              |
|      |              |
|      |              ↓
+------+----------------------------- 
|      |              ↑
|      |              |
|      |        stack of hart n
|      |              |
+------+--------      |
|      |   ↑          |
|      | scratch      |
|      |   ↓          ↓
+------+----------------------------- 
|      |              ↑
|      |              |
|      |        stack of hart n - 1
|      |              |
+------+--------      |
|      |   ↑          |
|      | scratch      |
|      |   ↓          ↓
+------+-----------------------------
|      |              ↑
|      |              |
|      |        stack of hart n - 2
|      |              |
+------+--------      |
|      |   ↑          |
|      | scratch      |
|      |   ↓          ↓
+------+-----------------------------
.
.
.
+------+-----------------------------
|      |              ↑
|      |              |
|      |        stack of hart 0
|      |              |
+------+--------      |
|      |   ↑          |
|      | scratch      |
|      |   ↓          ↓
+------+-----------------------------
```

scratch是一个内存块，开始位置保存了如下一个结构体

```c
struct sbi_scratch {
    /** 固件的起始地址 */
    unsigned long fw_start;
    /** 固件的大小 */
    unsigned long fw_size;
    /** opensbi运行结束后下一个阶段的arg1，rag0为hart id */
    unsigned long next_arg1;
    /** opensbi运行结束后下一个阶段的起始地址 */
    unsigned long next_addr;
    /** opensbi运行结束后下一个阶段的运行模式（特权等级） */
    unsigned long next_mode;
    /** 热启动地址 */
    unsigned long warmboot_addr;
    /** 平台相关的数据结构的地址 */
    unsigned long platform_addr;
    /** 函数的地址，用于把hart转换为scratch地址 */
    unsigned long hartid_to_scratch;
    /** 临时存储 */
    unsigned long tmp0;
    /** opensbi的选项 */
    unsigned long options;
} __packed;
```

## 堆内存

scratch当前保留了4k空间，当前只存放了一个结构体`struct sbi_scratch`，剩余的空间用来作堆使用。在src/sbi_scratch.c中实现了堆内存申请。

```c
unsigned long sbi_scratch_alloc_offset(unsigned long size, const char *owner)
{
    u32 i;
    void *ptr;
    unsigned long ret = 0;
    struct sbi_scratch *rscratch;

    /*
     * We have a simple brain-dead allocator which never expects
     * anything to be free-ed hence it keeps incrementing the
     * next allocation offset until it runs-out of space.
     *
     * In future, we will have more sophisticated allocator which
     * will allow us to re-claim free-ed space.
     */

    if (!size)
        return 0;

    if (size & (__SIZEOF_POINTER__ - 1))
        size = (size & ~(__SIZEOF_POINTER__ - 1)) + __SIZEOF_POINTER__;

    spin_lock(&extra_lock);

    if (SBI_SCRATCH_SIZE < (extra_offset + size))
        goto done;

    ret = extra_offset;
    extra_offset += size;

done:
    spin_unlock(&extra_lock);

    if (ret) {
        for (i = 0; i < SBI_HARTMASK_MAX_BITS; i++) {
            rscratch = sbi_hartid_to_scratch(i);
            if (!rscratch)
                continue;
            ptr = sbi_scratch_offset_ptr(rscratch, ret);
            sbi_memset(ptr, 0, size);
        }
    }

    return ret;
}
```

此方法在每一个hart的scratch空间申请一个内存，不支持释放，一般用于多核通讯。返回值是相对与scratch的偏移量

## 平台相关

opensbi为了移植和适配，把平台相关信息保存到一个变量`platform`中，类型如下

```c
/** Representation of a platform */
struct sbi_platform {
    /**
     * OpenSBI version this sbi_platform is based on.
     * It's a 32-bit value where upper 16-bits are major number
     * and lower 16-bits are minor number
     */
    u32 opensbi_version;
    /**
     * OpenSBI platform version released by vendor.
     * It's a 32-bit value where upper 16-bits are major number
     * and lower 16-bits are minor number
     */
    u32 platform_version;
    /** Name of the platform */
    char name[64];
    /** Supported features */
    u64 features;
    /** Total number of HARTs */
    u32 hart_count;
    /** Per-HART stack size for exception/interrupt handling */
    u32 hart_stack_size;
    /** Pointer to sbi platform operations */
    unsigned long platform_ops_addr;
    /** Pointer to system firmware specific context */
    unsigned long firmware_context;
    /**
     * HART index to HART id table
     *
     * For used HART index <abc>:
     *     hart_index2id[<abc>] = some HART id
     * For unused HART index <abc>:
     *     hart_index2id[<abc>] = -1U
     *
     * If hart_index2id == NULL then we assume identity mapping
     *     hart_index2id[<abc>] = <abc>
     *
     * We have only two restrictions:
     * 1. HART index < sbi_platform hart_count
     * 2. HART id < SBI_HARTMASK_MAX_BITS
     */
    const u32 *hart_index2id;
} __packed;
```

平台相关操着保存在`struct sbi_platform_operations`结构体中，结构体地址保存在`platform->platform_ops_addr`

## 初始化过程

```
               +--------------------------------------+
               | select a core to perform a cold boot |
               +--------------------------------------+
                                  |
               +------------------+-------------------+
               ↓ cold boot                  warm boot |
      +------------------------------+                |
      | init hartid_to_scratch_table |                |
      +------------------------------+                |
               ↓                                      |
      +----------+                                    |
      | hms init |                                    |
      +----------+                                    |
               ↓                                      |
      +-------------------+                           |
      | system early init |                           |
      +-------------------+                           |
               ↓                                      |
      +-----------+                                   |
      | hart init |                                   |
      +-----------+                                   |
               ↓                                      |
      +--------------+                                |
      | console init |                                |
      +--------------+                                |
               ↓                                      |
      +--------------+                                |
      | irqchip init |                                |
      +--------------+                                |
               ↓                                      |
      +----------+                                    |
      | ipi init |                                    |
      +----------+                                    |
               ↓                                      |
      +----------+                                    |
      | tlb init |                                    |
      +----------+                                    |
               ↓                                      |
      +------------+                                  |
      | timer init |                                  |
      +------------+                                  |
               ↓                                      |
      +------------+                                  |
      | ecall init |                                  |
      +------------+                                  |
               ↓                                      |
      +-------------------+                           |
      | system final init |                           |
      +-------------------+                           |
               ↓                                      |
      +---------------------+                         |
      | wake_coldboot_harts |                         |
      +---------------------+                         ↓
               |                             +------------------------+
               |                             | waitting coldboot done |
               ↓                             +------------------------+
      +-------------------+                           ↓
      | run to next stage |                  +----------+
      +-------------------+                  | hsm init |
                                             +----------+
                                                      ↓
                                             +-------------------+
                                             | system early init |
                                             +-------------------+
                                                      ↓
                                             +-----------+
                                             | hart init |
                                             +-----------+
                                                      ↓
                                             +--------------+
                                             | irqchip_init |
                                             +--------------+
                                                      ↓
                                             +----------+
                                             | ipi init |
                                             +----------+
                                                      ↓
                                             +----------+
                                             | tlb init |
                                             +----------+
                                                      ↓
                                             +------------+
                                             | timer init |
                                             +------------+
                                                      ↓
                                             +-------------------+
                                             | system final init |
                                             +-------------------+
                                                      ↓
                                             +-------------------+
                                             | run to next stage |
                                             +-------------------+
```

其中有两个系统预留的初始化接口，用于执行特定平台需要执行的初始化操着，句柄保存在`struct sbi_platform_operations`中

```c
    /** Platform early initialization */
    int (*early_init)(bool cold_boot); /* 对应上图中system early init */
    /** Platform final initialization */
    int (*final_init)(bool cold_boot); /* 对应上图中system final init */
```

其中，`hart init`执行hart相关的初始化  
其中，`irqchip init`用于初始化中断芯片  
其中，`console init`用于初始化终端  
其中，`ecall init`用于初始化SBI调用  
其他的用于初始化SBI的具体的服务  

## 反初始化

opensbi提供了反初始化操着，用于反初始化单个hart相关资源，反初始化过程如下

```
         +-------------------------+
         | sbi_platform_early_exit |
         +-------------------------+
                     ↓
             +----------------+
             | sbi_timer_exit |
             +----------------+
                     ↓
              +--------------+
              | sbi_ipi_exit |
              +--------------+
                     ↓
        +---------------------------+
        | sbi_platform_irqchip_exit |
        +---------------------------+
                     ↓
          +-------------------------+
          | sbi_platform_final_exit |
          +-------------------------+
                     ↓
               +--------------+
               | sbi_hsm_exit |
               +--------------+
```

其中由两个系统预留的反初始化接口，用于特定平台需要执行的反初始化操着，句柄保存在`struct sbi_platform_operations`中

```c
    /** Platform early exit */
    void (*early_exit)(void); /* 对应上图的sbi_platform_early_exit */
    /** Platform final exit */
    void (*final_exit)(void); /* 对应上图的sbi_platform_final_exit */
```

其中，`sbi_timer_exit`用于反初始化定时器  
其中，`sbi_ipi_exit`用于反初始化ipi  
其中，`sbi_platform_irqchip_exit`用于反初始化中断  
其中，`sbi_hsm_exit`用于反初始化hsm，跳转到热启动入口，hart进入`SBI_HART_STOPPED`状态  

## SBI服务相关

SBI服务，通过a7指定调用的扩展号，通过a6指定调用的功能号，a0-a5用于参数传递

为了便于扩展SBI服务，系统定义了一个结构体用于描述SBI服务。结构如下

```c
struct sbi_ecall_extension {
    struct sbi_dlist head;
    unsigned long extid_start;
    unsigned long extid_end;
    /* 用于判断具体的扩展号是否可用 */
    int (* probe)(struct sbi_scratch *scratch,
              unsigned long extid, unsigned long *out_val);
    /* SBI服务句柄 */
    int (* handle)(struct sbi_scratch *scratch,
               unsigned long extid, unsigned long funcid,
               unsigned long *args, unsigned long *out_val,
               struct sbi_trap_info *out_trap);
};
```

在`lib/sbi/sbi_ecall.c`中定义了一个链表`ecall_exts_list`，用于管理SBI服务，方法如下

```c
/* 注册复位，把ext加到ecall_exts_list */
int sbi_ecall_register_extension(struct sbi_ecall_extension *ext);

/* 反注册，把ext从ecall_exts_list中删除 */
void sbi_ecall_unregister_extension(struct sbi_ecall_extension *ext);

/* 在ecall_exts_list中查找有无扩展号为extid的扩展 */
struct sbi_ecall_extension *sbi_ecall_find_extension(unsigned long extid);

/* sbi服务句柄，查找出服务并执行 */
int sbi_ecall_handler(u32 hartid, ulong mcause, struct sbi_trap_regs *regs,
              struct sbi_scratch *scratch);

/* 把各种扩展添加到链表 */
int sbi_ecall_init(void);
```

当前系统添加了如下扩展

```c
extern struct sbi_ecall_extension ecall_base;
extern struct sbi_ecall_extension ecall_legacy;
extern struct sbi_ecall_extension ecall_time;
extern struct sbi_ecall_extension ecall_rfence;
extern struct sbi_ecall_extension ecall_ipi;
extern struct sbi_ecall_extension ecall_vendor;
extern struct sbi_ecall_extension ecall_hsm;
```

### ipi

riscv可以有一个映射的内存的软件中断标识位，通过向这个标识写1可以向需要的hart发送软件中断。

为了使ipi可以做更多事，系统为每个hart创建了一个`struct sbi_ipi_data`，通过置位标识哪种软件中断被触发。

```c
struct sbi_ipi_data {
    unsigned long ipi_type;
};
```

具体的软件中断如何处理，通过`struct sbi_ipi_event_ops`来描述

```c
/** IPI event operations or callbacks */
struct sbi_ipi_event_ops {
    /** Name of the IPI event operations */
    char name[32];

    /**
     * Update callback to save/enqueue data for remote HART
     * Note: This is an optional callback and it is called just before
     * triggering IPI to remote HART.
     */
    int (* update)(struct sbi_scratch *scratch,
            struct sbi_scratch *remote_scratch,
            u32 remote_hartid, void *data);

    /**
     * Sync callback to wait for remote HART
     * Note: This is an optional callback and it is called just after
     * triggering IPI to remote HART.
     */
    void (* sync)(struct sbi_scratch *scratch);

    /**
     * Process callback to handle IPI event
     * Note: This is a mandatory callback and it is called on the
     * remote HART after IPI is triggered.
     */
    void (* process)(struct sbi_scratch *scratch);
};
```

这些结构体构成一个数组`ipi_ops_array`，当`sbi_ipi_data->ipi_type`的第i为为1时，通过`ipi_ops_array[i]`中的句柄来处理，其中主要的三个句柄描述如下：

- update，发送ipi消息的hart在发送ipi消息之前调用此函数，用于向接受ipi消息的hart传递数据  
- sync，发送ipi消息的hart在发送ipi消息之后调用此函数，用于同步处理  
- process，接受到ipi消息的hart，执行此方法  

主要接口：

```c
/** 注册一个ipi操着，反回在ipi_ops_array中的位置，这个位置作为事件编号event */
int sbi_ipi_event_create(const struct sbi_ipi_event_ops *ops);

/** 反注册一个ipi操着 */
void sbi_ipi_event_destroy(u32 event);

/** 
 * 发生一个ipi消息给选定的hart
 * hbase 为一个整数指定接受消息的起始hartid
 * hmask 为一个掩码，标识那些hart可以接受消息
 * data 是要传递给接受ipi消息的hart的数据
 */
int sbi_ipi_send_many(struct sbi_scratch *scratch, ulong hmask, ulong hbase,
            u32 event, void *data);
```

当前创建了两种ipi:

- `ipi_smode_ops`，用于向s-mode发送软件中断  
- `ipi_halt_ops`，用于执行hms stop  

### hsm

hsm为Hart State Management的简写，它提供了一种能力用来管理hart。hsm创建了一个堆空间，用于记录当前内核的状态，可以方便其他hart访问修改

hart具有如下几种状态

```c
/** Hart state values **/
#define SBI_HART_STOPPED    0
#define SBI_HART_STOPPING    1
#define SBI_HART_STARTING    2
#define SBI_HART_STARTED     3
#define SBI_HART_UNKNOWN    4
```

主要接口如下

```c
/**
 * 初始化hsm
 * cold boot的hart将初始化所有hart的状态，cold boot为SBI_HART_STARTING，其他hart为SBI_HART_STOPPED
 * warm boot将进入等待，等待ipi以及hsm status变为SBI_HART_STARTING
 */
int sbi_hsm_init(struct sbi_scratch *scratch, u32 hartid, bool cold_boot)


/** 
 * 唤醒一个SBI_HART_STOPPED的hart，使其进入SBI_HART_STARTING状态
 * saddr 为下一阶段的起始地址
 * priv 为下一阶段的参数
 */
int sbi_hsm_hart_start(struct sbi_scratch *scratch, u32 hartid,
               ulong saddr, ulong priv);

/** 把状态从SBI_HART_STARTING切换到SBI_HART_STARTED
 * 此方法一般在opensbi运行完成之后执行，然后跳转到下一阶段
 */
void sbi_hsm_prepare_next_jump(struct sbi_scratch *scratch, u32 hartid);

/**
 * 把状态从SBI_HART_STARTED切换到SBI_HART_STOPPING
 * 如果exitnow为true，将进一步把状态切换到SBI_HART_STOPPED，然后跳转到热启动入口
 */
int sbi_hsm_hart_stop(struct sbi_scratch *scratch, bool exitnow);
```

opensbi实现了hsm的几个sbi调用，用于查询一个hart的状态，唤醒一个hart，退出。并且在ipi调用中也实现了退出。

实现了如下功能：

- hart可以通过hsm调用停止自己  
- hart可以通过ipi停止其他hart  
- hart可以通过hsm调用启动一个hart，并运行指定的程序  

关机的ecall也是由hsm实现的

### timer

riscv具有两个CSR寄存器mtime/mtimecmp，这两个寄存器可以用于定时器。opensbi主要实现了如下接口

```c
/* 此接口用于设置mtime */
void sbi_timer_event_start(struct sbi_scratch *scratch, u64 next_event);

/* 此接口用于处理定时器中断，清除m-mode中断中断标识，并触发s-mode中断（写MIP.STIP） */
void sbi_timer_process(struct sbi_scratch *scratch);

/* 此接口用于初始定时器 */
int sbi_timer_init(struct sbi_scratch *scratch, bool cold_boot);

/* 此接口用于关闭定时器 */
void sbi_timer_exit(struct sbi_scratch *scratch);
```

这些接口主要在`ecall_legacy` / `ecall_time`中被使用

### fifo

opensbi实现了一个先进先出的队列，结构体如下

```
struct sbi_fifo {
    void *queue;      /* 队列使用的内存的起始地址 */
    spinlock_t qlock; /* 锁，用于防止多线程并发破坏数据 */
    u16 entry_size;   /* 队列中存放的每个数据的大小 */
    u16 num_entries;  /* 队列中最多存放数据的个数，entry_size × num_entries为队列内存的大小 */
    u16 avail;        /* 队列中多少个数据有效 */
    u16 tail;         /* 最早进入队列的数据的索引 */
};
```

主要接口如下：

```c
/**
 * 初始化队列
 * fifo用于维护记录队列的信息
 * queue_mem 分配给队列的内存
 * entries 队列中最多存放数据的个数
 * entry_size 队列中存放的每个数据的大小
 */
void sbi_fifo_init(struct sbi_fifo *fifo, void *queue_mem, u16 entries,
           u16 entry_size);

/* 判断队列为空 */
bool sbi_fifo_is_empty(struct sbi_fifo *fifo);

/* 判断队列以满 */
bool sbi_fifo_is_full(struct sbi_fifo *fifo);

/* 返回队列中数据的个数 */
u16 sbi_fifo_avail(struct sbi_fifo *fifo);

/* 向队列中添加一个数据 */
int sbi_fifo_enqueue(struct sbi_fifo *fifo, void *data);

/* 从队列中取出一个数据 */
int sbi_fifo_enqueue(struct sbi_fifo *fifo, void *data);

/**
 * 更新队列中的数据，此方法会遍历队列中的每一个数据
 * fptr为更新队列的方法
 *     in 为更新函数的参数
 *     data 为队列中的一个数据
 */
int sbi_fifo_inplace_update(struct sbi_fifo *fifo, void *in,
                int (*fptr)(void *in, void *data));
```

### tlb

tlb用于在多核直接进行缓存同步，它使用了前面介绍的ipi和fifo。opensbi在每个hart创建了一个fifo用于接受tlb操作指令，并创建了一个整数用来同步。tlb操作指令定义如下

```c
/* tlb操作类型 */
enum sbi_tlb_info_types {
    SBI_TLB_FLUSH_VMA,
    SBI_TLB_FLUSH_VMA_ASID,
    SBI_TLB_FLUSH_GVMA,
    SBI_TLB_FLUSH_GVMA_VMID,
    SBI_TLB_FLUSH_VVMA,
    SBI_TLB_FLUSH_VVMA_ASID,
    SBI_ITLB_FLUSH
};

struct sbi_tlb_info {
    unsigned long start;       /* 起始地址 */
    unsigned long size;        /* 大小 */
    unsigned long asid;        /* 虚拟内存标识 */
    unsigned long type;        /* tlb操作类型 */
    struct sbi_hartmask smask; /* 发生指令的hart的掩码 */
};
```

作为一个ipi服务，主要实现了三个函数：

```c
/* 用于向接受ipi的hart传递数据，添加sbi_tlb_info到目标hart的fifo */
static int sbi_tlb_update(struct sbi_scratch *scratch,
              struct sbi_scratch *remote_scratch,
              u32 remote_hartid, void *data);

/* 接受ipi的hart处理ipi请求，
 * 请求完成后，根据sbi_tlb_info->smask向发送请求的hart同步（同步变量写1）
 */
static void sbi_tlb_process(struct sbi_scratch *scratch);

/* 监视同步变量等待收到1 */
static void sbi_tlb_sync(struct sbi_scratch *scratch);
```

## 异常服务

代码入口`lib/sbi/sbi_trap.c`，主要实现了以下服务

1. 时钟中断，处理定时器事件转发中断给s-mode  
2. 软件中断，处理软件中断时间转发中断给s-mode  
3. 非法指令异常，软件模拟一些指令  
4. 非对齐内存访问，通过字节操作实现非对齐内存访问  
5. ecall，sbi服务入口  
6. 内存访问错误也错误，转给s-mode处理  




