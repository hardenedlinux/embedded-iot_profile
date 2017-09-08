# 概要

coreboot是一个开源项目，旨在替换大多数计算机中的专有固件（BIOS）。coreboot之前被称为LinuxBIOS。coreboot在进行一点硬件相关的初始化后引导一个payload，这个payload可以是标准固件的实现、操作系统的引导程序或者操作系统本身。

# 编译和调试

当前riscv没有具体硬件，需要使用**spike**进行仿真。这里我选择linux作为coreboot的payload。

## 编译

### 构建工具以及模拟器

1. 获取源码，**git clone https://github.com/riscv/riscv-tools.git**
2. 更新源码，**cd riscv-tools ; git submodule update --init --recursive**
3. 修正代码（https://github.com/riscv/riscv-isa-sim/pull/53），添加8250串口支持
4. 修正代码添加configstring支持
5. 设置环境变量，**export RISCV=path**，path为软件希望安装的路径
6. 编译，**./build.sh**

### 构建内核

1. 获取linux内核源码，**git clone https://github.com/torvalds/linux.git**
2. 获取riscv linux内核源码，**git clone https://github.com/riscv/riscv-linux**
3. 创建符号链接，**cd linux-4.6.x/arch ; ln -s ../../riscv-linux/arch/riscv  .**
4. 重置为默认配置，**make ARCH=riscv defconfig**
5. 设置交叉编译器，**make ARCH=riscv menuconfig**，修改setup->Cross-compiler tool prefix
6. 编译，**make ARCH=riscv**

### 构建coreboot

1. 获取源码，**git clone https://review.coreboot.org/coreboot.git**
2. 构建交叉编译器，**make crossgcc-ricsv**
3. 配置，**make menuconfig**
4. 编译，**make**


make menuconfig需要注意：
>修改Mainboard->Mainboard vendor为Emulation
>修改Mainboard->Mainboard model为SPIKE ucb riscv
>修改Payload->Add a payload为An ELF executable payload
>修改Payload->Payload path and filename为编译产生的Linux镜像文件的路径
>修改Chipset->Location of pointer to RISCV config string，地址需要与模拟器中configstring地址匹配

## 调试

### 串口

coreboot使用8250串口作为终端，这并没有合并到spike代码中，这需要手动合并。合并过程可以参考clint实现。

带8250串口的源码位于https://github.com/riscv/riscv-isa-sim/pull/53

### 寄存器错误

这和特权指令集的变更相关，mcounter寄存器地址由0x320变更为0x306。

这个patch我已经提交，https://review.coreboot.org/#/c/20043/

### 架构判定宏

这和工具链的更改相关，老版本gcc通过**\_\_riscv__**来标示架构，新版本gcc使用**__riscv**。这导致加载文件到内存时出现内存访问不对齐的异常。

这个patch我已经提交，https://review.coreboot.org/#/c/20125/

### configstring问题

configstring是一个字符串，用于描述一些设备信息。当前通过spike硬编码进ROM实现，但官方的spike并没有这部分代码，这需要自己添加。

而且configstring是一个临时解决办法，将来coreboot会使用linux的设备描述信息fdt。可以参见https://mail.coreboot.org/pipermail/coreboot/2017-June/084534.html

### linux内核加载问题

内核编译出来后，起始地址为0xffffffff80000000。而spike内存范围0x80000000-0xffffffff，这会导致加载失败。可以通过链接器再链接一次修改内核的起始地址。这里使用0x90000000作为起始地址，因为0x80000000处有coreboot的镜像文件。

创建链接脚本

```
ENTRY(_start)
SECTIONS
{
	. = 0x90000000;
	.data : {
		*.*
	}
}
```

执行命令**riscv64-unknown-elf-ld -T payload.ld vmlinux -o vmlinux.payload**

输出的文件即可作为payload

# 代码分析

## 基本结构

coreboot引导过程被分为多个过程。

- bootblock : 初始化部分硬件（Flash），引导加载romstage
- romstage : 初始化存储器和部分芯片组，并引导执行ramstage
- ramstage : 设备枚举资源分配，创建ACPI表和SMM句柄，并加载执行payload
- payload : 操作系统、操作系统引导程序或标准固件等

## bootblock

此部分是机器上电最先执行的代码，负责初始化部分硬件然后引导romstage。

此不分主要有两个文件组成，一个汇编文件用于初始化C运行环境然后跳转到C代码。一个通用的框架文件，初始化部分硬件后启动下一阶段（romstage）。

两个文件分别位于：

- **src/arch/riscv/bootblock.S**
- **src/lib/bootblock.c**

### 汇编入口

此部分位于**src/arch/riscv/bootblock.S**

主要负责初始化堆栈、遗常、线程数据

```assembly
.section ".text._start", "ax", %progbits

.globl _stack
.global _estack				//栈内存的起始和结束符号（_stack < _estack）

.globl _start
_start:

	# N.B. This only works on low 4G of the address space
	# and the stack must be page-aligned.
	la sp, _estack			//初始化堆栈

	# poison the stack
	la t1, _stack
	li t0, 0xdeadbeef
	sd t0, 0(t1)			//在栈顶放一个标记

	# make room for HLS and initialize it
	addi sp, sp, -HLS_SIZE	//开辟空间存放线程数据，线程数据存放在堆栈开始位置

	// Once again, the docs and toolchain disagree.
	// Rather than get fancy I'll just lock this down
	// until it all stabilizes.
	//csrr a0, mhartid
	csrr a0, 0xf14			//获取hart ID
	call hls_init			//初始化线程数据

	la t0, trap_entry
	csrw mtvec, t0			//初始化异常处理

	# clear any pending interrupts
	csrwi mip, 0			//清除中断标记

	# set up the mstatus register for VM
	call mstatus_init		//中断设置、初始化定时器
	tail main				//跳转到bootblock的主程序
```

### 框架

此部分位于**src/lib/bootblock.c**

此文件提供了一个初始化框架。

其中需要提供时钟相关初始化**init_timer**

具体设备根据自身情况实现4个函数（riscv当前只使用软件模拟，所有没有实现）

>bootblock_soc_early_init
>bootblock_mainboard_early_init
>bootblock_soc_init
>bootblock_mainboard_init


```c
//声明一些弱符号的空函数，当具体架构有实现时调用具体架构的函数，如果没有使用这些函数
__attribute__((weak)) void bootblock_mainboard_early_init(void) { /* no-op */ }
__attribute__((weak)) void bootblock_soc_early_init(void) { /* do nothing */ }
__attribute__((weak)) void bootblock_soc_init(void) { /* do nothing */ }
__attribute__((weak)) void bootblock_mainboard_init(void) { /* do nothing */ }

asmlinkage void bootblock_main_with_timestamp(uint64_t base_timestamp)
{
	/* Initialize timestamps if we have TIMESTAMP region in memlayout.ld. */
	if (IS_ENABLED(CONFIG_COLLECT_TIMESTAMPS) && _timestamp_size > 0)
		timestamp_init(base_timestamp);

	cmos_post_init();						//cmos初始化

	bootblock_soc_early_init();				//soc早期初始化
	bootblock_mainboard_early_init();		//主板早期初始化

	if (IS_ENABLED(CONFIG_BOOTBLOCK_CONSOLE)) {
		console_init();						//串口初始化
		exception_init();					//异常初始化
	}

	bootblock_soc_init();					//与bootblock相关的SOC初始化
	bootblock_mainboard_init();				//与bootblock相关的主板初始化

	run_romstage();							//运行下一个阶段程序（romstage）
}

void main(void)
{
	uint64_t base_timestamp = 0;

	init_timer();							//时钟初始化
	
	if (IS_ENABLED(CONFIG_COLLECT_TIMESTAMPS))
		base_timestamp = timestamp_get();	//获取时间戳

	bootblock_main_with_timestamp(base_timestamp);
}
```

在riscv中，只使用了串口uart8250。对于uart8250只需要实现一个函数（**uart_platform_base**），用来获取设备地址。此函数在**src/mainboard/emulation/spike-riscv/uart.c**中实现

### 异常处理分析

从bootblock.S中看出，异常入口为**trap_entry**。此部分实现也分了两个部分，**trap_util.S**用于处理现场保护和恢复，trap_hendler为中断逻辑部分。

#### 现场保护结构体

```c
typedef struct
{
	uintptr_t gpr[32];
	uintptr_t status;
	uintptr_t epc;
	uintptr_t badvaddr;
	uintptr_t cause;
	uintptr_t insn;
} trapframe;
```

#### 现场保护代码分析

```assembly
#define STORE    sd		# 用于处理32位、64位、128位系统寄存器宽度不一样的问题
#define LOAD     ld		# 用于处理32位、64位、128位系统寄存器宽度不一样的问题

#define LOG_REGBYTES 3
#define REGBYTES (1 << LOG_REGBYTES)	# 用于描述一个寄存器有多少个字节

.macro save_tf
  # save gprs
  # 保存通用寄存器，此时X2（sp）已经保存在mscratch中
  STORE  x1,1*REGBYTES(x2)
  STORE  x3,3*REGBYTES(x2)
  STORE  x4,4*REGBYTES(x2)
  STORE  x5,5*REGBYTES(x2)
  STORE  x6,6*REGBYTES(x2)
  STORE  x7,7*REGBYTES(x2)
  STORE  x8,8*REGBYTES(x2)
  STORE  x9,9*REGBYTES(x2)
  STORE  x10,10*REGBYTES(x2)
  STORE  x11,11*REGBYTES(x2)
  STORE  x12,12*REGBYTES(x2)
  STORE  x13,13*REGBYTES(x2)
  STORE  x14,14*REGBYTES(x2)
  STORE  x15,15*REGBYTES(x2)
  STORE  x16,16*REGBYTES(x2)
  STORE  x17,17*REGBYTES(x2)
  STORE  x18,18*REGBYTES(x2)
  STORE  x19,19*REGBYTES(x2)
  STORE  x20,20*REGBYTES(x2)
  STORE  x21,21*REGBYTES(x2)
  STORE  x22,22*REGBYTES(x2)
  STORE  x23,23*REGBYTES(x2)
  STORE  x24,24*REGBYTES(x2)
  STORE  x25,25*REGBYTES(x2)
  STORE  x26,26*REGBYTES(x2)
  STORE  x27,27*REGBYTES(x2)
  STORE  x28,28*REGBYTES(x2)
  STORE  x29,29*REGBYTES(x2)
  STORE  x30,30*REGBYTES(x2)
  STORE  x31,31*REGBYTES(x2)
  
  # get sr, epc, badvaddr, cause
  csrrw  t0,mscratch,x0		# 从mscratch中取出X2
  
  # 取出其他异常相关寄存器
  csrr   s0,mstatus
  csrr   t1,mepc
  csrr   t2,mbadaddr
  csrr   t3,mcause
  STORE  t0,2*REGBYTES(x2)	#保存X2
  
  #保存其他异常相关的寄存器
  STORE  s0,32*REGBYTES(x2)
  STORE  t1,33*REGBYTES(x2)
  STORE  t2,34*REGBYTES(x2)
  STORE  t3,35*REGBYTES(x2)
```

以上定义了一个宏**save_tf**来进行现场保护，在调用此宏之前需要保存用户的X2到mscratch。并且初始化X2（指向现场保护地址）

#### 现场恢复

```assembly
.macro restore_regs
    # restore x registers
    LOAD  x1,1*REGBYTES(a0)
    LOAD  x2,2*REGBYTES(a0)
    LOAD  x3,3*REGBYTES(a0)
    LOAD  x4,4*REGBYTES(a0)
    LOAD  x5,5*REGBYTES(a0)
    LOAD  x6,6*REGBYTES(a0)
    LOAD  x7,7*REGBYTES(a0)
    LOAD  x8,8*REGBYTES(a0)
    LOAD  x9,9*REGBYTES(a0)
    LOAD  x11,11*REGBYTES(a0)
    LOAD  x12,12*REGBYTES(a0)
    LOAD  x13,13*REGBYTES(a0)
    LOAD  x14,14*REGBYTES(a0)
    LOAD  x15,15*REGBYTES(a0)
    LOAD  x16,16*REGBYTES(a0)
    LOAD  x17,17*REGBYTES(a0)
    LOAD  x18,18*REGBYTES(a0)
    LOAD  x19,19*REGBYTES(a0)
    LOAD  x20,20*REGBYTES(a0)
    LOAD  x21,21*REGBYTES(a0)
    LOAD  x22,22*REGBYTES(a0)
    LOAD  x23,23*REGBYTES(a0)
    LOAD  x24,24*REGBYTES(a0)
    LOAD  x25,25*REGBYTES(a0)
    LOAD  x26,26*REGBYTES(a0)
    LOAD  x27,27*REGBYTES(a0)
    LOAD  x28,28*REGBYTES(a0)
    LOAD  x29,29*REGBYTES(a0)
    LOAD  x30,30*REGBYTES(a0)
    LOAD  x31,31*REGBYTES(a0)
    # restore a0 last
    LOAD  x10,10*REGBYTES(a0)
```

现场恢复比较简单，此宏在使用前需要初始化a0（执行现场保护的内存）

#### 异常处理

异常处理主要处理了，内存非对齐访问、S-Mode系统调用、以及中断。其他异常会打印异常信息，然后停止运行。

```c
void trap_handler(trapframe *tf)
{
	write_csr(mscratch, tf);	/* 把异常保护地址保存到mscratch */
	if (tf->cause & 0x8000000000000000ULL) {
		interrupt_handler(tf);	/* 中断处理 */
		return;
	}
	
    /* 异常处理 */
	switch(tf->cause) {
		case CAUSE_MISALIGNED_FETCH:	/* 取指不对齐 */
		case CAUSE_FAULT_FETCH:			/* 取指失败 */
		case CAUSE_ILLEGAL_INSTRUCTION:	/* 非法指令 */
		case CAUSE_BREAKPOINT:			/* 断点指令 */
		case CAUSE_FAULT_LOAD:			/* LOAD失败 */
		case CAUSE_FAULT_STORE:			/* STORE失败 */
		case CAUSE_USER_ECALL:			/* 来自U-Mode的系统调用 */
		case CAUSE_HYPERVISOR_ECALL:	/* 来自H-Mode的系统调用 */
		case CAUSE_MACHINE_ECALL:		/* 来自M-Mode的系统调用 */
			print_trap_information(tf);	/* 打印异常信息 */
			break;
		case CAUSE_MISALIGNED_LOAD:
			print_trap_information(tf);	/* 打印异常信息 */
			handle_misaligned_load(tf);	/* 通过字节访问实现非对齐访问内存 */
			break;
		case CAUSE_MISALIGNED_STORE:
			print_trap_information(tf);	/* 打印异常信息 */
			handle_misaligned_store(tf);/* 通过字节访问实现非对齐访问内存 */
			break;
		case CAUSE_SUPERVISOR_ECALL:
			/* Don't print so we make console putchar calls look
			   the way they should */
			handle_supervisor_call(tf);	/* 处理S—Mode的系统调用 */
			break;
		default:
			printk(BIOS_EMERG, "================================\n");
			printk(BIOS_EMERG, "Coreboot: can not handle a trap:\n");
			printk(BIOS_EMERG, "================================\n");
			print_trap_information(tf);	/* 打印异常信息 */
			break;
	}
	die("Can't recover from trap. Halting.\n");
}
```

##### 内存非对齐访问

此部分处理非对齐访问内存引起的异常，并通过字节操作实现非对齐内存访问。（只实现了64位）

```c
//获取指令
static uint32_t fetch_instruction(uintptr_t vaddr) {
	printk(BIOS_SPEW, "fetching instruction at 0x%016zx\n", (size_t)vaddr);
	return mprv_read_u32((uint32_t *) vaddr);
}

void handle_misaligned_load(trapframe *tf) {
	printk(BIOS_DEBUG, "Trapframe ptr:      %p\n", tf);
  
    //获取异常发生的地址
	uintptr_t faultingInstructionAddr = tf->epc;
  
  	//获取异常的指令
	insn_t faultingInstruction = fetch_instruction(faultingInstructionAddr);
	printk(BIOS_DEBUG, "Faulting instruction: 0x%x\n", faultingInstruction);
  	
  	//获取访问内存的宽度
	insn_t widthMask = 0x7000;
	insn_t memWidth = (faultingInstruction & widthMask) >> 12;
  	
  	//获取目标寄存器编号
	insn_t destMask = 0xF80;
	insn_t destRegister = (faultingInstruction & destMask) >> 7;
  
	printk(BIOS_DEBUG, "Width: 0x%x\n", memWidth);
	if (memWidth == 3) {//只处理64位内存访问
		// load double, handle the issue
		void* badAddress = (void*) tf->badvaddr;
		uint64_t value = 0;
		for (int i = 0; i < 8; i++) {//按字节读取
			value <<= 8;
			value += mprv_read_u8(badAddress+i);
		}
		tf->gpr[destRegister] = value;//回写到现场保护
	} else {
		// panic, this should not have happened
		die("Code should not reach this path, misaligned on a non-64 bit store/load\n");
	}

	// return to where we came from
	write_csr(mepc, read_csr(mepc) + 4);
	asm volatile("j machine_call_return");
}

void handle_misaligned_store(trapframe *tf) {
	printk(BIOS_DEBUG, "Trapframe ptr:      %p\n", tf);
  	
  	//获取异常发生的地址
	uintptr_t faultingInstructionAddr = tf->epc;
  
  	//获取异常的指令
	insn_t faultingInstruction = fetch_instruction(faultingInstructionAddr);
	printk(BIOS_DEBUG, "Faulting instruction: 0x%x\n", faultingInstruction);
  	
  	//获取访问内存的宽度
	insn_t widthMask = 0x7000;
	insn_t memWidth = (faultingInstruction & widthMask) >> 12;
  	
  	//获取源寄存器编号
	insn_t srcMask = 0x1F00000;
	insn_t srcRegister = (faultingInstruction & srcMask) >> 20;
  
	printk(BIOS_DEBUG, "Width: 0x%x\n", memWidth);
	if (memWidth == 3) {//只处理64位内存访问
		// store double, handle the issue
		void* badAddress = (void*) tf->badvaddr;
		uint64_t value = tf->gpr[srcRegister];//从现场保护获取寄存器的值
		for (int i = 0; i < 8; i++) {//按字节回写到内存中去
			mprv_write_u8(badAddress+i, value);
			value >>= 8;
		}
	} else {
		// panic, this should not have happened
		die("Code should not reach this path, misaligned on a non-64 bit store/load\n");
	}

	// return to where we came from
	write_csr(mepc, read_csr(mepc) + 4);
	asm volatile("j machine_call_return");
}
```

##### S-Mode系统调用

```c
void handle_supervisor_call(trapframe *tf) {
	uintptr_t call = tf->gpr[17]; /* a7 */
	uintptr_t arg0 = tf->gpr[10]; /* a0 */
	uintptr_t arg1 = tf->gpr[11]; /* a1 */
	uintptr_t returnValue;
	switch(call) {
		case MCALL_HART_ID:			//获取核心编号
			printk(BIOS_DEBUG, "Getting hart id...\n");
			returnValue = read_csr(0xf14);//mhartid);
			break;
		case MCALL_NUM_HARTS:		//获取核心数
			/* TODO: parse the hardware-supplied config string and
			   return the correct value */
			returnValue = 1;
			break;
		case MCALL_CONSOLE_PUTCHAR:	//向终端打印字符
			returnValue = mcall_console_putchar(arg0);
			break;
        
        //SBI相关，当前未实现
		case MCALL_SEND_IPI:
			printk(BIOS_DEBUG, "Sending IPI...\n");
			returnValue = mcall_send_ipi(arg0);
			break;
		case MCALL_CLEAR_IPI:
			printk(BIOS_DEBUG, "Clearing IPI...\n");
			returnValue = mcall_clear_ipi();
			break;
        
		case MCALL_SHUTDOWN:		//关机
			printk(BIOS_DEBUG, "Shutting down...\n");
			returnValue = mcall_shutdown();
			break;
		case MCALL_SET_TIMER:		//设置定时器
			returnValue = mcall_set_timer(arg0);
			break;
		case MCALL_QUERY_MEMORY:	//获取内存信息
			printk(BIOS_DEBUG, "Querying memory, CPU #%lld...\n", arg0);
			returnValue = mcall_query_memory(arg0, (memory_block_info*) arg1);
			break;
		default:
			printk(BIOS_DEBUG, "ERROR! Unrecognized SBI call\n");
			returnValue = 0;
			break; // note: system call we do not know how to handle
	}
	tf->gpr[10] = returnValue;
	write_csr(mepc, read_csr(mepc) + 4);
	asm volatile("j supervisor_call_return");
}
```

##### 中断

中断只处理了M-Mode的时钟中断。**gettime**函数用于从configstring获取定时器寄存器映射到内存的地址。

```c
static void interrupt_handler(trapframe *tf)
{
	uint64_t cause = tf->cause & ~0x8000000000000000ULL;
	uint32_t msip, ssie;

	switch (cause) {
	case IRQ_M_TIMER:
		// The only way to reset the timer interrupt is to
		// write mtimecmp. But we also have to ensure the
		// comparison fails, for a long time, to let
		// supervisor interrupt handler compute a new value
		// and set it. Finally, it fires if mtimecmp is <=
		// mtime, not =, so setting mtimecmp to 0 won't work
		// to clear the interrupt and disable a new one. We
		// have to set the mtimecmp far into the future.
		// Akward!
		//
		// Further, maybe the platform doesn't have the
		// hardware or the payload never uses it. We hold off
		// querying some things until we are sure we need
		// them. What to do if we can not find them? There are
		// no good options.

		// This hart may have disabled timer interrupts.  If
		// so, just return. Kernels should only enable timer
		// interrupts on one hart, and that should be hart 0
		// at present, as we only search for
		// "core{0{0{timecmp" above.
		ssie = read_csr(sie);
		if (!(ssie & SIE_STIE))
			break;

		if (!timecmp)
			gettimer();
		//printk(BIOS_SPEW, "timer interrupt\n");
		*timecmp = (uint64_t) -1;
		msip = read_csr(mip);
		msip |= SIP_STIP;
		write_csr(mip, msip);
		break;
	default:
		printk(BIOS_EMERG, "======================================\n");
		printk(BIOS_EMERG, "Coreboot: Unknown machine interrupt: 0x%llx\n",
		       cause);
		printk(BIOS_EMERG, "======================================\n");
		print_trap_information(tf);
		break;
	}
}
```

## romstage

romstage位于**src/mainboard/emulation/spike-riscv/romstage.c**中

此文件比较简单，只有对串口的初始化，然后直接引导ramstage

```c
void main(void)
{
	uintptr_t base;
	size_t size;
	
  	//初始化串口终端
	console_init();
    
    //从configstring中获取内存配置信息
	query_mem(configstring(), &base, &size);
	printk(BIOS_SPEW, "0x%zx bytes of memory at 0x%llx\n", size, base);
  
    //引导ramstage
	run_ramstage();
}
```

## ramstage

ramstage位于**src/lib/hardwaremain.c**中，与romstage类似，他提供了一个框架，初始化设备，然后引导下一级payload。但与romstage相比，ramstage具有更强的灵活性。

主程序位于hardwaremain.c文件中的main函数。

```c
void main(void)
{
	/*
	 * We can generally jump between C and Ada code back and forth
	 * without trouble. But since we don't have an Ada main() we
	 * have to do some Ada package initializations that GNAT would
	 * do there. This has to be done before calling any Ada code.
	 *
	 * The package initializations should not have any dependen-
	 * cies on C code. So we can call them here early, and don't
	 * have to worry at which point we can start to use Ada.
	 */
	//给Ada语言初始化环境，便于Ada和C混合编程
	//在代码中暂时没有发现Ada的代码
	ramstage_adainit();

	/* TODO: Understand why this is here and move to arch/platform code. */
	/* For MMIO UART this needs to be called before any other printk. */
	if (IS_ENABLED(CONFIG_ARCH_X86))
		init_timer();//时钟初始化

	/* console_init() MUST PRECEDE ALL printk()! Additionally, ensure
	 * it is the very first thing done in ramstage.*/
	console_init();//终端初始化

	post_code(POST_CONSOLE_READY);

	/*
	 * CBMEM needs to be recovered in the EARLY_CBMEM_INIT case because
	 * timestamps, APCI, etc rely on the cbmem infrastructure being
	 * around. Explicitly recover it.
	 */
	if (IS_ENABLED(CONFIG_EARLY_CBMEM_INIT))
		cbmem_initialize();

	/* Record current time, try to locate timestamps in CBMEM. */
	timestamp_init(timestamp_get());

	timestamp_add_now(TS_START_RAMSTAGE);
	post_code(POST_ENTRY_RAMSTAGE);

	/* Handoff sleep type from romstage. */
#if CONFIG_HAVE_ACPI_RESUME
	acpi_is_wakeup();
#endif

	exception_init();
	threads_initialize();

	/* Schedule the static boot state entries. */
	boot_state_schedule_static_entries();

	bs_walk_state_machine();

	die("Boot state machine failure.\n");
}
```

### 启动步骤

ramstage把启动分为如下步骤

```c
typedef enum {
	BS_PRE_DEVICE,
	BS_DEV_INIT_CHIPS,
	BS_DEV_ENUMERATE,
	BS_DEV_RESOURCES,
	BS_DEV_ENABLE,
	BS_DEV_INIT,
	BS_POST_DEVICE,
	BS_OS_RESUME_CHECK,
	BS_OS_RESUME,
	BS_WRITE_TABLES,
	BS_PAYLOAD_LOAD,
	BS_PAYLOAD_BOOT,
} boot_state_t;
```

每一个步骤对应一个结构体**boot_state**。每个步骤被分为3个阶段，前后通过枚举**boot_state_sequence_t**描述。**boot_state_callback**标示一个回调对象列表，即一个步骤前后可以执行一系列的方法。

```c
struct boot_state_callback {			/* 回调对象 */
	void *arg;							/* 参数 */
	void (*callback)(void *arg);		/* 回调的方法 */
	/* For use internal to the boot state machine. */
	struct boot_state_callback *next;	/* 用于多个回调对象构成链表 */
#if IS_ENABLED(CONFIG_DEBUG_BOOT_STATE)
	const char *location;
#endif
};

typedef enum {							/* 用于描述一个步骤的执行状态 */
	BS_ON_ENTRY,
	BS_ON_EXIT
} boot_state_sequence_t;

struct boot_phase {						/* 用于处理每个步骤情后的事 */
  										/* 回调对象 */
	struct boot_state_callback *callbacks;
	int blockers;
};

struct boot_state {
	const char *name;					/* 名字 */
	boot_state_t id;					/* id对应一个boot_state_t的枚举对象 */
	u8 post_code;						/* 调试用，可以输出到终端 */
	struct boot_phase phases[2];		/* 一个步骤前后需要处理的回调，下标对应boot_state_sequence_t */
	boot_state_t (*run_state)(void *arg);/* 一个步骤的主方法 */
	void *arg;							/* 步骤方法的参数 */
	int complete : 1;					/* 标记步骤处理完成 */
#if CONFIG_HAVE_MONOTONIC_TIMER
	struct boot_state_times times;
#endif
};
```

对每一个步骤定义了一个宏，用于快速生成一个**boot_state**，宏定义如下

```c
#define BS_INIT(state_, run_func_)				\
	{							\
		.name = #state_,				\
		.id = state_,					\
		.post_code = POST_ ## state_,			\
		.phases = { { NULL, 0 }, { NULL, 0 } },		\
		.run_state = run_func_,				\
		.arg = NULL,					\
		.complete = 0,					\
	}
```

所有的步骤通过一个**boot_state**数组描述

```c
#define BS_INIT_ENTRY(state_, run_func_)	\
	[state_] = BS_INIT(state_, run_func_)

static struct boot_state boot_states[] = {
	BS_INIT_ENTRY(BS_PRE_DEVICE, bs_pre_device),
	BS_INIT_ENTRY(BS_DEV_INIT_CHIPS, bs_dev_init_chips),
	BS_INIT_ENTRY(BS_DEV_ENUMERATE, bs_dev_enumerate),
	BS_INIT_ENTRY(BS_DEV_RESOURCES, bs_dev_resources),
	BS_INIT_ENTRY(BS_DEV_ENABLE, bs_dev_enable),
	BS_INIT_ENTRY(BS_DEV_INIT, bs_dev_init),
	BS_INIT_ENTRY(BS_POST_DEVICE, bs_post_device),
	BS_INIT_ENTRY(BS_OS_RESUME_CHECK, bs_os_resume_check),
	BS_INIT_ENTRY(BS_OS_RESUME, bs_os_resume),
	BS_INIT_ENTRY(BS_WRITE_TABLES, bs_write_tables),
	BS_INIT_ENTRY(BS_PAYLOAD_LOAD, bs_payload_load),
	BS_INIT_ENTRY(BS_PAYLOAD_BOOT, bs_payload_boot),
};
```

### 启动步骤定制

特定平台可以对启动步骤进行定制，主要是在步骤前后添加回调对象。**boot_states**中的对象，前后回调是空的。

通过**boot_state_init_entry**结构体描述，启动步骤的什么位置需要调用那个方法。

```c
struct boot_state_init_entry {
	boot_state_t state;					/* 指定步骤 */
	boot_state_sequence_t when;			/* 指定步骤的前后 */
	struct boot_state_callback bscb;	/* 回调对象 */
};
```

软件通过扫描所有的**boot_state_init_entry**对**boot_states**中的**phases**初始化

为了软件便于访问到所有的**boot_state_init_entry**对象，定义了如下宏

```c
//定义了一个属性，声明成此属性的变量将被存放到.bs_init段中
#define BOOT_STATE_INIT_ATTR  __attribute__ ((used, section(".bs_init")))

//定义一个boot_state_init_entry结构体，并把它的指针放到.bs_init段中
#define BOOT_STATE_INIT_ENTRY(state_, when_, func_, arg_)		\
	static struct boot_state_init_entry func_ ##_## state_ ##_## when_ = \
	{								\
		.state = state_,					\
		.when = when_,						\
		.bscb = BOOT_STATE_CALLBACK_INIT(func_, arg_),		\
	};								\
	static struct boot_state_init_entry *				\
		bsie_ ## func_ ##_## state_ ##_## when_ BOOT_STATE_INIT_ATTR = \
		&func_ ##_## state_ ##_## when_;
```

链接脚本中做了如下处理，开放.bs_init段的起始符号，并在指针的结尾放了一个0。这样访问到空指针就可以结束了。脚本位于**src/lib/program.ld**

```
110     . = ALIGN(ARCH_POINTER_ALIGN_SIZE);
111     _bs_init_begin = .;                                                                                                         
112     KEEP(*(.bs_init));
113     LONG(0);
```

然后通过**boot_state_schedule_static_entries**函数绑定**boot_state_init_entry**到**boot_states**

```c
static void boot_state_schedule_static_entries(void)
{
	extern struct boot_state_init_entry *_bs_init_begin[];
	struct boot_state_init_entry **slot;
	
    /* 扫描所有的boot_state_init_entry结构体指针 */
	for (slot = &_bs_init_begin[0]; *slot != NULL; slot++) {
		struct boot_state_init_entry *cur = *slot;

		if (cur->when == BS_ON_ENTRY)
			boot_state_sched_on_entry(&cur->bscb, cur->state);
		else
			boot_state_sched_on_exit(&cur->bscb, cur->state);
	}
}
```

# coreboot riscv实现当前存在的问题

## 多核

当前由于不确定系统调用接口问题，导致不知道那些寄存器可以改变，现在保存了所有的现场导致没有更多的寄存器去处理当前那些寄存器可以使用。这部分讨论可以参见**https://mail.coreboot.org/pipermail/coreboot/2017-May/084378.html**，相关回复引用如下:

> I never tested the code on an SMP system or properly thought through supporting SMP, but I remember the following problem: I wanted to give each hart its own stack (IOW its own initial stack pointer value) to avoid race conditions, but I didn't have enough free registers to calculate it. So I gave up and only allowed hart 0 to do SBI calls. What I didn't think about is that I could *probably* use all registers that are defined as caller-saved by the user-level spec. ("probably" because I don't know if that's a valid thing to do. The Privileged Spec should specify which registers are saved by the M-mode code across SBI calls, and which aren't.)

## FDT

coreboot当前使用configstring来传入配置信息，未来涉想使用类似ARM的FDT来实现。这部分讨论可以参见https://mail.coreboot.org/pipermail/coreboot/2017-June/084534.html，相关回复引用如下：


>On Mon, Jun 12, 2017 at 12:13 AM 王翔 <merle at tya.email> wrote:
>
>>
>> The configstring implement by spike or other SOC?
>>
>
>yes.
>
>But it seems they will eventually use FDT, since most of the people in the
>discussion think Linux compatibility is the single most important thing.
>ron

## SBI

SBI为操作系统调用接口，此部分当前基本没有实现，不清除具体需要实现那些功能。


