# 概要
ARM Trusted Firmware实现了一个可信引导过程，并给操作系统提供运行环境（SMC调用服务）。本文分析基于ARM Trusted Firmware - version 1.3，代码托管与[GitHub](https://github.com/ARM-software/arm-trusted-firmware)。

引导过程分为多步，每一步为一个独立的程序。上一步负责验证下一步的程序（可能是多个，BL2会加载认证其他所有的bootloader），并把控制权交给下一级程序。每一级程序需要为下一级程序提供运行环境。最终引导一个通用的bootloader，并实现SMC调用服务。

本文主要分析AArch64相关部分代码

# 安全相关
## auth_common
此模块主要用于声明一些公共的数据结构，这些数据结构主要用与描述认证的方式`auth_method_desc_s`和认证的参数`auth_param_desc_t`。

认证参数`auth_param_desc_t`是由两个基本结构组成：类型`auth_param_type_desc_t`和数据`auth_param_data_desc_t`。

并定义了两个宏用于快速实现一个类型和数据
```
#define AUTH_PARAM_TYPE_DESC(_type, _cookie) \
	{ \
		.type = _type, \
		.cookie = (void *)_cookie \
	}

#define AUTH_PARAM_DATA_DESC(_ptr, _len) \
	{ \
		.ptr = (void *)_ptr, \
		.len = (unsigned int)_len \
	}
```

## crypto_mod
此模块主要实现一个框架，并非具体实现。主要用于校验哈希和签名。

此框架主要基于一个结构体，结构体定义如下:
```
typedef struct crypto_lib_desc_s {
	const char *name;/* 名称， */

	void (*init)(void);/* 初始化方法 */

	/* 校验签名的方法 */
	int (*verify_signature)(
                void *data_ptr, unsigned int data_len,   /* 要签名的数据 */
				void *sig_ptr, unsigned int sig_len,     /* 签名 */
				void *sig_alg, unsigned int sig_alg_len, /* 签名算法 */
				void *pk_ptr, unsigned int pk_len);      /* 公钥 */

	/* 校验哈希的方法 */
	int (*verify_hash)(
               void *data_ptr, unsigned int data_len,       /* 要计算哈希的数据 */
			   void *digest_info_ptr, unsigned int digest_info_len);/* 哈希值 */
} crypto_lib_desc_t;
```

通过`REGISTER_CRYPTO_LIB`宏实现一个名为`crypto_lib_desc`类型为`crypto_lib_desc_t`结构体。宏实现如下:
```
#define REGISTER_CRYPTO_LIB(_name, _init, _verify_signature, _verify_hash) \
	const crypto_lib_desc_t crypto_lib_desc = { \
		.name = _name, \
		.init = _init, \
		.verify_signature = _verify_signature, \
		.verify_hash = _verify_hash \
	}
```

此模块通过操作crypto_lib_desc变量，实现模块初始化、校验签名、校验哈希。函数声明如下：
```
/* 模块初始化 */
void crypto_mod_init(void);

/* 校验签名 */
int crypto_mod_verify_signature(void *data_ptr, unsigned int data_len,
				void *sig_ptr, unsigned int sig_len,
				void *sig_alg, unsigned int sig_alg_len,
				void *pk_ptr, unsigned int pk_len);
/* 校验哈希值 */
int crypto_mod_verify_hash(void *data_ptr, unsigned int data_len,
			   void *digest_info_ptr, unsigned int digest_info_len);
```

## img_parser_mod
此模块实现一个框架，并非具体实现。主要用于校验镜像完整性，以及从镜像中提取内容。

镜像被分为以下几种类型
```
typedef enum img_type_enum {
	IMG_RAW,			/* Binary image */
	IMG_PLAT,			/* Platform specific format */
	IMG_CERT,			/* X509v3 certificate */
	IMG_MAX_TYPES,
} img_type_t;
```

镜像解析器被包装成一个结构体，定义如下
```
typedef struct img_parser_lib_desc_s {
	img_type_t img_type;   /* 镜像解析器操作的镜像的类型 */
	const char *name;      /* 镜像解析器的名字 */

	void (*init)(void);    /* 初始化镜像解析器 */

    /* 校验镜像完整性 */
	int (*check_integrity)(void *img, unsigned int img_len);

    /* 从镜像中提取内容 */
	int (*get_auth_param)(
            const auth_param_type_desc_t *type_desc,  /* 要提取的内容的描述信息 */
			void *img, unsigned int img_len,          /* 镜像内容 */
			void **param, unsigned int *param_len);   /* 输出：提取的内容 */
} img_parser_lib_desc_t;
```

通过宏`REGISTER_IMG_PARSER_LIB`实现一个解析器，定义如下：
```
#define REGISTER_IMG_PARSER_LIB(_type, _name, _init, _check_int, _get_param) \
	static const img_parser_lib_desc_t __img_parser_lib_desc_##_type \
	__section(".img_parser_lib_descs") __used = { \
		.img_type = _type, \
		.name = _name, \
		.init = _init, \
		.check_integrity = _check_int, \
		.get_auth_param = _get_param \
	}
```
多个解析器被放在同一个段`.img_parser_lib_descs`。配合链接脚本此段开始符号为`__PARSER_LIB_DESCS_START__`，结束符号为`__PARSER_LIB_DESCS_END__`。两个符号可以看作解析器数组的起始符号和结束符号。

`img_parser_init`通过遍历解析器数组，调用解析器的初始化函数`init`，对解析器进行初始化。并根据`img_type`构建索引（通过镜像类型找到解析器下标）。并防止有多个相同类型的镜像解析器存在。

`img_parser_check_integrity`校验镜像完整性

`img_parser_get_auth_param`从镜像中提取需要的内容

## auth_mod
auth_mod实现了一个校验镜像的模型，此模型通过结构体`auth_img_desc_t`描述。
```
typedef struct auth_img_desc_s {
	/* 镜像的ID,标志是哪一个镜像 */
	unsigned int img_id;

	/* 镜像类型(Binary、证书等) */
	img_type_t img_type;

	/* 父镜像，保存了认证当前镜像的公钥、哈希等 */
	const struct auth_img_desc_s *parent;

	/* 认证当前镜像的方法 */
	auth_method_desc_t img_auth_methods[AUTH_METHOD_NUM];

	/* 用于校验子镜像的公钥、哈希等 */
	auth_param_desc_t authenticated_data[COT_MAX_VERIFIED_PARAMS];
} auth_img_desc_t;
```

并定义一个宏`REGISTER_COT`，用于注册`auth_img_desc_t`数组
```
#define REGISTER_COT(_cot) \
	const auth_img_desc_t *const cot_desc_ptr = \
			(const auth_img_desc_t *const)&_cot[0]; \
	unsigned int auth_img_flags[sizeof(_cot)/sizeof(_cot[0])]
```

`auth_mod_verify_img`通过`img_id`访问`cot_desc_ptr`数组，找到对应的镜像描述符`auth_method_desc_t`，即可知道当前镜像的认证方式，访问父节点找到签名的公钥或哈希，即可认证当前镜像是否合法。在认证完当前镜像后，从镜像中解析出公钥哈希等放入当前的镜像描述符中，便于对下一级镜像校验
```
int auth_mod_verify_img(unsigned int img_id,
			void *img_ptr,
			unsigned int img_len)
{
	const auth_img_desc_t *img_desc = NULL;
	const auth_method_desc_t *auth_method = NULL;
	void *param_ptr;
	unsigned int param_len;
	int rc, i;

	/* 根据img_id获取镜像描述符 */
	img_desc = &cot_desc_ptr[img_id];

	/* 校验镜像完整性 */
	rc = img_parser_check_integrity(img_desc->img_type, img_ptr, img_len);
	return_if_error(rc);

	/* 根据镜像描述符的仍正方式对镜像进行认证 */
	for (i = 0 ; i < AUTH_METHOD_NUM ; i++) {
		auth_method = &img_desc->img_auth_methods[i];
		switch (auth_method->type) {
		case AUTH_METHOD_NONE:/* 不需要认证 */
			rc = 0;
			break;
		case AUTH_METHOD_HASH:/* 哈希认证 */
			rc = auth_hash(&auth_method->param.hash,
					img_desc, img_ptr, img_len);
			break;
		case AUTH_METHOD_SIG:/* 签名认证 */
			rc = auth_signature(&auth_method->param.sig,
					img_desc, img_ptr, img_len);
			break;
		case AUTH_METHOD_NV_CTR:/* Non-Volatile counter认证？ */
			rc = auth_nvctr(&auth_method->param.nv_ctr,
					img_desc, img_ptr, img_len);
			break;
		default:
			/* 未知认证类型，报错 */
			rc = 1;
			break;
		}
		return_if_error(rc);
	}

	/* 从镜像中解析出公钥哈希等，以便对下一级镜像进行认证 */
	for (i = 0 ; i < COT_MAX_VERIFIED_PARAMS ; i++) {
		if (img_desc->authenticated_data[i].type_desc == NULL) {
			continue;
		}

		/* 通过镜像解析器从镜像中提取内容 */
		rc = img_parser_get_auth_param(img_desc->img_type,
				img_desc->authenticated_data[i].type_desc,
				img_ptr, img_len, &param_ptr, &param_len);
		return_if_error(rc);

		/* 异常检查
		   防止从镜像中解析出的数据字节数大于镜像描述符中的字节数
		   出现内存访问溢出 */
		if (param_len > img_desc->authenticated_data[i].data.len) {
			return 1;
		}

		/* 把解析出的内容拷贝到镜像描述符中，便于解析下一级BL */
		memcpy((void *)img_desc->authenticated_data[i].data.ptr,
				(void *)param_ptr, param_len);
	}

	/* 标记镜像以认证过 */
	auth_img_flags[img_desc->img_id] |= IMG_FLAG_AUTHENTICATED;

	return 0;
}
```


# 设备抽象

为了便于移植ARM Trusted Firmware提供了一种设备抽象机制，这里主要用来抽象镜像读取操作，以便加载镜像到系统内存中。

## 主要数据结构
### io_dev_info_t
io_dev_info_t对应一个设备描述符
```
/* 设备操作相关的函数句柄 */
typedef struct io_dev_funcs {
	io_type_t (*type)(void);
	int (*open)(io_dev_info_t *dev_info, const uintptr_t spec,
			io_entity_t *entity);
	int (*seek)(io_entity_t *entity, int mode, ssize_t offset);
	int (*size)(io_entity_t *entity, size_t *length);
	int (*read)(io_entity_t *entity, uintptr_t buffer, size_t length,
			size_t *length_read);
	int (*write)(io_entity_t *entity, const uintptr_t buffer,
			size_t length, size_t *length_written);
	int (*close)(io_entity_t *entity);
	int (*dev_init)(io_dev_info_t *dev_info, const uintptr_t init_params);
	int (*dev_close)(io_dev_info_t *dev_info);
} io_dev_funcs_t;

typedef struct io_dev_info {
	const struct io_dev_funcs *funcs;/* 设备操作相关的函数句柄 */
	uintptr_t info;/* 设备信息（比如设备地址） */
} io_dev_info_t;
```

### io_entity_t
io_entity_t对应一个文件描述符
```
typedef struct io_entity {
	struct io_dev_info *dev_handle;/* 设备信息 */
	uintptr_t info;/* 文件有关信息 */
} io_entity_t;
```

### io_dev_connector_t
io_dev_connector_t对应一个驱动
```
typedef struct io_dev_connector {
	/* dev_open opens a connection to a particular device driver */
	int (*dev_open)(const uintptr_t dev_spec, io_dev_info_t **dev_info);
} io_dev_connector_t;
```

### 关联
系统通过`dev_count`、`devices`，来维护一个注册的设备列表
```
/* 注册一个设备此变量加1 */
static unsigned int dev_count;
/* 指向设备的指针数组，注册的设备会被添加进来 */
static const io_dev_info_t *devices[MAX_IO_DEVICES];
```

系统通过`entity_pool`、`entity_map`、`entity_count`来维护已经打开的文件
```
/* 实际的文件句柄 */
static io_entity_t entity_pool[MAX_IO_HANDLES];

/* 指针数组用于标记entity_pool中的文件句柄是否空闲 */
static io_entity_t *entity_map[MAX_IO_HANDLES];

/* 记录当前打开的文件句柄书目 */
static unsigned int entity_count;
```

## 接口
### 打开设备
```
int io_dev_open(const struct io_dev_connector *dev_con,/* 驱动描述符 */
		const uintptr_t dev_spec,/* 用于指定打开那个设备，与具体驱动有关 */
		uintptr_t *dev_handle);/* 返回值：设备句柄（io_dev_info_t） */
```

### 设备初始化
```
/*
 * dev_handle设备句柄
 * init_params设备初始化的参数（特定设备相关）
 */
int io_dev_init(uintptr_t dev_handle, const uintptr_t init_params);
```

### 关闭设备
```
int io_dev_close(uintptr_t dev_handle);
```

### 文件相关操作
```
/*
 * 打开文件
 * dev_handle设备句柄
 * spec文件信息（特定设备相关）
 * handle文件句柄
 */
int io_open(uintptr_t dev_handle, const uintptr_t spec, uintptr_t *handle);

/*
 * 移动文件指针
 * handle文件句柄
 * mode移动文件指针的相对初始地址
 * offset移动文件指针的相对地址
 */
int io_seek(uintptr_t handle, io_seek_mode_t mode, ssize_t offset);

/*
 * 获取文件大小
 * handle文件句柄
 * length返回值，文件大小
 */
int io_size(uintptr_t handle, size_t *length);

/*
 * 读文件
 * handle文件句柄
 * buffer是读出缓存的地址
 * length是读出缓存的大小
 * length_read返回值，实际读出的大小
 */
int io_read(uintptr_t handle, uintptr_t buffer, size_t length,
		size_t *length_read);

/*
 * 写文件
 * handle文件句柄
 * buffer要写入文件的缓存的起始地址
 * length要写如文件的缓存的大小
 * length_written返回值，实际写入的大小
 */
int io_write(uintptr_t handle, const uintptr_t buffer, size_t length,
		size_t *length_written);

/* 关闭文件，handle文件句柄 */
int io_close(uintptr_t handle);
```

# BL1分析

## 功能概要

BL1主要负责对BL2进行认证加载并执行，并且提供SMC中断服务，便于执行在EL1-Secure的BL2能给把BL31加载到EL3。

## 主过程

BL1的入口代码位于`bl1/aarch64/bl1_entrypoint.S`文件中，代码注释如下。

```asm
	.globl	bl1_entrypoint
func bl1_entrypoint /* BL1的程序入口 */
	 /* EL3通用的初始化宏 */
	el3_entrypoint_common					\
		_set_endian=1		/* 大小端初始化 */\
		_warm_boot_mailbox=!PROGRAMMABLE_RESET_ADDRESS	/* 热启动分支执行 */\
		_secondary_cold_boot=!COLD_BOOT_SINGLE_CPU /* 次处理器进入低功耗模式 */\
		_init_memory=1	/* 内存初始化 */\
		_init_c_runtime=1	/* 初始化堆栈（SP）等 */\
		_exception_vectors=bl1_exceptions /* 设置异常向量 */
	
	/*--------------------------------------------------
	 * bl1_early_platform_setup
	 * 初始化看梦狗、串口（用于输出调试信息）
	 * 更新BL1内存布局到bl1_tzram_layout
	 *--------------------------------------------------
	 */
	bl	bl1_early_platform_setup
	
	/*--------------------------------------------------
	 * bl1_plat_arch_setup
	 * 根据内存布局设置页表并使能MMU
	 *--------------------------------------------------
	 */
	bl	bl1_plat_arch_setup

	/* --------------------------------------------------
	 * 转到主程序
	 * --------------------------------------------------
	 */
	bl	bl1_main

	/* --------------------------------------------------
	 * 退出异常，执行下一级BL
	 * --------------------------------------------------
	 */
	b	el3_exit
endfunc bl1_entrypoint
```

bl1\_main是bl1的主程序，位于`bl1/bl1_main.c`中

```c
void bl1_main(void)
{
	unsigned int image_id;

	/* 输出一些提示信息 */
	NOTICE(FIRMWARE_WELCOME_STR);
	NOTICE("BL1: %s\n", version_string);
	NOTICE("BL1: %s\n", build_message);

	INFO("BL1: RAM %p - %p\n", (void *)BL1_RAM_BASE,
					(void *)BL1_RAM_LIMIT);

	print_errata_status();

	/*
	 * 异常检测，查看汇编中初始化是否正常
	 */
#if DEBUG
	u_register_t val;
#ifdef AARCH32
	val = read_sctlr();
#else
	val = read_sctlr_el3();
#endif
	assert(val & SCTLR_M_BIT);
	assert(val & SCTLR_C_BIT);
	assert(val & SCTLR_I_BIT);
	val = (read_ctr_el0() >> CTR_CWG_SHIFT) & CTR_CWG_MASK;
	if (val != 0)
		assert(CACHE_WRITEBACK_GRANULE == SIZE_FROM_LOG2_WORDS(val));
	else
		assert(CACHE_WRITEBACK_GRANULE <= MAX_CACHE_LINE_SIZE);
#endif

	/* 设置下一级EL可以运行在64bit */
	bl1_arch_setup();

#if TRUSTED_BOARD_BOOT
	/* 初始化认证模块 */
	auth_mod_init();
#endif /* TRUSTED_BOARD_BOOT */

	/* 具体硬件相关设置（IO外设等） */
	bl1_platform_setup();

	/* 获取下一级要执行的镜像id */
	image_id = bl1_plat_get_next_image_id();
	
	if (image_id == BL2_IMAGE_ID)
		bl1_load_bl2();/* 加载BL2到内存 */
	else
		NOTICE("BL1-FWU: *******FWU Process Started*******\n");

	/* 执行环境配置，在程序结束后可以执行下一个镜像 */
	bl1_prepare_next_image(image_id);
}
```

## 中断服务

BL1异常服务程序位于`bl1/aarch64/bl1_exceptions.S`中

```assembly
/*
 * BL1的异常向量表
 * plat_report_exception用与指示异常与具体架构有关
 * no_ret 是一个bl的宏，用于指示被调用的函数没有返回值
 * plat_panic_handler是一个死循环，在程序出错时停止程序的运行
 * check_vecror_size用于检查代码有没有越界(一个异常向量只有128Byte)
 * BL1只处理低异常等级的SMC调用，其他异常全部非法
 */
	.globl	bl1_exceptions

vector_base bl1_exceptions

	/* -----------------------------------------------------
	 * Current EL with SP0 : 0x0 - 0x200
	 * -----------------------------------------------------
	 */
vector_entry SynchronousExceptionSP0
	mov	x0, #SYNC_EXCEPTION_SP_EL0
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size SynchronousExceptionSP0

vector_entry IrqSP0
	mov	x0, #IRQ_SP_EL0
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size IrqSP0

vector_entry FiqSP0
	mov	x0, #FIQ_SP_EL0
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size FiqSP0

vector_entry SErrorSP0
	mov	x0, #SERROR_SP_EL0
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size SErrorSP0

	/* -----------------------------------------------------
	 * Current EL with SPx: 0x200 - 0x400
	 * -----------------------------------------------------
	 */
vector_entry SynchronousExceptionSPx
	mov	x0, #SYNC_EXCEPTION_SP_ELX
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size SynchronousExceptionSPx

vector_entry IrqSPx
	mov	x0, #IRQ_SP_ELX
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size IrqSPx

vector_entry FiqSPx
	mov	x0, #FIQ_SP_ELX
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size FiqSPx

vector_entry SErrorSPx
	mov	x0, #SERROR_SP_ELX
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size SErrorSPx

	/* -----------------------------------------------------
	 * Lower EL using AArch64 : 0x400 - 0x600
	 * -----------------------------------------------------
	 */
vector_entry SynchronousExceptionA64
	/* 使能SError中断 */
	msr	daifclr, #DAIF_ABT_BIT

	/* 保存x30下面需要使用x30作临时寄存器分析异常的原因 */
	str	x30, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_LR]

	/* Expect only SMC exceptions */
	mrs	x30, esr_el3 /* 获取异常的原因 */
	ubfx	x30, x30, #ESR_EC_SHIFT, #ESR_EC_LENGTH
	cmp	x30, #EC_AARCH64_SMC
	b.ne	unexpected_sync_exception /* 报告非法异常（只处理AArch64 SMC调用） */

	b	smc_handler64 /* 处理低异常等级SMC调用 */
	check_vector_size SynchronousExceptionA64

vector_entry IrqA64
	mov	x0, #IRQ_AARCH64
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size IrqA64

vector_entry FiqA64
	mov	x0, #FIQ_AARCH64
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size FiqA64

vector_entry SErrorA64
	mov	x0, #SERROR_AARCH64
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size SErrorA64

	/* -----------------------------------------------------
	 * Lower EL using AArch32 : 0x600 - 0x800
	 * -----------------------------------------------------
	 */
vector_entry SynchronousExceptionA32
	mov	x0, #SYNC_EXCEPTION_AARCH32
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size SynchronousExceptionA32

vector_entry IrqA32
	mov	x0, #IRQ_AARCH32
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size IrqA32

vector_entry FiqA32
	mov	x0, #FIQ_AARCH32
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size FiqA32

vector_entry SErrorA32
	mov	x0, #SERROR_AARCH32
	bl	plat_report_exception
	no_ret	plat_panic_handler
	check_vector_size SErrorA32


func smc_handler64

	/* RUN_IMAGE SMC调用，用于运行低异常等级的镜像，并执行在EL3，
	 * 即不返回调用的位置，所以需要特别处理 */
	mov	x30, #BL1_SMC_RUN_IMAGE
	cmp	x30, x0
	b.ne	smc_handler/* 其他SMC调用 */
	
	/* 确定在secure world执行RUN_IMAGE SMC调用 */
	mrs	x30, scr_el3
	tst	x30, #SCR_NS_BIT
	b.ne	unexpected_sync_exception/* 报告异常 */
	
	/* 加载出异常退出时保存的SP，切换到SP_EL0 */
	ldr	x30, [sp, #CTX_EL3STATE_OFFSET + CTX_RUNTIME_SP]
	msr	spsel, #0
	mov	sp, x30

	/*
	 * bl1_print_next_bl_ep_info用于打印要执行的镜像文件的地址参数
	 * 这些参数保存在entry_point_info_t结构体中
	 * x1为指针指向entry_point_info_t
	 */
	mov	x20, x1
	mov	x0, x20
	bl	bl1_print_next_bl_ep_info

	ldp	x0, x1, [x20, #ENTRY_POINT_INFO_PC_OFFSET]/* 获取pc、spsr */
	msr	elr_el3, x0/* 修改返回地址，用来在异常退出时执行指定的镜像文件 */
	msr	spsr_el3, x1/* 修改返回的cpsr,可以修改异常等级 */
	ubfx	x0, x1, #MODE_EL_SHIFT, #2
	cmp	x0, #MODE_EL3/* 执行的镜像必须为EL3，为后面的代码放开权限 */
	b.ne	unexpected_sync_exception

	bl	disable_mmu_icache_el3
	tlbi	alle3

#if SPIN_ON_BL1_EXIT
	bl	print_debug_loop_message
debug_loop:
	b	debug_loop
#endif

	mov	x0, x20
	bl	bl1_plat_prepare_exit
	/* 从x1指定的结构体中获取参数，并退出异常 */
	ldp	x6, x7, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x30)]
	ldp	x4, x5, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x20)]
	ldp	x2, x3, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x10)]
	ldp	x0, x1, [x20, #(ENTRY_POINT_INFO_ARGS_OFFSET + 0x0)]
	eret
endfunc smc_handler64

unexpected_sync_exception:
	mov	x0, #SYNC_EXCEPTION_AARCH64
	bl	plat_report_exception
	no_ret	plat_panic_handler

smc_handler:
	bl	save_gp_registers/* 保存x0-x29，LR(x30)在之前已经保存过了 */
	mov	x5, xzr/* x5作为cookie参数传递给异常函数现在未用 */
	mov	x6, sp /* x6为作为handle传递给异常处理函数，x6为堆栈的指针，便于遗产函数设置返回值 */
	
	/* 加载出上一次异常退出的SP指针 */
	ldr	x12, [x6, #CTX_EL3STATE_OFFSET + CTX_RUNTIME_SP]

	/* ---------------------------------------------
	 * Switch back to SP_EL0 for the C runtime stack.
	 * ---------------------------------------------
	 */
	msr	spsel, #0
	mov	sp, x12

	/* -----------------------------------------------------
	 * 保存spsr_el3、elr_el3、scr_el3寄存器
	 * 在el3_exit时可以退出到进入遗产的位置
	 * -----------------------------------------------------
	 */
	mrs	x16, spsr_el3
	mrs	x17, elr_el3
	mrs	x18, scr_el3
	stp	x16, x17, [x6, #CTX_EL3STATE_OFFSET + CTX_SPSR_EL3]
	str	x18, [x6, #CTX_EL3STATE_OFFSET + CTX_SCR_EL3]

	/* 传递给异常处理函数的参数flag，标记之前是否在Secure Word */
	bfi	x7, x18, #0, #1

	/* -----------------------------------------------------
	 * 跳转到证实的异常处理函数
	 * -----------------------------------------------------
	 */
	bl	bl1_smc_handler

	/* -----------------------------------------------------
	 * 遗产调用返回，返回到调用位置
	 * -----------------------------------------------------
	 */
	b	el3_exit
```

BL1使用SP_EL0作为其C程序运行的堆栈，在进入异常时默认认为SPSEL=1，使用的是SP_EL3。这样，保存的线程在SP_EL3中。切换到SP_EL0执行程序，并通过SP_EL3来修改堆栈。

# BL2分析

## 功能概要

BL2主要负责对其他所有BL进行认证和加载，并执行EL31。

## 主过程

BL2入口位于`bl2/aarch64/bl2_entrypoint.S`中

```assembly
	.globl	bl2_entrypoint



func bl2_entrypoint
	mov	x20, x1 /* x1保存了内存布局信息 */

    /* 设置异常处理函数 */
	adr	x0, early_exceptions
	msr	vbar_el1, x0
	isb

	/* ---------------------------------------------
	 * 使能异常
	 * ---------------------------------------------
	 */
	msr	daifclr, #DAIF_ABT_BIT

	/* ---------------------------------------------
	 * 使能指令缓存，使能堆栈数据访问对齐检查
	 * ---------------------------------------------
	 */
	mov	x1, #(SCTLR_I_BIT | SCTLR_A_BIT | SCTLR_SA_BIT)
	mrs	x0, sctlr_el1
	orr	x0, x0, x1
	msr	sctlr_el1, x0
	isb

	/* ---------------------------------------------
	 * 失效内存
	 * ---------------------------------------------
	 */
	adr	x0, __RW_START__
	adr	x1, __RW_END__
	sub	x1, x1, x0
	bl	inv_dcache_range

	/* ---------------------------------------------
	 * BSS内存初始化
	 * ---------------------------------------------
	 */
	ldr	x0, =__BSS_START__
	ldr	x1, =__BSS_SIZE__
	bl	zeromem16

#if USE_COHERENT_MEM
	ldr	x0, =__COHERENT_RAM_START__
	ldr	x1, =__COHERENT_RAM_UNALIGNED_SIZE__
	bl	zeromem16
#endif

	/* --------------------------------------------
	 * 设置SP指针
	 * --------------------------------------------
	 */
	bl	plat_set_my_stack

	/* ---------------------------------------------
	 * 串口初始化，更新内存布局信息，并初始化页表使能mmu
	 * ---------------------------------------------
	 */
	mov	x0, x20
	bl	bl2_early_platform_setup
	bl	bl2_plat_arch_setup

	/* ---------------------------------------------
	 * 跳转到主函数（通过SMC执行下一级BL不会返回）
	 * ---------------------------------------------
	 */
	bl	bl2_main

	/* ---------------------------------------------
	 * 下面的代码不会执行
	 * ---------------------------------------------
	 */
	no_ret	plat_panic_handler

endfunc bl2_entrypoint
```

bl2\_main为bl2的主程序，位于bl2/bl2_main.c中

```c
void bl2_main(void)
{
	entry_point_info_t *next_bl_ep_info;
	/* 输出提示信息 */
	NOTICE("BL2: %s\n", version_string);
	NOTICE("BL2: %s\n", build_message);

	/* 初始化，这里开启FP/SIMD的访问权限 */
	bl2_arch_setup();

#if TRUSTED_BOARD_BOOT
	/*
	 * 初始化认证模块
	 * 初始化加密库（crypto_mod），加密库可以用于校验签名和哈希
	 * 初始化镜像解析模块（img_parser_mod），用于校验镜像完整性以及从镜像中提取内容
	 */
	auth_mod_init();
#endif /* TRUSTED_BOARD_BOOT */

	/* 此处不不只是加载BL2，此处会加载校验所有的BL */
	next_bl_ep_info = bl2_load_images();

#ifdef AARCH32
	/*
	 * For AArch32 state BL1 and BL2 share the MMU setup.
	 * Given that BL2 does not map BL1 regions, MMU needs
	 * to be disabled in order to go back to BL1.
	 */
	disable_mmu_icache_secure();
#endif /* AARCH32 */

	 /* 通过SMC系统调用运行下一级BL，下一级BL将跳转到EL3 */
	smc(BL1_SMC_RUN_IMAGE, (unsigned long)next_bl_ep_info, 0, 0, 0, 0, 0, 0);
}
```



## 中断服务

# BL31分析

## 功能概要

BL31主要提供了SMC服务框架，并于SMC服务的添加扩展。并执行下一级BL

## 主过程

BL31入口位于`bl31/aarch64/bl31_entrypoint.S`中

```assembly
func bl31_entrypoint
#if !RESET_TO_BL31
     /* 由前端bootloader引导进入，这时cpu会通过X0/X1传递信息
      * X0指向`bl31_params`结构体
      * X1指向特定架构相关的结构体
      */
	mov	x20, x0
	mov	x21, x1

     /* 因为前端bootloader已经初始化了一部分类容，这里只要初始化当前内核的运行环境 */
	el3_entrypoint_common					\
		_set_endian=0					\
		_warm_boot_mailbox=0				\
		_secondary_cold_boot=0				\
		_init_memory=0					\
		_init_c_runtime=1				\
		_exception_vectors=runtime_exceptions

     /* 恢复之前保存的参数 */
	mov	x0, x20
	mov	x1, x21
#else
     /* 由前端cpu复位进入，这时需要重新初始化，非主核cpu不需要初始化 */
	el3_entrypoint_common					\
		_set_endian=1					\
		_warm_boot_mailbox=!PROGRAMMABLE_RESET_ADDRESS	\
		_secondary_cold_boot=!COLD_BOOT_SINGLE_CPU	\
		_init_memory=1					\
		_init_c_runtime=1				\
		_exception_vectors=runtime_exceptions

     /* 前端没有参数传递进来，给指针赋值0 */
	mov	x0, 0
	mov	x1, 0
#endif /* RESET_TO_BL31 */

	bl	bl31_early_platform_setup/* 初始化cpu内部初始化，初始化执行环境 */
	bl	bl31_plat_arch_setup/* 初始化页表 */

	/* ---------------------------------------------
	 * Jump to main function.
	 * ---------------------------------------------
	 */
	bl	bl31_main
     /* 把BL31的data段同步到内存 */
	adr	x0, __DATA_START__
	adr	x1, __DATA_END__
	sub	x1, x1, x0
	bl	clean_dcache_range

    /* 把BL31的bss段同步到内存 */
	adr	x0, __BSS_START__
	adr	x1, __BSS_END__
	sub	x1, x1, x0
	bl	clean_dcache_range

    /* 退出异常，并执行下一个镜像 */
	b	el3_exit
endfunc bl31_entrypoint
```

BL31主程序位于`bl31/bl31_main.c`中

```c
void bl31_main(void)
{
	NOTICE("BL31: %s\n", version_string);
	NOTICE("BL31: %s\n", build_message);

	/* 系统相关的初始化 */
	bl31_platform_setup();
	bl31_lib_init();

	/* 初始化系统服务，并构造索引rt_svc_descs_indices */
	INFO("BL31: Initializing runtime services\n");
	runtime_svc_init();
	
  	/* 判对有无bl31，执行对应的初始化 */
	if (bl32_init) {
		INFO("BL31: Initializing BL32\n");
		(*bl32_init)();
	}
	
	 /* 初始化下一级BL的执行环境，以便异常返回时执行下一级BL */
	bl31_prepare_next_image_entry();

	 /* 从BL31退出前执行与特定平台相关的初始化工作 */
	bl31_plat_runtime_setup();
}
```



## 中断服务

bl31中断服务程序位于`bl31/aarch64/runtime_exceptions.S`中

```assembly
	.globl	runtime_exceptions

	/* ---------------------------------------------------------------------
	 * This macro handles Synchronous exceptions.
	 * Only SMC exceptions are supported.
	 * ---------------------------------------------------------------------
	 */
     /* 同步异常处理只处理smc相关的调用 */
	.macro	handle_sync_exception
	/* Enable the SError interrupt */
	msr	daifclr, #DAIF_ABT_BIT

	/* 保存x30下面要分析异常来源 */
	str	x30, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_LR]

#if ENABLE_RUNTIME_INSTRUMENTATION
	/*
	 * Read the timestamp value and store it in per-cpu data. The value
	 * will be extracted from per-cpu data by the C level SMC handler and
	 * saved to the PMF timestamp region.
	 */
	mrs	x30, cntpct_el0/* 时钟 */
	str	x29, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X29]
	mrs	x29, tpidr_el3 /* tpidr_el3 cpu标示 */
	str	x30, [x29, #CPU_DATA_PMF_TS0_OFFSET]
	ldr	x29, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X29]
#endif
	/* 获取exceptions code */
	mrs	x30, esr_el3
	ubfx	x30, x30, #ESR_EC_SHIFT, #ESR_EC_LENGTH

	/* Handle SMC exceptions separately from other synchronous exceptions */
	cmp	x30, #EC_AARCH32_SMC
	b.eq	smc_handler32 /* 32bit SMC调用 */

	cmp	x30, #EC_AARCH64_SMC
	b.eq	smc_handler64 /* 64bit SMC调用 */

	/* Other kinds of synchronous exceptions are not handled */
	no_ret	report_unhandled_exception
	.endm


	/* ---------------------------------------------------------------------
	 * This macro handles FIQ or IRQ interrupts i.e. EL3, S-EL1 and NS
	 * interrupts.
	 * ---------------------------------------------------------------------
	 */
	.macro	handle_interrupt_exception label
	/* Enable the SError interrupt */
	msr	daifclr, #DAIF_ABT_BIT

	/* 保存x0-x30 sp_el1 */
	str	x30, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_LR]
	bl	save_gp_registers/* 因为此处为函数调用会修改lr(x30),所以lr需要提前保存 */

	/* Save the EL3 system registers needed to return from this exception */
	mrs	x0, spsr_el3 /* 获取进入异常前的程序状态寄存器 */
	mrs	x1, elr_el3  /* 获取程序被中断的异常地址 */
	/* 保存进入异常的程序状态和异常地址 */
	stp	x0, x1, [sp, #CTX_EL3STATE_OFFSET + CTX_SPSR_EL3]

	/* Switch to the runtime stack i.e. SP_EL0 */
	ldr	x2, [sp, #CTX_EL3STATE_OFFSET + CTX_RUNTIME_SP]
	mov	x20, sp
	msr	spsel, #0
	mov	sp, x2 /* 切换到EL0的SP指针 */

	/*
	 * Find out whether this is a valid interrupt type.
	 * If the interrupt controller reports a spurious interrupt then return
	 * to where we came from.
	 */
	bl	plat_ic_get_pending_interrupt_type
	cmp	x0, #INTR_TYPE_INVAL
	b.eq	interrupt_exit_\label

	/*
	 * Get the registered handler for this interrupt type.
	 * A NULL return value could be 'cause of the following conditions:
	 *
	 * a. An interrupt of a type was routed correctly but a handler for its
	 *    type was not registered.
	 *
	 * b. An interrupt of a type was not routed correctly so a handler for
	 *    its type was not registered.
	 *
	 * c. An interrupt of a type was routed correctly to EL3, but was
	 *    deasserted before its pending state could be read. Another
	 *    interrupt of a different type pended at the same time and its
	 *    type was reported as pending instead. However, a handler for this
	 *    type was not registered.
	 *
	 * a. and b. can only happen due to a programming error. The
	 * occurrence of c. could be beyond the control of Trusted Firmware.
	 * It makes sense to return from this exception instead of reporting an
	 * error.
	 */
	bl	get_interrupt_type_handler
	cbz	x0, interrupt_exit_\label
	mov	x21, x0

	mov	x0, #INTR_ID_UNAVAILABLE

	/* Set the current security state in the 'flags' parameter */
	mrs	x2, scr_el3
	ubfx	x1, x2, #0, #1

	/* Restore the reference to the 'handle' i.e. SP_EL3 */
	mov	x2, x20

	/* x3 will point to a cookie (not used now) */
	mov	x3, xzr

	/* Call the interrupt type handler */
	blr	x21

interrupt_exit_\label:
	/* Return from exception, possibly in a different security state */
	b	el3_exit

	.endm


	.macro save_x18_to_x29_sp_el0
	stp	x18, x19, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X18]
	stp	x20, x21, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X20]
	stp	x22, x23, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X22]
	stp	x24, x25, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X24]
	stp	x26, x27, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X26]
	stp	x28, x29, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X28]
	mrs	x18, sp_el0
	str	x18, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_SP_EL0]
	.endm


vector_base runtime_exceptions

	/* ---------------------------------------------------------------------
	 * Current EL with SP_EL0 : 0x0 - 0x200
	 * ---------------------------------------------------------------------
	 */
vector_entry sync_exception_sp_el0
	/* We don't expect any synchronous exceptions from EL3 */
	no_ret	report_unhandled_exception
	check_vector_size sync_exception_sp_el0

vector_entry irq_sp_el0
	/*
	 * EL3 code is non-reentrant. Any asynchronous exception is a serious
	 * error. Loop infinitely.
	 */
	no_ret	report_unhandled_interrupt
	check_vector_size irq_sp_el0


vector_entry fiq_sp_el0
	no_ret	report_unhandled_interrupt
	check_vector_size fiq_sp_el0


vector_entry serror_sp_el0
	no_ret	report_unhandled_exception
	check_vector_size serror_sp_el0

	/* ---------------------------------------------------------------------
	 * Current EL with SP_ELx: 0x200 - 0x400
	 * ---------------------------------------------------------------------
	 */
vector_entry sync_exception_sp_elx
	/*
	 * This exception will trigger if anything went wrong during a previous
	 * exception entry or exit or while handling an earlier unexpected
	 * synchronous exception. There is a high probability that SP_EL3 is
	 * corrupted.
	 */
	no_ret	report_unhandled_exception
	check_vector_size sync_exception_sp_elx

vector_entry irq_sp_elx
	no_ret	report_unhandled_interrupt
	check_vector_size irq_sp_elx

vector_entry fiq_sp_elx
	no_ret	report_unhandled_interrupt
	check_vector_size fiq_sp_elx

vector_entry serror_sp_elx
	no_ret	report_unhandled_exception
	check_vector_size serror_sp_elx

	/* ---------------------------------------------------------------------
	 * Lower EL using AArch64 : 0x400 - 0x600
	 * ---------------------------------------------------------------------
	 */
vector_entry sync_exception_aarch64
	/*
	 * This exception vector will be the entry point for SMCs and traps
	 * that are unhandled at lower ELs most commonly. SP_EL3 should point
	 * to a valid cpu context where the general purpose and system register
	 * state can be saved.
	 */
	handle_sync_exception
	check_vector_size sync_exception_aarch64

vector_entry irq_aarch64
	handle_interrupt_exception irq_aarch64
	check_vector_size irq_aarch64

vector_entry fiq_aarch64
	handle_interrupt_exception fiq_aarch64
	check_vector_size fiq_aarch64

vector_entry serror_aarch64
	/*
	 * SError exceptions from lower ELs are not currently supported.
	 * Report their occurrence.
	 */
	no_ret	report_unhandled_exception
	check_vector_size serror_aarch64

	/* ---------------------------------------------------------------------
	 * Lower EL using AArch32 : 0x600 - 0x800
	 * ---------------------------------------------------------------------
	 */
vector_entry sync_exception_aarch32
	/*
	 * This exception vector will be the entry point for SMCs and traps
	 * that are unhandled at lower ELs most commonly. SP_EL3 should point
	 * to a valid cpu context where the general purpose and system register
	 * state can be saved.
	 */
	handle_sync_exception
	check_vector_size sync_exception_aarch32

vector_entry irq_aarch32
	handle_interrupt_exception irq_aarch32
	check_vector_size irq_aarch32

vector_entry fiq_aarch32
	handle_interrupt_exception fiq_aarch32
	check_vector_size fiq_aarch32

vector_entry serror_aarch32
	/*
	 * SError exceptions from lower ELs are not currently supported.
	 * Report their occurrence.
	 */
	no_ret	report_unhandled_exception
	check_vector_size serror_aarch32


	/* ---------------------------------------------------------------------
	 * The following code handles secure monitor calls.
	 * Depending upon the execution state from where the SMC has been
	 * invoked, it frees some general purpose registers to perform the
	 * remaining tasks. They involve finding the runtime service handler
	 * that is the target of the SMC & switching to runtime stacks (SP_EL0)
	 * before calling the handler.
	 *
	 * Note that x30 has been explicitly saved and can be used here
	 * ---------------------------------------------------------------------
	 */
func smc_handler
smc_handler32:
	/* Check whether aarch32 issued an SMC64 */
	tbnz	x0, #FUNCID_CC_SHIFT, smc_prohibited

	/*
	 * Since we're are coming from aarch32, x8-x18 need to be saved as per
	 * SMC32 calling convention. If a lower EL in aarch64 is making an
	 * SMC32 call then it must have saved x8-x17 already therein.
	 */
	stp	x8, x9, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X8]
	stp	x10, x11, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X10]
	stp	x12, x13, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X12]
	stp	x14, x15, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X14]
	stp	x16, x17, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X16]

	/* x4-x7, x18, sp_el0 are saved below */

smc_handler64:
	/*
	 * Populate the parameters for the SMC handler.
	 * We already have x0-x4 in place. x5 will point to a cookie (not used
	 * now). x6 will point to the context structure (SP_EL3) and x7 will
	 * contain flags we need to pass to the handler Hence save x5-x7.
	 *
	 * Note: x4 only needs to be preserved for AArch32 callers but we do it
	 *       for AArch64 callers as well for convenience
	 */
	stp	x4, x5, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X4]
	stp	x6, x7, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_X6]

	/* Save rest of the gpregs and sp_el0*/
	save_x18_to_x29_sp_el0

	mov	x5, xzr
	mov	x6, sp

	/* Get the unique owning entity number */
    /* X0用于确定调用的功能
     * b31    调用类型 1->Fast Call 0->Yielding Call
     * b30    标示32bit还是64bit调用
     * b29:24 服务类型
     *           0x0  ARM架构相关
     *           0x1  CPU服务
     *           0x2  SiP服务
     *           0x3  OEM服务
     *           0x4  标准安全服务
     *           0x5  标准Hypervisor服务
     *           0x6  厂商定制的Hypervisor服务
     *     0x07-0x2f  预留
     *     0x30-0x31  Trusted Application Calls
     *     0x32-0x3f  Trusted OS Calls
     * b23:16 在Fast Call时这些比特位必须为0，其他值保留未用
     *        ARMv7传统的Trusted OS在Fast Call时必须为1
     * b15:0  函数号
     */
	ubfx	x16, x0, #FUNCID_OEN_SHIFT, #FUNCID_OEN_WIDTH   /* 服务类型 */
	ubfx	x15, x0, #FUNCID_TYPE_SHIFT, #FUNCID_TYPE_WIDTH /* 调用类型 */
	orr	x16, x16, x15, lsl #FUNCID_OEN_WIDTH

    /* __RT_SVC_DESCS_START__为rt_svc_descs段的起始地址，此段存放服务描述符
     * RT_SVC_DESC_HANDLE为服务句柄在服务描述符结构体中的偏移量
     * 以下代码保存第一个复位句柄的地址到x11中
     */
	adr	x11, (__RT_SVC_DESCS_START__ + RT_SVC_DESC_HANDLE)

	/* Load descriptor index from array of indices */
    /* 因为描述符在内存中布局是乱的通过rt_svc_descs_indices数组来查找描述符在内存中的位置
     * rt_svc_descs_indices为rt_svc_descs的索引
     */
	adr	x14, rt_svc_descs_indices
	ldrb	w15, [x14, x16]

	/*
	 * Restore the saved C runtime stack value which will become the new
	 * SP_EL0 i.e. EL3 runtime stack. It was saved in the 'cpu_context'
	 * structure prior to the last ERET from EL3.
	 */
	ldr	x12, [x6, #CTX_EL3STATE_OFFSET + CTX_RUNTIME_SP]

	/*
	 * Any index greater than 127 is invalid. Check bit 7 for
	 * a valid index
	 */
	tbnz	w15, 7, smc_unknown/* 索引的值不应该大于127 */

	/* Switch to SP_EL0 */
	msr	spsel, #0

	/*
	 * Get the descriptor using the index
	 * x11 = (base + off), x15 = index
	 *
	 * handler = (base + off) + (index << log2(size))
	 */
	lsl	w10, w15, #RT_SVC_SIZE_LOG2/* 计算偏移量 */
	ldr	x15, [x11, w10, uxtw]/* 加载出对应服务的句柄 */

	/*
	 * Save the SPSR_EL3, ELR_EL3, & SCR_EL3 in case there is a world
	 * switch during SMC handling.
	 * TODO: Revisit if all system registers can be saved later.
	 */
	mrs	x16, spsr_el3
	mrs	x17, elr_el3
	mrs	x18, scr_el3
	stp	x16, x17, [x6, #CTX_EL3STATE_OFFSET + CTX_SPSR_EL3]
	str	x18, [x6, #CTX_EL3STATE_OFFSET + CTX_SCR_EL3]

	/* Copy SCR_EL3.NS bit to the flag to indicate caller's security */
	bfi	x7, x18, #0, #1/* x7用于传递给服务函数security state */

	mov	sp, x12

	/*
	 * Call the Secure Monitor Call handler and then drop directly into
	 * el3_exit() which will program any remaining architectural state
	 * prior to issuing the ERET to the desired lower EL.
	 */
#if DEBUG
	cbz	x15, rt_svc_fw_critical_error/* x15等于0说明有错误存在 */
#endif
	blr	x15 /* 调用对应的服务 */

	b	el3_exit/* 异常恢复 */

smc_unknown:
	/*
	 * Here we restore x4-x18 regardless of where we came from. AArch32
	 * callers will find the registers contents unchanged, but AArch64
	 * callers will find the registers modified (with stale earlier NS
	 * content). Either way, we aren't leaking any secure information
	 * through them.
	 */
	mov	w0, #SMC_UNK
	b	restore_gp_registers_callee_eret

smc_prohibited:
	ldr	x30, [sp, #CTX_GPREGS_OFFSET + CTX_GPREG_LR]
	mov	w0, #SMC_UNK
	eret

rt_svc_fw_critical_error:
	/* Switch to SP_ELx */
	msr	spsel, #1
	no_ret	report_unhandled_exception
endfunc smc_handler
```

## SMC服务框架

SMC调用通过W0标示当前调用的类型。W0根据如下方式标记调用类型

```
W0用于确定调用的功能
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
b23:16 在Fast Call时这些比特位必须为0，其他值保留未用
       ARMv7传统的Trusted OS在Fast Call时必须为1
b15:0  函数号
```

SMC服务通过结构体描述

```C
typedef struct rt_svc_desc {
  	/* start_oen-end_oen 对应W0 b29:24中的一个片段 */
	uint8_t start_oen;     /* 当前服务可以处理的SMC调用编号的起始编号 */
	uint8_t end_oen;       /* 当前服务可以处理的SMC调用编号的结束编号 */
	uint8_t call_type;     /* 调用的类型，对应W0 b30 */
	const char *name;      /* 名字 */
	rt_svc_init_t init;    /* 服务初始化函数 */
	rt_svc_handle_t handle;/* 服务句柄 */
} rt_svc_desc_t;
```

通过如下的宏声明一个SMC服务，服务被放在rt_svc_descs段中，并在链接脚本中指定rt_svc_descs的起始结束符号（\_\_RT\_SVC\_DESCS\_START\_\_/\_\_RT\_SVC\_DESCS\_END\_\_）

```c
#define DECLARE_RT_SVC(_name, _start, _end, _type, _setup, _smch) \
	static const rt_svc_desc_t __svc_desc_ ## _name \
		__section("rt_svc_descs") __used = { \
			.start_oen = _start, \
			.end_oen = _end, \
			.call_type = _type, \
			.name = #_name, \
			.init = _setup, \
			.handle = _smch }
```

SMC服务初始化

```c
void runtime_svc_init(void)
{
	int rc = 0, index, start_idx, end_idx;

	/* 异常检查 */
	assert((RT_SVC_DESCS_END >= RT_SVC_DESCS_START) &&
			(RT_SVC_DECS_NUM < MAX_RT_SVCS));

	/* 没有系统服务退出 */
	if (RT_SVC_DECS_NUM == 0)
		return;

	/* 初始化索引*/
	memset(rt_svc_descs_indices, -1, sizeof(rt_svc_descs_indices));
	
    /* 遍历所有的SMC服务 */
    rt_svc_descs = (rt_svc_desc_t *) RT_SVC_DESCS_START;
	for (index = 0; index < RT_SVC_DECS_NUM; index++) {
		rt_svc_desc_t *service = &rt_svc_descs[index];
		
      	/* 检查SMC服务是否合法 */
		rc = validate_rt_svc_desc(service);
		if (rc) {
			ERROR("Invalid runtime service descriptor %p\n",
				(void *) service);
			panic();
		}

		/*
		 * 调用初始话函数
		 */
		if (service->init) {
			rc = service->init();
			if (rc) {
				ERROR("Error initializing runtime service %s\n",
						service->name);
				continue;
			}
		}

		/*
		 * 构建索引
		 * 通过调用类型和服务类型查找服务在rt_svc_descs中的下标
		 */
		start_idx = get_unique_oen(rt_svc_descs[index].start_oen,
				service->call_type);
		assert(start_idx < MAX_RT_SVCS);
		end_idx = get_unique_oen(rt_svc_descs[index].end_oen,
				service->call_type);
		assert(end_idx < MAX_RT_SVCS);
		for (; start_idx <= end_idx; start_idx++)
			rt_svc_descs_indices[start_idx] = index;
	}
}
```





