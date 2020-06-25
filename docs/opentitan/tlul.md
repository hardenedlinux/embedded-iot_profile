# 概要

tlul是TileLink的轻量级未缓存的变体

# 接口

它的接口信号定义在`hw/ip/tlul/rtl/tlul_pkg.sv`中
```
  parameter ArbiterImpl = "PPC";

  // 总线上的host可以向设备发送的命令
  typedef enum logic [2:0] {
    PutFullData    = 3'h 0, // 写整个字
    PutPartialData = 3'h 1, // 写一部分，对应sram接口中的be信号
    Get            = 3'h 4  // 读取
  } tl_a_op_e;
  
  // 总线上的device可以响应host的命令
  typedef enum logic [2:0] {
    AccessAck     = 3'h 0, // 应答，这种应答不带数据响应，表示操作完成
    AccessAckData = 3'h 1  // 应答，这种应答带数据响应
  } tl_d_op_e;

  // 总线上host到device的信号
  typedef struct packed {
    logic                         a_valid;   // 信号有效标志
    tl_a_op_e                     a_opcode;  // host请求的操作码
    logic                  [2:0]  a_param;   // 未用
    logic  [top_pkg::TL_SZW-1:0]  a_size;    // 要操作的大小
    logic  [top_pkg::TL_AIW-1:0]  a_source;
    logic   [top_pkg::TL_AW-1:0]  a_address; // 地址
    logic  [top_pkg::TL_DBW-1:0]  a_mask;    // 掩码，用来标识那些字节要写，在tl_a_op_e==PutPartialData时有效
    logic   [top_pkg::TL_DW-1:0]  a_data;    // 写操作时，要写的数据
    tl_a_user_t                   a_user;

    logic                         d_ready;   // host就绪
  } tl_h2d_t;

  // 总线上device到host的信号
  typedef struct packed {
    logic                         d_valid;  // 信号有效
    tl_d_op_e                     d_opcode; // device响应的操作码
    logic                  [2:0]  d_param;  // 未用
    logic  [top_pkg::TL_SZW-1:0]  d_size;   // Bouncing back a_size
    logic  [top_pkg::TL_AIW-1:0]  d_source;
    logic  [top_pkg::TL_DIW-1:0]  d_sink;
    logic   [top_pkg::TL_DW-1:0]  d_data;   // 读操作时，返回读取到的数据
    logic  [top_pkg::TL_DUW-1:0]  d_user;
    logic                         d_error;  // 设备发生设备

    logic                         a_ready;  // device就绪
  } tl_d2h_t;
```

对于一个host设备应该有如下接口
```
	output tl_h2d_t tl_o,
	input  tl_d2h_t tl_i
```

对于一个device设备应该有如下接口
```
	input  tl_h2d_t td_i
	output tl_d2h_t td_o,	
```

如果host和device直接链接，链接方式如下
```
	tl_o ------> td_i
	tl_i <------ td_o
```

# fifo

tlul总线有使用到队列（fifo），队列的实现在`hw/ip/prim/prim_fifo_sync.sv` `hw/ip/prim/prim_fifo_async.sv`。其中：prim\_fifo\_sync是同步队列，读写端口使用相同的时钟和复位信号；prim\_fifo\_async是异步队列，读写端口使用不同的时钟和复位信号。

prim\_fifo\_sync同步队列的接口如下：
```systemverilog
module prim_fifo_sync #(
  parameter int unsigned Width       = 16,
  parameter bit Pass                 = 1'b1, // if == 1 allow requests to pass through empty FIFO
  parameter int unsigned Depth       = 4,
  // derived parameter
  localparam int unsigned DepthWNorm = $clog2(Depth+1),
  localparam int unsigned DepthW     = (DepthWNorm == 0) ? 1 : DepthWNorm
) (
  input                   clk_i,    // 时钟信号
  input                   rst_ni,   // 复位信号
  // synchronous clear / flush port
  input                   clr_i,    // 此信号用于清除队列内缓存的内容
  // write port
  input                   wvalid,   // 写端口，信号有效标识
  output                  wready,   // 写端口，队列就绪可以接受写操着
  input   [Width-1:0]     wdata,    // 写端口，要写的数据
  // read port
  output                  rvalid,   // 读端口，信号有效标识
  input                   rready,   // 读端口，读的设备就绪队列可以输出数据
  output  [Width-1:0]     rdata,    // 读端口，读取到的数据
  // occupancy
  output  [DepthW-1:0]    depth
);
```

prim\_fifo\_async异步队列的接口如下：
```systemverilog
module prim_fifo_async #(
  parameter  int unsigned Width  = 16,
  parameter  int unsigned Depth  = 3,
  localparam int unsigned DepthW = $clog2(Depth+1) // derived parameter representing [0..Depth]
) (
  // write port
  input                  clk_wr_i,  // 写端口，时钟信号
  input                  rst_wr_ni, // 写端口，复位信号
  input                  wvalid,    // 写端口，信号有效标识
  output                 wready,    // 写端口，队列就绪可以接受写操作
  input [Width-1:0]      wdata,     // 写端口，要写的数据
  output [DepthW-1:0]    wdepth,

  // read port
  input                  clk_rd_i,  // 读端口，时钟信号
  input                  rst_rd_ni, // 读端口，复位信号
  output                 rvalid,    // 读端口，信号有效标识
  input                  rready,    // 读端口，读的设备准备就绪队列可以输出数据
  output [Width-1:0]     rdata,     // 读端口，读取到的数据
  output [DepthW-1:0]    rdepth
);
```

其中，当前tlul主要使用的是同步队列prim\_fifo\_sync。队列用来中转缓存信号，接口定义如下
```systemverilog
module tlul_fifo_sync #(
  parameter int unsigned ReqPass  = 1'b1,
  parameter int unsigned RspPass  = 1'b1,
  parameter int unsigned ReqDepth = 2,
  parameter int unsigned RspDepth = 2,
  parameter int unsigned SpareReqW = 1,
  parameter int unsigned SpareRspW = 1
) (
  input                     clk_i,        // 时钟信号
  input                     rst_ni,       // 复位信号
  input  tlul_pkg::tl_h2d_t tl_h_i,       // host接口，host到device的信号
  output tlul_pkg::tl_d2h_t tl_h_o,       // host接口，device到host的信号
  output tlul_pkg::tl_h2d_t tl_d_o,       // device接口，host到device的信号
  input  tlul_pkg::tl_d2h_t tl_d_i,       // device接口，device到host的信号
  input  [SpareReqW-1:0]    spare_req_i,  // 缓存host请求的设备的设备号
  output [SpareReqW-1:0]    spare_req_o,  // 缓存host请求的设备的设备号
  input  [SpareRspW-1:0]    spare_rsp_i,  // 缓存device响应的host请求的host编码
  output [SpareRspW-1:0]    spare_rsp_o   // 缓存device响应的host请求的host编码
);
```

在模块内部维护以下队列
```
    host                         device

   tl_h_i    ---------------->   tl_d_o      此队列维护host发生给设备的信号
 spare_req_i ----------------> spare_req_o   此信号，用于设备选择

   tl_h_o    <----------------   tl_d_i      此队列用于维护device发生给host的响应
 spare_rsp_o <---------------- spare_rsp_i   此信号，用于标识响应的host的编号
```

# 总线逻辑tlul_socket_1n

此模块创造了一个tlul总线，一个host多个device。接口如下
```systemverilog
module tlul_socket_1n #(
  parameter int unsigned  N         = 4,          // 设备个数
  parameter bit           HReqPass  = 1'b1,
  parameter bit           HRspPass  = 1'b1,
  parameter bit [N-1:0]   DReqPass  = {N{1'b1}},
  parameter bit [N-1:0]   DRspPass  = {N{1'b1}},
  parameter bit [3:0]     HReqDepth = 4'h2,
  parameter bit [3:0]     HRspDepth = 4'h2,
  parameter bit [N*4-1:0] DReqDepth = {N{4'h2}},
  parameter bit [N*4-1:0] DRspDepth = {N{4'h2}},
  localparam int unsigned NWD       = $clog2(N+1) // derived parameter
) (
  input                     clk_i,         // 时钟信号
  input                     rst_ni,        // 复位信号
  input  tlul_pkg::tl_h2d_t tl_h_i,        // host接口，host->device的信号
  output tlul_pkg::tl_d2h_t tl_h_o,        // host接口，device->host的信号
  output tlul_pkg::tl_h2d_t tl_d_o    [N], // device接口，host->device的信号
  input  tlul_pkg::tl_d2h_t tl_d_i    [N], // device接口，device->host的信号
  input  [NWD-1:0]          dev_select     // 设备选择信号
);
```

此模块内部给每个接口创建了一个队列，用来缓存信号，如下图
```
host :
 dev_select --------> dev_select_t
   tl_h_i   -------->   tl_t_o
   tl_h_o   <--------   tl_t_i
 
devices:
 tl_u_o[i] -----------> tl_d_i[i]
 tl_u_i[i] <----------- tl_d_o[i]
```

总线要处理的就是把tl\_t\_o连接到tl\_u\_o中的一个，把tl\_u\_i中的一个链接到tl\_t\_i。

系统通过如下代码连接tl\_t\_o连接到tl\_u\_o。其中，通过`(dev_select_t == NWD'(i)`确定只有一路`tl_u_o`是有效的，`hold_all_requests`用于错误检测。
```
  for (genvar i = 0 ; i < N ; i++) begin : gen_u_o
    assign tl_u_o[i].a_valid   = tl_t_o.a_valid &
                                 (dev_select_t == NWD'(i)) &
                                 ~hold_all_requests;
    assign tl_u_o[i].a_opcode  = tl_t_o.a_opcode;
    assign tl_u_o[i].a_param   = tl_t_o.a_param;
    assign tl_u_o[i].a_size    = tl_t_o.a_size;
    assign tl_u_o[i].a_source  = tl_t_o.a_source;
    assign tl_u_o[i].a_address = tl_t_o.a_address;
    assign tl_u_o[i].a_mask    = tl_t_o.a_mask;
    assign tl_u_o[i].a_data    = tl_t_o.a_data;
    assign tl_u_o[i].a_user    = tl_t_o.a_user;
  end
```

通过如下逻辑连接tl\_u\_i到tl\_t\_i
```
  tlul_pkg::tl_d2h_t tl_t_p ;

  // for the returning reqready, only look at the slave we're addressing
  logic hfifo_reqready;
  always_comb begin
    hfifo_reqready = tl_u_i[N].a_ready; // default to error
    for (int idx = 0 ; idx < N ; idx++) begin
      //if (dev_select_outstanding == NWD'(idx)) hfifo_reqready = tl_u_i[idx].a_ready;
      if (dev_select_t == NWD'(idx)) hfifo_reqready = tl_u_i[idx].a_ready;
    end
    if (hold_all_requests) hfifo_reqready = 1'b0;
  end
  // Adding a_valid as a qualifier. This prevents the a_ready from having unknown value
  // when the address is unknown and the Host TL-UL FIFO is bypass mode.
  assign tl_t_i.a_ready = tl_t_o.a_valid & hfifo_reqready;

  always_comb begin
    tl_t_p = tl_u_i[N];
    for (int idx = 0 ; idx < N ; idx++) begin
      if (dev_select_outstanding == NWD'(idx)) tl_t_p = tl_u_i[idx];
    end
  end
  assign tl_t_i.d_valid  = tl_t_p.d_valid ;
  assign tl_t_i.d_opcode = tl_t_p.d_opcode;
  assign tl_t_i.d_param  = tl_t_p.d_param ;
  assign tl_t_i.d_size   = tl_t_p.d_size  ;
  assign tl_t_i.d_source = tl_t_p.d_source;
  assign tl_t_i.d_sink   = tl_t_p.d_sink  ;
  assign tl_t_i.d_data   = tl_t_p.d_data  ;
  assign tl_t_i.d_user   = tl_t_p.d_user  ;
  assign tl_t_i.d_error  = tl_t_p.d_error ;
```

# socket_m1

此模块实现了一个多host单device的总线结构。其中，为了竞争总线权限使用到了仲裁器。仲裁器接口如下：
```
module prim_arbiter_ppc #(
  parameter int unsigned N  = 4,
  parameter int unsigned DW = 32,

  // Configurations
  // EnDataPort: {0, 1}, if 0, input data will be ignored
  parameter int EnDataPort = 1
) (
  input clk_i,                              // 时钟信号
  input rst_ni,                             // 复位信号

  input        [ N-1:0]        req_i,       // 请求信号
  input        [DW-1:0]        data_i [N],  // 输入信号
  output logic [ N-1:0]        gnt_o,       // 
  output logic [$clog2(N)-1:0] idx_o,       // 输出当前选择信号的编号

  output logic          valid_o,            // 输出有效标识
  output logic [DW-1:0] data_o,             // 输出选择的信号的值
  input                 ready_i             // 输入信号有效
);
```

此模块有N组信号输入`data_i[N]`，`req_i`标识请求那些信号，仲裁器根据自己的逻辑选择出一组信号输出。

socket\_m1接口如下
```
module tlul_socket_m1 #(
  parameter int unsigned  M         = 4,
  parameter bit [M-1:0]   HReqPass  = {M{1'b1}},
  parameter bit [M-1:0]   HRspPass  = {M{1'b1}},
  parameter bit [M*4-1:0] HReqDepth = {M{4'h2}},
  parameter bit [M*4-1:0] HRspDepth = {M{4'h2}},
  parameter bit           DReqPass  = 1'b1,
  parameter bit           DRspPass  = 1'b1,
  parameter bit [3:0]     DReqDepth = 4'h2,
  parameter bit [3:0]     DRspDepth = 4'h2
) (
  input                     clk_i,       // 时钟信号
  input                     rst_ni,      // 复位信号

  input  tlul_pkg::tl_h2d_t tl_h_i [M],  // host接口，host->device的信号
  output tlul_pkg::tl_d2h_t tl_h_o [M],  // host接口，device->host的信号

  output tlul_pkg::tl_h2d_t tl_d_o,      // device接口，host->device的信号
  input  tlul_pkg::tl_d2h_t tl_d_i       // device接口，device->host的信号
);
```

模块内部创建了一些如下队列来缓冲信号
```
 host:
   tl_h_i[M] ------------> hreq_fifo_o[M]
   tl_h_o[M] <------------ hrsp_fifo_i[M]

 device:
   dreq_fifo_i -----------> tl_d_o
   drsp_fifo_o <----------- tl_d_i
```

为了使设备知道信号来自哪个主机，把主机编号编码进了a\_source。代码如下：
```
    // ID Shifting
    logic [STIDW-1:0] reqid_sub;
    logic [IDW-1:0] shifted_id;
    assign reqid_sub = i;   // i为设备编号
    assign shifted_id = {
      tl_h_i[i].a_source[0+:(IDW-STIDW)], // 高位保留原本的内容
      reqid_sub                           // 低位存放设备编号
    };
    
    assign hreq_fifo_i = '{
      a_valid:    tl_h_i[i].a_valid,
      a_opcode:   tl_h_i[i].a_opcode,
      a_param:    tl_h_i[i].a_param,
      a_size:     tl_h_i[i].a_size,
      a_source:   shifted_id,
      a_address:  tl_h_i[i].a_address,
      a_mask:     tl_h_i[i].a_mask,
      a_data:     tl_h_i[i].a_data,
      a_user:     tl_h_i[i].a_user,
      d_ready:    tl_h_i[i].d_ready
    };
```

然后通过仲裁器，选出一组主机信号hreq\_fifo\_o连接到dreq\_fifo\_i。

总裁器，选择信号的相关代码如下
```
  // 构建一个标识那些信号有效
  for (genvar i = 0 ; i < M ; i++) begin : gen_arbreqgnt
    assign hrequest[i] = hreq_fifo_o[i].a_valid;
  end

  assign arb_ready = drsp_fifo_o.a_ready;
 
    prim_arbiter_ppc #(
      .N      (M),
      .DW     ($bits(tlul_pkg::tl_h2d_t))
    ) u_reqarb (
      .clk_i,
      .rst_ni,
      .req_i   ( hrequest    ),
      .data_i  ( hreq_fifo_o ),
      .gnt_o   ( hgrant      ),
      .idx_o   (             ),
      .valid_o ( arb_valid   ),
      .data_o  ( arb_data    ),
      .ready_i ( arb_ready   )
    );
```

选择出的信号通过arb\_data输出，然后连接到dreq\_fifo\_i。代码如下
```
  assign dreq_fifo_i = '{
    a_valid:   arb_valid,
    a_opcode:  arb_data.a_opcode,
    a_param:   arb_data.a_param,
    a_size:    arb_data.a_size,
    a_source:  arb_data.a_source,
    a_address: arb_data.a_address,
    a_mask:    arb_data.a_mask,
    a_data:    arb_data.a_data,
    a_user:    arb_data.a_user,

    d_ready:   dfifo_rspready_merged
  };
```

连接drsp\_fifo\_o到hrsp\_fifo\_i时会根据d_source，来设定信号是否有效，来把一组信号拆分为多组信号。代码如下：
```
  for (genvar i = 0 ; i < M ; i++) begin : gen_idrouting
    assign hfifo_rspvalid[i] = drsp_fifo_o.d_valid &
                               (drsp_fifo_o.d_source[0+:STIDW] == i);  // 通过判断d_source是否等于i，来设置d_valid
    assign dfifo_rspready[i] = hreq_fifo_o[i].d_ready                &
                               (drsp_fifo_o.d_source[0+:STIDW] == i) &
                              drsp_fifo_o.d_valid;

    assign hrsp_fifo_i[i] = '{
      d_valid:  hfifo_rspvalid[i],
      d_opcode: drsp_fifo_o.d_opcode,
      d_param:  drsp_fifo_o.d_param,
      d_size:   drsp_fifo_o.d_size,
      d_source: hfifo_rspid,
      d_sink:   drsp_fifo_o.d_sink,
      d_data:   drsp_fifo_o.d_data,
      d_user:   drsp_fifo_o.d_user,
      d_error:  drsp_fifo_o.d_error,
      a_ready:  hgrant[i]
    };
  end
```

# 适配器

适配器用于把其他类型的接口转换为tlul，系统实现了三个适配器：
- tlul\_adapter\_host sram接口主设备转tlul host接口
- tlul\_adapter\_sram sram接口从设备转tlul device设备
- tlul\_adapter\_reg reg接口转tlul device设备




















