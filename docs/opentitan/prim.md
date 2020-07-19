# prim

prim是一组模块组成的基础设施，代码实现位于`hw/ip/prim/rtl`。此文档用于分析相关基础设施的实现。prim依赖于一些基础模块：

- `prim_generic`位于`hw/ip/prim_generic/rtl`，通用的实现方式
- `prim_xilinx`位于`hw/ip/prim_xilinx/rtl`，xilinx平台下的实现

## 底层模块

### clock gating

数字电路可以等效为一个RC电路，每一个时钟周期进行一次冲放电，为了省点会在电路中插入时钟门控（clock gating），让需要工作的电路连接到时钟，不需要工作的电路从时钟断开。此模块就是用于控制时钟是否输出的，通用实现如下：

```systemverilog
module prim_generic_clock_gating (
  input        clk_i,      // 时钟输入
  input        en_i,       // 使能时钟输出
  input        test_en_i,  // 测试用的使能
  output logic clk_o       // 时钟输出
);

  // Assume en_i synchronized, if not put synchronizer prior to en_i
  logic en_latch;
  always_latch begin
    if (!clk_i) begin
      en_latch = en_i | test_en_i;
    end
  end
  assign clk_o = en_latch & clk_i;

endmodule
```

### clock mux2

此模块用于在两个时钟信号中选择一个时钟信号，通用实现如下：

```systemverilog
module prim_generic_clock_mux2 (
  input        clk0_i,
  input        clk1_i,
  input        sel_i,
  output logic clk_o
);

  assign clk_o = (sel_i) ? clk1_i : clk0_i;

  // make sure sel is never X (including during reset)
  // need to use ##1 as this could break with inverted clocks that
  // start with a rising edge at the beginning of the simulation.
  `ASSERT(selKnown0, ##1 !$isunknown(sel_i), clk0_i, 0)
  `ASSERT(selKnown1, ##1 !$isunknown(sel_i), clk1_i, 0)

endmodule : prim_generic_clock_mux2
```

### pad wrapper

此模块把分开的输入输出信号，合并为一个输入输出双向IO，模块接口如下：

```systemverilog
module prim_generic_pad_wrapper #(
  parameter int unsigned AttrDw = 6
) (
  inout wire         inout_io, // 双向IO的pad
  output logic       in_o,     // 连接到输入脚
  input              out_i,    // 连接到输出脚
  input              oe_i,     // 输出使能信号
  // 副加塑性 {drive strength, keeper, pull-up, pull-down, open-drain, invert}
  input [AttrDw-1:0] attr_i
);
```

其中，注意`open-drain`（集电极开路只输出低电平和高阻态）

### ram 1p

一个接口的ram存储器，接口如下：

```systemverilog
module prim_generic_ram_1p #(
  parameter  int Width           = 32, // bit
  parameter  int Depth           = 128,
  parameter  int DataBitsPerMask = 1, // Number of data bits per bit of write mask
  parameter      MemInitFile     = "", // VMEM file to initialize the memory with

  localparam int Aw              = $clog2(Depth)  // derived parameter
) (
  input  logic             clk_i,

  input  logic             req_i,
  input  logic             write_i,
  input  logic [Aw-1:0]    addr_i,
  input  logic [Width-1:0] wdata_i,
  input  logic [Width-1:0] wmask_i,
  output logic [Width-1:0] rdata_o // Read data. Data is returned one cycle after req_i is high.
);
```

### ram 2p

两个接口的ram存储器，每个接口有独立的时钟信号，接口如下：

```systemverilog
module prim_generic_ram_2p #(
  parameter  int Width           = 32, // bit
  parameter  int Depth           = 128,
  parameter  int DataBitsPerMask = 1, // Number of data bits per bit of write mask
  parameter      MemInitFile     = "", // VMEM file to initialize the memory with

  localparam int Aw              = $clog2(Depth)  // derived parameter
) (
  input clk_a_i,
  input clk_b_i,

  input                    a_req_i,
  input                    a_write_i,
  input        [Aw-1:0]    a_addr_i,
  input        [Width-1:0] a_wdata_i,
  input  logic [Width-1:0] a_wmask_i,
  output logic [Width-1:0] a_rdata_o,


  input                    b_req_i,
  input                    b_write_i,
  input        [Aw-1:0]    b_addr_i,
  input        [Width-1:0] b_wdata_i,
  input  logic [Width-1:0] b_wmask_i,
  output logic [Width-1:0] b_rdata_o
);
```

### rom

此模块实现了一个rom存储器，实现如下：

```systemverilog
module prim_generic_rom #(
  parameter  int Width       = 32,
  parameter  int Depth       = 2048, // 8kB default
  parameter      MemInitFile = "", // VMEM file to initialize the memory with

  localparam int Aw          = $clog2(Depth)
) (
  input  logic             clk_i,
  input  logic             req_i,
  input  logic [Aw-1:0]    addr_i,
  output logic [Width-1:0] rdata_o
);

  logic [Width-1:0] mem [Depth];

  always_ff @(posedge clk_i) begin
    if (req_i) begin
      rdata_o <= mem[addr_i];
    end
  end

  `include "prim_util_memload.sv"

  ////////////////
  // ASSERTIONS //
  ////////////////

  // Control Signals should never be X
  `ASSERT(noXOnCsI, !$isunknown(req_i), clk_i, '0)
endmodule
```

### otp(one time programmable memory)

此模块实现了一个一次编程存储器，接口如下：

```systemverilog
module prim_generic_otp #(
  parameter  int Width     = 8,            // 数据的位宽
  parameter  int Depth     = 1024,         // 地址个数
  parameter  int ErrWidth  = 8,            // 错误码的宽度
  localparam int AddrWidth = $clog2(Depth) // 地址的宽度
) (
  input                        clk_i,
  input                        rst_ni,
  // TODO: power sequencing signals from/to AST
  // Test interface
  input  tlul_pkg::tl_h2d_t    test_tl_i,
  output tlul_pkg::tl_d2h_t    test_tl_o,
  // Start Macro init sequence
  // init shall be asserted until acknowledged with done
  input                        init_req_i,
  output logic                 init_done_o,
  // Macro error output
  // TODO: define error codes
  output logic                 err_valid_o,
  output logic [ErrWidth-1:0]  err_code_o,
  // Ready valid handshake for read/write command
  output logic                 ready_o, 
  input                        valid_i,
  input [AddrWidth-1:0]        addr_i,  // 地址
  input [Width-1:0]            wdata_i, // 要写如的数据
  input                        wren_i, // 0: read command, 1: write command
  // Read data output
  output logic [Width-1:0]     rdata_o, // 读出的数据
  output logic                 rvalid_o // 标识有有效的数据读出 
);
```

为了保证每个位只能由`0`翻转到`1`，每一次写操着前会执行一次读，并把要写的信息缓存

```systemverilog
  // 用上一个周期读出的值和需要写入的值或，防止`1`到`0`的翻转
  assign wdata = rdata_o | wdata_q;

  always @(posedge clk_i) begin
    if (req) begin
      if (write_q) begin
        mem[addr] <= wdata;
      end else begin
        // 当收到写操作时，第一个周期会进行一次读操作
        rdata_o <= mem[addr];
      end
    end
  end

  // 缓存写相关的信号
  always_ff @(posedge clk_i or negedge rst_ni) begin : p_regs
    if (!rst_ni) begin
      read_q  <= 1'b0;
      write_q <= 1'b0;
      wdata_q <= '0;
      waddr_q <= '0;
    end else begin
      read_q  <= read_d;
      write_q <= write_d;
      wdata_q <= wdata_d;
      waddr_q <= waddr_d;
    end
  end
```

### flash

此模块通过ram模拟了一个flash，接口如下：

```systemverilog
module prim_generic_flash #(
  parameter int InfosPerBank = 1,   // info pages per bank
  parameter int PagesPerBank = 256, // data pages per bank
  parameter int WordsPerPage = 256, // words per page
  parameter int DataWidth   = 32,   // bits per word
  parameter bit SkipInit = 1,       // this is an option to reset flash to all F's at reset

  // Derived parameters
  localparam int PageW = $clog2(PagesPerBank),
  localparam int WordW = $clog2(WordsPerPage),
  localparam int AddrW = PageW + WordW
) (
  input                              clk_i,       // 时钟
  input                              rst_ni,      // 复位信号
  input                              rd_i,        // 读
  input                              prog_i,      // 编程
  input                              pg_erase_i,  // 页擦除
  input                              bk_erase_i,  // 块擦除
  input [AddrW-1:0]                  addr_i,      // 地址
  input flash_ctrl_pkg::flash_part_e part_i,      // 指定要操作的页：info/data page
  input [DataWidth-1:0]              prog_data_i, // 要写入的数据
  output logic                       ack_o,       // 响应信号
  output logic [DataWidth-1:0]       rd_data_o,   // 读出的数据
  output logic                       init_busy_o  // 忙信号
);
```

## prim_alert

此功能由两个模块组成，用于SoC内部总线传递alert，可以用于中断。包含以下两个模块:

- prim_alert_receiver：用于接受alert，实现位于`hw/ip/prim/rtl/prim_alert_receiver.sv`
- prim_alert_sender: 用于发送alert，实现位于`hw/ip/prim/rtl/prim_alert_sender.sv`

prim_alert_receiver与prim_alert_sender通过三组差分信号连接，如下

```systemverilog
package prim_alert_pkg;
  // 从prim_alert_sender连接到prim_alert_receiver
  typedef struct packed {
    logic alert_p;
    logic alert_n;
  } alert_tx_t;

  // 从prim_alert_receiver连接到prim_alert_sender
  typedef struct packed {
    // 用于ping测试
    logic ping_p;
    logic ping_n;
    // 用于发生应答响应
    logic ack_p;
    logic ack_n;
  } alert_rx_t;
endpackage
```

接口如下：

```systemverilog
module prim_alert_receiver
  import prim_alert_pkg::*;
#(
  // enables additional synchronization logic
  parameter bit AsyncOn = 1'b0
) (
  // 时钟和复位信号
  input             clk_i,
  input             rst_ni,
  // 用于触发测试
  input             ping_en_i,
  output logic      ping_ok_o,
  // 信号完整性出错
  output logic      integ_fail_o,
  // 检测到alert
  output logic      alert_o,
  // 与prim_alert_receiver通讯的差分信号
  output alert_rx_t alert_rx_o,
  input alert_tx_t  alert_tx_i
);

module prim_alert_sender
  import prim_alert_pkg::*;
#(
  // enables additional synchronization logic
  parameter bit AsyncOn = 1'b1
) (
  // 时钟和复位信号
  input             clk_i,
  input             rst_ni,
  // 此引脚连接到外设，用于触发alert
  input             alert_i,
  // 与prim_alert_receiver通讯的差分信号
  input alert_rx_t  alert_rx_i,
  output alert_tx_t alert_tx_o
);
```

## prim_esc

此模块与`prim_alert`类似，差别在于发起器连接cpu还是接收器连接cpu。由于发送器连接cpu，所以少一组ping测试用的差分信号。

## prim_assert

此模块定义了一些宏，用于断言，它屏蔽了一些具体平台的差异。具体平台需要实现9个宏：

```systemverilog
`define ASSERT_I(__name, __prop)
`define ASSERT_INIT(__name, __prop)
`define ASSERT_FINAL(__name, __prop)
`define ASSERT(__name, __prop, __clk = `ASSERT_DEFAULT_CLK, __rst = `ASSERT_DEFAULT_RST)
`define ASSERT_NEVER(__name, __prop, __clk = `ASSERT_DEFAULT_CLK, __rst = `ASSERT_DEFAULT_RST)
`define ASSERT_KNOWN(__name, __sig, __clk = `ASSERT_DEFAULT_CLK, __rst = `ASSERT_DEFAULT_RST)
`define COVER(__name, __prop, __clk = `ASSERT_DEFAULT_CLK, __rst = `ASSERT_DEFAULT_RST)
`define ASSUME(__name, __prop, __clk = `ASSERT_DEFAULT_CLK, __rst = `ASSERT_DEFAULT_RST)
`define ASSUME_I(__name, __prop)
```

当前系统有三个实现了：

- prim_assert_dummy_macros.svh，用于完全不支持断言的平台
- prim_assert_yosys_macros.svh，用于yosys
- prim_assert_standard_macros.svh，用于完整支持SystemVerilog和SVA的工具

然后在这个9个宏基础上实现了`ASSERT_PULSE`/`ASSERT_IF`/`ASSERT_KNOWN_IF`/`ASSUME_FPV`/`ASSUME_I_FPV`/`COVER_FPV`

## prim_arbiter

此模块实现了一个仲裁器，可以从多组信号中选择一组信号。此模块有两种实现方式（`prim_arbiter_ppc`/`prim_arbiter_tree`），功能类似。模块接口如下：

```systemverilog
module prim_arbiter_ppc #(
  parameter int unsigned N  = 8,  // 输入信号的个数
  parameter int unsigned DW = 32, // 每个信号的宽度

  // 配置
  // 如果EnDataPort = 0 data_o始终输出0
  // 如果EnDataPort = 1 data_o输出当前选择的信号
  parameter bit EnDataPort = 1,

  // Non-functional parameter to switch on the request stability assertion
  parameter bit EnReqStabA = 1,

  // 计算一个数据宽度用于输出当前选择的信号
  localparam int IdxW = $clog2(N)
) (
  // 时钟和复位信号
  input clk_i,
  input rst_ni,

  input        [ N-1:0]    req_i,      // 请求信号
  input        [DW-1:0]    data_i [N], // 数据输入
  output logic [ N-1:0]    gnt_o,      // 请求响应
  output logic [IdxW-1:0]  idx_o,      // 输出当前仲裁选择的信号的编号

  output logic             valid_o,    // 输出信号data_o有效标识
  output logic [DW-1:0]    data_o,     // 输出信号
  input                    ready_i     // 后端模块准备就绪
);
```

代码中定义了一个寄存器`mask`，它每一个周期左移移位。用这个寄存器去与上`req_i`，然后响应最低位的请求。仲裁器通过如下代码选择出信号：

```systemverilog
  // arb_req = req_i & mask
  // 此段代码，把最低为`1`的位左侧所有的位置`1`
  always_comb begin
    ppc_out[0] = arb_req[0];
    for (int i = 1 ; i < N ; i++) begin
      ppc_out[i] = ppc_out[i-1] | arb_req[i];
    end
  end

  // 计算出最低位为1的位置 
  assign winner = ppc_out ^ {ppc_out[N-2:0], 1'b0};
```

## prim_clock_inverter

此模块用于翻转时钟信号，接口如下：

```systemverilog
module prim_clock_inverter #(
  parameter bit HasScanMode = 1'b1 // 此信号用于设置scanmode_i是否有效
) (
  input        clk_i,              // 输入信号
  input        scanmode_i,         // 此信号有效时，clk_no = clk_i
  output logic clk_no              // 输出信号，clk_no = (scanmode_i & HasScanMode) ? clk_i : ~clk_i
);
```

## prim_diff_decode

此模块用于给差分信号解码，模块接口如下：

```systemverilog
module prim_diff_decode #(
  // enables additional synchronization logic
  parameter bit AsyncOn = 1'b0
) (
  input        clk_i,    // 时钟信号
  input        rst_ni,   // 复位信号
  // input diff pair
  input        diff_pi,  // 差分信号的正输入
  input        diff_ni,  // 差分信号的负输入
  // logical level and
  // detected edges
  output logic level_o,  // 差分信号的值: diff_pi > diff_ni -> 1, diff_pi < diff_ni -> 0
  output logic rise_o,   // 差分信号上升沿
  output logic fall_o,   // 差分信号下降沿
  // either rise or fall
  output logic event_o,  // 差分信号发生变化，rise_o | fall_o
  //signal integrity issue detected
  output logic sigint_o  // 差分信号发生错误： diff_pi == diff_ni
);
```

## prim_filter

此模块是基于移位寄存器的滤波器，可以滤除杂波，接口如下：

```systemverilog
module prim_filter #(parameter int Cycles = 4) (
  input  clk_i,    // 时钟
  input  rst_ni,   // 复位
  input  enable_i, // 使能信号，禁用时filter_o=filter_i
  input  filter_i, // 信号输入
  output filter_o  // 信号输出
);
```

其中，参数`Cycles`用于设定输入信号需要稳定的时钟周期数。每个周期把`filter_i`移入一个移位寄存器`stored_vector_q`，当移位寄存器全部为`0`或为`1`时，就更新`stored_value_q`。`stored_value_q`就为滤波后的信号。

```systemverilog
  // 信号稳定后更新stored_value_q
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      stored_value_q <= 1'b0;
    end else if (update_stored_value) begin
      stored_value_q <= filter_i;
    end
  end
  
  assign stored_vector_d = {stored_vector_q[Cycles-2:0],filter_i}; // 把filter_i移近寄存器stored_vector_q
  assign unused_stored_vector_q_msb = stored_vector_q[Cycles-1];
  
  // 移位寄存器更新
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      stored_vector_q <= {Cycles{1'b0}};
    end else begin
      stored_vector_q <= stored_vector_d;
    end
  end
  
  // 当移位寄存器全为0,或全为1时，可以更新stored_value_q
  assign update_stored_value =
             (stored_vector_d == {Cycles{1'b0}}) |
             (stored_vector_d == {Cycles{1'b1}});

  // 使能时，输出过滤值；禁用时直接输出
  assign filter_o = enable_i ? stored_value_q : filter_i;
```

## prim_filter_ctr

此模块是一个基于定时器的滤波模块，可以滤除杂波，接口如下：

```systemverilog
module prim_filter_ctr #(parameter int unsigned Cycles = 4) (
  input  clk_i,    // 时钟
  input  rst_ni,   // 复位
  input  enable_i, // 使能信号，禁用时filter_o=filter_i
  input  filter_i, // 信号输入
  output filter_o  // 信号输出
);
```

其中，参数`Cycles`用于设定输入信号需要稳定的时钟周期数。当输入信号`filter_i`发生变化时计数器`diff_ctr_q`清零，如果输入信号不变，每一个周期计数器`diff_ctr_q`加一，计数器`diff_ctr_q`达到最大值时不再变化，并且更新`stored_value_q`为输入的值。`stored_value_q`就为滤波后的信号。

```systemverilog
  // filter_q用于记录上一个周期，filter_i的状态
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      filter_q <= 1'b0;
    end else begin
      filter_q <= filter_i;
    end
  end
  
  // 当到达Cycles时记录下稳定态
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      stored_value_q <= 1'b0;
    end else if (update_stored_value) begin
      stored_value_q <= filter_i;
    end
  end

  // 周期更新
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      diff_ctr_q <= {CTR_WIDTH{1'b0}};
    end else begin
      diff_ctr_q <= diff_ctr_d;
    end
  end

  // always look for differences, even if not filter enabled
  assign diff_ctr_d =
             (filter_i != filter_q)           ? '0       : // 当输入发生变化时，复位计数器
                     (diff_ctr_q == CYCLESM1) ? CYCLESM1 : // 当计数器到达CYCLESM1时，计数器不在增加
                         (diff_ctr_q + 1'b1);              // 计数器增加
  assign update_stored_value = (diff_ctr_d == CYCLESM1);   // 计数器到达时，更新存储值

  assign filter_o = enable_i ? stored_value_q : filter_i;  // 使能时，输出过滤值；禁用时直接输出
```

## prim_cipher

prim实现了一些块加密算法（`prim_present` / `prim_prince`）。主要由以下文件实现

- hw/ip/prim/rtl/prim_cipher_pkg.sv，此文件中实现了一些函数，这些函数在两种加密算法中都会使用到  
- hw/ip/prim/rtl/prim_subst_perm.sv，此模块可以对任何长度数据进行扩散
- hw/ip/prim/rtl/prim_present.sv，此模块实现了present加解密算法  
- hw/ip/prim/rtl/prim_prince.sv，此模块实现了prince加解密算法  

### prim_subst_perm

`prim_subst_perm`，此模块可以对任意长度数据进行扩散，此算法不能用于加密，安全系数不够。此模块接口如下：

```systemverilog
module prim_subst_perm #(
  parameter int DataWidth = 64,  // 一次处理的数据的宽度
  parameter int NumRounds = 31,  // 通过多少轮来进行数据扩散
  parameter bit Decrypt   = 0    // 0: encrypt, 1: decrypt
) (
  input        [DataWidth-1:0] data_i, // 输入数据
  input        [DataWidth-1:0] key_i,  // 密钥
  output logic [DataWidth-1:0] data_o  // 输出
);
```

此算法主要用到三种方法：

- 异或
- sbox替换，系统定义了一个表（被叫作sbox），通过此表进行4bit对4bit的特换
- 位翻转，翻转后大小端调换
- 位重排，重排后所有的偶数bit被调到前端

每一轮执行一次以上算法

### present

此算法用于块加密解密，此算法优点在于易于硬件实现。此算法的块大小密码长度以及加密轮数是可配置的。以下分析以默认配置说明，默认配置为：64位块，128位密码长度，31轮。

此算法有一个16个4比特的sbox，此sbox用于对原文进行替换。

每一轮都会对密码进行变换，然后每一轮主要执行两个操作：用sbox进行替换，对位进行重新排序。加密过程伪代码如下

```c
void present_encrypt(
    u64  * data_i, // 需要加密的块
    u128 * key_i,  // 输入的加密密钥
    u64  * data_o, // 加密结果
    u128 * key_o,  // 用于加密下一个数据块的加密密钥
) {
    u128 round_key = key_i;
    u64 data_state = data_i;
    for (int NumRound = 0; NumRound < 31; NumRound++) {
        data_state = data_state ^ (round_key >> 64);
        data_state = sbox_64bit(data_state);
        data_state = perm_64bit(data_state);
        round_key  = present_update_key128(round_key, NumRound + 1);
    }
    data_o = data_state ^ (round_key >> 64);
    key_o  = round_key;
}
```

## prim_keccak

`prim_keccak`模块实现了SHA3的核心算法`keccakf`算法。此模块为组合逻辑，模块接口如下：

```
module prim_keccak #(
  parameter int Width = 1600, // b= {25, 50, 100, 200, 400, 800, 1600}

  // Derived
  localparam int W        = Width/25,
  localparam int L        = $clog2(W),
  localparam int MaxRound = 12 + 2*L, // Keccak-f only
  localparam int RndW     = $clog2(MaxRound) // Representing up to MaxRound-1
) (
  input        [RndW-1:0]  rnd_i,   // 轮数
  input        [Width-1:0] s_i,     // 输入
  output logic [Width-1:0] s_o      // 输出
);
```

此算法在SHA3中会迭代调用很多次这个算法，每次从原文中取出1600位与上一次计算的输出异或，然后作为新的输入，直到所有的原文被遍历完成。

此算法会把输入转换为一个5×5×W的数组。然后在此数组上进行一系列的操作：`theta`/`rho`/`pi`/`chi`/`iota`

### theta

`theta`执行以下操作：

```c
  // A为5×5×W的数组,C D为中间变量，theta为计算结果
  C[x,z] = A[x,0,z] ^ A[x,1,z] ^ A[x,2,z] ^ A[x,3,z] ^ A[x,4,z]
  //此处的x-1/x+1需要考虑溢出，实际代码为 
  // x-1 -> x == 0 ? 4 : x - 1
  // x+1 -> x == 4 ? 0 : x + 1
  D[x,z] = C[x-1,z] ^ C[x+1,z-1] 
  theta = A[x,y,z] ^ D[x,z]
```

### rho

`rho`可以看作对每个W宽度的元素进行循环右移，并且每个元素移动的位数不同。通过如下表格计算实际需要移位的位数：

```c
// A为5×5×W的数组
// A[x,y]需要循环右移的位数为： W - RhoOffset[x][y] % W
int RhoOffset [5][5]  = '{
 //y  0    1    2    3    4     x
  {   0,  36,   3, 105, 210},// 0
  {   1, 300,  10,  45,  66},// 1
  { 190,   6, 171,  15, 253},// 2
  {  28,  55, 153,  21, 120},// 3
  {  91, 276, 231, 136,  78} // 4
};
```

### pi

`pi`进行位置替换，替换规则如下：

```c
// pi为计算结果，state为输入
pi[x,y,z] = state[((x + 3y) % 5, x ,z]
```

### chi

`chi`进行如下计算： 

```c
// chi为计算结果，state为输入
// 其中x+1,x+2需要考虑溢出，实际如下：
// x + 1 -> x == 4 ? 0 : x + 1
// x + 2 -> x > 2 ? x - 3 : x + 2
chi[x,y,z] = state[x,y,z] ^ ((state[x+1,y,z] ^ 1) & state[x+2,y,z])
```

### iota

`iota`对5×5的第一个元素处理，异或上一个数，这个数与轮次有关。这个需要异或上的数通过表记录：

```systemverilog
  // 需要进行异或的表，通过轮次查表
  localparam logic [63:0] RC [24] = '{
     64'h 0000_0000_0000_0001, // Round 0
     64'h 0000_0000_0000_8082, // Round 1
     64'h 8000_0000_0000_808A, // Round 2
     64'h 8000_0000_8000_8000, // Round 3
     64'h 0000_0000_0000_808B, // Round 4
     64'h 0000_0000_8000_0001, // Round 5
     64'h 8000_0000_8000_8081, // Round 6
     64'h 8000_0000_0000_8009, // Round 7
     64'h 0000_0000_0000_008A, // Round 8
     64'h 0000_0000_0000_0088, // Round 9
     64'h 0000_0000_8000_8009, // Round 10
     64'h 0000_0000_8000_000A, // Round 11
     64'h 0000_0000_8000_808B, // Round 12
     64'h 8000_0000_0000_008B, // Round 13
     64'h 8000_0000_0000_8089, // Round 14
     64'h 8000_0000_0000_8003, // Round 15
     64'h 8000_0000_0000_8002, // Round 16
     64'h 8000_0000_0000_0080, // Round 17
     64'h 0000_0000_0000_800A, // Round 18
     64'h 8000_0000_8000_000A, // Round 19
     64'h 8000_0000_8000_8081, // Round 20
     64'h 8000_0000_0000_8080, // Round 21
     64'h 0000_0000_8000_0001, // Round 22
     64'h 8000_0000_8000_8008  // Round 23
  };
  // iota为计算结果，state为输入
  iota = state;
  iota[0][0][W-1:0] = state[0][0][W-1:0] ^ RC[rnd][W-1:0];
```
## prim_lfsr

`prim_lfsr`模块实现了一个线性反馈移位寄存器，此模块用于生成伪随机数，接口如下：

```systemverilog
module prim_lfsr #(
  // Lfsr Type, can be FIB_XNOR or GAL_XOR
  parameter                    LfsrType     = "GAL_XOR",
  // Lfsr width
  parameter int unsigned       LfsrDw       = 32,
  // Width of the entropy input to be XOR'd into state (lfsr_q[EntropyDw-1:0])
  parameter int unsigned       EntropyDw    =  8,
  // Width of output tap (from lfsr_q[StateOutDw-1:0])
  parameter int unsigned       StateOutDw   =  8,
  // Lfsr reset state, must be nonzero!
  parameter logic [LfsrDw-1:0] DefaultSeed  = LfsrDw'(1),
  // Custom polynomial coeffs
  parameter logic [LfsrDw-1:0] CustomCoeffs = '0,
  // Enable this for DV, disable this for long LFSRs in FPV
  parameter bit                MaxLenSVA    = 1'b1,
  // Can be disabled in cases where seed and entropy
  // inputs are unused in order to not distort coverage
  // (the SVA will be unreachable in such cases)
  parameter bit                LockupSVA    = 1'b1,
  parameter bit                ExtSeedSVA   = 1'b1
) (
  input                         clk_i,
  input                         rst_ni,
  input                         seed_en_i, // 加载种子的信号
  input        [LfsrDw-1:0]     seed_i,    // 种子
  input                         lfsr_en_i, // 使能信号
  input        [EntropyDw-1:0]  entropy_i, // 输入信号用来异或增加熵
  output logic [StateOutDw-1:0] state_o    // 输出信号
);
```

此电路是一个时序逻辑，当前状态`lfsr_q`，下一个状态`lfsr_d`，通过如下代码进行状态更新

```systemverilog
  if (64'(LfsrType) == 64'("GAL_XOR")) begin : gen_gal_xor
    // 基于XOR的线性反馈移位寄存器
    assign next_lfsr_state = LfsrDw'(entropy_i) ^ ({LfsrDw{lfsr_q[0]}} & coeffs) ^ (lfsr_q >> 1);
    assign lockup = ~(|lfsr_q);
    ...
  end else if (64'(LfsrType) == "FIB_XNOR") begin : gen_fib_xnor
    // 基于XNOR的线性反馈移位寄存器
    assign next_lfsr_state = LfsrDw'(entropy_i) ^ {lfsr_q[LfsrDw-2:0], ~(^(lfsr_q & coeffs))};
    assign lockup = &lfsr_q;
    ...
  end

  assign lfsr_d = (seed_en_i)           ? seed_i          : // 种子更新
                  (lfsr_en_i && lockup) ? DefaultSeed     : // 进入特殊状态，加载默认种子
                  (lfsr_en_i)           ? next_lfsr_state : // 使能时加载下一个值
                                          lfsr_q;           // 禁用时，不更新
  always_ff @(posedge clk_i or negedge rst_ni) begin : p_reg
    if (!rst_ni) begin
      lfsr_q <= DefaultSeed;
    end else begin
      lfsr_q <= lfsr_d;
    end
  end
  
  // 输出
  assign state_o  = lfsr_q[StateOutDw-1:0];
```

## prim_packer

此模块用于把数据打包，接口如下:

```systemverilog
module prim_packer #(
  parameter int InW  = 32, // 输入数据的最大宽度
  parameter int OutW = 32  // 输出数据的最大宽度
) (
  // 时钟和复位信号
  input clk_i ,
  input rst_ni,
  
  // 输入端口
  input                   valid_i,
  input        [InW-1:0]  data_i,
  input        [InW-1:0]  mask_i,
  output                  ready_o,

  // 输出端口
  output logic            valid_o,
  output logic [OutW-1:0] data_o,
  output logic [OutW-1:0] mask_o,
  input                   ready_i,

  input                   flush_i,     // 清除内部缓存
  output logic            flush_done_o // 缓存清除完成
);
```

输入口最多一次输入`InW`位，输出口最多一次输出`OutW`位。输入的数据可以长度可以小于`InW`，通过`mask_i`描述，`mask_i`中的`1`必须是连续的。

通过如下代码计算，输入数据的实际宽度：

```systemverilog
  // Computing next position
  always_comb begin
    // counting mask_i ones
    inmask_ones = '0;
    for (int i = 0 ; i < InW ; i++) begin
      inmask_ones = inmask_ones + mask_i[i];
    end
  end
```

通过如下代码计算最高位的`1`的位置：

```systemverilog
  // Leading one detector for mask_i
  always_comb begin
    lod_idx = 0;
    for (int i = InW-1; i >= 0 ; i--) begin
      if (mask_i[i] == 1'b1) begin
        lod_idx = i;
      end
    end
  end
```

其中，有一个`pos`寄存器用于记录内部缓存的位置。通过此变量，计算内部存储的记录和新输入的数据的连接。代码如下：

```systemverilog
  // 计算pos
  assign pos_next = (valid_i) ? pos + PtrW'(inmask_ones) : pos;  // pos always stays (% OutW)
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      pos <= '0;
    end else if (flush_ready) begin
      pos <= '0;
    end else if (ack_out) begin
      `ASSERT_I(pos_next_gte_outw_p, pos_next >= OutW)
      pos <= pos_next - OutW;  // 读出OutW位
    end else if (ack_in) begin
      pos <= pos_next;
    end
  end

  // 数据处理，连接内部记录和新输入的数据
  assign shiftl_data = (valid_i) ? Width'(data_i >> lod_idx) << pos : '0;
  assign shiftl_mask = (valid_i) ? Width'(mask_i >> lod_idx) << pos : '0;
  assign concat_data = {{(Width-MaxW){1'b0}}, stored_data & stored_mask} | (shiftl_data & shiftl_mask);
  assign concat_mask = {{(Width-MaxW){1'b0}}, stored_mask} | shiftl_mask;
```

## prim_flop_2sync

`prim_flop_2sync`，此模块实现了双缓冲同步触发器。此模块，内部为两个串联的触发器，输出信号晚输入信号一个周期。实现比较简单，代码如下：

```systemverilog
module prim_flop_2sync #(
  parameter int Width      = 16,              // 位宽
  parameter logic [Width-1:0] ResetValue = '0 // 复位值
) (
  input                    clk_i,    // 时钟
  input                    rst_ni,   // 复位信号
  input        [Width-1:0] d,        // 输入
  output logic [Width-1:0] q         // 输出
);

  logic [Width-1:0] intq;

  always_ff @(posedge clk_i or negedge rst_ni)
    if (!rst_ni) begin
      intq <= ResetValue;
      q    <= ResetValue;
    end else begin
      intq <= d;
      q    <= intq;
    end

endmodule
```

## prim_pulse_sync

`prim_pulse_sync`，此模块实现了一个脉冲同步器，把脉冲信号同步到不同的时钟域。接口如下：

```systemverilog
module prim_pulse_sync (
  input  logic clk_src_i,   // 输入脉冲信号的时钟信号
  input  logic rst_src_ni,  // 输入脉冲信号的复位信号
  input  logic src_pulse_i, // 输入脉冲信号

  input  logic clk_dst_i,   // 输出脉冲信号的时钟信号
  input  logic rst_dst_ni,  // 输出脉冲信号的复位信号
  output logic dst_pulse_o  // 输出脉冲信号
);
```

此模块首先在输入时钟域把输入脉冲转换为电平信号，代码如下：

```systemverilog
  always_ff @(posedge clk_src_i or negedge rst_src_ni) begin
    if (!rst_src_ni) begin
      src_level <= 1'b0;
    end else begin
      src_level <= src_level ^ src_pulse_i;
    end
  end
```

然后，通过双缓冲同步触发器，把电平信号同步到目标时钟域：

```systemverilog
  prim_flop_2sync #(.Width(1)) prim_flop_2sync (
    // source clock domain
    .d      (src_level),
    // destination clock domain
    .clk_i  (clk_dst_i),
    .rst_ni (rst_dst_ni),
    .q      (dst_level)
  );
```

最后，把电平信号转换为脉冲信号：

```systemverilog
  // delay dst_level by 1 cycle
  always_ff @(posedge clk_dst_i or negedge rst_dst_ni) begin
    if (!rst_dst_ni) begin
      dst_level_q <= 1'b0;
    end else begin
      dst_level_q <= dst_level;
    end
  end

  // edge detection
  assign dst_pulse_o = dst_level_q ^ dst_level;
```

## prim_subreg

`prim_subreg`，此模块实现了一个寄存器的子域，并实现了读写权限，模块接口如下：

```systemverilog
module prim_subreg #(
  parameter int            DW       = 32  ,  // 位宽
  parameter                SWACCESS = "RW",  // 权限 {RW, RO, WO, W1C, W1S, W0C, RC}
  parameter logic [DW-1:0] RESVAL   = '0     // 复位值
) (
  input clk_i,
  input rst_ni,

  // 连接到寄存器的写逻辑，受权限控制
  input          we,  // 写使能
  input [DW-1:0] wd,  // 数据

  // 连接到实际的寄存器硬件，不受权限控制
  input          de,  // 写使能
  input [DW-1:0] d,   // 数据

  // 输出信号
  output logic          qe, // 缓存写使能信号
  output logic [DW-1:0] q,  // 输出，连接到实际的寄存器硬件
  output logic [DW-1:0] qs  // 输出，连接到寄存器的读逻辑
);
```

此模块内有大量的代码负责处理权限，把它转换为写信号`wr_en`/`wr_data`


## prim_subreg_ext

`prim_subreg_ext`，此模块实现了一个简单的寄存器，没有硬件写端口，把写入内热直接输出。接口如下：

```systemverilog
module prim_subreg_ext #(
  parameter int unsigned DW = 32  // 位宽
) (
  input          re,  // 读使能
  input          we,  // 写使能
  input [DW-1:0] wd,  // 来自软件的数据

  input [DW-1:0] d,   // 来实际寄存器的值

  // output to HW and Reg Read
  output logic          qe,
  output logic          qre,
  output logic [DW-1:0] q,
  output logic [DW-1:0] qs
);

  assign qs = d;
  assign q = wd;
  assign qe = we;
  assign qre = re;

endmodule
```

## prim_rom_adv

`prim_rom_adv`此模块对rom封装，添加一个`rvalid`信号

```systemverilog
module prim_rom_adv #(
  // Parameters passed on the the ROM primitive.
  parameter  int Width       = 32,    // 数据线宽度 
  parameter  int Depth       = 2048,  // 地址数目
  parameter      MemInitFile = "",    // VMEM file to initialize the memory with

  parameter  int CfgW        = 8,     // WTC, RTC, etc

  localparam int Aw          = $clog2(Depth) // 计算出地址宽度
) (
  // 时钟和复位
  input  logic             clk_i,
  input  logic             rst_ni,

  input  logic             req_i,    // 数据请求
  input  logic [Aw-1:0]    addr_i,   // 地址输入
  output logic             rvalid_o, // 数据有效输出
  output logic [Width-1:0] rdata_o,  // 数据

  input        [CfgW-1:0]  cfg_i
);
```

## prim_sram_arbiter

此模块可以把单口sram转换为多口sram，数据请求通过仲裁器选择。接口如下：

```systemverilog
module prim_sram_arbiter #(
  parameter int N  = 4,         // 接口数
  parameter int SramDw = 32,    // sram的数据宽度
  parameter int SramAw = 12,    // sram的地址宽度
  parameter ArbiterImpl = "PPC" // 仲裁器类型
) (
  // 时钟和复位信号
  input clk_i,
  input rst_ni,
  
  // 拆分出来的N组sram接口
  input        [     N-1:0] req,             // 请求信号
  input        [SramAw-1:0] req_addr   [N],  // 地址
  input                     req_write  [N],  // 写使能
  input        [SramDw-1:0] req_wdata  [N],  // 要写入的数据
  output logic [     N-1:0] gnt,             // 请求响应

  output logic [     N-1:0] rsp_rvalid,      // 读取到的数据有效信号
  output logic [SramDw-1:0] rsp_rdata  [N],  // 读取到的数据
  output logic [       1:0] rsp_error  [N],  // 出错

  // 连接到单口sram
  output logic              sram_req,        // 请求信号
  output logic [SramAw-1:0] sram_addr,       // 地址
  output logic              sram_write,      // 写使能信号
  output logic [SramDw-1:0] sram_wdata,      // 要写入的数据
  input                     sram_rvalid,     // 读取到的数据有效信号
  input        [SramDw-1:0] sram_rdata,      // 读取到的数据
  input        [1:0]        sram_rerror      // 出错
);
```

## prim_secded

此模块实现了secded编码。在了解此编码之前，需要了解汉明码。汉明码用于防止数据传输保存时出错，它在k比特后附加m比特，用与校验数据并可以检测修复单个比特出错的情况。下面是一个(k=4,m=3)的C语言示例：

```c
#define GET_BIT(data,bit_num) \
  ((data >> bit_num) & 1)

#define SET_BIT(data,bit_num,val) \
  data = (data & (~(1 << bit_num)) | ((!!val) << bit_num))

// 编码
uint8_t encode(uint8_t in) {
  SET_BIT(in, 4, GET_BIT(in, 0) ^ GET_BIT(in, 1) ^ GET_BIT(in, 3));
  SET_BIT(in, 5, GET_BIT(in, 0) ^ GET_BIT(in, 2) ^ GET_BIT(in, 3));
  SET_BIT(in, 6, GET_BIT(in, 1) ^ GET_BIT(in, 2) ^ GET_BIT(in, 3));
  return in;
}

// 解码
int decode(uint8_t in, uint8_t *out) {
  uint8_t syndrome = 0;
  SET_BIT(syndrome, 0, GET_BIT(in, 4) ^ GET_BIT(in, 0) ^ GET_BIT(in, 1) ^ GET_BIT(in, 3));
  SET_BIT(syndrome, 1, GET_BIT(in, 5) ^ GET_BIT(in, 0) ^ GET_BIT(in, 2) ^ GET_BIT(in, 3));
  SET_BIT(syndrome, 2, GET_BIT(in, 6) ^ GET_BIT(in, 1) ^ GET_BIT(in, 2) ^ GET_BIT(in, 3));
  *out = 0;
  SET_BIT(*out, 0, (syndrome == 0x3) ^ GET_BIT(in, 0));
  SET_BIT(*out, 1, (syndrome == 0x5) ^ GET_BIT(in, 1));
  SET_BIT(*out, 2, (syndrome == 0x6) ^ GET_BIT(in, 2));
  SET_BIT(*out, 3, (syndrome == 0x7) ^ GET_BIT(in, 3));
}
```

此编码要求可以检测出一位出错的情况并修正，通过解码，我们知道添加的m比特数被转换为`syndrome`后：$0$标识成功；$2^n$标识添加的m比特的第n位出错；其他的值用于标识数据比特出错。所以要对k比特数据校验，需要添加m比特校验位，m需要满足以下条件：$2^m > m + k$。

编码方式如下：

- 列出m比特数可以的取值序列a1：$0,1,2,3,4,5,6,7...2^m$
- 去除$0$和$2^n$的序列a2：$3,5,7...$
- m比特中的第i位，由$d_k$比特异或得到，k满足$(a2[k] >> i) \& 1 == 1$

上面编码可以看出，用于修正错误的值没有规则，这样不能确定是单比特出错还是多比特出错，为了使编码出的值有规律一般会把m比特多一位，并使`syndrome`中有计数个`1`,这就是secded算法。

prim实现了三组此算法：

- prim_secded_28_22_enc / prim_secded_28_22_dnc，(k=22, m=6)情况下的编码解码
- prim_secded_39_32_enc / prim_secded_39_32_dnc，(k=32, m=7)情况下的编码解码
- prim_secded_72_64_enc / prim_secded_72_64_dnc，(k=64, m=8)情况下的编码解码

下面介绍编码解码接口以(k=22, m=6)为例，编码接口：

```systemverilog
module prim_secded_28_22_enc (
  input        [21:0] in, // 数据输入
  output logic [27:0] out // 数据输出
);
```

解码接口：

```systemverilog
module prim_secded_28_22_dec (
  input        [27:0] in,           // 数据输入
  output logic [21:0] d_o,          // 数据输出
  output logic [5:0] syndrome_o,    // 输出译码后的m位的值
  output logic [1:0] err_o          // 错误信息：bit0，单比特出错；bit1，双比特出错
);
```

## prim ram adv

带校验的ram，校验方式可以是奇偶校验或secded或者没有校验。模块接口如下：

```systemverilog
module prim_ram_1p_adv #(
  parameter  int Depth                = 512, // 地址个数
  parameter  int Width                = 32,  // 数据宽度
  parameter  int DataBitsPerMask      = 1,  // Number of data bits per bit of write mask
  parameter  int CfgW                 = 8,  // WTC, RTC, etc
  parameter      MemInitFile          = "", // VMEM file to initialize the memory with

  // Configurations
  parameter  bit EnableECC            = 0, // 使能ecc校验，校验方式为上面提到的secded
  parameter  bit EnableParity         = 0, // 使能奇偶校验
  parameter  bit EnableInputPipeline  = 0, // Adds an input register (read latency +1)
  parameter  bit EnableOutputPipeline = 0, // Adds an output register (read latency +1)

  localparam int Aw                   = prim_util_pkg::vbits(Depth)
) (
  input clk_i,
  input rst_ni,

  input                      req_i,
  input                      write_i,
  input        [Aw-1:0]      addr_i,
  input        [Width-1:0]   wdata_i,
  input        [Width-1:0]   wmask_i,
  output logic [Width-1:0]   rdata_o,
  output logic               rvalid_o, // read response (rdata_o) is valid
  output logic [1:0]         rerror_o, // Bit1: Uncorrectable, Bit0: Correctable

  // config
  input [CfgW-1:0] cfg_i
);
```

此模块内部创建了一个位宽可以同时保存原始数据和校验位的内存。位宽通过如下代码计算：

```systemverilog
  // 计算校验需要的位数
  localparam int ParWidth  = (EnableParity) ? Width/8 : // 奇偶校验每个字（8比特）节需要一个比特校验位
                             (!EnableECC)   ? 0 :       // 没有校验
                             (Width <=   4) ? 4 :       // 使用ECC校验，位宽小于等于  4时，需要4个比特校验位
                             (Width <=  11) ? 5 :       // 使用ECC校验，位宽小于等于 11时，需要5个比特校验位
                             (Width <=  26) ? 6 :       // 使用ECC校验，位宽小于等于 26时，需要6个比特校验位
                             (Width <=  57) ? 7 :       // 使用ECC校验，位宽小于等于 57时，需要7个比特校验位
                             (Width <= 120) ? 8 : 8 ;   // 使用ECC校验，位宽小于等于120时，需要8个比特校验位
  localparam int TotalWidth = Width + ParWidth;         // 计算保存一个数据和一个校验信息所需要的位宽
```

写操作时实际输入数据为`wdata_i`，添加校验信息后为`wdata_d`。读数据时从实际的内存读取到的值为`rdata_sram`，解码去除掉校验信息后为`rdata_d`。通过ECC校验代码如下：

```systemverilog
      prim_secded_39_32_enc u_enc (.in(wdata_i), .out(wdata_d));
      prim_secded_39_32_dec u_dec (
        .in         (rdata_sram),
        .d_o        (rdata_d[0+:Width]),
        .syndrome_o ( ),
        .err_o      (rerror_d)
      );
```

系统还实现了两个读写端口的带校验信息的sram，代码位于`hw/ip/prim/rtl/prim_ram_2p_adv.sv`

## prim_ram_1p_scr

此模块实现了一个内存加密算法，它会对写入的地址进行混淆，并对写入的数据进行加密，并且添加ECC校验防止数据出错。接口如下：

```systemverilog
module prim_ram_1p_scr #(
  parameter  int Depth                = 512, // Needs to be a power of 2 if NumAddrScrRounds > 0.
  parameter  int Width                = 256, // Needs to be Byte aligned for parity
  parameter  int DataBitsPerMask      = 8,   // Currently only 8 is supported
  parameter  int CfgW                 = 8,   // WTC, RTC, etc

  // Scrambling parameters. Note that this needs to be low-latency, hence we have to keep the
  // amount of cipher rounds low. PRINCE has 5 half rounds in its original form, which corresponds
  // to 2*5 + 1 effective rounds. Setting this to 2 halves this to approximately 5 effective rounds.
  parameter  int NumPrinceRoundsHalf  = 2,   // Number of PRINCE half rounds, can be [1..5]
  // Number of extra intra-Byte diffusion rounds. Setting this to 0 disables intra-Byte diffusion.
  parameter  int NumByteScrRounds     = 2,
  // Number of address scrambling rounds. Setting this to 0 disables address scrambling.
  parameter  int NumAddrScrRounds     = 2,
  // If set to 1, the same 64bit key stream is replicated if the data port is wider than 64bit.
  // If set to 0, the cipher primitive is replicated, and together with a wider nonce input,
  // a unique keystream is generated for the full data width.
  parameter  bit ReplicateKeyStream   = 1'b0,

  // Derived parameters
  localparam int AddrWidth            = prim_util_pkg::vbits(Depth),
  // Depending on the data width, we need to instantiate multiple parallel cipher primitives to
  // create a keystream that is wide enough (PRINCE has a block size of 64bit)
  localparam int NumParScr            = (ReplicateKeyStream) ? 1 : (Width + 63) / 64,
  localparam int NumParKeystr         = (ReplicateKeyStream) ? (Width + 63) / 64 : 1,
  // This is given by the PRINCE cipher primitive. All parallel cipher modules
  // use the same key, but they use a different IV
  localparam int DataKeyWidth         = 128,
  // Each scrambling primitive requires a 64bit IV composed of {nonce, address}
  localparam int DataNonceWidth       = (64-AddrWidth) * NumParScr
) (
  // 时钟复位
  input                             clk_i,
  input                             rst_ni,

  input        [DataKeyWidth-1:0]   key_i,         // 数据加密的密码
  input        [DataNonceWidth-1:0] data_nonce_i,  // 数据噪音
  input        [AddrWidth-1:0]      addr_nonce_i,  // 地址混淆噪音

  // sram接口
  input                             req_i,
  input                             write_i,
  input        [AddrWidth-1:0]      addr_i,
  input        [Width-1:0]          wdata_i,
  input        [Width-1:0]          wmask_i,  // Needs to be Byte-aligned for parity
  output logic [Width-1:0]          rdata_o,
  output logic                      rvalid_o, // Read response (rdata_o) is valid
  output logic [1:0]                rerror_o, // Bit1: Uncorrectable, Bit0: Correctable

  // config
  input [CfgW-1:0]                  cfg_i
);
```

地址混淆，使用地址噪音`addr_nonce_i`通过`prim_subst_perm`算法对地址混淆，代码如下

```systemverilog
  prim_subst_perm #(
    .DataWidth ( AddrWidth        ),
    .NumRounds ( NumAddrScrRounds ),
    .Decrypt   ( 0                )
  ) i_prim_subst_perm (
    .data_i ( addr_mux ),
    .key_i  ( addr_nonce_i ),
    .data_o ( addr_scr     )
  );
```

数据加密。使用加密密钥`key_i`和数据噪音`data_nonce_i`以及地址`addr_i`通过加密算法`prim_prince`计算出一个用于加密的中间变量`keystream_repl`。然后通过此中间变量与要写入的数据`wdata_q`异或，异或后在通过`prim_subst_perm`算法混淆，最终获得要写入的数据，相关代码如下：

```systemverilog
logic [NumParScr*64-1:0] keystream;
  for (genvar k = 0; k < NumParScr; k++) begin : gen_par_scr
    prim_prince #(
      .DataWidth      (64),
      .KeyWidth       (128),
      .NumRoundsHalf  (NumPrinceRoundsHalf),
      .UseOldKeySched (1'b0),
      .HalfwayDataReg (1'b1), // instantiate a register halfway in the primitive
      .HalfwayKeyReg  (1'b0)  // no need to instantiate a key register as the key remains static
    ) u_prim_prince (
      .clk_i,
      .rst_ni,
      .valid_i ( req_i ),
      // The IV is composed of a nonce and the row address
      .data_i  ( {data_nonce_i[k * (64 - AddrWidth) +: (64 - AddrWidth)], addr_i} ),
      // All parallel scramblers use the same key
      .key_i,
      // Since we operate in counter mode, this can always be set to encryption mode
      .dec_i   ( 1'b0 ),
      // Output keystream to be XOR'ed
      .data_o  ( keystream[k * 64 +: 64] ),
      .valid_o ( )
    );
  end

  // Replicate keystream if needed
  logic [Width-1:0] keystream_repl;
  assign keystream_repl = Width'({NumParKeystr{keystream}});

  logic [Width-1:0] wdata_scr_d, wdata_scr_q;
  for (genvar k = 0; k < Width/8; k++) begin : gen_diffuse_wdata
    // Apply the keystream first
    logic [7:0] wdata_xor;
    assign wdata_xor = wdata_q[k*8 +: 8] ^ keystream_repl[k*8 +: 8];

    // Byte aligned diffusion using a substitution / permutation network
    prim_subst_perm #(
      .DataWidth ( 8                ),
      .NumRounds ( NumByteScrRounds ),
      .Decrypt   ( 0                )
    ) i_prim_subst_perm (
      .data_i ( wdata_xor             ),
      .key_i  ( '0                    ),
      .data_o ( wdata_scr_d[k*8 +: 8] )
    );
  end
```