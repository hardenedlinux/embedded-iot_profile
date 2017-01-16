# Device Tree分析
此文档主要用于分析Device Tree的源码格式与其二进制文件表现，此文主要参照[Devicetree Specification](http://www.devicetree.org/specifications-pdf)
## 1 DT介绍
DT(Device Tree)用于描述硬件信息，是一种树形结构。DT由节点组成，每个节点有自己的属性（名字和值的键值对）和子节点（非必须，叶子节点就没有）。
DT有两种形式

- DTS(Device Tree Source)，文本格式遍与阅读和修改
- FDT(Flat Device Tree),DTS通过DTC编译产生的二进制文件
- DTB(Device Tree BLOB)，由多个FDT打包出的二进制文件

DTC(Device Tree Compile)用于把DTS转换成FDT的工具

## 2 DTS(device tree source)格式
### 2.1 节点
````
[lable:]node-name[@unit-address]{
	[properties definitions]
	[child nodes]
 }
````

`lable`，节点的标签，可以被其他节点引用
`node-name`，节点名，1-31个字符，正则表达式`[a-zA-Z][a-zA-Z0-9,._+-]*`
`unit-address`，设备的地址必须与属性reg匹配
`properties definitions`，为属性定义
`child nodes`，为子节点，子节点必须在塑性定义之后
### 2.2 属性
属性定义有两种格式
```
[label:] property-name = value;//此格式属性有值
[label:] property-name;//此格式属性没有值
```
`label`，塑性标签，可以被其他节点引用
`property-name`，属性名，1-31个字符，正则表达式[a-zA-Z0-9,._+?#]+
`value`，属性值，具有多种格式

- 字符串，使用引号（"）包围起来的字符序列
- 字符串列表，用逗号（,）分隔的字符串
- 数组（cells），用尖括号（<>）包围起来的数值（C语言32比特的整数）序列，用空格分隔。要表示64比特整数时可以使用连续的两个整数，第一个为高32比特第二个为低32比特。当数组只有一个32比特值时可以简写成一个数字不加尖括号。
- 字节序列，用放括号（[]）包围起来的数字，数字必须是16进制，可以在每个字节之间加空格，也可以不加。如[00 01 02]和[000102]是一样的
- 混合值，使用逗号分隔的多值（字符串、数组、字节序列中的一种或几种）

### 2.3 注释
可以使用C语言风格的注释，如下

格式1：
> //单行注释

格式2：
> /\*
> 多行注释
> \*/

### 2.4 文件格式
```
/dts-v1/;
[memory reservations]
/ {
	[property definitions]
	[child nodes]
};
```
`/dts-v1/`，标识dts的版本，如果没有dtc将认为是过时的版本（版本号0）
`memory reservations`，用于表示保留的内存（不可用），格式`/memreserve/ <address> <length>`其中`<address> ` `<length>`为C语言风格的64比特的整数
`/{}`为根节点
### 2.5 节点全路径
从根节点开始到目标节点路径上的所有节点，用'/'符号分隔的字符串，类似文件路径

在属性引用节点时就有以下一些方式
```
<&lable>
<&{fullpath}>
&lable
&{fullpath}
```
### 2.6 标准属性
#### 2.6.1 compatible
值，字符串列表
标识兼容的驱动，操作系统通过此数字确定适用的驱动程序
#### 2.6.2 model
值，字符串列表
推荐用于记录设备的名称，格式"公司注册码：设备名"
#### 2.6.3 phandle
值，<u32>
用于标记节点，必须是唯一的值，可以在其他节点中被引用
#### 2.6.4 status
值，字符串

- "okay"      标识设备可用
   -"disabled"  标识设备不可用（预留或者硬件未连接）
- "fail"      设备检查到错误，并且设备不能被操作在修复之前
- "fail-sss"  设备检查到错误，并且设备不能被操作在修复之前。sss和具体设备相关，标识错位的原因。
#### 2.6.5 \#addess-cells \#size-cells
此属性对子节点的的reg有影响，#address-cells表示address使用几个32比特，#size-cells表示size使用几个32比特
默认#address-cells=2、#size-cells=1
#### 2.6.6 reg
此属性用于表示当前设备的地址
格式
```
reg <address size [address size [address size [address size ...]]]>
```
`address`、`size`大小和#addess-cells、#size-cells相关
#### 2.6.7 ranges
格式
```
ranges <child-bus-address parent-bus-address length [child-bus-address parent-bus-address length ...]>
```
此塑性性标记子节点和父节点的地址映射关系，列：
`ranges <0x0 0x100 0x200>`
表示子设备地址0x0-0x200映射到父总线0x100-0x300
#### 2.6.8 dma-ranges
类似与ranges，表示dma设备访问内存，表示dma内存和主内存的映射关系
2.6.10 name
值，字符串
表示节点的名字，此属性以弃用。
#### 2.6.9 device_type
值，字符串
标识设备类型，此属性以弃用。只在CPU和memory节点中使用。
### 2.7 标准节点
#### 2.7.1 根节点
根节点无名字/{}
根节点必须具备以下塑性
\#address-cells
\#size-cells
model
compatible
#### 2.7.1 /aliases
此节点用于定义节点别名
别名长度1-31个字符，正则[0-9a-z-]*
```
aliases {
serial0 = "/simple-bus@fe000000/serial@llc500";
ethernet0 = "/simple-bus@fe000000/ethernet@31c000";
}
```
#### 2.7.2 /memory
此节点用于指定内存信息
其中device-type必须指定为"memory"
其中reg指定内存范围
#### 2.7.3 /cpus /cpu
用与配置处理器信息
#### 2.7.4 /chosen
此节点不是必须的
此节点有三个属性
bootargs 值为字符串，命令行参数
stdout-path 值为字符串，指定启动时的输出设备
stdin-path 值为字符串，指定启动时的输入设备

## 3 FDT(Flat Device Tree)
FDT是DTS经过DTC处理得到的文件，是DTS的二进制格式，FDT库在kernel源码中的位置（scripts/dtc/libfdt），FDT库在little kernel源码中的位置（lib/libfdt）。理解FDT可以借助fdtdump工具，ubuntu下安装：`sudo apt install device-tree-compiler`，FDT使用大端格式，FDT文件格式如下：
```
---------------------------
fdt_header
magic
totalsize
off_dt_struct
off_dt_strings
off_mem_rsvmap
version
last_comp_version
boot_cpuid_phys
size_dt_strings
size_dt_struct
---------------------------
(free spaces)
---------------------------
memory reservation block
---------------------------
(free spaces)
---------------------------
structure block
---------------------------
(free spaces)
---------------------------
strings block
---------------------------
(free spaces)
---------------------------
```
fdt_header中主要保存了memory reservation block、structure block、strings block的位置信息，以及structure block、strings block的大小信息

其中free spaces不一定存在，只是用来处理内存对齐的

memory reservation block对应dts根节点之前定义的memory reservations，其中以两个64比特的数据保存（对应address和size），以一个size为0的结束

```c
struct fdt_reserve_entry {
    uint64_t address;   /* 保留内存的起始地址 */
    uint64_t size;      /* 保留内存的大小 */
};/* 此数据块以一个size为0的结构结束 */
```
strings block字符串表，用于存储字符串
structure block是文件的主体，记节点信息

从内存中读取一个32比特数，确定当前记录类型tag（节点、属性），有以下类型的记录
```c
#define FDT_BEGIN_NODE	0x1		/* 节点头记录，后面紧接着节点的名字 */
#define FDT_END_NODE	0x2		/* 节点尾记录 */
#define FDT_PROP	    0x3		/* 属性记录，后面接着属性值的长度、属性名在字符串表中的偏移、属性 */
#define FDT_NOP		    0x4		/* 空的记录 */
#define FDT_END		    0x9     /* 记录结束标记 */
```
其中FDT_BEGIN_NODE、FDT_END_NODE、FDT_PROP为常用记录，FDT_NOP、FDT_END在我分析的DTB中没有出现

FDT_BEGIN_NODE：节点头记录，后面紧接着节点的名字，如果名字长度不足4的倍数在后面补0x00,对应C语言结构体。
```c
struct fdt_node_header {
uint32_t tag;       /* 节点头固定为FDT_BEGIN_NODE */
char name[0];       /* 节点名字，长度不足4的倍数会在后面补0 */
};
```
节点名没有保存到字符串表中，因为节点名是唯一的，保存到字符串表中不能减少文件的大小
FDT_END_NODE： 节点尾记录，和FDT_BEGIN_NODE成对出现，无任何数据
FDT_PROP属性记录：用于记录属性信息，对应结构体如下
```c
struct fdt_property {
uint32_t tag;       /* 固定为FDT_PROP */
uint32_t len;       /* 属性值的长度，属性值的类型和属性名有关 */
uint32_t nameoff;   /* 属性名在字符串表中的偏移量 */
char data[0];       /* 属性值，长度不为4的倍数会在后面补0 */
};
```
其中属性名保存在字符串表中，因为同一个属性会在多个节点中出现，保存到字符串表中可以大量减少文件大小

nameoff为字符串在字符串表中的偏移量，off_dt_strings + nameoff为对应字符串在文件中的偏移量

数据类型和属性名相关，在这里不作类型处理，全部保存在字符数组中，需要解析程序自行处理

## 4 DTB(Device Tree BLOB)

DTB由多个FDT打包得到，DTB文件格式如下
```
--------------------------------------------------------------
dt_table
    magic
    version
    num_entries
--------------------------------------------------------------
dt_entry(entry 0)
    platform_id
    variant_id
    board_hw_subtype
    soc_rev
    pmic_rev[4]
    offset
    size
--------------------------------------------------------------
dt_entry(entry 1)
--------------------------------------------------------------
dt_entry(entry 2)
--------------------------------------------------------------
...
--------------------------------------------------------------
dt_entry(entry n, n == dt_table.num_entries)
--------------------------------------------------------------
FDT 0
--------------------------------------------------------------
FDT 1
--------------------------------------------------------------
FDT 2
--------------------------------------------------------------
...
--------------------------------------------------------------
FDT n(n == dt_table.num_entries)
--------------------------------------------------------------
```
dt_table主要记录有多个dt_entry
dt_entry记录fdt在文件中的位置（offset）和大小（size），以及fdt对应的硬件信息（platform_id、variant_id、board_hw_subtype、soc_rev）

