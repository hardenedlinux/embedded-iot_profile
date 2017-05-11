# ARM Trusted Firmware历史漏洞分析

## CVE-2016-10319

### 概要

在ATF（ARM Trusted Firmware）的BL1中，有一个FWU（Firmware Update feature）功能。此功能用于更新固件，BL1中实现了Secure部分。如果要实现更新固件功能需要实现BL1U，Non-Secure部分。不过BL1U在ATF中并没有实现。

BL1的FWU存在一个整数溢出的漏洞，此漏洞可以导致在Non-Secure中通过SMC调用实现缓冲区溢出攻击。

漏洞代码获取
```shell
git clone git@github.com:ARM-software/arm-trusted-firmware.git CVE-2016-10319
cd  CVE-2016-10319
git checkout 48bfb88
```

### 漏洞代码分析

FWU功能主要在`bl1/bl1_fwu.c`文件中实现，`bl1_fwu_image_copy`用于把镜像从Non-Secure空间拷贝到Secure空间，此代码存在漏洞，代码如下

```c
/*******************************************************************************
 * This function is responsible for copying secure images in AP Secure RAM.
 ******************************************************************************/
static int bl1_fwu_image_copy(unsigned int image_id,
			uintptr_t image_src,
			unsigned int block_size,
			unsigned int image_size,
			unsigned int flags)
{
	uintptr_t base_addr;
	meminfo_t *mem_layout;

	/* Get the image descriptor. */
	image_desc_t *image_desc = bl1_plat_get_image_desc(image_id);

	/* Check if we are in correct state. */
    /* FWU功能由多次SMC调用组合完才，服务程序有一个状态机来记录执行到那一步，此处验证状态是否正常 */
	if ((!image_desc) ||
		((image_desc->state != IMAGE_STATE_RESET) &&
		 (image_desc->state != IMAGE_STATE_COPYING))) {
		WARN("BL1-FWU: Copy not allowed due to invalid state\n");
		return -EPERM;
	}

	/* Only Normal world is allowed to copy a Secure image. */
    /* 此SMC服务，只能由Nor-Secure调用，用于更新Secure的固件 */
	if ((GET_SEC_STATE(flags) == SECURE) ||
		(GET_SEC_STATE(image_desc->ep_info.h.attr) == NON_SECURE)) {
		WARN("BL1-FWU: Copy not allowed for Non-Secure "
			 "image from Secure-world\n");
		return -EPERM;
	}

	if ((!image_src) || (!block_size)) {
		WARN("BL1-FWU: Copy not allowed due to invalid image source"
			" or block size\n");
		return -ENOMEM;
	}

	/* Get the image base address. */
	base_addr = image_desc->image_info.image_base;

	if (image_desc->state == IMAGE_STATE_COPYING) {
      	/* 拷贝可以通过多次SMC调用组合完成
      	   IMAGE_STATE_COPYING状态用于标示 执行了第一次拷贝但拷贝还没有完才 */
		/*
		 * If last block is more than expected then
		 * clip the block to the required image size.
		 */
      	/* BUG 此处加法存在整数溢出风险，block_size由SMC调用传入
      	 * 此处可以传入一个很大的block_size使加法溢出，从而通过条件判定
      	 * 导致下面拷贝（memcpy）溢出
         */
		if (image_desc->image_info.copied_size + block_size >
			 image_desc->image_info.image_size) {
			block_size = image_desc->image_info.image_size -
				image_desc->image_info.copied_size;
			WARN("BL1-FWU: Copy argument block_size > remaining image size."
				" Clipping block_size\n");
		}

		/* Make sure the image src/size is mapped. */
		if (bl1_plat_mem_check(image_src, block_size, flags)) {
			WARN("BL1-FWU: Copy arguments source/size not mapped\n");
			return -ENOMEM;
		}

		INFO("BL1-FWU: Continuing image copy in blocks\n");

		/* Copy image for given block size. */
		base_addr += image_desc->image_info.copied_size;
		image_desc->image_info.copied_size += block_size;
		memcpy((void *)base_addr, (const void *)image_src, block_size);
		flush_dcache_range(base_addr, block_size);

		/* Update the state if last block. */
		if (image_desc->image_info.copied_size ==
				image_desc->image_info.image_size) {
			image_desc->state = IMAGE_STATE_COPIED;
			INFO("BL1-FWU: Image copy in blocks completed\n");
		}
	} else {
		/* This means image is in RESET state and ready to be copied. */
		INFO("BL1-FWU: Fresh call to copy an image\n");

		if (!image_size) {
			WARN("BL1-FWU: Copy not allowed due to invalid image size\n");
			return -ENOMEM;
		}

		/*
		 * If block size is more than total size then
		 * assume block size as the total image size.
		 */
		if (block_size > image_size) {
			block_size = image_size;
			WARN("BL1-FWU: Copy argument block_size > image size."
				" Clipping block_size\n");
		}

		/* Make sure the image src/size is mapped. */
		if (bl1_plat_mem_check(image_src, block_size, flags)) {
			WARN("BL1-FWU: Copy arguments source/size not mapped\n");
			return -ENOMEM;
		}

		/* Find out how much free trusted ram remains after BL1 load */
        /* BUG此处加法存在整数溢出风险，image_size由SMC调用传入
         * 可以传入一个很大的image_size使加法溢出，并绕过此条件判断
         * 从而导致后面的内存拷贝（memcpy）溢出
         */
		mem_layout = bl1_plat_sec_mem_layout();
		if ((image_desc->image_info.image_base < mem_layout->free_base) ||
			 (image_desc->image_info.image_base + image_size >
			  mem_layout->free_base + mem_layout->free_size)) {
			WARN("BL1-FWU: Memory not available to copy\n");
			return -ENOMEM;
		}

		/* Update the image size. */
		image_desc->image_info.image_size = image_size;

		/* Copy image for given size. */
		memcpy((void *)base_addr, (const void *)image_src, block_size);
		flush_dcache_range(base_addr, block_size);

		/* Update the state. */
		if (block_size == image_size) {
			image_desc->state = IMAGE_STATE_COPIED;
			INFO("BL1-FWU: Image is copied successfully\n");
		} else {
			image_desc->state = IMAGE_STATE_COPYING;
			INFO("BL1-FWU: Started image copy in blocks\n");
		}
		image_desc->image_info.copied_size = block_size;
	}

	return 0;
}
```

以上代码两出加法可能会造成整数溢出，导致后面的内存拷贝出现缓冲区溢出问题

### 可利用性

此SMC调用在BL1是建立，被BL31的SMC调用替代。BL1U和BL一起被加载，一般位于芯片内修改比较困难，BL2需要BL1认证后加载一般不会给攻击者替换。

## CVE-2017-7564

### 概要

此漏洞是ARM的一个调试寄存器设置错误。导致Non-Secure通过访问调试寄存器获取到Secure的信息。

代码获取

```
git clone git@github.com:ARM-software/arm-trusted-firmware.git CVE-2016-10319
cd CVE-2016-10319
git checkout 48bfb88
```

### 漏洞分析

ARM具有Self-Hosted调试模块（根一般桌面系统一样）。

MDCR_EL3.SDD是Secure Word Self-Hosted调试的使能位，类似的MDCR_EL3.SPD32为32比特模式下Self-Hosted调试模块的控制寄存器

MDCR_EL3.SDD

​	0 使能Secure Word Self-Hosted调试

​	1 关闭Secure Word Self-Hosted调试

MDCR_EL3.SPD32

​	00 传统模式

​	01 关闭Secure优先级调试，关闭Secure-EL1的调试异常

​	11 使能Secure优先级调试，使能Secure-EL1的调试异常

**el3_entrypoint_common**宏用于el3环境的初始化（大小端、主次处理器、内存、异常），其中调用**el3_arch_init_common**执行架构相关初始化。

漏洞代码

```assembly
/* ---------------------------------------------------------------------
 * Reset registers that may have architecturally unknown reset values
 * ---------------------------------------------------------------------
 */
msr	mdcr_el3, xzr
```

修复后

```assembly
/* MDCR definitions */
#define MDCR_SPD32(x)		((x) << 14)
#define MDCR_SPD32_LEGACY	0x0
#define MDCR_SPD32_DISABLE	0x2
#define MDCR_SPD32_ENABLE	0x3
#define MDCR_SDD_BIT		(1 << 16)

#define MDCR_DEF_VAL		(MDCR_SDD_BIT | MDCR_SPD32(MDCR_SPD32_DISABLE))

/* ---------------------------------------------------------------------
 * Disable secure self-hosted invasive debug.
 * ---------------------------------------------------------------------
 */
mov_imm	x0, MDCR_DEF_VAL
msr	mdcr_el3, x0
```

### 可利用性

Secure EL0、Secure EL1调试异常有两种路由方式Secure EL1、Non-Secure EL2，要利用此漏洞获取Secure信息，需要提前获取Non-Secure EL2的权限。