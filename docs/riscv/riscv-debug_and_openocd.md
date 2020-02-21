# riscv调试与openocd

## jtag

在理解调试之前需要先理解jtag接口。

jtag内部有两组寄存器，一个IR，一组DR。其中IR用于选择要操作的DR，即DR的地址选择寄存器。为了操作这两组寄存器，jtag内部定义了一个状态机，如下

![jtag状态机](https://www.xjtag.com/wp-content/uploads/tap_state_machine.gif)


jtag最少由以下几个信号组成，复位信号是可选的

- TMS 用于状态机转换
- TDO 数据输出
- TDI 数据输入
- TCK 时钟型号

TDO/TDI是串联的，随TCK信号的变化串入串出

TMS的变化会改变jtag的状态，具体参考jtag状态机图

其中IR和DR的位宽都是和具体平台相关的

## riscv调试

### jtag接口

riscv通过在jtag中添加两个DR (dmi/dtmcs)。其中dmi实现了一个可读写的地址空间，dmi结构如下
```
 abits+33     34 33            2 1  0
+---------------+---------------+----+
|    address    |      data     | op |
+---------------+---------------+----+
```
abits是来自dtmcs，表示地址的宽度

写dmi时，op表示要执行的操作<br/>
op=0，表示空操作<br/>
op=1，从address读取数据到data<br/>
op=2，把data写入到address<br/>
op=3，保留<br/>

读dmi是，op表示上一个操作的状态<br/>
op=0，上一个操作成功<br/>
op=1，保留<br/>
op=2，上一个操作失败<br/>
op=3，上一个操作还在进行中<br/>

### 调试模块

调试模块是映射到dmi空间的一段内存，第一个调试模块映射到dmi最底部的内存，调试模块中有一个字段nextdm，从而可以在dmi空间放下多个调试模块。

#### 核心选择

riscv支持一次选择一个核心，也支持一次选择多个核心。

选择单个核心时，可以直接把核心的id写入到dmcontrol的hartsel字段。这个字段有20比特，所以一个调试模块最多支持$2^{20}$个核心。

选择多个核心时通过两个寄存器进行，一个hawindowsel，一个hawindow。选择逻辑如下

> 如果要选择的核心编号为n<br/>
> hawindowsel = n / 32<br/>
> hawindow |= 1 << (n % 32)<br/>

hawindowsel为15比特，hawindow为32比特。所以最多可选择的核心数为：$2^{15} * 32 = 2^{20}$

#### 复位

复位相关的位位于dmcontrol/dmstatus中。可以复位单个或多个核心
1. 选择要复位的核心
2. 置位haltreset
3. 等待allhavereset被置位

#### 暂停

暂停相关的位位于dmcontrol/dmstatus中。可以暂停单个或多个核心
1. 选择要暂停的核心
2. 置位haltreq
3. 等待allhalted被置位

#### 恢复运行

恢复运行相关的位位于dmcontrol/dmstatus中。可以恢复运行单个或多个核心
1. 选择要恢复运行的核心
2. 置位resumereq
3. 等待allresumeack被置位

#### 抽象命令

通过抽象命令可以实习寄存器读写/内存访问以及运行一段特定的代码。

为了实习抽象命令，调试模块上添加了一些寄存器，如下表
| 寄存器名        |描述					|
|:---------------|:-------------------------------------|
| abstractcs     | 抽象命令的控制和状态			|
| command        | 命令类型和一些相关细节的控制		|
| abstractauto   | 此寄存器用于提高访问速度，是可选的		|
| data0 - data11 | command的参数和返回值			|

command高8位为cmdtype，用于标识命令类型。当前有三种类型如下表
| cmdtype | 描述			|
|:--------|:--------------------|
| 0       | 寄存器访问命令	|
| 1       | 快速访问命令		|
| 2       | 内存访问命令		|

command的低24位与具体的命令类型有关，即与高8位的值相关

data0 -data11用于做command的参数，使用方式如下表

| 参数宽度 | arg0 / return value     | arg1                    | arg2                      |
|:--------|:------------------------|:------------------------|:--------------------------|
|  32     | data0                   | data1                   | data2                     |
|  64     | data0,data1             | data2,data3             | data4,data5               |
| 128     | data0,data1,data2,data3 | data4,data5,data6,data7 | data8,data9,data10,data11 |

##### 寄存器访问命令

当为寄存器访问命令时，command寄存器试图如下：
```
 31     24  23  22     20         19             18         17       16    15     0
+---------+----+---------+------------------+----------+----------+-------+--------+
| cmdtype |  0 | aarsize | aarpostincrement | postexec | transfer | write |  regno |
+---------+----+---------+------------------+----------+----------+-------+--------+
     8       1      3              1              1          1        1       16
```
- arrsize
	- 2 访问低32位
	- 3 访问低64位
	- 4 访问低128位
- aarpostincrement
	- 0 没有影响
	- 1 访问成功后regno自动加一，此字段是可选实现的
- postexec
	- 0 没有影响
	- 1 寄存器传输完成后执行一次Program Buffer，此字段是可选实习的
- transfer
	- 0 不读写寄存器
	- 1 执行读写寄存器
- write
	- 0 读取regno指定的寄存器到arg0
	- 1 把arg0的值写入到regno制定的寄存器中
- regno
	-  要访问的寄存器编号

由于riscv中有多种类型的寄存器，需要编码到线性空间才能通过regno访问，编码方式如下表
|编码范围          |描述					|
|-----------------|-------------------------------------|
| 0x0000 - 0x0fff | 状态控制寄存器，通过访问dpc获取pc的值	|
| 0x1000 - 0x101f | 通用寄存器				|
| 0x1020 - 0x103f | 浮点寄存器				|
| 0xc000 - 0xffff | 保留给非标准扩展			|

##### 快速访问

快速访问命令，会暂停选定的核心，然后执行Program Buffer。这时command寄存器视图如下
```
 34     24 23                                0
+---------+-----------------------------------+
| cmdtype |                  0                |
+---------+-----------------------------------+
     8                      24
```

##### 内存访问命令

当为内存访问命令时，command视图如下
```
 34     24      23      22     20        19          18  17   16    15             14 13       0
+---------+------------+---------+------------------+------+-------+-----------------+----------+
| cmdtype | aamvirtual | aamsize | aampostincrement |   0  | write | target-specific |     0    |
+---------+------------+---------+------------------+------+-------+-----------------+----------+
     8          1           3             1             2      1            2             14
```
- aamvirtual
	- 0 访问物理地址
	- 1 访问虚拟地址
- aamsize
	- 0 访问的内存宽度8比特
	- 1 访问的内存宽度16比特
	- 2 访问的内存宽度32比特
	- 3 访问的内存宽度64比特
	- 4 访问的内存宽度128比特
- aampostincrement
	- 0 没影响
	- 1 访问内存后把arg1增加，具体加的值与aamsize有关，此位域可选实现
- write
	- 0 把arg1制定的内存读取到arg0
	- 1 把arg0的值写入到arg1制定的内存
- target-specific
	- 为特定平台保留

#### Program Buffer

在抽象命令中有提到Program Buffer，它可以使核心运行一小段代码。在调试模块的内存中有一小段空间存放程序。这段空间的大小由abstractcs中的字段progbufsize指定。

通过dmi可以写入一小段程序。如果程序比较复杂，可以添加调用指令，把具体的程序写入到内存中。在程序最后需要添加一个ebreak，使程序返回调试状态。

#### 系统总线访问

在调试模块中还提供了一组寄存器用于访问系统总线。
| 寄存器名                 | 描述                     |
|:------------------------|:------------------------|
| sbcs                    | 系统总线访问的控制状态寄存器 |
| sbaddress0 - sbaddress3 | 访问的地址                |
| sbdata0 - sbdata3       | 访问的数据                |

sbcs试图如下
```
 31       29 28 23      22          21         20        19      17        16              15        14     12 11      5       4             3            2            1            0
+-----------+-----+-------------+--------+--------------+----------+-----------------+--------------+---------+---------+-------------+------------+------------+------------+-----------+
| sbversion |  0  | sbbusyerror | sbbusy | sbreadonaddr | sbaccess | sbautoincrement | sbreadondata | sberror | sbasize | sbaccess128 | sbaccess64 | sbaccess32 | sbaccess16 | sbaccess8 |
+-----------+-----+-------------+--------+--------------+----------+-----------------+--------------+---------+---------+-------------+------------+------------+------------+-----------+
      3        6         1           1           1            3             1               1            3         7           1             1            1            1            1
```
- sbversion 系统总线访问的版本
- sbbusyerror 为1时表示，系统总线访问发送错误
- sbbusy 为1时表示总线忙
- sbreadonaddr 为1时表示，每次写sbaddress0，都会触发系统总线的访问
- sbaccess
	- 0 访问总线的地址宽度8比特
	- 1 访问总线的地址宽度16比特
	- 2 访问总线的地址宽度32比特
	- 3 访问总线的地址宽度64比特
	- 4访问总线的地址宽度128比特
- sbautoincrement 为1时表示，当每次系统总线访问后sbaddress自动增加，增加的大小与sbaccess相关
- sbreadondata 为1时表示，每一次读取sbdata0，都会触发系统总线的访问
- sberror
	- 0 没有错误
	- 1 超时
	- 2 访问了一个错误的地址
	- 3 非对齐访问错误
	- 4 访问了一个不支持的内存宽度
- sbasize 总线宽度，0表示不支持总线访问
- sbaccess128 为1表示，支持128比特宽度访问
- sbaccess64 为1表示，支持64比特宽度访问
- sbaccess32 为1表示，支持32比特宽度访问
- sbaccess16 为1表示，支持16比特宽度访问
- sbaccess8 为1表示，支持8比特宽度访问

## openocd

openocd是一个针对嵌入性系统的片上调试软件。它通过[gdb远程调试协议](https://sourceware.org/gdb/current/onlinedocs/gdb/Remote-Protocol.html)与gdb通信。

### server

源码位于:
> src/server/server.h<br/>
> src/server/server.c<br/>

openocd实习了一个抽象的server类，可以处理文件、管道和TCP链接，结构体描述如下：
```c
struct connection {
	int fd;
	int fd_out;	/* When using pipes we're writing to a different fd */
	struct sockaddr_in sin;
	struct command_context *cmd_ctx;
	struct service *service;
	int input_pending;
	void *priv;
	struct connection *next;
};

typedef int (*new_connection_handler_t)(struct connection *connection);
typedef int (*input_handler_t)(struct connection *connection);
typedef int (*connection_closed_handler_t)(struct connection *connection);

struct service {
	char *name;
	enum connection_type type;
	char *port;
	unsigned short portnumber;
	int fd;
	struct sockaddr_in sin;
	int max_connections;
	struct connection *connections;
	new_connection_handler_t new_connection;
	input_handler_t input;
	connection_closed_handler_t connection_closed;
	void *priv;
	struct service *next;
};
```

要实现一个服务，关键要实习三个方法：
> new_connection，用于处理新的链接<br/>
> input，用于处理客户端的输入<br/>
> connection_closed，用于处理链接关闭，释放资源<br/>

所有的服务构成一个链表，由系统提供的接口管理：
```c
/*
 * 添加一个服务
 * name 服务名
 * port 服务的端口
 *      port为数字时监听tcp
 *      port为"pipe"时监听标准输入
 *      其他情况port为文件名从，文件中获取输入
 * max_connections 最多的链接数
 * new_connection_handler 用于处理新的连接的方法
 * in_handler 用于处理客户端请求的方法
 * close_handler 用于处理链接关闭释放资源的方法
 */
int add_service(char *name, const char *port,
		int max_connections, new_connection_handler_t new_connection_handler,
		input_handler_t in_handler, connection_closed_handler_t close_handler,
		void *priv);

/* 移除一个服务 */
int remove_service(const char *name, const char *port);

/* 初始化服务 */
int server_init(struct command_context *cmd_ctx);

/* 此方法用于消息轮询 */
int server_loop(struct command_context *command_context);

/* 服务退出 */
int server_quit(void);
```

### gdb server

源码位于：
> src/server/gdb_server.h<br/>
> src/server/gdb_server.c<br/>

gdb server是openocd最主要的服务，它实现了绝大部分的gdb远程调试协议

它负责解析远程调试协议，然后调用target/rtos的接口执行具体的操作，然后格式化为远程调试协议的响应

### target

源码位于：
> src/target/target.h<br/>
> src/target/target_type.h<br/>
> src/target/target.c<br/>

target是调试的主要接口，它实习了一些与具体平台无关的操作，与平台相关操作由target_type实现。

#### event事件回调接口

target实现了一个事件回调接口，其他模块可以向target注册事件响应代码，在事件发生时被执行。target定义的事件通过`enum target_event`描述

```c
enum target_event {

	/* allow GDB to do stuff before others handle the halted event,
	 * this is in lieu of defining ordering of invocation of events,
	 * which would be more complicated
	 *
	 * Telling GDB to halt does not mean that the target stopped running,
	 * simply that we're dropping out of GDB's waiting for step or continue.
	 *
	 * This can be useful when e.g. detecting power dropout.
	 */
	TARGET_EVENT_GDB_HALT,
	TARGET_EVENT_HALTED,		/* target entered debug state from normal execution or reset */
	TARGET_EVENT_RESUMED,		/* target resumed to normal execution */
	TARGET_EVENT_RESUME_START,
	TARGET_EVENT_RESUME_END,

	TARGET_EVENT_GDB_START, /* debugger started execution (step/run) */
	TARGET_EVENT_GDB_END, /* debugger stopped execution (step/run) */

	TARGET_EVENT_RESET_START,
	TARGET_EVENT_RESET_ASSERT_PRE,
	TARGET_EVENT_RESET_ASSERT,	/* C code uses this instead of SRST */
	TARGET_EVENT_RESET_ASSERT_POST,
	TARGET_EVENT_RESET_DEASSERT_PRE,
	TARGET_EVENT_RESET_DEASSERT_POST,
	TARGET_EVENT_RESET_INIT,
	TARGET_EVENT_RESET_END,

	TARGET_EVENT_DEBUG_HALTED,	/* target entered debug state, but was executing on behalf of the debugger */
	TARGET_EVENT_DEBUG_RESUMED, /* target resumed to execute on behalf of the debugger */

	TARGET_EVENT_EXAMINE_START,
	TARGET_EVENT_EXAMINE_END,

	TARGET_EVENT_GDB_ATTACH,
	TARGET_EVENT_GDB_DETACH,

	TARGET_EVENT_GDB_FLASH_ERASE_START,
	TARGET_EVENT_GDB_FLASH_ERASE_END,
	TARGET_EVENT_GDB_FLASH_WRITE_START,
	TARGET_EVENT_GDB_FLASH_WRITE_END,

	TARGET_EVENT_TRACE_CONFIG,
};
```

事件发生时需要执行的方法通过`struct target_event_callback`描述
```c
struct target_event_callback {
	int (*callback)(struct target *target, enum target_event event, void *priv);
	void *priv;
	struct target_event_callback *next;
};
```

这些回调通过next构成链表保存在静态变量`target_event_callbacks`中，此变量位于`src/target/target.c`文件中

target还添加了一个tcl的接口，可以注册一些回调方法，在事件发生时被执行。通过`struct target_event_action`描述：
```c
struct target_event_action {
	enum target_event event;
	struct Jim_Interp *interp;
	struct Jim_Obj *body;
	struct target_event_action *next;
};
```
这些回调方法通过tcl接口注册到target->event_action

其他模块可以通过，以下函数来注册事件的回调
```c
int target_register_event_callback(
		int (*callback)(struct target *target,
		enum target_event event, void *priv),
		void *priv);
int target_unregister_event_callback(
		int (*callback)(struct target *target,
		enum target_event event, void *priv),
		void *priv);
```

其他模块可以通过调用以下方法执行事件回调
```c
int target_call_event_callbacks(struct target *target, enum target_event event);
```

#### reset事件回调接口

target还维护了一个复位的回调方法。根据复位后是否运行和初始化被分为3种模式，通过`enum target_reset_mode`描述
```c
enum target_reset_mode {
	RESET_UNKNOWN = 0,
	RESET_RUN = 1,		/* reset and let target run */
	RESET_HALT = 2,		/* reset and halt target out of reset */
	RESET_INIT = 3,		/* reset and halt target out of reset, then run init script */
};
```

复位时需要执行的方法通过`struct target_reset_callback`
```c
struct target_reset_callback {
	struct list_head list;
	void *priv;
	int (*callback)(struct target *target, enum target_reset_mode reset_mode, void *priv);
};
```
其他模块可以通过，以下函数来注册服务的回调
```c
int target_register_reset_callback(
		int (*callback)(struct target *target,
		enum target_reset_mode reset_mode, void *priv),
		void *priv);
int target_unregister_reset_callback(
		int (*callback)(struct target *target,
		enum target_reset_mode reset_mode, void *priv),
		void *priv);
```
其他模块可以通过调用以下方法执行事件回调
```c
int target_call_reset_callbacks(struct target *target, enum target_reset_mode reset_mode);
```

#### trace事件回调接口

trace事件用来监控一段内存的变化。需要执行的回调方法通过`struct target_trace_callback`描述
```
struct target_trace_callback {
	struct list_head list;
	void *priv;
	int (*callback)(struct target *target, size_t len, uint8_t *data, void *priv);
};
```
其他模块可以通过，以下函数来注册服务的回调
```c
int target_register_trace_callback(
		int (*callback)(struct target *target,
		size_t len, uint8_t *data, void *priv),
		void *priv);
int target_unregister_trace_callback(
		int (*callback)(struct target *target,
		size_t len, uint8_t *data, void *priv),
		void *priv);
```
其他模块可以通过调用以下方法执行事件回调
```c
int target_call_trace_callbacks(struct target *target, size_t len, uint8_t *data);
```

#### timer事件回调接口

timer用来定时执行。定时器分为是否周期执行，通过`enum target_timer_type`描述
```c
enum target_timer_type {
	TARGET_TIMER_TYPE_ONESHOT,
	TARGET_TIMER_TYPE_PERIODIC
};
```

需要执行的回调方法通过`struct target_trace_callback`描述
```c
struct target_timer_callback {
	int (*callback)(void *priv);
	unsigned int time_ms;
	enum target_timer_type type;
	bool removed;
	struct timeval when;
	void *priv;
	struct target_timer_callback *next;
};
```

其他模块可以通过，以下函数来注册服务的回调
```c
int target_register_timer_callback(int (*callback)(void *priv),
		unsigned int time_ms, enum target_timer_type type, void *priv);
int target_unregister_timer_callback(int (*callback)(void *priv), void *priv);
```

其他模块可以通过调用以下方法执行事件回调
```c
int target_call_timer_callbacks(void);
```

#### working_area

working_area用来维护一段调试目标的内存。这段内存可以是物理内存，也可以是虚拟地址的内存。与target中的一下字段相关
```c
struct target {
	// ......
	target_addr_t working_area;				/* working area (initialised RAM). Evaluated
										 * upon first allocation from virtual/physical address. */
	bool working_area_virt_spec;		/* virtual address specified? */
	target_addr_t working_area_virt;			/* virtual address */
	bool working_area_phys_spec;		/* physical address specified? */
	target_addr_t working_area_phys;			/* physical address */
	uint32_t working_area_size;			/* size in bytes */
	uint32_t backup_working_area;		/* whether the content of the working area has to be preserved */
	struct working_area *working_areas;/* list of allocated working areas */
	// ......
}
```

分配的内存通过`struct working_area`描述
```c
struct working_area {
	target_addr_t address;
	uint32_t size;
	bool free;
	uint8_t *backup;
	struct working_area **user;
	struct working_area *next;
};
```

主要接口如下
```c
/* DANGER!!!!!
 *
 * if "area" passed in to target_alloc_working_area() points to a memory
 * location that goes out of scope (e.g. a pointer on the stack), then
 * the caller of target_alloc_working_area() is responsible for invoking
 * target_free_working_area() before "area" goes out of scope.
 *
 * target_free_all_working_areas() will NULL out the "area" pointer
 * upon resuming or resetting the CPU.
 *
 */
int target_alloc_working_area(struct target *target,
		uint32_t size, struct working_area **area);
/* Same as target_alloc_working_area, except that no error is logged
 * when ERROR_TARGET_RESOURCE_NOT_AVAILABLE is returned.
 *
 * This allows the calling code to *try* to allocate target memory
 * and have a fallback to another behaviour(slower?).
 */
int target_alloc_working_area_try(struct target *target,
		uint32_t size, struct working_area **area);
int target_free_working_area(struct target *target, struct working_area *area);
void target_free_all_working_areas(struct target *target);
uint32_t target_get_working_area_avail(struct target *target);
```

#### target_type

target提供了大量的封装接口，用来处理target_type。

target_type提供了大量的接口方法，用于执行具体的调试操作，接口定义如下
```c
/**
 * This holds methods shared between all instances of a given target
 * type.  For example, all Cortex-M3 targets on a scan chain share
 * the same method table.
 */
struct target_type {
	/**
	 * Name of this type of target.  Do @b not access this
	 * field directly, use target_type_name() instead.
	 */
	const char *name;
	const char *deprecated_name;

	/* poll current target status */
	int (*poll)(struct target *target);
	/* Invoked only from target_arch_state().
	 * Issue USER() w/architecture specific status.  */
	int (*arch_state)(struct target *target);

	/* target request support */
	int (*target_request_data)(struct target *target, uint32_t size, uint8_t *buffer);

	/* halt will log a warning, but return ERROR_OK if the target is already halted. */
	int (*halt)(struct target *target);
	/* See target.c target_resume() for documentation. */
	int (*resume)(struct target *target, int current, target_addr_t address,
			int handle_breakpoints, int debug_execution);
	int (*step)(struct target *target, int current, target_addr_t address,
			int handle_breakpoints);
	/* target reset control. assert reset can be invoked when OpenOCD and
	 * the target is out of sync.
	 *
	 * A typical example is that the target was power cycled while OpenOCD
	 * thought the target was halted or running.
	 *
	 * assert_reset() can therefore make no assumptions whatsoever about the
	 * state of the target
	 *
	 * Before assert_reset() for the target is invoked, a TRST/tms and
	 * chain validation is executed. TRST should not be asserted
	 * during target assert unless there is no way around it due to
	 * the way reset's are configured.
	 *
	 */
	int (*assert_reset)(struct target *target);
	/**
	 * The implementation is responsible for polling the
	 * target such that target->state reflects the
	 * state correctly.
	 *
	 * Otherwise the following would fail, as there will not
	 * be any "poll" invoked inbetween the "reset run" and
	 * "halt".
	 *
	 * reset run; halt
	 */
	int (*deassert_reset)(struct target *target);
	int (*soft_reset_halt)(struct target *target);

	/**
	 * Target architecture for GDB.
	 *
	 * The string returned by this function will not be automatically freed;
	 * if dynamic allocation is used for this value, it must be managed by
	 * the target, ideally by caching the result for subsequent calls.
	 */
	const char *(*get_gdb_arch)(struct target *target);

	/**
	 * Target register access for GDB.  Do @b not call this function
	 * directly, use target_get_gdb_reg_list() instead.
	 *
	 * Danger! this function will succeed even if the target is running
	 * and return a register list with dummy values.
	 *
	 * The reason is that GDB connection will fail without a valid register
	 * list, however it is after GDB is connected that monitor commands can
	 * be run to properly initialize the target
	 */
	int (*get_gdb_reg_list)(struct target *target, struct reg **reg_list[],
			int *reg_list_size, enum target_register_class reg_class);

	/**
	 * Same as get_gdb_reg_list, but doesn't read the register values.
	 * */
	int (*get_gdb_reg_list_noread)(struct target *target,
			struct reg **reg_list[], int *reg_list_size,
			enum target_register_class reg_class);

	/* target memory access
	* size: 1 = byte (8bit), 2 = half-word (16bit), 4 = word (32bit)
	* count: number of items of <size>
	*/

	/**
	 * Target memory read callback.  Do @b not call this function
	 * directly, use target_read_memory() instead.
	 */
	int (*read_memory)(struct target *target, target_addr_t address,
			uint32_t size, uint32_t count, uint8_t *buffer);
	/**
	 * Target memory write callback.  Do @b not call this function
	 * directly, use target_write_memory() instead.
	 */
	int (*write_memory)(struct target *target, target_addr_t address,
			uint32_t size, uint32_t count, const uint8_t *buffer);

	/* Default implementation will do some fancy alignment to improve performance, target can override */
	int (*read_buffer)(struct target *target, target_addr_t address,
			uint32_t size, uint8_t *buffer);

	/* Default implementation will do some fancy alignment to improve performance, target can override */
	int (*write_buffer)(struct target *target, target_addr_t address,
			uint32_t size, const uint8_t *buffer);

	int (*checksum_memory)(struct target *target, target_addr_t address,
			uint32_t count, uint32_t *checksum);
	int (*blank_check_memory)(struct target *target,
			struct target_memory_check_block *blocks, int num_blocks,
			uint8_t erased_value);

	/*
	 * target break-/watchpoint control
	 * rw: 0 = write, 1 = read, 2 = access
	 *
	 * Target must be halted while this is invoked as this
	 * will actually set up breakpoints on target.
	 *
	 * The breakpoint hardware will be set up upon adding the
	 * first breakpoint.
	 *
	 * Upon GDB connection all breakpoints/watchpoints are cleared.
	 */
	int (*add_breakpoint)(struct target *target, struct breakpoint *breakpoint);
	int (*add_context_breakpoint)(struct target *target, struct breakpoint *breakpoint);
	int (*add_hybrid_breakpoint)(struct target *target, struct breakpoint *breakpoint);

	/* remove breakpoint. hw will only be updated if the target
	 * is currently halted.
	 * However, this method can be invoked on unresponsive targets.
	 */
	int (*remove_breakpoint)(struct target *target, struct breakpoint *breakpoint);

	/* add watchpoint ... see add_breakpoint() comment above. */
	int (*add_watchpoint)(struct target *target, struct watchpoint *watchpoint);

	/* remove watchpoint. hw will only be updated if the target
	 * is currently halted.
	 * However, this method can be invoked on unresponsive targets.
	 */
	int (*remove_watchpoint)(struct target *target, struct watchpoint *watchpoint);

	/* Find out just hit watchpoint. After the target hits a watchpoint, the
	 * information could assist gdb to locate where the modified/accessed memory is.
	 */
	int (*hit_watchpoint)(struct target *target, struct watchpoint **hit_watchpoint);

	/**
	 * Target algorithm support.  Do @b not call this method directly,
	 * use target_run_algorithm() instead.
	 */
	int (*run_algorithm)(struct target *target, int num_mem_params,
			struct mem_param *mem_params, int num_reg_params,
			struct reg_param *reg_param, target_addr_t entry_point,
			target_addr_t exit_point, int timeout_ms, void *arch_info);
	int (*start_algorithm)(struct target *target, int num_mem_params,
			struct mem_param *mem_params, int num_reg_params,
			struct reg_param *reg_param, target_addr_t entry_point,
			target_addr_t exit_point, void *arch_info);
	int (*wait_algorithm)(struct target *target, int num_mem_params,
			struct mem_param *mem_params, int num_reg_params,
			struct reg_param *reg_param, target_addr_t exit_point,
			int timeout_ms, void *arch_info);

	const struct command_registration *commands;

	/* called when target is created */
	int (*target_create)(struct target *target, Jim_Interp *interp);

	/* called for various config parameters */
	/* returns JIM_CONTINUE - if option not understood */
	/* otherwise: JIM_OK, or JIM_ERR, */
	int (*target_jim_configure)(struct target *target, Jim_GetOptInfo *goi);

	/* target commands specifically handled by the target */
	/* returns JIM_OK, or JIM_ERR, or JIM_CONTINUE - if option not understood */
	int (*target_jim_commands)(struct target *target, Jim_GetOptInfo *goi);

	/**
	 * This method is used to perform target setup that requires
	 * JTAG access.
	 *
	 * This may be called multiple times.  It is called after the
	 * scan chain is initially validated, or later after the target
	 * is enabled by a JRC.  It may also be called during some
	 * parts of the reset sequence.
	 *
	 * For one-time initialization tasks, use target_was_examined()
	 * and target_set_examined().  For example, probe the hardware
	 * before setting up chip-specific state, and then set that
	 * flag so you don't do that again.
	 */
	int (*examine)(struct target *target);

	/* Set up structures for target.
	 *
	 * It is illegal to talk to the target at this stage as this fn is invoked
	 * before the JTAG chain has been examined/verified
	 * */
	int (*init_target)(struct command_context *cmd_ctx, struct target *target);

	/**
	 * Free all the resources allocated by the target.
	 *
	 * @param target The target to deinit
	 */
	void (*deinit_target)(struct target *target);

	/* translate from virtual to physical address. Default implementation is successful
	 * no-op(i.e. virtual==physical).
	 */
	int (*virt2phys)(struct target *target, target_addr_t address, target_addr_t *physical);

	/* read directly from physical memory. caches are bypassed and untouched.
	 *
	 * If the target does not support disabling caches, leaving them untouched,
	 * then minimally the actual physical memory location will be read even
	 * if cache states are unchanged, flushed, etc.
	 *
	 * Default implementation is to call read_memory.
	 */
	int (*read_phys_memory)(struct target *target, target_addr_t phys_address,
			uint32_t size, uint32_t count, uint8_t *buffer);

	/*
	 * same as read_phys_memory, except that it writes...
	 */
	int (*write_phys_memory)(struct target *target, target_addr_t phys_address,
			uint32_t size, uint32_t count, const uint8_t *buffer);

	int (*mmu)(struct target *target, int *enabled);

	/* after reset is complete, the target can check if things are properly set up.
	 *
	 * This can be used to check if e.g. DCC memory writes have been enabled for
	 * arm7/9 targets, which they really should except in the most contrived
	 * circumstances.
	 */
	int (*check_reset)(struct target *target);

	/* get GDB file-I/O parameters from target
	 */
	int (*get_gdb_fileio_info)(struct target *target, struct gdb_fileio_info *fileio_info);

	/* pass GDB file-I/O response to target
	 */
	int (*gdb_fileio_end)(struct target *target, int retcode, int fileio_errno, bool ctrl_c);

	/* do target profiling
	 */
	int (*profiling)(struct target *target, uint32_t *samples,
			uint32_t max_num_samples, uint32_t *num_samples, uint32_t seconds);

	/* Return the number of address bits this target supports. This will
	 * typically be 32 for 32-bit targets, and 64 for 64-bit targets. If not
	 * implemented, it's assumed to be 32. */
	unsigned (*address_bits)(struct target *target);

	/* Return the number of system bus data bits this target supports. This
	 * will typically be 32 for 32-bit targets, and 64 for 64-bit targets. If
	 * not implemented, it's assumed to be 32. */
	unsigned (*data_bits)(struct target *target);
};
```

主要提供了如下接口
- 初始化
- 复位
- 寄存器读写
- 内存读写
- 物理内存虚拟内存地址处理
- 单步
- 运行
- 执行一小段程序
- breakpoint
- watchpoint
- 等等

### rtos

源代码位于：
> src/rtos/rtos.h<br/>
> src/rtos/rtos.c<br/>

rtos实现了gdb远程调试协议相关的部分，主要入口为`gdb_thread_packet`，位于`src/rtos/rtos.c`

rtos与具体调试目标相关的部分通过`struct rtos_type`描述
```c
struct rtos_type {
	const char *name;
	bool (*detect_rtos)(struct target *target);
	int (*create)(struct target *target);
	int (*smp_init)(struct target *target);
	int (*update_threads)(struct rtos *rtos);
	/** Return a list of general registers, with their values filled out. */
	int (*get_thread_reg_list)(struct rtos *rtos, int64_t thread_id,
			struct rtos_reg **reg_list, int *num_regs);
	int (*get_thread_reg)(struct rtos *rtos, int64_t thread_id,
			uint32_t reg_num, struct rtos_reg *reg);
	int (*get_symbol_list_to_lookup)(symbol_table_elem_t *symbol_list[]);
	int (*clean)(struct target *target);
	char * (*ps_command)(struct target *target);
	int (*set_reg)(struct rtos *rtos, uint32_t reg_num, uint8_t *reg_value);
	/**
	 * Possibly work around an annoying gdb behaviour: when the current thread
	 * is changed in gdb, it assumes that the target can follow and also make
	 * the thread current. This is an assumption that cannot hold for a real
	 * target running a multi-threading OS. If an RTOS can do this, override
	 * needs_fake_step(). */
	bool (*needs_fake_step)(struct target *target, int64_t thread_id);
};
```

### jtag

#### jtag状态机

代码实习位于：
> src/jtag/interface.h<br/>
> src/jtag/interface.c<br/>

此部分代码是平台无关的，主要用于状态机的维护和切换。注意观察jtag的状态机，有的状态机有一个指向自己的箭头，这种状态被称为稳定态，这种状态下可以执行一些操作，状态机切换也只需要考虑这种情况。主要通过一个二维数组来记录怎么切换状态机（TMS需要发送怎么一段数据）。数组中记录这样的数据，如下
```c
struct tms_sequences {
	uint8_t bits;
	uint8_t bit_count;
};
```
并通过一些宏来快速定义一个如上的结构
```c
#define HEX__(n) 0x##n##LU

#define B8__(x)	\
	((((x) & 0x0000000FLU) ? (1 << 0) : 0) \
	+(((x) & 0x000000F0LU) ? (1 << 1) : 0) \
	+(((x) & 0x00000F00LU) ? (1 << 2) : 0) \
	+(((x) & 0x0000F000LU) ? (1 << 3) : 0) \
	+(((x) & 0x000F0000LU) ? (1 << 4) : 0) \
	+(((x) & 0x00F00000LU) ? (1 << 5) : 0) \
	+(((x) & 0x0F000000LU) ? (1 << 6) : 0) \
	+(((x) & 0xF0000000LU) ? (1 << 7) : 0))

#define B8(bits, count) {((uint8_t)B8__(HEX__(bits))), (count)}
```

数组定义如下
```c
static const struct tms_sequences old_tms_seqs[6][6] = {	/* [from_state_ndx][to_state_ndx] */
	/* value clocked to TMS to move from one of six stable states to another.
	 * N.B. OOCD clocks TMS from LSB first, so read these right-to-left.
	 * N.B. Reset only needs to be 0b11111, but in JLink an even byte of 1's is more stable.
	 * These extra ones cause no TAP state problem, because we go into reset and stay in reset.
	 */

/* to state: */
/*	RESET		 IDLE			DRSHIFT			DRPAUSE			IRSHIFT			IRPAUSE		*/	/* from state: */
{B8(1111111, 7), B8(0000000, 7), B8(0010111, 7), B8(0001010, 7), B8(0011011, 7), B8(0010110, 7)},/* RESET */
{B8(1111111, 7), B8(0000000, 7), B8(0100101, 7), B8(0000101, 7), B8(0101011, 7), B8(0001011, 7)},/* IDLE */
{B8(1111111, 7), B8(0110001, 7), B8(0000000, 7), B8(0000001, 7), B8(0001111, 7), B8(0101111, 7)},/* DRSHIFT */
{B8(1111111, 7), B8(0110000, 7), B8(0100000, 7), B8(0010111, 7), B8(0011110, 7), B8(0101111, 7)},/* DRPAUSE */
{B8(1111111, 7), B8(0110001, 7), B8(0000111, 7), B8(0010111, 7), B8(0000000, 7), B8(0000001, 7)},/* IRSHIFT */
{B8(1111111, 7), B8(0110000, 7), B8(0011100, 7), B8(0010111, 7), B8(0011110, 7), B8(0101111, 7)},/* IRPAUSE */
};

static const struct tms_sequences short_tms_seqs[6][6] = { /* [from_state_ndx][to_state_ndx] */
	/* this is the table submitted by Jeff Williams on 3/30/2009 with this comment:

	OK, I added Peter's version of the state table, and it works OK for
	me on MC1322x. I've recreated the jlink portion of patch with this
	new state table. His changes to my state table are pretty minor in
	terms of total transitions, but Peter feels that his version fixes
	some long-standing problems.
	Jeff

	I added the bit count into the table, reduced RESET column to 7 bits from 8.
	Dick

	state specific comments:
	------------------------
	*->RESET		tried the 5 bit reset and it gave me problems, 7 bits seems to
					work better on ARM9 with ft2232 driver.  (Dick)

	RESET->DRSHIFT add 1 extra clock cycles in the RESET state before advancing.
					needed on ARM9 with ft2232 driver.  (Dick)
					(For a total of *THREE* extra clocks in RESET; NOP.)

	RESET->IRSHIFT add 1 extra clock cycles in the RESET state before advancing.
					needed on ARM9 with ft2232 driver.  (Dick)
					(For a total of *TWO* extra clocks in RESET; NOP.)

	RESET->*		always adds one or more clocks in the target state,
					which should be NOPS; except shift states which (as
					noted above) add those clocks in RESET.

	The X-to-X transitions always add clocks; from *SHIFT, they go
	via IDLE and thus *DO HAVE SIDE EFFECTS* (capture and update).
*/

/* to state: */
/*	RESET		IDLE			DRSHIFT			DRPAUSE			IRSHIFT			IRPAUSE */ /* from state: */
{B8(1111111, 7), B8(0000000, 7), B8(0010111, 7), B8(0001010, 7), B8(0011011, 7), B8(0010110, 7)}, /* RESET */
{B8(1111111, 7), B8(0000000, 7), B8(001, 3),	 B8(0101, 4),	 B8(0011, 4),	 B8(01011, 5)}, /* IDLE */
{B8(1111111, 7), B8(011, 3),	 B8(00111, 5),	 B8(01, 2),		 B8(001111, 6),	 B8(0101111, 7)}, /* DRSHIFT */
{B8(1111111, 7), B8(011, 3),	 B8(01, 2),		 B8(0, 1),		 B8(001111, 6),	 B8(0101111, 7)}, /* DRPAUSE */
{B8(1111111, 7), B8(011, 3),	 B8(00111, 5),	 B8(010111, 6),	 B8(001111, 6),	 B8(01, 2)}, /* IRSHIFT */
{B8(1111111, 7), B8(011, 3),	 B8(00111, 5),	 B8(010111, 6),	 B8(01, 2),		 B8(0, 1)} /* IRPAUSE */
};
```

主要接口如下
```c
void tap_set_state_impl(tap_state_t new_state);
/* 设置当前状态 */
#define tap_set_state(new_state) \
	do { \
		LOG_DEBUG_IO("tap_set_state(%s)", tap_state_name(new_state)); \
		tap_set_state_impl(new_state); \
	} while (0)

/* 获取当前状态 */
tap_state_t tap_get_state(void);
/* 设置结束状态 */
void tap_set_end_state(tap_state_t new_end_state);
/* 获取结束状态 */
tap_state_t tap_get_end_state(void);

/* 查表计算从from到to状态需要发送的TMS的数据，放回一个整数 */
int tap_get_tms_path(tap_state_t from, tap_state_t to);

/* 查表计算从from到to状态需要发送的TMS的数据的长度 */
int tap_get_tms_path_len(tap_state_t from, tap_state_t to);

/* 计算一个状态对应表中的序号，只有稳定态才在可以计算出序号 */
int tap_move_ndx(tap_state_t astate);

/* 判断一个状态是否为稳定态 */
bool tap_is_state_stable(tap_state_t astate);

/* 计算下一个状态 */
tap_state_t tap_state_transition(tap_state_t current_state, bool tms);

```

#### 驱动接口

#####  命令
系统定义了jtag接口可以执行的命令，源码位于
> src/jtag/command.h<br/>
> src/jtag/command.c<br/>

代码里实现了大量的结构用于描述jtag操作
```c
/* 各种操作的联合体体 */
union jtag_command_container {
	struct scan_command *scan;
	struct statemove_command *statemove;
	struct pathmove_command *pathmove;
	struct runtest_command *runtest;
	struct stableclocks_command *stableclocks;
	struct reset_command *reset;
	struct end_state_command *end_state;
	struct sleep_command *sleep;
	struct tms_command *tms;
};

/* 命令的顶层结构 */
struct jtag_command {
	union jtag_command_container cmd;
	enum jtag_command_type type;
	struct jtag_command *next;
};
```

###### 扫描操作

扫描操作，就是把状态机切换到恰当的位置，使IR或DR链接到TDO/TDI之间，然后通过TCK执行移位操作，用以读取IR/DR
```c
/**
 * The inferred type of a scan_command_s structure, indicating whether
 * the command has the host scan in from the device, the host scan out
 * to the device, or both.
 */
enum scan_type {
	/** From device to host, */
	SCAN_IN = 1,
	/** From host to device, */
	SCAN_OUT = 2,
	/** Full-duplex scan. */
	SCAN_IO = 3
};

/**
 * The scan_command provide a means of encapsulating a set of scan_field_s
 * structures that should be scanned in/out to the device.
 */
struct scan_command {
	/** instruction/not data scan */
	bool ir_scan;
	/** number of fields in *fields array */
	int num_fields;
	/** pointer to an array of data scan fields */
	struct scan_field *fields;
	/** state in which JTAG commands should finish */
	tap_state_t end_state;
};
```

###### 状态切换操作

状态机切换有两个命令，一个用于单次切换指明目标状态，一种用于多次切换指明切换状态的路径
```c
/* 单次切换，end_state为目标状态 */
struct statemove_command {
	/** state in which JTAG commands should finish */
	tap_state_t end_state;
};

/* 多次切换，path记录状态切换的路径 */
struct pathmove_command {
	/** number of states in *path */
	int num_states;
	/** states that have to be passed */
	tap_state_t *path;
};
```
###### runtest

次操作用于延时等待，在`idel run test`状态下等待多少个时钟
```c
struct runtest_command {
	/** number of cycles to spend in Run-Test/Idle state */
	int num_cycles;
	/** state in which JTAG commands should finish */
	tap_state_t end_state;
};
```

###### 在某个稳定态下发送时钟信号

次操作只在特定平台下使用
```c
struct stableclocks_command {
	/** number of clock cycles that should be sent */
	int num_cycles;
};
```

###### 复位命令

```c
struct reset_command {
	/** Set TRST output: 0 = deassert, 1 = assert, -1 = no change */
	int trst;
	/** Set SRST output: 0 = deassert, 1 = assert, -1 = no change */
	int srst;
};
```
###### 指定命令完成时的状态

```c
struct end_state_command {
	/** state in which JTAG commands should finish */
	tap_state_t end_state;
};
```
###### 延时
```c
struct sleep_command {
	/** number of microseconds to sleep */
	uint32_t us;
};
```
###### TMS命令

用于在TMS脚发送信号
```c
struct tms_command {
	/** How many bits should be clocked out. */
	unsigned num_bits;
	/** The bits to clock out; the LSB is bit 0 of bits[0]. */
	const uint8_t *bits;
};
```

##### 内存管理

源码在
> src/jtag/command.h<br/>
> src/jtag/command.c<br/>

应为系统需要不断的创建命令删除命令来操作jtag，为了防止内存碎片，系统实现了一个内存管理方法。通过如下结构记录一个内存块。
```c
struct cmd_queue_page {
	struct cmd_queue_page *next;
	void *address;
	size_t used;
};
```

内存块的大小为1M，如下
```c
#define CMD_QUEUE_PAGE_SIZE (1024 * 1024)
```

通过两个指针来维和内存，一个执行链表头，一个执行链表尾
```c
static struct cmd_queue_page *cmd_queue_pages;
static struct cmd_queue_page *cmd_queue_pages_tail;
/* 申请内存 */
void *cmd_queue_alloc(size_t size);
/* 释放所有内存 */
static void cmd_queue_free(void);
```

同时维护了一个命令的链表
```c
struct jtag_command *jtag_command_queue;
static struct jtag_command **next_command_pointer = &jtag_command_queue;
/* 把命令添加到链表尾 */
void jtag_queue_command(struct jtag_command *cmd);
/* 把命令链表清空并释放内存 */
void jtag_command_queue_reset(void);
```
##### 命令的辅助函数

源码位于
> src/jtag/minidriver.h<br/>
> src/jtag/drivers/driver.c<br/>

这些代码主要负责创建命令并添加到命令链表中，等待执行。主要接口如下
```c
int interface_jtag_add_ir_scan(struct jtag_tap *active,
		const struct scan_field *fields,
		tap_state_t endstate);
int interface_jtag_add_plain_ir_scan(
		int num_bits, const uint8_t *out_bits, uint8_t *in_bits,
		tap_state_t endstate);

int interface_jtag_add_dr_scan(struct jtag_tap *active,
		int num_fields, const struct scan_field *fields,
		tap_state_t endstate);
int interface_jtag_add_plain_dr_scan(
		int num_bits, const uint8_t *out_bits, uint8_t *in_bits,
		tap_state_t endstate);

int interface_jtag_add_tlr(void);
int interface_jtag_add_pathmove(int num_states, const tap_state_t *path);
int interface_jtag_add_runtest(int num_cycles, tap_state_t endstate);

int interface_add_tms_seq(unsigned num_bits,
		const uint8_t *bits, enum tap_state state);

/**
 * This drives the actual srst and trst pins. srst will always be 0
 * if jtag_reset_config & RESET_SRST_PULLS_TRST != 0 and ditto for
 * trst.
 *
 * the higher level jtag_add_reset will invoke jtag_add_tlr() if
 * approperiate
 */
int interface_jtag_add_reset(int trst, int srst);
int interface_jtag_add_sleep(uint32_t us);
int interface_jtag_add_clocks(int num_cycles);
int interface_jtag_execute_queue(void);

/**
 * Calls the interface callback to execute the queue.  This routine
 * is used by the JTAG driver layer and should not be called directly.
 */
int default_interface_jtag_execute_queue(void);
```

##### 驱动抽象结构

源码位于
> src/jtag/interface.h<br/>

驱动通过一个结构体描述
```c
struct jtag_interface {
	/** The name of the JTAG interface driver. */
	const char * const name;

	/**
	 * Bit vector listing capabilities exposed by this driver.
	 */
	unsigned supported;
#define DEBUG_CAP_TMS_SEQ	(1 << 0)

	/** transports supported in C code (NULL terminated vector) */
	const char * const *transports;

	const struct swd_driver *swd;

	/**
	 * Execute queued commands.
	 * @returns ERROR_OK on success, or an error code on failure.
	 */
	int (*execute_queue)(void);

	/**
	 * Set the interface speed.
	 * @param speed The new interface speed setting.
	 * @returns ERROR_OK on success, or an error code on failure.
	 */
	int (*speed)(int speed);

	/**
	 * The interface driver may register additional commands to expose
	 * additional features not covered by the standard command set.
	 */
	const struct command_registration *commands;

	/**
	 * Interface driver must initialize any resources and connect to a
	 * JTAG device.
	 *
	 * quit() is invoked if and only if init() succeeds. quit() is always
	 * invoked if init() succeeds. Same as malloc() + free(). Always
	 * invoke free() if malloc() succeeds and do not invoke free()
	 * otherwise.
	 *
	 * @returns ERROR_OK on success, or an error code on failure.
	 */
	int (*init)(void);

	/**
	 * Interface driver must tear down all resources and disconnect from
	 * the JTAG device.
	 *
	 * @returns ERROR_OK on success, or an error code on failure.
	 */
	int (*quit)(void);

	/**
	 * Returns JTAG maxium speed for KHz. 0 = RTCK. The function returns
	 *  a failure if it can't support the KHz/RTCK.
	 *
	 *  WARNING!!!! if RTCK is *slow* then think carefully about
	 *  whether you actually want to support this in the driver.
	 *  Many target scripts are written to handle the absence of RTCK
	 *  and use a fallback kHz TCK.
	 * @returns ERROR_OK on success, or an error code on failure.
	 */
	int (*khz)(int khz, int *jtag_speed);

	/**
	 * Calculate the clock frequency (in KHz) for the given @a speed.
	 * @param speed The desired interface speed setting.
	 * @param khz On return, contains the speed in KHz (0 for RTCK).
	 * @returns ERROR_OK on success, or an error code if the
	 * interface cannot support the specified speed (KHz or RTCK).
	 */
	int (*speed_div)(int speed, int *khz);

	/**
	 * Read and clear the power dropout flag. Note that a power dropout
	 * can be transitionary, easily much less than a ms.
	 *
	 * To find out if the power is *currently* on, one must invoke this
	 * method twice.  Once to clear the power dropout flag and a second
	 * time to read the current state.  The default implementation
	 * never reports power dropouts.
	 *
	 * @returns ERROR_OK on success, or an error code on failure.
	 */
	int (*power_dropout)(int *power_dropout);

	/**
	 * Read and clear the srst asserted detection flag.
	 *
	 * Like power_dropout this does *not* read the current
	 * state.  SRST assertion is transitionary and may be much
	 * less than 1ms, so the interface driver must watch for these
	 * events until this routine is called.
	 *
	 * @param srst_asserted On return, indicates whether SRST has
	 * been asserted.
	 * @returns ERROR_OK on success, or an error code on failure.
	 */
	int (*srst_asserted)(int *srst_asserted);

	/**
	 * Configure trace parameters for the adapter
	 *
	 * @param enabled Whether to enable trace
	 * @param pin_protocol Configured pin protocol
	 * @param port_size Trace port width for sync mode
	 * @param trace_freq A pointer to the configured trace
	 * frequency; if it points to 0, the adapter driver must write
	 * its maximum supported rate there
	 * @returns ERROR_OK on success, an error code on failure.
	 */
	int (*config_trace)(bool enabled, enum tpiu_pin_protocol pin_protocol,
			    uint32_t port_size, unsigned int *trace_freq);

	/**
	 * Poll for new trace data
	 *
	 * @param buf A pointer to buffer to store received data
	 * @param size A pointer to buffer size; must be filled with
	 * the actual amount of bytes written
	 *
	 * @returns ERROR_OK on success, an error code on failure.
	 */
	int (*poll_trace)(uint8_t *buf, size_t *size);
};
```
其中，最主要的方法是`int (*execute_queue)(void)`，此方法用于执行实际的jtag命令（`struct jtag_command`）

#### 对外接口

源代码位于
> src/jtag/jtag.h<br/>
> src/jtag/core.c<br/>

这部分主要是封装一些操作，以便其他模块调用。



## 参考

1. [riscv调试规范](https://riscv.org/specifications/debug-specification/)

2. [riscv-openocd源码](https://github.com/riscv/riscv-openocd)