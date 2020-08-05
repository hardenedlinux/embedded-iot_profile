# 概要

此文档用于介绍opentitan的外设的实现，作代码分析。opentitan使用tlul总线，所有的外设都挂载到此总线上，所以外设一般都具有tlul总线接口，外设实现一些具体功能需要一些对外的引脚，如果设备提供可以触发中断还有一些中断信号。

## 公共相似代码

由于外设都需要连接到tlul总线上，并对cpu提供一些寄存器接口。所以需要把tlul总线转换为寄存器接口，代码如下：

```systemverilog
  tlul_adapter_reg #(
    .RegAw(AW), // 地址宽度
    .RegDw(DW)  // 数据宽度
  ) u_reg_if (
    // 时钟和复位
    .clk_i,
    .rst_ni,

     // tlul总线接口
    .tl_i (tl_reg_h2d),
    .tl_o (tl_reg_d2h),

    // 寄存器接口
    .we_o    (reg_we),     // 写使能
    .re_o    (reg_re),     // 读使能
    .addr_o  (reg_addr),   // 地址
    .wdata_o (reg_wdata),  // 要写入的数据
    .be_o    (reg_be),     // 字节使能信号
    .rdata_i (reg_rdata),  // 读出的数据
    .error_i (reg_error)   // 错误信号
  );
```

对于一个外设有一系列的寄存器，地址可以连续也可以不连续，通过如下代码把地址信号转换为地址命中信号：

```systemverilog
  logic [11:0] addr_hit;
  always_comb begin
    addr_hit = '0;
    addr_hit[ 0] = (reg_addr == UART_INTR_STATE_OFFSET);
    addr_hit[ 1] = (reg_addr == UART_INTR_ENABLE_OFFSET);
    addr_hit[ 2] = (reg_addr == UART_INTR_TEST_OFFSET);
    addr_hit[ 3] = (reg_addr == UART_CTRL_OFFSET);
    addr_hit[ 4] = (reg_addr == UART_STATUS_OFFSET);
    addr_hit[ 5] = (reg_addr == UART_RDATA_OFFSET);
    addr_hit[ 6] = (reg_addr == UART_WDATA_OFFSET);
    addr_hit[ 7] = (reg_addr == UART_FIFO_CTRL_OFFSET);
    addr_hit[ 8] = (reg_addr == UART_FIFO_STATUS_OFFSET);
    addr_hit[ 9] = (reg_addr == UART_OVRD_OFFSET);
    addr_hit[10] = (reg_addr == UART_VAL_OFFSET);
    addr_hit[11] = (reg_addr == UART_TIMEOUT_CTRL_OFFSET);
  end
```

通过如下代码检测地址不命中的情况，即访问了不存在的地址。代码如下：

```systemverilog
  assign addrmiss = (reg_re || reg_we) ? ~|addr_hit : 1'b0 ;
```

riscv总线宽度为32/64/128位，为了此外设可以在各个位宽的平台下运行，寄存器设计一般小于32位，寄存器操作一般需要一次操着整个字，不容许进行子字节操作，通过如下代码实现。

```systemverilog
  // UART_PERMIT是预定于的数据，标识寄存器的宽度，可以是以下的值：
  //     0b0001 -> 8 位寄存器
  //     0b0011 -> 16位寄存器
  //     0b1111 -> 32位寄存器
  always_comb begin
    wr_err = 1'b0;
    if (addr_hit[ 0] && reg_we && (UART_PERMIT[ 0] != (UART_PERMIT[ 0] & reg_be))) wr_err = 1'b1 ;
    if (addr_hit[ 1] && reg_we && (UART_PERMIT[ 1] != (UART_PERMIT[ 1] & reg_be))) wr_err = 1'b1 ;
    if (addr_hit[ 2] && reg_we && (UART_PERMIT[ 2] != (UART_PERMIT[ 2] & reg_be))) wr_err = 1'b1 ;
    if (addr_hit[ 3] && reg_we && (UART_PERMIT[ 3] != (UART_PERMIT[ 3] & reg_be))) wr_err = 1'b1 ;
    if (addr_hit[ 4] && reg_we && (UART_PERMIT[ 4] != (UART_PERMIT[ 4] & reg_be))) wr_err = 1'b1 ;
    if (addr_hit[ 5] && reg_we && (UART_PERMIT[ 5] != (UART_PERMIT[ 5] & reg_be))) wr_err = 1'b1 ;
    if (addr_hit[ 6] && reg_we && (UART_PERMIT[ 6] != (UART_PERMIT[ 6] & reg_be))) wr_err = 1'b1 ;
    if (addr_hit[ 7] && reg_we && (UART_PERMIT[ 7] != (UART_PERMIT[ 7] & reg_be))) wr_err = 1'b1 ;
    if (addr_hit[ 8] && reg_we && (UART_PERMIT[ 8] != (UART_PERMIT[ 8] & reg_be))) wr_err = 1'b1 ;
    if (addr_hit[ 9] && reg_we && (UART_PERMIT[ 9] != (UART_PERMIT[ 9] & reg_be))) wr_err = 1'b1 ;
    if (addr_hit[10] && reg_we && (UART_PERMIT[10] != (UART_PERMIT[10] & reg_be))) wr_err = 1'b1 ;
    if (addr_hit[11] && reg_we && (UART_PERMIT[11] != (UART_PERMIT[11] & reg_be))) wr_err = 1'b1 ;
  end
```

当地址不命中或者访问不完整的寄存器需要报错，通过如下代码实现：

```systemverilog
// devmode_i是配置位，用于配置不存在的寄存器是否可以访问
assign reg_error = (devmode_i & addrmiss) | wr_err ;
```

具体的寄存器位域，通过prim提供的`prim_subreg` / `prim_subreg_ext`来实现。其中，`prim_subreg`是标准实现，提供软硬件读写接口，以及权限控制（RW/RO/WO/W1C/W1S/W0C/RC）。其中`prim_subreg_ext`，对应简单的配置用寄存器，即硬件不会对寄存器进行写操作。

系统一般会定义两个结构体`hw2reg`/`reg2hw`。这两个寄存器，真实的连接到寄存器实现的硬件接口上，用于和具体的硬件交互。

寄存器的位于有各种塑性，对应具体的位需要单独的读写代码，读写代码如下：

```systemverilog
  // 单比特的位域，写代码
  assign intr_state_tx_watermark_we = addr_hit[0] & reg_we & ~wr_err; 
  assign intr_state_tx_watermark_wd = reg_wdata[0];
  
  // 多比特的位域，写代码
  assign fifo_ctrl_rxilvl_we = addr_hit[7] & reg_we & ~wr_err;
  assign fifo_ctrl_rxilvl_wd = reg_wdata[4:2];

  // 读使能信号，由prim_subreg/prim_subreg_ext输出读出的内容
  assign status_txfull_re = addr_hit[4] && reg_re;
  ...
  
  // 整合读出的内容
  always_comb begin
    reg_rdata_next = '0;
    unique case (1'b1)
      addr_hit[0]: begin
        reg_rdata_next[0] = intr_state_tx_watermark_qs;
        reg_rdata_next[1] = intr_state_rx_watermark_qs;
        reg_rdata_next[2] = intr_state_tx_empty_qs;
        reg_rdata_next[3] = intr_state_rx_overflow_qs;
        reg_rdata_next[4] = intr_state_rx_frame_err_qs;
        reg_rdata_next[5] = intr_state_rx_break_err_qs;
        reg_rdata_next[6] = intr_state_rx_timeout_qs;
        reg_rdata_next[7] = intr_state_rx_parity_err_qs;
      end
    ...
    endcase
  end
```

有些外设会有一小段内存映射到cpu内存空间，这时就需要`tlul_adapter_sram`模块，把tlul接口转换为sram接口代码如下：

```systemverilog
  tlul_adapter_sram #(
    .SramAw      (SramAw),
    .SramDw      (SramDw),
    .Outstanding (1),
    .ByteAccess  (0)
  ) u_tlul2sram (
    .clk_i,
    .rst_ni,

    .tl_i (tl_sram_h2d [0]),
    .tl_o (tl_sram_d2h [0]),

    .req_o    (mem_a_req),
    .gnt_i    (mem_a_req),  //Always grant when request
    .we_o     (mem_a_write),
    .addr_o   (mem_a_addr),
    .wdata_o  (mem_a_wdata),
    .wmask_o  (),           // Not used
    .rdata_i  (mem_a_rdata),
    .rvalid_i (mem_a_rvalid),
    .rerror_i (mem_a_rerror)
  );
```

一般情况下这段内存用于和cpu交互，外设也需要访问这段内存，就需要使用双端口内存，代码如下：

```systemverilog
  prim_ram_2p_adv #(
    .Depth (512),
    .Width (SramDw),    // 32 x 512 --> 2kB
    .DataBitsPerMask (1),
    .CfgW  (8),

    .EnableECC           (1),
    .EnableParity        (0),
    .EnableInputPipeline (0),
    .EnableOutputPipeline(0)
  ) u_memory_2p (
    // 时钟和复位
    .clk_i,
    .rst_ni,
    
    // 端口a用于连接cpu
    .a_req_i    (mem_a_req),
    .a_write_i  (mem_a_write),
    .a_addr_i   (mem_a_addr),
    .a_wdata_i  (mem_a_wdata),
    .a_wmask_i  ({SramDw{1'b1}}),
    .a_rvalid_o (mem_a_rvalid),
    .a_rdata_o  (mem_a_rdata),
    .a_rerror_o (mem_a_rerror),

     // 端口b用于连接外设
    .b_req_i    (mem_b_req),
    .b_write_i  (mem_b_write),
    .b_addr_i   (mem_b_addr),
    .b_wdata_i  (mem_b_wdata),
    .b_wmask_i  ({SramDw{1'b1}}),
    .b_rvalid_o (mem_b_rvalid),
    .b_rdata_o  (mem_b_rdata),
    .b_rerror_o (mem_b_rerror),

    .cfg_i      ('0)
  );
```

## UART

代码位于`hw/ip/uart/rtl`中。其中，`hw/ip/uart/rtl/uart_reg_top.sv`实现了与uart相关的控制状态寄存器。接口如下：

```systemverilog
module uart_reg_top (
  // 时钟和复位
  input clk_i,
  input rst_ni,

  // tlul总线接口
  input  tlul_pkg::tl_h2d_t tl_i,
  output tlul_pkg::tl_d2h_t tl_o,
  
  // 连接硬件的接口
  output uart_reg_pkg::uart_reg2hw_t reg2hw, // Write
  input  uart_reg_pkg::uart_hw2reg_t hw2reg, // Read

  // 配置
  input devmode_i // 如果为1,写不存在的寄存器将触发错误
);
```

其中，`hw/ip/uart/rtl/uart_tx.sv`实现了uart的发送逻辑，接口如下：

```systemverilog
module uart_tx (
  // 时钟和复位
  input               clk_i,
  input               rst_ni,

  // 控制信号
  input               tx_enable,     // 使能发送
  input               tick_baud_x16, // 使能时钟16分频，此信号无效时将停止数据发送
  input  logic        parity_enable, // 奇偶校验位使能

  // 写端口，写入要发生的数据
  input               wr,            // 写使能信号
  input  logic        wr_parity,     // 校验位
  input   [7:0]       wr_data,       // 要发送的数据
  
  // 输出设备空闲
  output              idle,

  // uart发生端口
  output logic        tx
);
```

对时钟16分配的代码，通过一个计数器实现，代码如下：

```systemverilog
  logic    [3:0] baud_div_q;
  logic          tick_baud_q;

  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      baud_div_q  <= 4'h0;
      tick_baud_q <= 1'b0;
    end else if (tick_baud_x16) begin
      {tick_baud_q, baud_div_q} <= {1'b0,baud_div_q} + 5'h1;
    end else begin
      tick_baud_q <= 1'b0;
    end
  end
```

数据发送代码如下：

```systemverilog
 always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      bit_cnt_q <= 4'h0;
      sreg_q    <= 11'h7ff;
      tx_q      <= 1'b1;
    end else begin
      bit_cnt_q <= bit_cnt_d; // 记录还有多少比特需要发送
      sreg_q    <= sreg_d;    // 要发送的比特
      tx_q      <= tx_d;      // 需要被发送的值
    end
  end

  always_comb begin
    if (!tx_enable) begin // 发生失能
      bit_cnt_d = 4'h0;
      sreg_d    = 11'h7ff;
      tx_d      = 1'b1;   // tx引脚保持高电平
    end else begin        //发生使能
      bit_cnt_d = bit_cnt_q;
      sreg_d    = sreg_q;
      tx_d      = tx_q;
      if (wr) begin // 写入要发送的值
        sreg_d    = {1'b1, (parity_enable ? wr_parity : 1'b1), wr_data, 1'b0}; //起始位为0,结束位为1，不发生校验位时蛇校验位为1充当结束位
        bit_cnt_d = (parity_enable ? 4'd11 : 4'd10); // 计算要发送的长度
      end else if (tick_baud_q && (bit_cnt_q != 4'h0)) begin
        sreg_d    = {1'b1, sreg_q[10:1]};// 右移
        tx_d      = sreg_q[0];           // 发生最低位
        bit_cnt_d = bit_cnt_q - 4'h1;    // 计数器更新
      end
    end
  end
```

其中，`hw/ip/uart/rtl/uart_rx.sv`实现了uart的接受逻辑，接口如下：

```systemverilog
module uart_rx (
  // 时钟和复位
  input           clk_i,
  input           rst_ni,

  // 控制信号
  input           rx_enable,      // 接受使能
  input           tick_baud_x16,  // 使能时钟16分频，此信号无效时将停止数据接受
  input           parity_enable,  // 奇偶校验使能
  input           parity_odd,     // 奇校验

  // 输出端口
  output logic    tick_baud,      // 对时钟16分频后输出信号
  output logic    rx_valid,       // 输出有效，接受到一个字节
  output [7:0]    rx_data,        // uart接受到的数据
  output logic    idle,           // uart 空闲信号
  output          frame_err,      // 帧错误
  output          rx_parity_err,  // 校验位错误

  input           rx              // uart接受端口
);
```

主体代码如下：

```systemverilog
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin // 复位
      sreg_q      <= 11'h0;
      bit_cnt_q   <= 4'h0;
      baud_div_q  <= 4'h0;
      tick_baud_q <= 1'b0;
      idle_q      <= 1'b1;
    end else begin     // 工作时
      sreg_q      <= sreg_d;     // 接受数据的寄存器
      bit_cnt_q   <= bit_cnt_d;  // 计数器，计数接受到的比特数
      baud_div_q  <= baud_div_d; // 对时钟16分频的计算器
      tick_baud_q <= tick_baud_d;// 时钟分频有效有效信号
      idle_q      <= idle_d;     // 设备空闲
    end
  end

  always_comb begin
    if (!rx_enable) begin  // 接受失能
      sreg_d      = 11'h0;
      bit_cnt_d   = 4'h0;
      baud_div_d  = 4'h0;
      tick_baud_d = 1'b0;
      idle_d      = 1'b1;  // 输出设备空闲
    end else begin
      tick_baud_d = 1'b0;
      sreg_d      = sreg_q;
      bit_cnt_d   = bit_cnt_q;
      baud_div_d  = baud_div_q;
      idle_d      = idle_q;
      if (tick_baud_x16) begin
        {tick_baud_d, baud_div_d} = {1'b0,baud_div_q} + 5'h1; // 计数器溢出时触发一次发送
      end

      if (idle_q && !rx) begin
        // 空闲状态下，接受到一个低电平触发接受
        baud_div_d  = 4'd8;
        tick_baud_d = 1'b0;
        bit_cnt_d   = (parity_enable ? 4'd11 : 4'd10); // 计算要接受的比特数
        sreg_d      = 11'h0; // 接收寄存器清0
        idle_d      = 1'b0;  // 设备转为忙碌
      end else if (!idle_q && tick_baud_q) begin // 接受其他比特
        if ((bit_cnt_q == (parity_enable ? 4'd11 : 4'd10)) && rx) begin
          // must have been a glitch on the input, start bit is not set
          // in the middle of the bit time, abort
          idle_d    = 1'b1;
          bit_cnt_d = 4'h0;
        end else begin
          sreg_d    = {rx, sreg_q[10:1]}; // 右移，把接受到的值放到最高位
          bit_cnt_d = bit_cnt_q - 4'h1;   // 计数器减1
          idle_d    = (bit_cnt_q == 4'h1);// 接受到最后以比特后设备转为空闲
        end
      end
    end
  end

  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) rx_valid_q <= 1'b0;
    else         rx_valid_q <= tick_baud_q & (bit_cnt_q == 4'h1); // 接受到最后以比特后，接受到有效数据
  end

  assign rx_valid      = rx_valid_q; // 输出数据有效
  assign rx_data       = parity_enable ? sreg_q[8:1] : sreg_q[9:2]; // 计算接受到的数据
  assign frame_err     = rx_valid_q & ~sreg_q[10];
  assign rx_parity_err = parity_enable & rx_valid_q &
                         (^{sreg_q[9:1],parity_odd}); // 校验位计算
```

其中，`hw/ip/uart/rtl/uart_core.sv`整合了`uart_tx`/`uart_rx`，形成了一个完整的uart。接口如下：

```systemverilog
module uart_core (
  // 时钟和复位
  input                  clk_i,
  input                  rst_ni,

  // 寄存器接口
  input  uart_reg_pkg::uart_reg2hw_t reg2hw,
  output uart_reg_pkg::uart_hw2reg_t hw2reg,

  // uart的发生接受端口
  input                  rx,
  output logic           tx,

  // 中断信号
  output logic           intr_tx_watermark_o,
  output logic           intr_rx_watermark_o,
  output logic           intr_tx_empty_o,
  output logic           intr_rx_overflow_o,
  output logic           intr_rx_frame_err_o,
  output logic           intr_rx_break_err_o,
  output logic           intr_rx_timeout_o,
  output logic           intr_rx_parity_err_o
);
```

在理解此部分代码之前，请先参考文档[UART HWIP Technical Specification](https://docs.opentitan.org/hw/ip/uart/doc/index.html)。

opentitan uart具有接受和发生缓存，并且通过寄存器反映缓存的使用情况，以及缓存满或者空后触发中断

```systemverilog
  // 接受缓存
  prim_fifo_sync #(
    .Width (8),
    .Pass  (1'b0),
    .Depth (32)
  ) u_uart_rxfifo (
    .clk_i,
    .rst_ni,
    .clr_i  (uart_fifo_rxrst),
    .wvalid (rx_fifo_wvalid),
    .wready (rx_fifo_wready),
    .wdata  (rx_fifo_data),
    .depth  (rx_fifo_depth),
    .rvalid (rx_fifo_rvalid),
    .rready (reg2hw.rdata.re),
    .rdata  (uart_rdata)
  );
   
  // 发生缓存
  prim_fifo_sync #(
    .Width(8),
    .Pass (1'b0),
    .Depth(32)
  ) u_uart_txfifo (
    .clk_i,
    .rst_ni,
    .clr_i  (uart_fifo_txrst),
    .wvalid (reg2hw.wdata.qe),
    .wready (tx_fifo_wready),
    .wdata  (reg2hw.wdata.q),
    .depth  (tx_fifo_depth),
    .rvalid (tx_fifo_rvalid),
    .rready (tx_fifo_rready),
    .rdata  (tx_fifo_data)
  );

  // 接受缓存满后触发中断信号，让软件拿走数据
  always_comb begin
    unique case(uart_fifo_rxilvl)
      3'h0:    rx_watermark_d = (rx_fifo_depth >= 6'd1);
      3'h1:    rx_watermark_d = (rx_fifo_depth >= 6'd4);
      3'h2:    rx_watermark_d = (rx_fifo_depth >= 6'd8);
      3'h3:    rx_watermark_d = (rx_fifo_depth >= 6'd16);
      3'h4:    rx_watermark_d = (rx_fifo_depth >= 6'd30);
      default: rx_watermark_d = 1'b0;
    endcase
  end
  
  // 发生缓存空后触发中断，让软件补充要发生的数据
  always_comb begin
    unique case(uart_fifo_txilvl)
      2'h0:    tx_watermark_d = (tx_fifo_depth < 6'd2);
      2'h1:    tx_watermark_d = (tx_fifo_depth < 6'd4);
      2'h2:    tx_watermark_d = (tx_fifo_depth < 6'd8);
      default: tx_watermark_d = (tx_fifo_depth < 6'd16);
    endcase
  end

  // 缓存情况输出到寄存器
  assign hw2reg.fifo_status.rxlvl.d  = rx_fifo_depth;
  assign hw2reg.fifo_status.txlvl.d  = tx_fifo_depth;
  assign hw2reg.status.rxempty.d     = ~rx_fifo_rvalid;
  assign hw2reg.status.rxidle.d      = rx_uart_idle;
  assign hw2reg.status.txidle.d      = tx_uart_idle & ~tx_fifo_rvalid;
  assign hw2reg.status.txempty.d     = ~tx_fifo_rvalid;
  assign hw2reg.status.rxfull.d      = ~rx_fifo_wready;
  assign hw2reg.status.txfull.d      = ~tx_fifo_wready;
```

opentitan uart定义了一个特别的中断，当接收到指定个数的全零(起始位、数据位和结束位全为0)时触发中断，代码如下：

```systemverilog
  assign not_allzero_char = rx_valid & (~event_rx_frame_err | (rx_fifo_data != 8'h0));
  assign allzero_err = event_rx_frame_err & (rx_fifo_data == 8'h0);

  // 对接受到全零的情况计数
  assign allzero_cnt_d = (break_st_q == BRK_WAIT || not_allzero_char) ? 5'h0 :
                          //allzero_cnt_q[4] never be 1b without break_st_q as BRK_WAIT
                          //allzero_cnt_q[4] ? allzero_cnt_q :
                          allzero_err ? allzero_cnt_q + 5'd1 :
                          allzero_cnt_q;

  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni)        allzero_cnt_q <= '0;
    else if (rx_enable) allzero_cnt_q <= allzero_cnt_d; // 接受使能时计数
  end

  // break_err edges in same cycle as event_rx_frame_err edges ; that way the
  // reset-on-read works the same way for break and frame error interrupts.

  always_comb begin
    unique case (reg2hw.ctrl.rxblvl.q) // 根据配置触发break_err
      2'h0:    break_err = allzero_cnt_d >= 5'd2;
      2'h1:    break_err = allzero_cnt_d >= 5'd4;
      2'h2:    break_err = allzero_cnt_d >= 5'd8;
      default: break_err = allzero_cnt_d >= 5'd16;
    endcase
  end

  // 维护一个状态
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) break_st_q <= BRK_CHK;
    else begin
      unique case (break_st_q)
        BRK_CHK: begin
          if (event_rx_break_err) break_st_q <= BRK_WAIT;
        end

        BRK_WAIT: begin
          if (rx_in) break_st_q <= BRK_CHK;
        end

        default: begin
          break_st_q <= BRK_CHK;
        end
      endcase
    end
  end
  assign event_rx_break_err = break_err & (break_st_q == BRK_CHK);
```

opentitan uart提供接受超时，接受到多少个比特还没有结束当前字节接受，相关代码如下：

```systemverilog
  // 来自寄存器的配置信息
  assign uart_rxto_en  = reg2hw.timeout_ctrl.en.q;
  assign uart_rxto_val = reg2hw.timeout_ctrl.val.q;

  // 接受缓存发生变化
  assign rx_fifo_depth_changed = (rx_fifo_depth != rx_fifo_depth_prev_q);

  // 计数器
  assign rx_timeout_count_d =
              // don't count if timeout feature not enabled ;
              // will never reach timeout val + lower power
              (uart_rxto_en == 1'b0)              ? 24'd0 :
              // reset count if timeout interrupt is set
              event_rx_timeout                    ? 24'd0 :
              // reset count upon change in fifo level: covers both read and receiving a new byte
              rx_fifo_depth_changed               ? 24'd0 :
              // reset count if no bytes are pending
              (rx_fifo_depth == 5'd0)             ? 24'd0 :
              // stop the count at timeout value (this will set the interrupt)
              //   Removed below line as when the timeout reaches the value,
              //   event occured, and timeout value reset to 0h.
              //(rx_timeout_count_q == uart_rxto_val) ? rx_timeout_count_q :
              // increment if at rx baud tick
              rx_tick_baud                        ? (rx_timeout_count_q + 24'd1) :
              rx_timeout_count_q;

  // 触发异常信号
  assign event_rx_timeout = (rx_timeout_count_q == uart_rxto_val) & uart_rxto_en;
```

## GPIO

在理解此模块代码之前，请参考文档[GPIO HWIP Technical Specification](https://docs.opentitan.org/hw/ip/gpio/doc/)

模块接口如下：

```systemverilog
module gpio (
  // 时钟和复位信号
  input clk_i,
  input rst_ni,

  // 总线接口
  input  tlul_pkg::tl_h2d_t tl_i,
  output tlul_pkg::tl_d2h_t tl_o,

  // GPIO的物理口
  input        [31:0] cio_gpio_i,    // 输入
  output logic [31:0] cio_gpio_o,    // 输出
  output logic [31:0] cio_gpio_en_o, // 控制信号

  // 异常信号
  output logic [31:0] intr_gpio_o
);
```

opentitan提供对gpio输入信号滤波去除杂波，代码如下：

```systemverilog
  logic [31:0] data_in_d;
  for (genvar i = 0 ; i < 32 ; i++) begin : gen_filter
    prim_filter_ctr #(.Cycles(16)) filter (
      .clk_i,
      .rst_ni,
      .enable_i(reg2hw.ctrl_en_input_filter.q[i]), // 来自寄存器的配置信息
      .filter_i(cio_gpio_i[i]), // 物理引脚上的信号
      .filter_o(data_in_d[i])   // 滤波后的信号
    );
  end
```

对于GPIO的输出和输出使能，提供两种操着模式：直接输出，mask输出。直接输出，写入一个32比特的寄存器。mask输出，只修改mask为1的位。相关代码如下：

```systemverilog
  // GPIO_OUT
  assign cio_gpio_o                     = cio_gpio_q;
  
  // 写入寄存器
  assign hw2reg.direct_out.d            = cio_gpio_q;
  assign hw2reg.masked_out_upper.data.d = cio_gpio_q[31:16];
  assign hw2reg.masked_out_upper.mask.d = 16'h 0;
  assign hw2reg.masked_out_lower.data.d = cio_gpio_q[15:0];
  assign hw2reg.masked_out_lower.mask.d = 16'h 0;

  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      cio_gpio_q  <= '0;
    end else if (reg2hw.direct_out.qe) begin
      cio_gpio_q <= reg2hw.direct_out.q;
    end else if (reg2hw.masked_out_upper.data.qe) begin
      cio_gpio_q[31:16] <=
        ( reg2hw.masked_out_upper.mask.q & reg2hw.masked_out_upper.data.q) |
        (~reg2hw.masked_out_upper.mask.q & cio_gpio_q[31:16]);
    end else if (reg2hw.masked_out_lower.data.qe) begin
      cio_gpio_q[15:0] <=
        ( reg2hw.masked_out_lower.mask.q & reg2hw.masked_out_lower.data.q) |
        (~reg2hw.masked_out_lower.mask.q & cio_gpio_q[15:0]);
    end
  end

  // GPIO OE
  assign cio_gpio_en_o                  = cio_gpio_en_q;

  assign hw2reg.direct_oe.d = cio_gpio_en_q;
  assign hw2reg.masked_oe_upper.data.d = cio_gpio_en_q[31:16];
  assign hw2reg.masked_oe_upper.mask.d = 16'h 0;
  assign hw2reg.masked_oe_lower.data.d = cio_gpio_en_q[15:0];
  assign hw2reg.masked_oe_lower.mask.d = 16'h 0;

  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      cio_gpio_en_q  <= '0;
    end else if (reg2hw.direct_oe.qe) begin
      cio_gpio_en_q <= reg2hw.direct_oe.q;
    end else if (reg2hw.masked_oe_upper.data.qe) begin
      cio_gpio_en_q[31:16] <=
        ( reg2hw.masked_oe_upper.mask.q & reg2hw.masked_oe_upper.data.q) |
        (~reg2hw.masked_oe_upper.mask.q & cio_gpio_en_q[31:16]);
    end else if (reg2hw.masked_oe_lower.data.qe) begin
      cio_gpio_en_q[15:0] <=
        ( reg2hw.masked_oe_lower.mask.q & reg2hw.masked_oe_lower.data.q) |
        (~reg2hw.masked_oe_lower.mask.q & cio_gpio_en_q[15:0]);
    end
  end
```

opentitan gpio提供4中中断：上升沿中断、下降沿中断、高电平中断、低电平中断。相关代码如下：

```systemverilog
  // detect four possible individual interrupts
  assign event_intr_rise    = (~data_in_q &  data_in_d) & reg2hw.intr_ctrl_en_rising.q;
  assign event_intr_fall    = ( data_in_q & ~data_in_d) & reg2hw.intr_ctrl_en_falling.q;
  assign event_intr_acthigh =                data_in_d  & reg2hw.intr_ctrl_en_lvlhigh.q;
  assign event_intr_actlow  =               ~data_in_d  & reg2hw.intr_ctrl_en_lvllow.q;

  assign event_intr_combined = event_intr_rise   |
                               event_intr_fall   |
                               event_intr_actlow |
                               event_intr_acthigh;
```

## SPI

理解此部分代码之前，请参考[SPI Device HWIP Technical Specification](https://docs.opentitan.org/hw/ip/spi_device/doc/)

与SPI接受发送相关的代码主要位于`hw/ip/spi_device/rtl/spi_fwmode.sv`，接口如下：

```systemverilog
module spi_fwmode (
  // MOSI
  input clk_in_i,  // SPI输入端时钟
  input rst_in_ni, // SPI输入端的复位

  // MISO
  input clk_out_i, // SPI输出端时钟
  input rst_out_ni,// SPI输出端的复位

  // Configurations
  // No sync logic. Configuration should be static when SPI operating
  input                             cpha_i,
  input                             cfg_rxorder_i, // 1: 0->7 , 0:7->0
  input                             cfg_txorder_i, // 1: 0->7 , 0:7->0
  input  spi_device_pkg::spi_mode_e mode_i, // Only works at mode_i == FwMode

  // 接受到的字节输出到RX FIFO
  output logic                      rx_wvalid_o,
  input                             rx_wready_i,
  output spi_device_pkg::spi_byte_t rx_data_o,
  
  // 从TX FIFO读取要发送的数据
  input                             tx_rvalid_i,
  output logic                      tx_rready_o,
  input  spi_device_pkg::spi_byte_t tx_data_i,

  output logic                      rx_overflow_o, // 接受缓存区溢出异常
  output logic                      tx_underflow_o,// 发生缓冲区空异常

  // SPI Interface: clock is given (ckl_in_i, clk_out_i)
  input        csb_i,
  input        mosi,
  output logic miso,
  output logic miso_oe
);
```

SPI内部主要是两个移位寄存器，一个时钟移动移位，一个把接受到的数据移入，一个把移出的数据输出。代码如下：

```systemverilog
  // rx移位寄存器
  always_comb begin
    if (cfg_rxorder_i) begin
      rx_data_d = {mosi, rx_data_q[BITS-1:1]};
    end else begin
      rx_data_d = {rx_data_q[BITS-2:0], mosi};
    end
  end
  always_ff @(posedge clk_in_i) begin
    rx_data_q <= rx_data_d;
  end
  // 接受计数器
  always_ff @(posedge clk_in_i or negedge rst_in_ni) begin
    if (!rst_in_ni) begin
      rx_bitcount <= BITWIDTH'(BITS-1);
    end else begin
      if (rx_bitcount == '0) begin
        rx_bitcount <= BITWIDTH'(BITS-1);
      end else begin
        rx_bitcount <= rx_bitcount -1;
      end
    end
  end
  // 计数器为0,接受到完整的字节
  assign rx_wvalid_o = (rx_bitcount == '0);

  
  // 计算第一比特和最后一比特
  assign first_bit = (tx_bitcount == BITWIDTH'(BITS-1)) ? 1'b1 : 1'b0;
  assign last_bit  = (tx_bitcount == '0) ? 1'b1 : 1'b0;
  
  assign tx_rready_o = (tx_bitcount == BITWIDTH'(1)); // Pop at second bit transfer
  // 发送计算器
  always_ff @(posedge clk_out_i or negedge rst_out_ni) begin
    if (!rst_out_ni) begin
      tx_bitcount <= BITWIDTH'(BITS-1);
    end else begin
      if (last_bit) begin
        tx_bitcount <= BITWIDTH'(BITS-1);
      end else if (tx_state != TxIdle || cpha_i == 1'b0) begin
        tx_bitcount <= tx_bitcount - 1'b1;
      end
    end
  end
  always_ff @(posedge clk_out_i or negedge rst_out_ni) begin
    if (!rst_out_ni) begin
      tx_state <= TxIdle;
    end else begin
      tx_state <= TxActive;
    end
  end
  
  // 在miso输出移出位
  assign miso = (cfg_txorder_i) ? ((~first_bit) ? miso_shift[0] : tx_data_i[0]) :
                (~first_bit) ? miso_shift[7] : tx_data_i[7] ;
  assign miso_oe = ~csb_i;
  
  // tx移位寄存器
  always_ff @(posedge clk_out_i) begin
    if (cfg_txorder_i) begin
      if (first_bit) begin
        miso_shift <= {1'b0, tx_data_i[7:1]};
      end else begin
        miso_shift <= {1'b0, miso_shift[7:1]};
      end
    end else begin
      if (first_bit) begin
        // Dummy byte cannot be used. empty signal could be delayed two clocks to cross
        // async clock domain. It means even FW writes value to FIFO, empty signal deasserts
        // after two negative edge of SCK. HW module already in the middle of sending DUMMY.
        miso_shift <= {tx_data_i[6:0], 1'b0};
      end else begin
        miso_shift <= {miso_shift[6:0], 1'b0};
      end
    end
  end
```

`spi_fwmode`模块从接受缓冲取获取要发送的数据，把接受到的数据写入发生缓冲区。代码如下：

```systemverilog
  prim_fifo_async #(
    .Width (FifoWidth),
    .Depth (FifoDepth)
  ) u_rx_fifo (
    .clk_wr_i     (clk_spi_in),
    .rst_wr_ni    (rst_rxfifo_n),

    .clk_rd_i     (clk_i),
    .rst_rd_ni    (rst_rxfifo_n),

    // 来自fwmode的接受到的数据
    .wvalid       (rxf_wvalid),
    .wready       (rxf_wready),
    .wdata        (rxf_wdata),

    .rvalid       (rxf_rvalid),
    .rready       (rxf_rready),
    .rdata        (rxf_rdata),

    .wdepth       (),
    .rdepth       (as_rxfifo_depth)
  );

  prim_fifo_async #(
    .Width (FifoWidth),
    .Depth (FifoDepth)
  ) u_tx_fifo (
    .clk_wr_i     (clk_i),
    .rst_wr_ni    (rst_txfifo_n),

    .clk_rd_i     (clk_spi_out),
    .rst_rd_ni    (rst_txfifo_n),

    .wvalid       (txf_wvalid),
    .wready       (txf_wready),
    .wdata        (txf_wdata),

    // fwmode从fifo获取要发生的数据
    .rvalid       (txf_rvalid),
    .rready       (txf_rready),
    .rdata        (txf_rdata),

    .wdepth       (as_txfifo_depth),
    .rdepth       ()
  );
```

SPI内部有一个双端RAM，一头由cpu访问，一条由硬件访问。硬件这头需要由两个硬件逻辑访问，为了实现两个硬件访问内存，加入仲裁器，代码如下：

```systemverilog
  // 两组ram操作接口
  logic        [1:0] fwm_sram_req;
  logic [SramAw-1:0] fwm_sram_addr  [2];
  logic              fwm_sram_write [2];
  logic [SramDw-1:0] fwm_sram_wdata [2];
  logic        [1:0] fwm_sram_gnt;
  logic        [1:0] fwm_sram_rvalid;    // RXF doesn't use
  logic [SramDw-1:0] fwm_sram_rdata [2]; // RXF doesn't use
  logic        [1:0] fwm_sram_error [2];

  prim_sram_arbiter #(
    .N       (2),  // RXF, TXF
    .SramDw (SramDw),
    .SramAw (SramAw)   // 2kB
  ) u_fwmode_arb (
    .clk_i,
    .rst_ni,

    // 仲裁器后端，两组SRAM接口
    .req          (fwm_sram_req),
    .req_addr     (fwm_sram_addr),
    .req_write    (fwm_sram_write),
    .req_wdata    (fwm_sram_wdata),
    .gnt          (fwm_sram_gnt),

    .rsp_rvalid   (fwm_sram_rvalid),
    .rsp_rdata    (fwm_sram_rdata),
    .rsp_error    (fwm_sram_error),

    // 双端RAM的b口
    .sram_req     (mem_b_req),
    .sram_addr    (mem_b_addr),
    .sram_write   (mem_b_write),
    .sram_wdata   (mem_b_wdata),

    .sram_rvalid  (mem_b_rvalid),
    .sram_rdata   (mem_b_rdata),
    .sram_rerror  (mem_b_rerror)
  );
```

opentitan spi内部有两个硬件逻辑（`spi_fwm_rxf_ctrl`和`spi_fwm_txf_ctrl`），负责把从spi接受到的数据写入内存，负责把内存中的数据通过spi发生出去。这两个逻辑连接到fifo缓冲后，代码如下：

```systemverilog
  // RX Fifo control (FIFO Read port --> SRAM request)
  spi_fwm_rxf_ctrl #(
    .FifoDw (FifoWidth),
    .SramAw (SramAw),
    .SramDw (SramDw)
  ) u_rxf_ctrl (
    .clk_i,
    .rst_ni,

     // 接受模块使用的内存的起始地址和长度限制
    .base_index_i  (sram_rxf_bindex),
    .limit_index_i (sram_rxf_lindex),

    // 寄存器
    .timer_v      (timer_v),
    .rptr         (sram_rxf_rptr),  // Given by FW
    .wptr         (sram_rxf_wptr),  // to Register interface
    .depth        (sram_rxf_depth),
    .full         (sram_rxf_full),

    // 接受fifo缓存接口
    .fifo_valid  (rxf_rvalid),
    .fifo_ready  (rxf_rready),
    .fifo_rdata  (rxf_rdata),

    // 仲裁器后的ram接口，用于从内存读取数据通过SPI发送
    .sram_req    (fwm_sram_req   [FwModeRxFifo]),
    .sram_write  (fwm_sram_write [FwModeRxFifo]),
    .sram_addr   (fwm_sram_addr  [FwModeRxFifo]),
    .sram_wdata  (fwm_sram_wdata [FwModeRxFifo]),
    .sram_gnt    (fwm_sram_gnt   [FwModeRxFifo]),
    .sram_rvalid (fwm_sram_rvalid[FwModeRxFifo]),
    .sram_rdata  (fwm_sram_rdata [FwModeRxFifo]),
    .sram_error  (fwm_sram_error [FwModeRxFifo])
  );

// TX Fifo control (SRAM read request --> FIFO write)
  spi_fwm_txf_ctrl #(
    .FifoDw (FifoWidth),
    .SramAw (SramAw),
    .SramDw (SramDw)
  ) u_txf_ctrl (
    .clk_i,
    .rst_ni,

     // 发送模块使用的内存的起始地址和长度限制
    .base_index_i  (sram_txf_bindex),
    .limit_index_i (sram_txf_lindex),

    // 寄存器
    .abort        (abort),
    .rptr         (sram_txf_rptr),  // Given by FW
    .wptr         (sram_txf_wptr),  // to Register interface
    .depth        (sram_txf_depth),

    // 发生fifo缓冲接口
    .fifo_valid  (txf_wvalid),
    .fifo_ready  (txf_wready),
    .fifo_wdata  (txf_wdata),

     // 仲裁器后的ram接口，用于把接受到的数据写入内存
    .sram_req    (fwm_sram_req   [FwModeTxFifo]),
    .sram_write  (fwm_sram_write [FwModeTxFifo]),
    .sram_addr   (fwm_sram_addr  [FwModeTxFifo]),
    .sram_wdata  (fwm_sram_wdata [FwModeTxFifo]),
    .sram_gnt    (fwm_sram_gnt   [FwModeTxFifo]),
    .sram_rvalid (fwm_sram_rvalid[FwModeTxFifo]),
    .sram_rdata  (fwm_sram_rdata [FwModeTxFifo]),
    .sram_error  (fwm_sram_error [FwModeTxFifo])
  );
```

其中`spi_fwm_rxf_ctrl`代码实现位于`hw/ip/spi_device/rtl/spi_fwm_rxf_ctrl.sv`中。模块接口如下：

```systemverilog
module spi_fwm_rxf_ctrl #(
  parameter int unsigned FifoDw = 8,  // spi一次接受8个比特
  parameter int unsigned SramAw = 11, // sram地址宽度11位
  parameter int unsigned SramDw = 32, // 内存宽度32比特
  // Do not touch below
  // SramDw should be multiple of FifoDw
  localparam int unsigned NumBytes = SramDw/FifoDw,    // 从fifo中取NumBytes可以一次写入内存
  localparam int unsigned SDW      = $clog2(NumBytes), // 读写指针的低SDW位标识字节偏移量
  localparam int unsigned PtrW     = SramAw + SDW + 1  // 指针宽度添加一，用于记录指针翻转
) (
  // 时钟和复位
  input clk_i,
  input rst_ni,

  // 来自寄存器的配置信息
  input      [SramAw-1:0] base_index_i,  // 接收缓冲的起始地址
  input      [SramAw-1:0] limit_index_i, // 接收缓冲的结束地址
  input             [7:0] timer_v,       // 超时，当超时时fifo中的内容会写入到内存不管是否收到完整的字
  input        [PtrW-1:0] rptr,          // 读指针
  output logic [PtrW-1:0] wptr,          // 写指针
  output logic [PtrW-1:0] depth,         // 内存中接受到多少字节

  output logic            full,          // 缓存满了

  // 接收缓存fifo的接口
  input               fifo_valid,
  output logic        fifo_ready,
  input  [FifoDw-1:0] fifo_rdata,

  // sram的内存接口
  output logic              sram_req,
  output logic              sram_write,
  output logic [SramAw-1:0] sram_addr,
  output logic [SramDw-1:0] sram_wdata,
  input                     sram_gnt,
  input                     sram_rvalid,
  input        [SramDw-1:0] sram_rdata,
  input               [1:0] sram_error
);
```

此模块负责把spi接受到的数据从fifo中转移到内存中。由与spi每次发送接受一个字节，而opentitan的内存宽度是32位的，所以此模块要把多个字节合并为一个字节再写入到内存中。为了实现此功能内部使用了一个状态机，状态机如下：

![spi_fwm_rxf_ctrl状态机](https://raw.githubusercontent.com/hardenedlinux/embedded-iot_profile/master/resources/images/docs/opentitan/spi_fwm_rxf_ctrl_fsm.png)

- StIdel，等待fifo不为空进入StPop
- StPop，从fifo取字节，如果fifo中有连续的4个字节进入StWrite，否则进入StWait
- StWait，计时等待fifo不为空，超时进入StRead，否则返回StPop
- StRead，读取内存
- StModify，把从内存读取到的内容和从fifo读取到的内容合并为一个字
- StWrite，写内存等待内存响应后进入StUpdate
- StUpdate，更新wptr寄存器

`spi_fwm_rxf_ctrl`维护了两个指针rptr/wptr，wptr由硬件控制，rptr标识当前缓存的起始地址。通过如下代码判定缓存以满

```systemverilog
  logic [PtrW-1:0] ptr_cmp;
  assign ptr_cmp = rptr ^ wptr;
  // TODO: Check partial SRAM width read condition
  assign sramf_full = (ptr_cmp[PtrW-1] == 1'b1) && (ptr_cmp[PtrW-2:SDW] == '0);
  assign full = sramf_full;
```

wptr更新代码如下：

```systemverilog
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      // 复位
      wptr <= '0;
    end else if (update_wptr) begin
      if (byte_enable == '0) begin
        // as byte enable is cleared, it means full write was done
        if (wptr[PtrW-2:SDW] == sramf_limit) begin
          // 到达最大地址反转最高位
          wptr[PtrW-1] <= ~wptr[PtrW-1];
          wptr[PtrW-2:0] <= '0;
        end else begin
          // 地址加一
          wptr[PtrW-2:SDW] <= wptr[PtrW-2:SDW] + 1'b1;
          wptr[SDW-1:0] <= '0;
        end
      end else begin
        // 记录当前pos
        wptr[SDW-1:0] <= pos;
      end
    end
  end
```

`spi_fwm_rxf_ctrl`内部使用两个寄存器`byte_enable`/`pos`记录从fifo中取出的字节数，通过如下代码判断是否取出了完整的字。

```systemverilog
  // Byte Enable control
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      // 复位
      byte_enable <= '0;
      pos <= '0;
    end else if (update_wdata) begin
      // 从fifo取出一个字节时，更新wdata
      byte_enable[pos] <= 1'b1;                      // 标记取出的字节
      if (pos == SDW'(NumBytes-1)) pos <= '0;        // 取出完整字后清零pos
      else                         pos <= pos + 1'b1;// pos加一
    end else if (clr_byte_enable) begin
      byte_enable <= '0;
      pos <= '0;
    end
  end
```

通过如下代码把从fifo中取出的值输出到内存的写接口：

```systemverilog
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      // 复位
      sram_wdata <= '0;
    end else if (update_wdata) begin
      // 从fifo取出内容
      sram_wdata[8*pos+:8] <= fifo_rdata;
    end else if (sram_wdata_sel == 1'b1) begin
      // 把内存读取的内容和fifo读取的内容合并
      for (int i = 0 ; i < NumBytes ; i++) begin
        if (!byte_enable[i]) begin
          sram_wdata[8*i+:8] <= sram_rdata[8*i+:8];
        end
      end
    end
  end
```

其中`spi_fwm_txf_ctrl`代码实现位于`hw/ip/spi_device/rtl/spi_fwm_txf_ctrl.sv`中。模块接口如下：

```systemverilog
module spi_fwm_txf_ctrl #(
  parameter int FifoDw = 8,   // spi一次接受8个比特
  parameter int SramAw = 11,  // sram地址宽度
  parameter int SramDw = 32,  // sram数据宽度
  // SramDw should be multiple of FifoDw
  localparam int NumBytes = SramDw/FifoDw, // 从fifo中取出NumBytes个字节可以一次写入sram
  localparam int SDW = $clog2(NumBytes),   // 读写指针的低SDW位标识字节偏移量
  localparam int PtrW = SramAw + SDW + 1   // 指针宽度添加一，用于记录指针翻转
) (
  // 时钟和复位
  input clk_i,
  input rst_ni,

  // 来自寄存器的配置信息
  input [SramAw-1:0] base_index_i,  // 发送缓冲的起始地址
  input [SramAw-1:0] limit_index_i, // 发送缓冲的结束地址

  // 寄存器
  input                   abort, // Abort State Machine if TX Async at stuck
  input        [PtrW-1:0] wptr,
  output logic [PtrW-1:0] rptr,
  output logic [PtrW-1:0] depth,

  // fifo接口
  output logic              fifo_valid,
  input                     fifo_ready,
  output logic [FifoDw-1:0] fifo_wdata,

  // sram内存接口
  output logic              sram_req,
  output logic              sram_write,
  output logic [SramAw-1:0] sram_addr,
  output logic [SramDw-1:0] sram_wdata,
  input                     sram_gnt,
  input                     sram_rvalid,
  input        [SramDw-1:0] sram_rdata,
  input               [1:0] sram_error
);
```

此模块负责把内存中的内容写入到fifo中。由于spi一次发送一个字节，而内存宽度四个字节，此模块需要把一个字拆分为4个字节写入fifo中。为了实现此功能内部实现了一个状态机，状态机如下：

![spi_fwm_txf_ctrl状态机](https://raw.githubusercontent.com/hardenedlinux/embedded-iot_profile/master/resources/images/docs/opentitan/spi_fwm_txf_ctrl_fsm.png)

- StIdel，等待fifo准备就绪然后进入StRead
- StRead，发生内存请求信号等待内存响应，然后进入StLatch
- StLatch，等待读取到有效数据，然后进入StPush
- StPush，拆分成字节写入到fifo，然后进入StUpdate
- StUpdate，更新rptr，然后进入StIdel

`spi_fwm_txf_ctrl`维护了两个指针rptr/wptr，rptr由硬件控制，wptr标识当前缓存的起始地址。通过如下代码判定缓存为空：

```systemverilog
  assign sramf_empty = (rptr == wptr_q);

  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      wptr_q <= '0;
    end else if (latch_wptr) begin
      wptr_q <= wptr;
    end
  end
```

rptr更新代码如下：

```systemverilog
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      // 复位
      rptr <= '0;
    end else if (update_rptr) begin
      if (pos == '0) begin
        // full sram word is written.
        if (rptr[PtrW-2:SDW] != sramf_limit) begin
          // 指针加一
          rptr[PtrW-1:SDW] <= rptr[PtrW-1:SDW] + 1'b1;
          rptr[SDW-1:0] <= '0;
        end else begin
          // 指针翻转，高位记录
          rptr[PtrW-1] <= ~rptr[PtrW-1];
          rptr[PtrW-2:SDW] <= '0;
          rptr[SDW-1:0]    <= '0;
        end
      end else begin
        // Abort, or partial update (fifo_full), or wptr_q is at the same entry
        rptr[SDW-1:0] <= pos;
      end
    end
  end
```

通过如下代码，把从内存读出的内容写入到fifo

```systemverilog
  // 维护一个字节偏移量pos
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      pos <= '0;
    end else if (cnt_rst) begin
      // Reset to rptr to select bytes among fifo_wdata_d
      pos <= rptr[SDW-1:0];
    end else if (cnt_incr) begin
      // Increase position
      pos <= pos + 1'b1;
    end
  end

  // 缓存从内存读出的内容
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) sram_rdata_q <= '0;
    else if (sram_rvalid) sram_rdata_q <= sram_rdata;
  end

  assign fifo_wdata_d = (txf_sel) ? sram_rdata_q : sram_rdata ;

  // 写入fifo
  always_comb begin
    fifo_wdata = '0;
    for (int i = 0 ; i < NumBytes ; i++) begin
      if (pos == i) fifo_wdata = fifo_wdata_d[8*i+:8];
    end
  end
```

## I2C

在理解此部分代码前请参考[I2C HWIP Technical Specification](https://docs.opentitan.org/hw/ip/i2c/doc/)

与I2C发送和接收相关的代码主要位于`hw/ip/i2c/rtl/i2c_fsm.sv`，接口如下：

```systemverilog
module i2c_fsm (
  // 时钟和复位
  input        clk_i,  // clock
  input        rst_ni, // active low reset

  // i2c的io口，i2c采用开漏输出外部上拉，所以可以同时输出并获取引脚状态
  input        scl_i,  // serial clock input from i2c bus
  output       scl_o,  // serial clock output to i2c bus
  input        sda_i,  // serial data input from i2c bus
  output       sda_o,  // serial data output to i2c bus

  input        host_enable_i, // i2c host功能使能

  // fmt用于控制i2c如何工作，此处连接fmt的fifo
  input        fmt_fifo_rvalid_i, // indicates there is valid data in fmt_fifo
  input        fmt_fifo_wvalid_i, // indicates data is being put into fmt_fifo
  input [5:0]  fmt_fifo_depth_i,  // fmt_fifo_depth
  output logic fmt_fifo_rready_o, // populates fmt_fifo
  input [7:0]  fmt_byte_i,        // byte in fmt_fifo to be sent to target
  input        fmt_flag_start_before_i, // issue start before sending byte
  input        fmt_flag_stop_after_i,   // issue stop after sending byte
  input        fmt_flag_read_bytes_i,   // indicates byte is an number of reads
  input        fmt_flag_read_continue_i,// host to send Ack to final byte read
  input        fmt_flag_nak_ok_i,       // no Ack is expected

  // 把接受到的字节写入到接受fifo
  output logic       rx_fifo_wvalid_o, // high if there is valid data in rx_fifo
  output logic [7:0] rx_fifo_wdata_o,  // byte in rx_fifo read from target
  
  // 主机空闲信号
  output logic       host_idle_o,      // indicates the host is idle

  // 控制信息
  input [15:0] thigh_i,    // high period of the SCL in clock units
  input [15:0] tlow_i,     // low period of the SCL in clock units
  input [15:0] t_r_i,      // rise time of both SDA and SCL in clock units
  input [15:0] t_f_i,      // fall time of both SDA and SCL in clock units
  input [15:0] thd_sta_i,  // hold time for (repeated) START in clock units
  input [15:0] tsu_sta_i,  // setup time for repeated START in clock units
  input [15:0] tsu_sto_i,  // setup time for STOP in clock units
  input [15:0] tsu_dat_i,  // data setup time in clock units
  input [15:0] thd_dat_i,  // data hold time in clock units
  input [15:0] t_buf_i,    // bus free time between STOP and START in clock units
  input [30:0] stretch_timeout_i,  // max time target may stretch the clock
  input        timeout_enable_i,   // assert if target stretches clock past max

  // 输出异常和错误信息
  output logic event_nak_o,              // target didn't Ack when expected
  output logic event_scl_interference_o, // other device forcing SCL low
  output logic event_sda_interference_o, // other device forcing SDA low
  output logic event_stretch_timeout_o,  // target stretches clock past max time
  output logic event_sda_unstable_o,     // SDA is not constant during SCL pulse
  output logic event_trans_complete_o    // Transaction is complete
);
```

opentitan i2c把i2c操作简化为字节操作。内部有两个fifo，一个（rxfifo）用于保存接收到的字节，一个(fmtfifo)用于控制和发送。rxfiifo比较简单8比特宽度用于保存读取到的字节。fmtfifo 13比特，用于控制和发送。fmtfifo用于保存FDATA，格式如下：

| 字段 | 类型 | 复位值 | 名字 | 描述 |
|-----|------|--------|-----|------|
| 7:0 | 只写 | 不定值  | FBYTE | 格式字节，当其他位为零时，直接发生此字节 |
| 8   | 只写 | 不定值  | START | 在传送节前发送起始位 |
| 9   | 只写 | 不定值  | STOP  | 在传送字节后发送结束位 |
| 10  | 只写 | 不定值  | READ  | 读取，读取的字节数为，FBYTE != 0 ? FBYTE : 256 |
| 11  | 只写 | 不定值  | RCONT | 读取完最后一个字节不发生NAK |
| 12  | 只写 | 不定值  | NAKOK | 写字节时接受到NAK不触发异常 |

`i2c_fsm`模块就负责从fmtfifo中读取控制信息，并把接受到的字节回写到rxfifo中。为了实现此功能，内部维护了一个状态机，状态机如下：

![i2c_fsm状态机](https://raw.githubusercontent.com/hardenedlinux/embedded-iot_profile/master/resources/images/docs/opentitan/i2c_fsm.png)

- Idle，表示空闲状态
- SetupStart - ClockStart，用于发生起始位
- ClockStop - HoldStop，用于发送结束位
- ReadClockLow - ReadHoldBit，用于循环接收一个字节
- HostClockLowAck - HostHoldBitAck，用于向device发生响应，标识成功接收到一个字节
- ReadClockLow - HostHoldBitAck，用于循环接受多个字节
- ClockLow - HoldBit，用于循环发送一个字节
- ClockLowAck - HoldDevAck，用于从device接受响应，标识成功发送一个字节
- PopFmtFifo，用于从fmtfifo中取出一个新的操作
- Active，用于开始一个新操作的中间状态

I2C各个阶段的信号可以配置稳定多长时间，通过如下代码计算信号稳定的时间。

```systemverilog
  always_comb begin : counter_functions
    tcount_d = tcount_q;
    if (load_tcount) begin
      // 根据tcount_sel加载时间
      unique case (tcount_sel)
        tSetupStart : tcount_d = t_r_i + tsu_sta_i;
        tHoldStart  : tcount_d = t_f_i + thd_sta_i;
        tClockStart : tcount_d = 20'(thd_dat_i);
        tClockLow   : tcount_d = tlow_i - t_r_i - tsu_dat_i - thd_dat_i;
        tSetupBit   : tcount_d = t_r_i + tsu_dat_i;
        tClockPulse : tcount_d = t_r_i + thigh_i + t_f_i;
        tHoldBit    : tcount_d = t_f_i + thd_dat_i;
        tClockStop  : tcount_d = t_f_i + tlow_i - thd_dat_i;
        tSetupStop  : tcount_d = t_r_i + tsu_sto_i;
        tHoldStop   : tcount_d = t_r_i + t_buf_i - tsu_sta_i;
        tNoDelay    : tcount_d = 20'h00001;
        default     : tcount_d = 20'h00001;
      endcase
    end else if (stretch == 0) begin
      // 1个周期时钟减1
      tcount_d = tcount_q - 1'b1;
    end else begin
      tcount_d = tcount_q;  // pause timer if clock is stretched
    end
  end

  always_ff @ (posedge clk_i or negedge rst_ni) begin : clk_counter
    if (!rst_ni) begin
      tcount_q <= '1;
    end else begin
      tcount_q <= tcount_d;
    end
  end
```

在状态切换之前，上一个状态会给下一个状态设定号延时信息（`load_tcount`/`tcount_sel`）。

通过如下代码接受一个字节：

```systemverilog
  // Deserializer for a byte read from the bus
  always_ff @ (posedge clk_i or negedge rst_ni) begin : read_register
    if (!rst_ni) begin
      read_byte <= 8'h00;
    end else if (read_byte_clr) begin
      read_byte <= 8'h00;
    end else if (shift_data_en) begin
      read_byte[7:0] <= {read_byte[6:0], sda_i};  // MSB goes in first
    end
  end
```

当为接收时，通过如下代码计算要接收的字节数：

```systemverilog
  // Number of bytes to read
  always_comb begin : byte_number
    if (!fmt_flag_read_bytes_i) byte_num = 9'd0;
    else if (fmt_byte_i == 0) byte_num = 9'd256;
    else byte_num = 9'(fmt_byte_i);
  end
```

并通过如下代码，对接受字节数计数：

```systemverilog
  // Byte index implementation
  always_ff @ (posedge clk_i or negedge rst_ni) begin : byte_counter
    if (!rst_ni) begin
      byte_index <= '0;
    end else if (byte_clr) begin
      byte_index <= byte_num;
    end else if (byte_decr) begin
      byte_index <= byte_index - 1'b1;
    end else begin
      byte_index <= byte_index;
    end
  end
```

并在接收完一个fmt控制长度后发生结束位，或者进入PopFmtFifo状态从fmtfifo中取新的命令，代码如下：

```systemverilog
      HostHoldBitAck : begin
        if (tcount_q == 1) begin
          if (byte_index == 1) begin
            if (fmt_flag_stop_after_i) begin
              state_d = ClockStop;
              load_tcount = 1'b1;
              tcount_sel = tClockStop;
            end else begin
              state_d = PopFmtFifo;
              load_tcount = 1'b1;
              tcount_sel = tNoDelay;
            end
          end else begin
            state_d = ReadClockLow;
            load_tcount = 1'b1;
            tcount_sel = tClockLow;
            byte_decr = 1'b1;
          end
        end
      end
```

发生时，通过`bit_index`控制当前传输的比特的位置。`bit_index`通过如下代码控制

```systemverilog
  // Bit index implementation
  always_ff @ (posedge clk_i or negedge rst_ni) begin : bit_counter
    if (!rst_ni) begin
      bit_index <= 3'd7;
    end else if (bit_clr) begin
      bit_index <= 3'd7;
    end else if (bit_decr) begin
      bit_index <= bit_index - 1'b1;
    end else begin
      bit_index <= bit_index;
    end
  end
```

当发送完一个字节后，进入ClockLowAck状态，否则返回ClockLow状态进行发送下一个比特。代码如下：

```systemverilog
      HoldBit : begin
        if (tcount_q == 1) begin
          load_tcount = 1'b1;
          tcount_sel = tClockLow;
          if (bit_index == 0) begin
            state_d = ClockLowAck;
            bit_clr = 1'b1;
          end else begin
            state_d = ClockLow;
            bit_decr = 1'b1;
          end
        end
      end
```

当接受完一个字节后，进入HostClockLowAck状态，否则返回ReadClockLow状态接收下一个比特。代码如下：

```systemverilog
      ReadHoldBit : begin
        if (tcount_q == 1) begin
          load_tcount = 1'b1;
          tcount_sel = tClockLow;
          if (bit_index == 0) begin
            state_d = HostClockLowAck;
            bit_clr = 1'b1;
            read_byte_clr = 1'b1;
          end else begin
            state_d = ReadClockLow;
            bit_decr = 1'b1;
          end
        end
      end
```

在`hw/ip/i2c/rtl/i2c_core.sv`中把寄存器和硬件连接到一起。opentitan i2c具有缓冲区异常，代码如下：

```systemverilog
  // 控制发生缓冲区过少时触发异常，让软件添加
  always_comb begin
    unique case(i2c_fifo_fmtilvl)
      2'h0:    fmt_watermark_d = (fmt_fifo_depth <= 6'd1);
      2'h1:    fmt_watermark_d = (fmt_fifo_depth <= 6'd4);
      2'h2:    fmt_watermark_d = (fmt_fifo_depth <= 6'd8);
      default: fmt_watermark_d = (fmt_fifo_depth <= 6'd16);
    endcase
  end

  assign event_fmt_watermark = fmt_watermark_d & ~fmt_watermark_q;

  // 接受缓冲区快满时触发异常，让软件拿走数据
  always_comb begin
    unique case(i2c_fifo_rxilvl)
      3'h0:    rx_watermark_d = (rx_fifo_depth >= 6'd1);
      3'h1:    rx_watermark_d = (rx_fifo_depth >= 6'd4);
      3'h2:    rx_watermark_d = (rx_fifo_depth >= 6'd8);
      3'h3:    rx_watermark_d = (rx_fifo_depth >= 6'd16);
      3'h4:    rx_watermark_d = (rx_fifo_depth >= 6'd30);
      default: rx_watermark_d = 1'b0;
    endcase
  end

  assign event_rx_watermark = rx_watermark_d & ~rx_watermark_q;
```

opentitan i2c可以通过软件控制，代码如下：

```systemverilog
  // 获取寄存器override
  assign override = reg2hw.ovrd.txovrden;
  
  // 通过寄存器override，控制i2c输出由i2c_fsm或寄存器控制
  assign scl_o = override ? reg2hw.ovrd.sclval : scl_out_fsm;
  assign sda_o = override ? reg2hw.ovrd.sdaval : sda_out_fsm;

  // 接收的值，直接写入寄存器
  assign hw2reg.val.scl_rx.d = scl_rx_val;
  assign hw2reg.val.sda_rx.d = sda_rx_val;
    // Sample scl_i and sda_i at system clock
  always_ff @ (posedge clk_i or negedge rst_ni) begin : rx_oversampling
    if(!rst_ni) begin
       scl_rx_val <= 16'h0;
       sda_rx_val <= 16'h0;
    end else begin
       scl_rx_val <= {scl_rx_val[14:0], scl_i};
       sda_rx_val <= {sda_rx_val[14:0], sda_i};
    end
  end
```

在`hw/ip/i2c/rtl/i2c.sv`中，把tlul总线转换为寄存器再与i2c_core连接，实现了挂载在tlul上的i2c host设备。


## padctrl

此模块由好几个模块组成。由于padctrl的io控制模块需要靠近pad，寄存器相关的模块与soc比较靠近，所以padctrl被拆分为多个模块。在理解此部分代码前请参考[Padctrl Technical Specification](https://docs.opentitan.org/hw/ip/padctrl/doc/)

`padring`位于`hw/ip/padctrl/rtl/padring.sv`中，主要通过`prim_pad_wrapper`实现了一组端口输入输出以及属性控制。接口如下：

```systemverilog
module padring import padctrl_reg_pkg::*; #(
  // muxed IO
  parameter logic [NMioPads-1:0] ConnectMioIn = '1,  // 掩码用于标识io口是否支持输入
  parameter logic [NMioPads-1:0] ConnectMioOut = '1, // 掩码用于标识io口是否支持输出
  // dedicated IO
  parameter logic [NDioPads-1:0] ConnectDioIn = '1,  // 掩码用于标识io口是否支持输入
  parameter logic [NDioPads-1:0] ConnectDioOut = '1  // 掩码用于标识io口是否支持输
) (
  // pad input
  input wire                  clk_pad_i,
  input wire                  clk_usb_48mhz_pad_i,
  input wire                  rst_pad_ni,
  // to clocking/reset infrastructure
  output logic                clk_o,
  output logic                clk_usb_48mhz_o,
  output logic                rst_no,
  // pads
  inout wire   [NMioPads-1:0] mio_pad_io,
  inout wire   [NDioPads-1:0] dio_pad_io,
  // muxed IO signals coming from pinmux
  output logic [NMioPads-1:0] mio_in_o,
  input        [NMioPads-1:0] mio_out_i,
  input        [NMioPads-1:0] mio_oe_i,
  // dedicated IO signals coming from peripherals
  output logic [NDioPads-1:0] dio_in_o,
  input        [NDioPads-1:0] dio_out_i,
  input        [NDioPads-1:0] dio_oe_i,
  // pad attributes from top level instance
  input        [NMioPads-1:0][AttrDw-1:0] mio_attr_i,
  input        [NDioPads-1:0][AttrDw-1:0] dio_attr_i
);
```
对于时钟和复位信号为纯输入信号，代码如下：

```systemverilog
  prim_pad_wrapper #(
    .AttrDw(AttrDw)
  ) i_clk_pad (
    .inout_io ( clk   ),
    .in_o     ( clk_o ),
    .out_i    ( 1'b0  ),
    .oe_i     ( 1'b0  ),
    .attr_i   (   '0  )
  );
```

对于`muxed IO`和`dedicated IO`输入输出塑性通过`ConnectMioIn/ConnectMioOut/ConnectDioIn/ConnectDioOut`控制，通过如下代码判断输入输出属性。

```systemverilog
    if (ConnectMioIn[k] && ConnectMioOut[k]) begin : gen_mio_inout
      // 同时指出输入输出
      prim_pad_wrapper #(
        .AttrDw(AttrDw)
      ) i_mio_pad (
        .inout_io ( mio_pad_io[k] ),
        .in_o     ( mio_in_o[k]   ),
        .out_i    ( mio_out_i[k]  ),
        .oe_i     ( mio_oe_i[k]   ),
        .attr_i   ( mio_attr_i[k] )
      );
    end else if (ConnectMioOut[k]) begin : gen_mio_output
      // 只支持输出
      prim_pad_wrapper #(
        .AttrDw(AttrDw)
      ) i_mio_pad (
        .inout_io ( mio_pad_io[k] ),
        .in_o     (               ),
        .out_i    ( mio_out_i[k]  ),
        .oe_i     ( mio_oe_i[k]   ),
        .attr_i   ( mio_attr_i[k] )
      );

      assign mio_in_o[k]  = 1'b0;
    end else if (ConnectMioIn[k]) begin : gen_mio_input
      // 只支持输入
      prim_pad_wrapper #(
        .AttrDw(AttrDw)
      ) i_mio_pad (
        .inout_io ( mio_pad_io[k] ),
        .in_o     ( mio_in_o[k]   ),
        .out_i    ( 1'b0          ),
        .oe_i     ( 1'b0          ),
        .attr_i   ( mio_attr_i[k] )
      );

      logic unused_out, unused_oe;
      assign unused_out   = mio_out_i[k];
      assign unused_oe    = mio_oe_i[k];
    end else begin : gen_mio_tie_off
      // 不支持输入和输出
      logic unused_out, unused_oe;
      logic [AttrDw-1:0] unused_attr;
      assign mio_pad_io[k] = 1'bz;
      assign unused_out   = mio_out_i[k];
      assign unused_oe    = mio_oe_i[k];
      assign unused_attr  = mio_attr_i[k];
      assign mio_in_o[k]  = 1'b0;
    end
```

`padring`模块接受两个数组，用于标识IO口塑性。这来自寄存器模块，寄存器实现来自`padctrl`位于`hw/ip/padctrl/rtl/padctrl.sv`。模块接口如下：

```systemverilog
module padctrl import padctrl_reg_pkg::*; #(
  parameter prim_pkg::impl_e Impl = `PRIM_DEFAULT_IMPL
) (
  // 时钟和复位
  input                                  clk_i,
  input                                  rst_ni,
  // 总线接口
  input  tlul_pkg::tl_h2d_t              tl_i,
  output tlul_pkg::tl_d2h_t              tl_o,
  // 输出给padring用于控制io口属性
  output logic[NMioPads-1:0][AttrDw-1:0] mio_attr_o,
  output logic[NDioPads-1:0][AttrDw-1:0] dio_attr_o
);
```

此模块内部通过获取寄存器输出值，合并为两个连续数组输出。寄存器的实际实现来自`padctrl_reg_top`位于`hw/ip/padctrl/rtl/padctrl_reg_top.sv`

## pinmux

opentitan有两种外设：一种叫`Muxed Peripheral`没有专用io端口；一种叫`Dedicated Peripheral`具有专用的io端口。pinmux提供`Muxed IO`与`pad`的映射关系，并且提供休眠，休眠时输出状态可配置，并在io口上出现特定信号时唤醒。在理解此部分代码之前请参考[Pinmux Technical Specification](https://docs.opentitan.org/hw/ip/pinmux/doc/)

模块顶层接口如下：

```systemverilog
module pinmux import pinmux_pkg::*; import pinmux_reg_pkg::*; (
  // 时钟和复位
  input                            clk_i,
  input                            rst_ni,

  // 一直开启的慢速时钟，在休眠时依旧工作
  input                            clk_aon_i,
  input                            rst_aon_ni,

  // 接受到唤醒信号
  output logic                     aon_wkup_req_o,

  // 休眠使能
  input                            sleep_en_i,

  // Strap sample request
  input  lc_strap_req_t            lc_pinmux_strap_i,
  output lc_strap_rsp_t            lc_pinmux_strap_o,

  // 总线接口
  input  tlul_pkg::tl_h2d_t        tl_i,
  output tlul_pkg::tl_d2h_t        tl_o,

  // Muxed Peripheral 接口
  input        [NMioPeriphOut-1:0] periph_to_mio_i,
  input        [NMioPeriphOut-1:0] periph_to_mio_oe_i,
  output logic [NMioPeriphIn-1:0]  mio_to_periph_o,

  // Dedicated Peripheral 接口
  input        [NDioPads-1:0]      periph_to_dio_i,
  input        [NDioPads-1:0]      periph_to_dio_oe_i,
  output logic [NDioPads-1:0]      dio_to_periph_o,

  // Pad side 连接到padring
  // MIOs
  output logic [NMioPads-1:0]      mio_out_o,
  output logic [NMioPads-1:0]      mio_oe_o,
  input        [NMioPads-1:0]      mio_in_i,
  // DIOs
  output logic [NDioPads-1:0]      dio_out_o,
  output logic [NDioPads-1:0]      dio_oe_o,
  input        [NDioPads-1:0]      dio_in_i
);
```

`Dedicated Peripheral`不需要进行io映射，直接连接，代码如下：

```systemverilog
  // Inputs are just fed through
  assign dio_to_periph_o = dio_in_i;

  for (genvar k = 0; k < NDioPads; k++) begin : gen_dio_out
    // Since this is a DIO, this can be determined at design time
    if (DioPeriphHasSleepMode[k]) begin : gen_sleep
      assign dio_out_o[k] = (sleep_en_q) ? dio_out_sleep_q[k] : periph_to_dio_i[k];
      assign dio_oe_o[k]  = (sleep_en_q) ? dio_oe_sleep_q[k]  : periph_to_dio_oe_i[k];
    end else begin : gen_nosleep
      assign dio_out_o[k] = periph_to_dio_i[k];
      assign dio_oe_o[k]  = periph_to_dio_oe_i[k];
    end
  end
```

`Muxed Peripheral` io需要映射。通过两个寄存器分别配置输入映射（`periph_insel`）和输出映射（`mio_outsel`）。寄存器含义如下：

| periph_insel | 含义 |
|--------------|------|
| 0            | 输入连接到低电平 |
| 1            | 输入连接到高电平 |
| n>=2         | 输入连接到第n-2个 MIO |

| mio_outsel | 含义 |
|------------|------|
| 0          | 输出低电平 |
| 1          | 输出高电平 |
| 2          | 输出高阻态 |
| n >=3      | 输出连接到低n-3个 Muxed Peripheral |

代码如下：

```systemverilog
  //////////
  // 输入 //
  //////////

  for (genvar k = 0; k < NMioPeriphIn; k++) begin : gen_mio_periph_in
    logic [2**$clog2(NMioPads+2)-1:0] data_mux;
    // 低位补两个分别对应：低电平 高电平
    assign data_mux = $bits(data_mux)'({mio_in_i, 1'b1, 1'b0});
    // 通过寄存器选择输入由
    assign mio_to_periph_o[k] = data_mux[reg2hw.periph_insel[k].q];
  end

  //////////
  // 输出 //
  //////////

  for (genvar k = 0; k < NMioPads; k++) begin : gen_mio_out
    logic sleep_en;
    logic [2**$clog2(NMioPeriphOut+3)-1:0] data_mux, oe_mux, sleep_mux;
    // 低位补三位分别对应：低电平 高电平 高阻态（设置为输入）
    assign data_mux  = $bits(data_mux)'({periph_to_mio_i, 1'b0, 1'b1, 1'b0});
    assign oe_mux    = $bits(oe_mux)'({periph_to_mio_oe_i,  1'b0, 1'b1, 1'b1});
    assign sleep_mux = $bits(sleep_mux)'({MioPeriphHasSleepMode,  1'b1, 1'b1, 1'b1});

    // check whether this peripheral can actually go to sleep
    assign sleep_en = sleep_mux[reg2hw.mio_outsel[k].q] & sleep_en_q;
    // index using configured outsel
    assign mio_out_o[k] = (sleep_en) ? mio_out_sleep_q[k] : data_mux[reg2hw.mio_outsel[k].q];
    assign mio_oe_o[k]  = (sleep_en) ? mio_oe_sleep_q[k]  : oe_mux[reg2hw.mio_outsel[k].q];
  end
```

上面`mio_out_sleep_q`和`mio_oe_sleep_q`用于休眠状态下的输出控制。休眠控制由一个寄存器`sleep_val`控制，寄存器定义如下：

| sleep_val | 含义 |
|-----------|------|
| 0         | 低电平 |
| 1         | 高电平 |
| 2         | 高阻态（设置为输入） |
| 3         | 保持休眠之前的状态 |

代码如下：

```systemverilog
  for (genvar k = 0; k < NDioPads; k++) begin : gen_dio_sleep
    // DioPeriphHasSleepMode[k]用于使能低k个io的休眠功能
    if (DioPeriphHasSleepMode[k]) begin : gen_warl_connect
      assign hw2reg.dio_out_sleep_val[k].d = dio_out_sleep_val_q[k];

      assign dio_out_sleep_val_d[k] = (reg2hw.dio_out_sleep_val[k].qe) ?
                                      reg2hw.dio_out_sleep_val[k].q :
                                      dio_out_sleep_val_q[k];
      // 休眠输出值
      assign dio_out_sleep_d[k] = (dio_out_sleep_val_q[k] == 0) ? 1'b0 :
                                  (dio_out_sleep_val_q[k] == 1) ? 1'b1 :
                                  (dio_out_sleep_val_q[k] == 2) ? 1'b0 : dio_out_o[k];

      // 休眠输出使能
      assign dio_oe_sleep_d[k] = (dio_out_sleep_val_q[k] == 0) ? 1'b1 :
                                 (dio_out_sleep_val_q[k] == 1) ? 1'b1 :
                                 (dio_out_sleep_val_q[k] == 2) ? 1'b0 : dio_oe_o[k];
    end else begin : gen_warl_tie0
      assign hw2reg.dio_out_sleep_val[k].d = 2'b10;
      assign dio_out_sleep_val_d[k] = 2'b10;
      assign dio_out_sleep_d[k]     = '0;
      assign dio_oe_sleep_d[k]      = '0;
    end
  end
```

休眠唤醒由多组，每一组可以：选择使用`Muxed Peripheral`或`Dedicated Peripheral`，并选择第几个io；设定唤醒方式（禁用、下降沿、上升沿、低电平、高电平），电平触发时可以设定电平稳定时间。io选择代码如下：

```systemverilog
    assign pin_value = (reg2hw.wkup_detector[k].miodio.q)           ?
                       dio_data_mux[reg2hw.wkup_detector_padsel[k]] :
                       mio_data_mux[reg2hw.wkup_detector_padsel[k]];
```

唤醒代码由`pinmux_wkup`实现，接口如下：

```systemverilog
module pinmux_wkup import pinmux_pkg::*; import pinmux_reg_pkg::*; #(
  parameter int Cycles = 4
) (
  // 时钟和复位
  input                    clk_i,
  input                    rst_ni,

  // 一直开启的慢速时钟和复位
  input                    clk_aon_i,
  input                    rst_aon_ni,

  input                    wkup_en_i,    // 唤醒使能
  input                    filter_en_i,  // 滤除杂波使能
  input wkup_mode_e        wkup_mode_i,  // 唤醒模式
  input [WkupCntWidth-1:0] wkup_cnt_th_i,// 电平唤醒的计数器
  input                    pin_value_i,  // 引脚值
  // Signals to/from cause register.
  // They are synched to/from the AON clock internally
  input                    wkup_cause_valid_i,
  input                    wkup_cause_data_i,
  output                   wkup_cause_data_o,
  // This signal is running on the AON clock
  // and is held high as long as the cause register
  // has not been cleared.
  output logic             aon_wkup_req_o//输出唤醒请求
);
```

通过`prim_filter`滤除杂波，代码如下：

```systemverilog
  prim_filter #(
    .Cycles(Cycles)
  ) i_prim_filter (
    .clk_i    ( clk_aon_i       ),
    .rst_ni   ( rst_aon_ni      ),
    .enable_i ( aon_filter_en_q ),
    .filter_i ( pin_value_i     ),
    .filter_o ( aon_filter_out  )
  );
```

通过如下代码检测唤醒条件：

```systemverilog
  logic aon_rising, aon_falling;
  assign aon_falling = ~aon_filter_out_d &  aon_filter_out_q; // 检测下降沿
  assign aon_rising  =  aon_filter_out_d & ~aon_filter_out_q; // 检测上升沿

  logic aon_cnt_en, aon_cnt_eq_th;
  logic [WkupCntWidth-1:0] aon_cnt_d, aon_cnt_q;
  assign aon_cnt_d = (aon_cnt_eq_th) ? '0                :
                     (aon_cnt_en)    ?  aon_cnt_q + 1'b1 : '0; // 对电平计数

  assign aon_cnt_eq_th = aon_cnt_q == aon_wkup_cnt_th_q; // 检测计数器达到触发条件

  logic aon_wkup_pulse;
  always_comb begin : p_mode
    aon_wkup_pulse = 1'b0;
    aon_cnt_en     = 1'b0;
    if (aon_wkup_en_q) begin
      // 根据模式输出唤醒信号
      unique case (aon_wkup_mode_q)
        Negedge:   aon_wkup_pulse = aon_falling;
        Posedge:   aon_wkup_pulse = aon_rising;
        Edge:      aon_wkup_pulse = aon_rising | aon_falling;
        LowTimed: begin
          aon_cnt_en = ~aon_filter_out_d; // 低电平时控制计数器自增
          aon_wkup_pulse = aon_cnt_eq_th;
        end
        HighTimed: begin
          aon_cnt_en = aon_filter_out_d;  // 高电平时控制计数器自增
          aon_wkup_pulse = aon_cnt_eq_th;
        end
        default: ; // also covers "Disabled"
      endcase
    end
  end
```

## timer

在理解此部分代码前请参考[Timer HWIP Technical Specification](https://docs.opentitan.org/hw/ip/rv_timer/doc/)

opentitan的timer有两个控制寄存器`prescaler`和`setp`。`prescaler`用于分频，`setp`用于一次增加的值。伪代码如下：

```c
    while (1) {
        tick_count++;
        if (tick_count >= prescaler) {
            tick_count = 0;
            mtime = mtime + setp
        }
    }
```

组要代码位于`hw/ip/rv_timer/rtl/time_core.sv`中，接口如下：

```systemverilog
module timer_core #(
  /* N为计时器个数
   * 计时器共用mtime，有独立的mtimecmp
   */
  parameter int N = 1
) (
  // 时钟和复位信号
  input clk_i,
  input rst_ni,

  // 计时器控制信号
  input        active,
  input [11:0] prescaler,
  input [ 7:0] step,

  output logic        tick,    // 当内部计数器大于等于prescaler时有效，用于锁存mtime_d
  output logic [63:0] mtime_d,
  input        [63:0] mtime,
  input        [63:0] mtimecmp [N],

  output logic [N-1:0] intr   // 中断信号
);
```

内部维护了一个计数器`tick_count`，代码如下：

```systemverilog
  logic [11:0] tick_count;

  always_ff @(posedge clk_i or negedge rst_ni) begin : generate_tick
    if (!rst_ni) begin // 复位清零
      tick_count <= 12'h0;
    end else if (!active) begin // active无效时，计算器不工作
      tick_count <= 12'h0;
    end else if (tick_count == prescaler) begin // 达到计算目标清零
      tick_count <= 12'h0;
    end else begin
      tick_count <= tick_count + 1'b1; // 计数器加1
    end
  end
```

当计数器达到`prescaler`后更新`mtime`，并尝试比较`mtime`与`mtimecmp`输出中断信息，代码如下：

```systemverilog
  // 输出此信号，用于锁存mtime_d
  assign tick = active & (tick_count >= prescaler);

  // 下一个mtime
  assign mtime_d = mtime + 64'(step);

  // 比较mtime与mtimecmp输出中断信号
  for (genvar t = 0 ; t < N ; t++) begin : gen_intr
    assign intr[t] = active & (mtime >= mtimecmp[t]);
  end
```

在`hw/ip/rv_timer/rtl/rv_timer_reg_top.sv`中实现了寄存器。在`hw/ip/rv_timer/rtl/rv_timer.sv`中整合了寄存器和计时器，并连接到总线。

## plic

plic为中断控制器，用于从多个中断源中挑出一个中断输出给cpu。在理解此模块之前请参考[Interrupt Controller Technical Specification](https://docs.opentitan.org/hw/ip/rv_plic/doc/)

opentitan plic中断的关键代码位于`hw/ip/rv_plic/rtl/rv_plic_gateway.sv`中，`rv_plic_gateway`接口如下：

```systemverilog
module rv_plic_gateway #(
  parameter int N_SOURCE = 32 // 中断源个数
) (
  // 时钟和复位
  input clk_i,
  input rst_ni,

  input [N_SOURCE-1:0] src,     // 中断源
  input [N_SOURCE-1:0] le,      // 设置中断触发模式：0->电平，1->上升沿

  input [N_SOURCE-1:0] claim,   // $onehot0(claim)，中断挂起（正在处理的中断）
  input [N_SOURCE-1:0] complete,// $onehot0(complete)，中断完成

  output logic [N_SOURCE-1:0] ip// 输出中断标识位
);
```

通过如下代码，处理中断的触发模式：

```systemverilog
  // 记录上一个周期src到src_d
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) src_d <= '0;
    else         src_d <= src;
  end

  // 根据模式计算中断请求
  always_comb begin
    for (int i = 0 ; i < N_SOURCE; i++) begin
      set[i] = (le[i]) ? src[i] & ~src_d[i] : src[i] ;
    end
  end
```

通过如下代码更新计算`ip`:

```systemverilog
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      ip <= '0;
    end else begin
      // 挂起中断从ip中排除
      ip <= (ip | (set & ~ia & ~ip)) & (~(ip & claim));
    end
  end

  // ia为active的中断
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      ia <= '0;
    end else begin
      // 完成的中断需要从ia中排除
      ia <= (ia | (set & ~ia)) & (~(ia & complete & ~ip));
    end
  end
```

`rv_plic_gateway`一边和中断请求引脚连接，一边和`rc_plic_target`连接。每一个riscv的hart对应一个`rc_plic_target`。`rc_plic_target`用于选择出优先级最高的中断传递给riscv核处理。相关配置有：中断使能，优先级，阀值。`rc_plic_target`接口如下：

```systemverilog
module rv_plic_target #(
  parameter int N_SOURCE = 32,
  parameter int MAX_PRIO = 7,

  // Local param (Do not change this through parameter
  localparam int SrcWidth  = $clog2(N_SOURCE+1),  // derived parameter
  localparam int PrioWidth = $clog2(MAX_PRIO+1)   // derived parameter
) (
  // 时钟和复位
  input clk_i,
  input rst_ni,

  input [N_SOURCE-1:0]  ip, // 中断挂起寄存器
  input [N_SOURCE-1:0]  ie, // 中断使能寄存器

  input [PrioWidth-1:0] prio [N_SOURCE], // 中断优先级
  input [PrioWidth-1:0] threshold,       // 中断阀值

  output logic            irq,           // 连接到hart的中断请求信息
  output logic [SrcWidth-1:0] irq_id     // 选择出的中断的中断编号
);
```

为了选择出优先级最高的异常，创建了一颗树形的比较器，代码如下：

```systemverilog
  // 计算N_SOURCE个叶子节点的完全二叉树有多高
  localparam int NumLevels = $clog2(N_SOURCE);

  // 记录中断信息
  logic [2**(NumLevels+1)-2:0]            is_tree;
  logic [2**(NumLevels+1)-2:0][SrcWidth-1:0]  id_tree;
  logic [2**(NumLevels+1)-2:0][PrioWidth-1:0] max_tree;

  for (genvar level = 0; level < NumLevels+1; level++) begin : gen_tree
    localparam int Base0 = (2**level)-1;
    localparam int Base1 = (2**(level+1))-1;

    for (genvar offset = 0; offset < 2**level; offset++) begin : gen_level
      localparam int Pa = Base0 + offset;
      localparam int C0 = Base1 + 2*offset;
      localparam int C1 = Base1 + 2*offset + 1;

      if (level == NumLevels) begin : gen_leafs
        // 树的叶子节点，填入中断信息
        if (offset < N_SOURCE) begin : gen_assign
          assign is_tree[Pa]  = ip[offset] & ie[offset]; // 有效中断请求
          assign id_tree[Pa]  = offset; //中断编号
          assign max_tree[Pa] = prio[offset]; // 优先级
        end else begin : gen_tie_off
          assign is_tree[Pa]  = '0;
          assign id_tree[Pa]  = '0;
          assign max_tree[Pa] = '0;
        end
      end else begin : gen_nodes
        logic sel;
        // 比较选择信号
        assign sel = (~is_tree[C0] & is_tree[C1]) |
                     (is_tree[C0] & is_tree[C1] & logic'(max_tree[C1] > max_tree[C0]));
        // 选择出高优先级的中断传递给上一层
        assign is_tree[Pa]  = (sel              & is_tree[C1])  | ((~sel)            & is_tree[C0]);
        assign id_tree[Pa]  = ({SrcWidth{sel}}  & id_tree[C1])  | ({SrcWidth{~sel}}  & id_tree[C0]);
        assign max_tree[Pa] = ({PrioWidth{sel}} & max_tree[C1]) | ({PrioWidth{~sel}} & max_tree[C0]);
      end
    end : gen_level
  end : gen_tree
```

在树的根节点就是选择出来的优先级最高的异常。通过如下代码，判断优先级是否超过阀值，并输出：

```systemverilog
  assign irq_d    = (max_tree[0] > threshold) ? is_tree[0] : 1'b0; // 判断优先级是否超过阀值
  assign irq_id_d = (is_tree[0]) ? id_tree[0] : '0;                // 判断是否为有效的中断，输出中断编号
```

在`hw/ip/rv_plic/rtl/rv_plic_reg_top.sv`中实现了中断控制器的寄存器。在`hw/ip/rv_plic/rtl/rv_plic.sv`中整合寄存器、`rv_plic_gateway`和`rv_plic_target`并连接到总线。
