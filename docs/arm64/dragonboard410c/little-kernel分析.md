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
通过链接脚本arch/arm/system-onesegment.ld第4行ENTRY(\_start)，可以知道程序从\_start开始执行，\_start位于arch/arm/crt0.S中，此文件实现了异常向量表、堆栈初始化、数据段初始化（data、BSS），最后跳转到kmain，kmain位于kernel/main.c中，kmain中做了大量初始化，最后创建线程bootstrap2，bootstrap2中依旧有初始化相关的操作，最后通过apps_init启动little kernel的应用程序

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







