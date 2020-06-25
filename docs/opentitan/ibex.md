# 简介

opentitan使用的cpu内核是来自lowRISC的ibex。此文档用于分析ibex的架构。

## 指令架构

当前ibex为32位RISC-V处理器。支持I/E/C/M/B扩展。根据RISC-V规范，I/E为基础扩展，二选一。ibex把C扩展设计为默认支持的扩展。M/B为可选支持的扩展。

## 流水线

当前ibex流水线，可以配置为两级流水或三级流水。分别如下：  

- 取指 -> 译码/执行
- 取指 -> 译码/执行 -> 回写

## 异常

根据RISC-V规范，m-mode异常入口通过mtvec设置，可以多个异常共用一个异常地址base，也可以每个异常使用独立的如可base + 4 * cause。具体使用哪种方式，由mtvec低两比特决定，为0时共享异常地址，为1是使用各自的异常地址。

ibex的mtvec低两位固定为1。这种模式下mtvec指向的内存，连续存放一些跳转指令。ibex把复位向量存放到此处，mcause = 32。并且ibex\_if\_stage会在启动时初始化mtvec为boot\_addr，并到boot\_addr + 4 * 32处取指。

## 计数器

RISC-V规范只规定了几个计算器，用来对什么事件计数，ibex计数器对以下事件计数

```systemverilog
  // event selection (hardwired) & control
  always_comb begin : gen_mhpmcounter_incr

    // Assign inactive counters (first to prevent latch inference)
    for (int unsigned i=0; i<32; i++) begin : gen_mhpmcounter_incr_inactive
      mhpmcounter_incr[i] = 1'b0;
    end

    // When adding or altering performance counter meanings and default
    // mappings please update dv/verilator/pcount/cpp/ibex_pcounts.cc
    // appropriately.
    //
    // active counters
    mhpmcounter_incr[0]  = 1'b1;                   // mcycle
    mhpmcounter_incr[1]  = 1'b0;                   // reserved
    mhpmcounter_incr[2]  = instr_ret_i;            // minstret
    mhpmcounter_incr[3]  = dside_wait_i;           // cycles waiting for data memory
    mhpmcounter_incr[4]  = iside_wait_i;           // cycles waiting for instr fetches
    mhpmcounter_incr[5]  = mem_load_i;             // num of loads
    mhpmcounter_incr[6]  = mem_store_i;            // num of stores
    mhpmcounter_incr[7]  = jump_i;                 // num of jumps (unconditional)
    mhpmcounter_incr[8]  = branch_i;               // num of branches (conditional)
    mhpmcounter_incr[9]  = branch_taken_i;         // num of taken branches (conditional)
    mhpmcounter_incr[10] = instr_ret_compressed_i; // num of compressed instr
    mhpmcounter_incr[11] = mul_wait_i;             // cycles waiting for multiply
    mhpmcounter_incr[12] = div_wait_i;             // cycles waiting for divide
  end
```

# 源码分析

## ibex_core

此模块为ibex的顶层模块，此模块为一个riscv的hart。接口如下：  

```systemverilog
module ibex_core #(
    parameter bit          PMPEnable                = 1'b0,
    parameter int unsigned PMPGranularity           = 0,
    parameter int unsigned PMPNumRegions            = 4,
    parameter int unsigned MHPMCounterNum           = 0,
    parameter int unsigned MHPMCounterWidth         = 40,
    parameter bit          RV32E                    = 1'b0, // 用于配置是否支持E扩展
    parameter bit          RV32M                    = 1'b1, // 用于配置是否支持M扩展
    parameter bit          RV32B                    = 1'b0, // 用于配置是否支持B扩展
    parameter bit          BranchTargetALU          = 1'b0,
    parameter bit          WritebackStage           = 1'b0,
    parameter              MultiplierImplementation = "fast",
    parameter bit          ICache                   = 1'b0,
    parameter bit          ICacheECC                = 1'b0,
    parameter bit          DbgTriggerEn             = 1'b0,
    parameter bit          SecureIbex               = 1'b0,
    parameter int unsigned DmHaltAddr               = 32'h1A110800,
    parameter int unsigned DmExceptionAddr          = 32'h1A110808
) (
    // 时钟和复位信号
    input  logic        clk_i,
    input  logic        rst_ni,

    input  logic        test_en_i,     // enable all clock gates for testing

    input  logic [31:0] hart_id_i,     // hartid
    input  logic [31:0] boot_addr_i,   // 启动地址，此地址存放异常向量表，复位向量为32

    // 指令内存接口
    output logic        instr_req_o,
    input  logic        instr_gnt_i,
    input  logic        instr_rvalid_i,
    output logic [31:0] instr_addr_o,
    input  logic [31:0] instr_rdata_i,
    input  logic        instr_err_i,

    // 数据内存接口
    output logic        data_req_o,
    input  logic        data_gnt_i,
    input  logic        data_rvalid_i,
    output logic        data_we_o,
    output logic [3:0]  data_be_o,
    output logic [31:0] data_addr_o,
    output logic [31:0] data_wdata_o,
    input  logic [31:0] data_rdata_i,
    input  logic        data_err_i,

    // 外设链接到hart的中断信号
    input  logic        irq_software_i,
    input  logic        irq_timer_i,
    input  logic        irq_external_i,
    input  logic [14:0] irq_fast_i,
    input  logic        irq_nm_i,       // non-maskeable interrupt

    // Debug Interface
    input  logic        debug_req_i,

    // RISC-V Formal Interface
    // Does not comply with the coding standards of _i/_o suffixes, but follows
    // the convention of RISC-V Formal Interface Specification.
`ifdef RVFI
    output logic        rvfi_valid,
    output logic [63:0] rvfi_order,
    output logic [31:0] rvfi_insn,
    output logic        rvfi_trap,
    output logic        rvfi_halt,
    output logic        rvfi_intr,
    output logic [ 1:0] rvfi_mode,
    output logic [ 1:0] rvfi_ixl,
    output logic [ 4:0] rvfi_rs1_addr,
    output logic [ 4:0] rvfi_rs2_addr,
    output logic [ 4:0] rvfi_rs3_addr,
    output logic [31:0] rvfi_rs1_rdata,
    output logic [31:0] rvfi_rs2_rdata,
    output logic [31:0] rvfi_rs3_rdata,
    output logic [ 4:0] rvfi_rd_addr,
    output logic [31:0] rvfi_rd_wdata,
    output logic [31:0] rvfi_pc_rdata,
    output logic [31:0] rvfi_pc_wdata,
    output logic [31:0] rvfi_mem_addr,
    output logic [ 3:0] rvfi_mem_rmask,
    output logic [ 3:0] rvfi_mem_wmask,
    output logic [31:0] rvfi_mem_rdata,
    output logic [31:0] rvfi_mem_wdata,
`endif

    // cpu控制信号
    input  logic        fetch_enable_i,   // 此信号在cpu复位后控制cpu是否开始执行
    output logic        core_sleep_o
);
```

ibex\_core中包含以下模块分别负责如下工作：  

- ibex\_if\_stage : 此模块负责从指令存储器获取指令，并解码压缩指令
- ibex\_id\_stage : 此模块负责指令解码和控制
- ibex\_ex\_block : 此模块由译码执行模块控制负责计算
- ibex\_load\_store\_unit : 此模块由译码执行模块控制，负责数据内存读写
- ibex\_wb\_stage : 此模块由译码执行模块控制，负责回写计算结构到寄存器
- ibex\_register\_file : 寄存器模块
- ibex\_cs\_registers : 控制状态寄存器，并由取指译码模块控制负责异常状态的保存和恢复
- ibex_pmp : 此模块监控pmp寄存器和数据指令存储器的地址，计算输出pmp错误信息

具体连接方式如下：  

```
+--------------------+                         +--------------------+
| instruction memory |                         | data memory        |
+--------------------+                         +--------------------+
    |                                                   ↑
    |                  +--------------------------------+
    |                  ↓                                |
    |            +----------+     +--------+            |
    +----------->| PMP      |<----|  CSRs  |            |
    |            +----------+     +--------+            |
    |              |      |        |  ↑                 ↓
    |  +-----------|------|--------+  |          +-----------------+
    |  |  +--------+      |           |          | load store unit |<----+
    ↓  ↓  ↓               ↓           ↓          +-----------------+     |
+-----------+      +---------------------+          ↑   ↑                |
| IF        |      |  ID                 |<---------+   |         +------------+
|           |<---->|                     |              |         | EX         |
|           |      |                     |<-------------|-------->|            |
+-----------+      +---------------------+              |         +------------+
    ↑                     ↑      ↑    |                 |                |
    |                     |      |    |          +----------------+      |
    |           +---------+      |    +--------->| WB             |      |
    |           |                |               +----------------+      |
    |           |                |                       ↓               |
    |  +--------------------+  +----------------------------------+      |
    |  | interrupt pins     |  | Register file                    |      |
    |  +--------------------+  +----------------------------------+      |
    +--------------------------------------------------------------------+
```

## ibex_if_stage

此模块被ID模块控制，从指令存储器获取指令并传递给ID模块。接口如下：
```
module ibex_if_stage #(
    parameter int unsigned DmHaltAddr        = 32'h1A110800,
    parameter int unsigned DmExceptionAddr   = 32'h1A110808,
    parameter bit          DummyInstructions = 1'b0,
    parameter bit          ICache            = 1'b0,
    parameter bit          ICacheECC         = 1'b0
) (
	// 时钟和复位
    input  logic                   clk_i,
    input  logic                   rst_ni,

    input  logic [31:0]            boot_addr_i,              // 启动地址，复位时mtvec会被设置为此值
    input  logic                   req_i,                    // 此信号来自ID，用于请求指令

    // 指令存储器接口
    output logic                  instr_req_o,
    output logic [31:0]           instr_addr_o,
    input  logic                  instr_gnt_i,
    input  logic                  instr_rvalid_i,
    input  logic [31:0]           instr_rdata_i,
    input  logic                  instr_err_i,
    input  logic                  instr_pmp_err_i,          // PMP模块传递过来的错误信息

    // 输出到ID的指令信息
    output logic                  instr_valid_id_o,         // instr in IF-ID is valid
    output logic                  instr_new_id_o,           // instr in IF-ID is new
    output logic [31:0]           instr_rdata_id_o,         // instr for ID stage
    output logic [31:0]           instr_rdata_alu_id_o,     // replicated instr for ID stage
                                                            // to reduce fan-out
    output logic [15:0]           instr_rdata_c_id_o,       // compressed instr for ID stage
                                                            // (mtval), meaningful only if
                                                            // instr_is_compressed_id_o = 1'b1
    output logic                  instr_is_compressed_id_o, // compressed decoder thinks this
                                                            // is a compressed instr
    output logic                  instr_fetch_err_o,        // bus error on fetch
    output logic                  instr_fetch_err_plus2_o,  // bus error misaligned
    output logic                  illegal_c_insn_id_o,      // compressed decoder thinks this
                                                            // is an invalid instr
    output logic                  dummy_instr_id_o,         // Instruction is a dummy
    output logic [31:0]           pc_if_o,                  // IF阶段的PC
    output logic [31:0]           pc_id_o,                  // ID阶段的PC

    // 控制信号，来自ID
    input  logic                  instr_valid_clear_i,      // clear instr valid bit in IF-ID
    input  logic                  pc_set_i,                 // set the PC to a new value
    input  logic                  pc_set_spec_i,
    input  ibex_pkg::pc_sel_e     pc_mux_i,                 // selector for PC multiplexer
    input  ibex_pkg::exc_pc_sel_e exc_pc_mux_i,             // selects ISR address
    input  ibex_pkg::exc_cause_e  exc_cause,                // selects ISR address for
                                                            // vectorized interrupt lines
    input logic                   dummy_instr_en_i,
    input logic [2:0]             dummy_instr_mask_i,
    input logic                   dummy_instr_seed_en_i,
    input logic [31:0]            dummy_instr_seed_i,
    input logic                   icache_enable_i,
    input logic                   icache_inval_i,

    // 跳转和分支的地址，来自ibex_ex_block
    input  logic [31:0]           branch_target_ex_i,       // branch/jump target address

    // 来自CSRs的一些值
    input  logic [31:0]           csr_mepc_i,               // PC to restore after handling
                                                            // the interrupt/exception
    input  logic [31:0]           csr_depc_i,               // PC to restore after handling
                                                            // the debug request
    input  logic [31:0]           csr_mtvec_i,              // base PC to jump to on exception
    output logic                  csr_mtvec_init_o,         // tell CS regfile to init mtvec

    // pipeline stall
    input  logic                  id_in_ready_i,            // 此信号来自ID模块，用于控制IF模块取指

    // misc signals
    output logic                  if_busy_o                 // IF stage is busy fetching instr
);
```

此模块与其他模块的连接关系如下：
- 从指令内存读取指令
- 访问指令内存的地址传递给PMP，PMP告知是否触发PMP错误
- 从CSR读取一些寄存器，这些寄存器主要用于异常发生和异常退出时地址选择（mepc/depc/mtvec）
- 从EX读取反回值(ALU计算跳转目标地址)，用来在分支跳转时设置PC
- 把从内存获取到的指令传递给ID，并被ID模块控制。


取指地址选择相关代码如下：
```
  // extract interrupt ID from exception cause
  assign irq_id         = {exc_cause}; // ID模块传递过来的异常原因
  assign unused_irq_bit = irq_id[5];   // MSB distinguishes interrupts from exceptions

  // exception PC selection mux
  always_comb begin : exc_pc_mux
    unique case (exc_pc_mux_i) // 根据ID模块传递过来的异常PC选择信号选择异常入口地址
      EXC_PC_EXC:     exc_pc = { csr_mtvec_i[31:8], 8'h00                    }; // 异常
      EXC_PC_IRQ:     exc_pc = { csr_mtvec_i[31:8], 1'b0, irq_id[4:0], 2'b00 }; // 中断
      EXC_PC_DBD:     exc_pc = DmHaltAddr;                                      // 调试
      EXC_PC_DBG_EXC: exc_pc = DmExceptionAddr;                                 // 
      default:        exc_pc = { csr_mtvec_i[31:8], 8'h00                    };
    endcase
  end

  // fetch address selection mux
  always_comb begin : fetch_addr_mux
    unique case (pc_mux_i)   // 根据ID模块传递过来的PC选择信号选择取指地址
      PC_BOOT: fetch_addr_n = { boot_addr_i[31:8], 8'h80 }; // 复位，复位异常位于异常向量表32, 偏移量32 × 4 -> 0x80
      PC_JUMP: fetch_addr_n = branch_target_ex_i;           // 分支指令从ALU获取目标地址
      PC_EXC:  fetch_addr_n = exc_pc;                       // 异常发生时
      PC_ERET: fetch_addr_n = csr_mepc_i;                   // 异常返回
      PC_DRET: fetch_addr_n = csr_depc_i;                   // 调试异常返回
      default: fetch_addr_n = { boot_addr_i[31:8], 8'h80 };
    endcase
  end
```

上面代码将产生一个取指的地址fetch\_addr\_n，此地址将配合pc\_set\_i，传递给一个实际负责取指的模块ibex\_icache / ibex\_prefetch\_buffer，用于设定当前从何处开始取指令，此模块负责取指令并维护地址信息。它们由类似的接口，接口如下：

```
module ibex_prefetch_buffer (
	// 时钟和复位信号
    input  logic        clk_i,
    input  logic        rst_ni,
	
    input  logic        req_i,             // 取指请求

    input  logic        branch_i,          // 此信号连接到pc_set_i，用来设定新的取指地址addr_i
    input  logic        branch_spec_i,
    input  logic [31:0] addr_i,


    input  logic        ready_i,           // 后续设备就绪，可以接受新指令
    output logic        valid_o,           // 输出的数据有效
    output logic [31:0] rdata_o,           // 输出的指令
    output logic [31:0] addr_o,            // 当前输出指令的地址
    output logic        err_o,
    output logic        err_plus2_o,


    // 指令内存内存接口
    output logic        instr_req_o,
    input  logic        instr_gnt_i,
    output logic [31:0] instr_addr_o,
    input  logic [31:0] instr_rdata_i,
    input  logic        instr_err_i,
    input  logic        instr_pmp_err_i,
    input  logic        instr_rvalid_i,

    // Prefetch Buffer Status
    output logic        busy_o
);
```

以上模块会判断当前指令是否为压缩指令，来维护取指地址。输出的32bit数据需要通过ibex\_compressed\_decoder模块进行指令解压。指令解压模块是一个组合逻辑模块，把对应的压缩指令转换为对应的普通指令。ibex\_compressed\_decoder接口如下：  
```
module ibex_compressed_decoder (
	// 时钟复位信号
    input  logic        clk_i,
    input  logic        rst_ni,
    
    input  logic        valid_i,          // 输入有效信号
    input  logic [31:0] instr_i,          // 指令输入
    output logic [31:0] instr_o,          // 指令输出
    output logic        is_compressed_o,  // 输出指令是否为压缩指令
    output logic        illegal_instr_o   // 输出非法指令（压缩指令编码出错）
);
```

ibex为了防止测信道攻击，在IF中添加了一个模块ibex\_dummy\_instr，此模块可以随机生成一些指令（ADD/MUL/DIV/AND，目标寄存器为X0，不影响执行结果）插入到正常的指令序列中。此模块使用线性反馈移位寄存器来产生伪随机数，然后用伪随机数来产生随机指令。为了实现此功能ibex添加了两个寄存器CPUCTRL/SECURESEED，这两个寄存器的功能如下：
- CPUCTRL，其中由两个位于与此功能相关：dummy\_instr\_en，dummy\_instr\_mask。dummy\_instr\_en用于使能此功能，dummy\_instr\_mask用于设定插入指令的频率。
- SECURESEED，用于设定伪随机数发生器的种子

| 值   | 描述                              |
|:-----|:----------------------------------|
|0b000 | 每0 -  4条真实指令插入一条dummy指令 |
|0b001 | 每0 -  8条真实指令插入一条dummy指令 |
|0b011 | 每0 - 16条真实指令插入一条dummy指令 |
|0b111 | 每0 - 32条真实指令插入一条dummy指令 |


## ibex_id_stage

此模块是整个ibex的核心，其他模块都由此模块控制。它主要由译码模块和控制模块组成。

译码模块是一个组合逻辑模块，主要负责解析出指令中要访问的寄存器地址立即数等，接口如下：
```
module ibex_decoder #(
    parameter bit RV32E           = 0, // 是否支持E扩展，不支持需要输出非法指令信号
    parameter bit RV32M           = 1, // 是否支持M扩展，不支持需要输出非法指令信号
    parameter bit RV32B           = 0, // 是否支持B扩展，不支持需要输出非法指令信号
    parameter bit BranchTargetALU = 0
) (
	// 时钟和复位信号
    input  logic                 clk_i,
    input  logic                 rst_ni,

    // 与控制器的接口
    output logic                 illegal_insn_o,        // 当前指令为非法指令
    output logic                 ebrk_insn_o,           // 当前指令为ebreak
    output logic                 mret_insn_o,           // 当前指令为mret
    output logic                 dret_insn_o,           // 当前指令为dret
    output logic                 ecall_insn_o,          // 当前指令为mcall
    output logic                 wfi_insn_o,            // 当前指令为wfi
    output logic                 jump_set_o,            // 当前指令为跳转指令
    input  logic                 branch_taken_i,        // 控制器输入信号，控制计算操作数B的选择
    
    output logic                 icache_inval_o,        // 此信号传递给控制器用于清除指令缓存

    // from IF-ID pipeline register
    input  logic                 instr_first_cycle_i,   // instruction read is in its first cycle
    input  logic [31:0]          instr_rdata_i,         // 来自IF模块的指令
    input  logic [31:0]          instr_rdata_alu_i,     // 来自IF模块的指令，与instr_rdata_i相同，防止扇出过多

    input  logic                 illegal_c_insn_i,      // compressed instruction decode failed

    // 立即数
    output ibex_pkg::imm_a_sel_e  imm_a_mux_sel_o,       // immediate selection for operand a
    output ibex_pkg::imm_b_sel_e  imm_b_mux_sel_o,       // immediate selection for operand b
    output ibex_pkg::op_a_sel_e   bt_a_mux_sel_o,        // branch target selection operand a
    output ibex_pkg::imm_b_sel_e  bt_b_mux_sel_o,        // branch target selection operand b
    output logic [31:0]           imm_i_type_o,
    output logic [31:0]           imm_s_type_o,
    output logic [31:0]           imm_b_type_o,
    output logic [31:0]           imm_u_type_o,
    output logic [31:0]           imm_j_type_o,
    output logic [31:0]           zimm_rs1_type_o,       // CSR相关操作（CSRRWI/CSRRSI/CSRRCI）使用的立即数

    // 寄存器
    output ibex_pkg::rf_wd_sel_e rf_wdata_sel_o,      // 要被写入的寄存器（EX计算结果 或CSR）
    output logic                 rf_we_o,             // 写使能信号
    output logic [4:0]           rf_raddr_a_o,        // 要读出的寄存器地址
    output logic [4:0]           rf_raddr_b_o,        // 要读出的寄存器地址
    output logic [4:0]           rf_waddr_o,          // 要被写的寄存器地址
    output logic                 rf_ren_a_o,          // Instruction reads from RF addr A
    output logic                 rf_ren_b_o,          // Instruction reads from RF addr B

    // ALU
    output ibex_pkg::alu_op_e    alu_operator_o,        // ALU operation selection
    output ibex_pkg::op_a_sel_e  alu_op_a_mux_sel_o,    // operand a selection: reg value, PC,
                                                        // immediate or zero
    output ibex_pkg::op_b_sel_e  alu_op_b_mux_sel_o,    // operand b selection: reg value or
                                                        // immediate
    output logic                 alu_multicycle_o,      // ternary bitmanip instruction

    // MULT & DIV
    output logic                 mult_en_o,             // perform integer multiplication
    output logic                 div_en_o,              // perform integer division or remainder
    output logic                 mult_sel_o,            // as above but static, for data muxes
    output logic                 div_sel_o,             // as above but static, for data muxes

    output ibex_pkg::md_op_e     multdiv_operator_o,
    output logic [1:0]           multdiv_signed_mode_o,

    // CSRs
    output logic                 csr_access_o,          // access to CSR
    output ibex_pkg::csr_op_e    csr_op_o,              // CSR要执行的操着（READ/WRITE/SET/CLEAR）

    // LSU
    output logic                 data_req_o,            // 数据访问请求信号
    output logic                 data_we_o,             // 写使能
    output logic [1:0]           data_type_o,           // 内存访问类型，字节/半字/字
    output logic                 data_sign_extension_o, // 读取时是否执行符号位扩展

    // jump/branches
    output logic                 jump_in_dec_o,         // jump is being calculated in ALU
    output logic                 branch_in_dec_o
);
```
控制器接口如下：  
```
module ibex_controller #(
    parameter bit WritebackStage = 0
 ) (
    // 时钟和复位信号
    input  logic                  clk_i,
    input  logic                  rst_ni,

    input  logic                  fetch_enable_i,        // cpu的控制信号，在cpu启动时，控制cpu开始取指
    output logic                  ctrl_busy_o,           // 输出信号标识当前cpu正在处理指令

    // 译码器输出的信号
    input  logic                  illegal_insn_i,          // decoder has an invalid instr
    input  logic                  ecall_insn_i,            // decoder has ECALL instr
    input  logic                  mret_insn_i,             // decoder has MRET instr
    input  logic                  dret_insn_i,             // decoder has DRET instr
    input  logic                  wfi_insn_i,              // decoder has WFI instr
    input  logic                  ebrk_insn_i,             // decoder has EBREAK instr
    input  logic                  csr_pipe_flush_i,        // do CSR-related pipeline flush

    // 来自取指模块的信号
    input  logic                  instr_valid_i,           // instr from IF-ID reg is valid
    input  logic [31:0]           instr_i,                 // instr from IF-ID reg, for mtval
    input  logic [15:0]           instr_compressed_i,      // instr from IF-ID reg, for mtval
    input  logic                  instr_is_compressed_i,   // instr from IF-ID reg is compressed
    input  logic                  instr_fetch_err_i,       // instr from IF-ID reg has error
    input  logic                  instr_fetch_err_plus2_i, // instr from IF-ID reg error is x32
    input  logic [31:0]           pc_id_i,                 // instr from IF-ID reg address

    // 传递给取指模块的控制信号
    output logic                  instr_valid_clear_o,     // kill instr in IF-ID reg
    output logic                  id_in_ready_o,           // ID stage is ready for new instr
    output logic                  controller_run_o,        // Controller is in standard instruction
                                                           // run mode

    // 传递给预取模块的信号，用于分支地址选择
    output logic                  instr_req_o,             // start fetching instructions
    output logic                  pc_set_o,                // jump to address set by pc_mux
    output logic                  pc_set_spec_o,           // speculative branch
    output ibex_pkg::pc_sel_e     pc_mux_o,                // IF stage fetch address selector
                                                           // (boot, normal, exception...)
    output ibex_pkg::exc_pc_sel_e exc_pc_mux_o,            // IF stage selector for exception PC
    output ibex_pkg::exc_cause_e  exc_cause_o,             // for IF stage, CSRs

    // LSU
    input  logic [31:0]           lsu_addr_last_i,         // for mtval
    input  logic                  load_err_i,
    input  logic                  store_err_i,
    output logic                  wb_exception_o,          // Instruction in WB taking an exception

    // 跳转分支信号
    input  logic                  branch_set_i,            // branch taken set signal
    input  logic                  branch_set_spec_i,       // speculative branch signal
    input  logic                  jump_set_i,              // jump taken set signal

    // 中断信号
    input  logic                  csr_mstatus_mie_i,       // M-mode interrupt enable bit
    input  logic                  irq_pending_i,           // interrupt request pending
    input  ibex_pkg::irqs_t       irqs_i,                  // interrupt requests qualified with
                                                           // mie CSR
    input  logic                  irq_nm_i,                // non-maskeable interrupt
    output logic                  nmi_mode_o,              // core executing NMI handler

    // 调试信号
    input  logic                  debug_req_i,
    output ibex_pkg::dbg_cause_e  debug_cause_o,
    output logic                  debug_csr_save_o,
    output logic                  debug_mode_o,
    input  logic                  debug_single_step_i,
    input  logic                  debug_ebreakm_i,
    input  logic                  debug_ebreaku_i,
    input  logic                  trigger_match_i,

    output logic                  csr_save_if_o,
    output logic                  csr_save_id_o,
    output logic                  csr_save_wb_o,
    output logic                  csr_restore_mret_id_o,
    output logic                  csr_restore_dret_id_o,
    output logic                  csr_save_cause_o,
    output logic [31:0]           csr_mtval_o,
    input  ibex_pkg::priv_lvl_e   priv_mode_i,
    input  logic                  csr_mstatus_tw_i,

    // stall & flush signals
    input  logic                  lsu_req_in_id_i,
    input  logic                  stall_id_i,
    input  logic                  stall_wb_i,
    output logic                  flush_id_o,
    input  logic                  ready_wb_i,

    // performance monitors
    output logic                  perf_jump_o,             // we are executing a jump
                                                           // instruction (j, jr, jal, jalr)
    output logic                  perf_tbranch_o           // we are executing a taken branch
                                                           // instruction
);
```
控制器内部维护了一个状态机，控制cpu的启动、运行和异常处理等操作，状态机如下：  

![控制器状态机](.ibex_controller_status.png)

- RESET: 复位后的默认状态  
- BOOT\_SET: 复位后的一个状态用于，用于拷贝启动地址到取指单元  
- FIRST\_FETCH：此状态用来等待id阶段准备就绪，然后进入DECODE。也能处理中断进入IRQ\_TAKEN，或处理调试异常进入DBG\_TAKEN\_IF  
- DECODE: 指令正常执行处于的状态。  
- FLUSH: 在DECODE状态下检测到异常，需要清空IF缓存时，进入此状态。  
- IRQ_TAKEN: 中断发生后跳转到此状态，用于设置异常原因（mcause），并控制IF跳转到异常入口  
- DBG\_TAKEN\_ID: 通过异常指令进入调试异常  
- DBG\_TAKEN\_IF: 异步事件进入异常  
- WAIT\_SLEEP: WFI指令使CPU进入等待状态，此状态暂停IF  
- SLEEP: 此状态等待异常唤醒  


## ibex_cs_register

此模块有一个类sram的接口，用于读写CSR寄存器。并且被ID模块控制，在中断发生和退出时进行状态保存和恢复。并输出一些寄存器的值给其他模块：输出一些寄存器的值给取指模块，在异常发生和异常退出时进行取指地址选择；输出一些pmp相关的寄存器的值给pmp模块，进行pmp地址访问控制。以及接受一些事件信号，并对这些事件计数。其接口如下：
```
module ibex_cs_registers #(
    parameter bit          DbgTriggerEn      = 0,
    parameter bit          DataIndTiming     = 1'b0,
    parameter bit          DummyInstructions = 1'b0,
    parameter bit          ICache            = 1'b0,
    parameter int unsigned MHPMCounterNum    = 10,
    parameter int unsigned MHPMCounterWidth  = 40,
    parameter bit          PMPEnable         = 0,
    parameter int unsigned PMPGranularity    = 0,
    parameter int unsigned PMPNumRegions     = 4,
    parameter bit          RV32E             = 0,
    parameter bit          RV32M             = 0
) (
    // Clock and Reset
    input  logic                 clk_i,                  // 时钟
    input  logic                 rst_ni,                 // 复位

    // Hart ID
    input  logic [31:0]          hart_id_i,              // hart id，每一个核心需要定义一组ibex_cs_registers 

    // Privilege mode
    output ibex_pkg::priv_lvl_e  priv_mode_id_o,         // ibex_id_stage的特权等级
    output ibex_pkg::priv_lvl_e  priv_mode_if_o,         // ibex_if_stage的特权等级
    output ibex_pkg::priv_lvl_e  priv_mode_lsu_o,        // ibex_load_store_unit的特权等级
    output logic                 csr_mstatus_tw_o,       // mstatus的tw位域

    // mtvec
    output logic [31:0]          csr_mtvec_o,            // mtvec
    input  logic                 csr_mtvec_init_i,       // 初始化mtvec的信号
    input  logic [31:0]          boot_addr_i,            // 用于初始化mtvec，开机时mtvec = boot_addr_i

    // Interface to registers (SRAM like)
    input  logic                 csr_access_i,           // csr访问信号
    input  ibex_pkg::csr_num_e   csr_addr_i,             // csr地址
    input  logic [31:0]          csr_wdata_i,            // 要写入的数据
    input  ibex_pkg::csr_op_e    csr_op_i,               // csr操着类型（READ/WRITE/SET/CLR）
    input                        csr_op_en_i,            // csr_op_i使能信号
    output logic [31:0]          csr_rdata_o,            // 读出的数据

    // interrupts
    input  logic                 irq_software_i,         // 软件中断信号
    input  logic                 irq_timer_i,            // 时钟中断信号
    input  logic                 irq_external_i,         // 外设中断信号
    input  logic [14:0]          irq_fast_i,             // 快速中断信号
    input  logic                 nmi_mode_i,             // 不可屏蔽中断信号
    output logic                 irq_pending_o,          // interrupt request pending
    output ibex_pkg::irqs_t      irqs_o,                 // interrupt requests qualified with mie
    output logic                 csr_mstatus_mie_o,      // mstatus的mie位域
    output logic [31:0]          csr_mepc_o,             // mepc

    // PMP
    output ibex_pkg::pmp_cfg_t   csr_pmp_cfg_o  [PMPNumRegions],  // pmp_cfg
    output logic [33:0]          csr_pmp_addr_o [PMPNumRegions],  // pmp_addr

    // debug
    input  logic                 debug_mode_i,
    input  ibex_pkg::dbg_cause_e debug_cause_i,
    input  logic                 debug_csr_save_i,
    output logic [31:0]          csr_depc_o,
    output logic                 debug_single_step_o,
    output logic                 debug_ebreakm_o,
    output logic                 debug_ebreaku_o,
    output logic                 trigger_match_o,

    input  logic [31:0]          pc_if_i,                // 来自取指单元的pc
    input  logic [31:0]          pc_id_i,                // 来自译码和执行单元的pc
    input  logic [31:0]          pc_wb_i,                // 来自回写单元的pc

    // CPU control bits
    output logic                 data_ind_timing_o,
    output logic                 dummy_instr_en_o,
    output logic [2:0]           dummy_instr_mask_o,
    output logic                 dummy_instr_seed_en_o,
    output logic [31:0]          dummy_instr_seed_o,
    output logic                 icache_enable_o,

    // 异常保存和恢复信号
    input  logic                 csr_save_if_i,          // 异常控制信号，保存取指单元的PC
    input  logic                 csr_save_id_i,          // 异常控制信号，保存译码执行单元的PC
    input  logic                 csr_save_wb_i,          // 异常控制信号，保存回写单元的
    input  logic                 csr_restore_mret_i,     // 异常控制信号，mret触发异常返回，恢复一些状态
    input  logic                 csr_restore_dret_i,     // 异常控制信号，dret触发异常返回，恢复一些状态
    input  logic                 csr_save_cause_i,       // 异常控制信号，异常发生时触发现场保存
    input  ibex_pkg::exc_cause_e csr_mcause_i,           // 传递异常原因，将被写入到macuse寄存器
    input  logic [31:0]          csr_mtval_i,            // 传递异常的一些附加信息，将被写入到mtval
    output logic                 illegal_csr_insn_o,     // access to non-existent CSR,
                                                         // with wrong priviledge level, or
                                                         // missing write permissions
    // 性能计数器的输入信号
    input  logic                 instr_ret_i,            // instr retired in ID/EX stage
    input  logic                 instr_ret_compressed_i, // compressed instr retired
    input  logic                 iside_wait_i,           // core waiting for the iside
    input  logic                 jump_i,                 // jump instr seen (j, jr, jal, jalr)
    input  logic                 branch_i,               // branch instr seen (bf, bnf)
    input  logic                 branch_taken_i,         // branch was taken
    input  logic                 mem_load_i,             // load from memory in this cycle
    input  logic                 mem_store_i,            // store to memory in this cycle
    input  logic                 dside_wait_i,           // core waiting for the dside
    input  logic                 mul_wait_i,             // core waiting for multiply
    input  logic                 div_wait_i              // core waiting for divide
);
```

此模块中有大量的寄存器定义，主要实现读写寄存器操着。读写逻辑如下：  
```
// 读的简化逻辑如下
always_comb begin
  unique case (csr_addr_i)
    CSR_MHARTID: csr_rdata_int = hart_id_i;
    ...
  endcase
end

// 写操作的简化逻辑如下
always_comb begin
  if (csr_we_int) begin
    unique case (csr_addr_i)
      CSR_MSCRATCH: mscratch_d = csr_wdata_int;
      ...
    endcase
  end
end

/*
 * 大部分的读操作是组合逻辑，
 * csr_rdata_o是读操着的结果
 * csr_wdata_i是指令中的操着数
 * csr_wdata_int是要最终要写入的值
 */
always_comb begin
    unique case (csr_op_i)
      CSR_OP_WRITE: csr_wdata_int =  csr_wdata_i;
      CSR_OP_SET:   csr_wdata_int =  csr_wdata_i | csr_rdata_o;
      CSR_OP_CLEAR: csr_wdata_int = ~csr_wdata_i & csr_rdata_o;
      CSR_OP_READ:  csr_wdata_int = csr_wdata_i;
      default:      csr_wdata_int = csr_wdata_i;
    endcase
  end
```

其中，有些寄存器只读，比如misa：  
```
  // misa
  localparam logic [31:0] MISA_VALUE =
      (0                 <<  0)  // A - Atomic Instructions extension
    | (1                 <<  2)  // C - Compressed extension
    | (0                 <<  3)  // D - Double precision floating-point extension
    | (32'(RV32E)        <<  4)  // E - RV32E base ISA
    | (0                 <<  5)  // F - Single precision floating-point extension
    | (32'(!RV32E)       <<  8)  // I - RV32I/64I/128I base ISA
    | (32'(RV32M)        << 12)  // M - Integer Multiply/Divide extension
    | (0                 << 13)  // N - User level interrupts supported
    | (0                 << 18)  // S - Supervisor mode implemented
    | (1                 << 20)  // U - User mode implemented
    | (0                 << 23)  // X - Non-standard extensions present
    | (32'(CSR_MISA_MXL) << 30); // M-XLEN
    
  always_comb begin
    unique case (csr_addr_i)
      CSR_MISA: csr_rdata_int = MISA_VALUE;
      ...
    endcase
  end
```

其中，有一些寄存器由外部信号构成，比如mip：  
```
  assign mip.irq_software = irq_software_i;
  assign mip.irq_timer    = irq_timer_i;
  assign mip.irq_external = irq_external_i;
  assign mip.irq_fast     = irq_fast_i; 
  ...
  always_comb begin
    unique case (csr_addr_i)
      CSR_MIP: begin
        csr_rdata_int                                     = '0;
        csr_rdata_int[CSR_MSIX_BIT]                       = mip.irq_software;
        csr_rdata_int[CSR_MTIX_BIT]                       = mip.irq_timer;
        csr_rdata_int[CSR_MEIX_BIT]                       = mip.irq_external;
        csr_rdata_int[CSR_MFIX_BIT_HIGH:CSR_MFIX_BIT_LOW] = mip.irq_fast;
      end
      ...
    endcase
  end
```
pmp的特殊处理，pmp的写信号与pmp_cfg是否上锁有关，相关代码如下
```
      // 判断是否加锁，加锁写入无效
      assign pmp_cfg_we[i] = csr_we_int & ~pmp_cfg[i].lock &
                             (csr_addr == (CSR_OFF_PMP_CFG + (i[11:0] >> 2)));

      // 上面有代码负责拆分csr_wdata_int的值为多个pmp_cfg_wdata
      
      // 写pmp_cfg的代码
      always_ff @(posedge clk_i or negedge rst_ni) begin
        if (!rst_ni) begin
          pmp_cfg[i] <= pmp_cfg_t'('b0); // 复位为0
        end else if (pmp_cfg_we[i]) begin
          pmp_cfg[i] <= pmp_cfg_wdata[i];
        end
      end
```

还有一些代码用于在异常发生时保存恢复状态
```
    // exception controller gets priority over other writes
    unique case (1'b1)
      
      csr_save_cause_i: begin              // 异常保存信号
        unique case (1'b1)                 // 选择要保存的pc
          csr_save_if_i: begin
            exception_pc = pc_if_i;
          end
          csr_save_id_i: begin
            exception_pc = pc_id_i;
          end
          csr_save_wb_i: begin
            exception_pc = pc_wb_i;
          end
          default:;
        endcase

        // Any exception, including debug mode, causes a switch to M-mode
        priv_lvl_d = PRIV_LVL_M;           // 保存特权等级

        if (debug_csr_save_i) begin        // 进入调试模式，执行调试模式相关信息保存
          // all interrupts are masked
          // do not update cause, epc, tval, epc and status
          dcsr_d.prv   = priv_lvl_q;
          dcsr_d.cause = debug_cause_i;
          depc_d       = exception_pc;
        end else if (!debug_mode_i) begin  // 进入普通异常，执行异常相关信息保存
          // In debug mode, "exceptions do not update any registers. That
          // includes cause, epc, tval, dpc and mstatus." [Debug Spec v0.13.2, p.39]
          mtval_d        = csr_mtval_i;
          mstatus_d.mie  = 1'b0; // disable interrupts
          // save current status
          mstatus_d.mpie = mstatus_q.mie;
          mstatus_d.mpp  = priv_lvl_q;
          mepc_d         = exception_pc;
          mcause_d       = {csr_mcause_i};
          // save previous status for recoverable NMI
          mstack_d.mpie  = mstatus_q.mpie;
          mstack_d.mpp   = mstatus_q.mpp;
          mstack_epc_d   = mepc_q;
          mstack_cause_d = mcause_q;
        end
      end // csr_save_cause_i

      csr_restore_dret_i: begin // DRET，从调试模式返回，恢复特权等级
        priv_lvl_d = dcsr_q.prv;
      end // csr_restore_dret_i

      csr_restore_mret_i: begin // MRET，从m模式返回
        priv_lvl_d     = mstatus_q.mpp;  // 恢复特权等级
        mstatus_d.mie  = mstatus_q.mpie; // 恢复中断使能位

        if (nmi_mode_i) begin
          // when returning from an NMI restore state from mstack CSR
          mstatus_d.mpie = mstack_q.mpie;
          mstatus_d.mpp  = mstack_q.mpp;
          mepc_d         = mstack_epc_q;
          mcause_d       = mstack_cause_q;
        end else begin
          // otherwise just set mstatus.MPIE/MPP
          // See RISC-V Privileged Specification, version 1.11, Section 3.1.6.1
          mstatus_d.mpie = 1'b1;
          mstatus_d.mpp  = PRIV_LVL_U;
        end
      end // csr_restore_mret_i
```

riscv规范定义了一组性能计数器，只有几个计数器含义被确定了，其他的可以由具体平台自定义。ibex计数事件如下
```
  // event selection (hardwired) & control
  always_comb begin : gen_mhpmcounter_incr

    // Assign inactive counters (first to prevent latch inference)
    for (int unsigned i=0; i<32; i++) begin : gen_mhpmcounter_incr_inactive
      mhpmcounter_incr[i] = 1'b0;
    end

    // When adding or altering performance counter meanings and default
    // mappings please update dv/verilator/pcount/cpp/ibex_pcounts.cc
    // appropriately.
    //
    // active counters
    mhpmcounter_incr[0]  = 1'b1;                   // mcycle
    mhpmcounter_incr[1]  = 1'b0;                   // reserved
    mhpmcounter_incr[2]  = instr_ret_i;            // minstret
    mhpmcounter_incr[3]  = dside_wait_i;           // cycles waiting for data memory
    mhpmcounter_incr[4]  = iside_wait_i;           // cycles waiting for instr fetches
    mhpmcounter_incr[5]  = mem_load_i;             // num of loads
    mhpmcounter_incr[6]  = mem_store_i;            // num of stores
    mhpmcounter_incr[7]  = jump_i;                 // num of jumps (unconditional)
    mhpmcounter_incr[8]  = branch_i;               // num of branches (conditional)
    mhpmcounter_incr[9]  = branch_taken_i;         // num of taken branches (conditional)
    mhpmcounter_incr[10] = instr_ret_compressed_i; // num of compressed instr
    mhpmcounter_incr[11] = mul_wait_i;             // cycles waiting for multiply
    mhpmcounter_incr[12] = div_wait_i;             // cycles waiting for divide
  end
```

计数器模块接口如下
```
module ibex_counter #(
  parameter int CounterWidth = 32 // 计数器宽度
) (
  // 时钟和复位信号
  input  logic        clk_i,
  input  logic        rst_ni,

  input  logic        counter_inc_i, // 计数器信号，信号有效时计数器将在时钟上升沿加一
  input  logic        counterh_we_i, // 此信号用于在写计数器时选择写高32位还是低32位，0写低32位，1写高32位
  input  logic        counter_we_i,  // 写使能信号
  input  logic [31:0] counter_val_i, // 要被写入的值
  output logic [63:0] counter_val_o  // 计数器的值
);
```

## ibex_ex_block

此模块负责主要的计算任务，模块接口如下：
```
module ibex_ex_block #(
    parameter bit RV32M                    = 1,     // 是否支持M扩展
    parameter bit RV32B                    = 0,     // 是否支持B扩展
    parameter bit BranchTargetALU          = 0,     // 是否由EX模块计算跳转目标地址
    parameter     MultiplierImplementation = "fast" // 乘除法的实现 
) (
	// 时钟和复位信号
    input  logic                  clk_i,
    input  logic                  rst_ni,

    // ALU
    input  ibex_pkg::alu_op_e     alu_operator_i,   // alu要进行的操作
    input  logic [31:0]           alu_operand_a_i,  // 操作数a
    input  logic [31:0]           alu_operand_b_i,  // 操作数b
    input  logic                  alu_instr_first_cycle_i, // 多周期指令输出信号，第一个周期

    // Branch Target ALU
    // All of these signals are unusued when BranchTargetALU == 0
    input  logic [31:0]           bt_a_operand_i,   // 分支指令计算目标地址，操作数a
    input  logic [31:0]           bt_b_operand_i,   // 分支指令计算目标地址，操作数b

    // Multiplier/Divider
    input  ibex_pkg::md_op_e      multdiv_operator_i,    // 乘除法要执行的操作
    input  logic                  mult_en_i,             // dynamic enable signal, for FSM control
    input  logic                  div_en_i,              // dynamic enable signal, for FSM control
    input  logic                  mult_sel_i,            // static decoder output, for data muxes
    input  logic                  div_sel_i,             // static decoder output, for data muxes
    input  logic  [1:0]           multdiv_signed_mode_i, // 输入的操作数的符号，两位分别表示两个操作数的符号
    input  logic [31:0]           multdiv_operand_a_i,   // 操作数a
    input  logic [31:0]           multdiv_operand_b_i,   // 操作数b
    input  logic                  multdiv_ready_id_i,
    input  logic                  data_ind_timing_i,

    // 多周期指令的中间结果，当imd_val_we_o=1时，在时钟上升沿锁存imd_val_d_o到imd_val_q_i
    output logic                  imd_val_we_o,
    output logic [33:0]           imd_val_d_o,
    input  logic [33:0]           imd_val_q_i,

    // Outputs
    output logic [31:0]           alu_adder_result_ex_o, // to LSU
    output logic [31:0]           result_ex_o,           // 运算结构
    output logic [31:0]           branch_target_o,       // to IF，计算的分支地址bt_a_operand_i + bt_b_operand_i
    output logic                  branch_decision_o,     // to ID，输出比较结果给ID，控制是否需要执行跳转操作

    output logic                  ex_valid_o             // EX has valid output
);
```

ibex\_ex\_block内部有两个子模块：alu和乘除法模块。alu模块中由很多算法，用于减少门电路的使用。

### 加法器实现

加法器需要计算加减法，比较指令也需要执行减法。我们知道要执行a-b可以通过加法实现a+~b+1，但写成这样需要两个加法器，这样不仅会增加电路成本，还会增加电路延时妨碍cpu主频提高。

ibex实现比较巧妙：
- 在操作数a低位补一位1，获得33位的adder\_in\_a
- 在操作数b低位补一位0，如果要执行减法，把补位的数取反，获得33位的adder\_in\_b
- 然后计算\$unsigned(adder\_in\_a) + \$unsigned(adder\_in\_b)获得34位结果adder\_result\_ext\_o
- 加法器结构adder\_result\_ext\_o[32:1]


### 移位实现

一个32位的单周期移位器，需要一个庞大的多路选择器。为了节省门电路ibex实现比较巧妙。

ibex只使用了一个33比特的数学右移。通常最高位清0，数学右移时符号扩展操作数赋值给33比特的移位器输入。要执行左移时，操作数先进行位反转，在执行右移，在把结果位反转。

### 循环移位

循环移位，可以转换为两次移位操着`rotate_right(rs1, rs2) -> logic_right(rs1, rs2 & 31) | logic_lift(rs1, 32 - rs2 & 31)`。

为了节省门电路，使用两个周期实现循环移位。第一周期移位结构通过`imd_val_d_o` `imd_val_we_o`锁存到`imd_val_q_i`。在第二周期时把移位结果或上`imd_val_q_i`得到计算结果。


## ibex_register_file

接口如下：
```
module ibex_register_file #(
    parameter bit          RV32E             = 0,
    parameter int unsigned DataWidth         = 32,
    parameter bit          DummyInstructions = 0
) (
    // Clock and Reset
    input  logic                 clk_i,             // 时钟
    input  logic                 rst_ni,            // 复位

    input  logic                 test_en_i,
    input  logic                 dummy_instr_id_i,

    //Read port R1
    input  logic [4:0]           raddr_a_i,         // 读取端口1，寄存器地址
    output logic [DataWidth-1:0] rdata_a_o,         // 读取端口1，输出读出的值

    //Read port R2
    input  logic [4:0]           raddr_b_i,         // 读取端口2，寄存器地址
    output logic [DataWidth-1:0] rdata_b_o,         // 读取端口2，输出读出的值


    // Write port W1
    input  logic [4:0]           waddr_a_i,         // 写端口，寄存器地址
    input  logic [DataWidth-1:0] wdata_a_i,         // 写端口，要写入的值
    input  logic                 we_a_i             // 写端口，写使能信号
);
```

此模块比较简单，但ibex使用三种方式来实现ibex\_register\_file，分别适用于不同的应用场景：  
- 基于触发器的实现，此方案对各种模拟器比较友好
- 基于SRAM的实现，此方案可以使用FPGA内部的SRAM资源，可以大量节约LUT的使用
- 基于锁存器的实现，此方案比较适合集成电路，可以大量节约门电路的使用

## ibex_load_store_unit

此模块用于连接数据存储器，支持非对齐内存访问。其接口如下：  
```
module ibex_load_store_unit
(
    // 时钟和复位信号
    input  logic         clk_i,
    input  logic         rst_ni,

    // 数据存储器接口
    output logic         data_req_o,           // 数据请求信号
    input  logic         data_gnt_i,           // 数据响应信号
    input  logic         data_rvalid_i,        // 数据有效信号
    input  logic         data_err_i,           // 数据存储器出错
    input  logic         data_pmp_err_i,       // 来自pmp的错误信号
    output logic [31:0]  data_addr_o,          // 要访问的数据地址
    output logic         data_we_o,            // 写使能信号
    output logic [3:0]   data_be_o,            // 写操作是的字节写使能信号
    output logic [31:0]  data_wdata_o,         // 要写入的值
    input  logic [31:0]  data_rdata_i,         // 读出的值，通过data_rvalid_i标识读出的值有效

    // 来自ID模块的信号
    input  logic         lsu_we_i,             // 写使能                           -> from ID/EX
    input  logic [1:0]   lsu_type_i,           // 数据类型: word, half word, byte  -> from ID/EX
    input  logic [31:0]  lsu_wdata_i,          // 要写入内存的数据                  -> from ID/EX
    input  logic         lsu_sign_ext_i,       // 读操作时是否进行符号位扩展         -> from ID/EX
    output logic [31:0]  lsu_rdata_o,          // 读取到的数据                      -> to ID/EX
    output logic         lsu_rdata_valid_o,    // 数据有效标识
    input  logic         lsu_req_i,            // 数据请求信号                      -> from ID/EX

    input  logic [31:0]  adder_result_ex_i,    // 来自ALU的计算结果                 -> from ID/EX

    output logic         addr_incr_req_o,      // 输出信号，用于告知ID模块正在进行
                                               // 非对齐地址访问                    -> to ID/EX
    output logic [31:0]  addr_last_o,          // address of last transaction      -> to controller
                                               // -> mtval
                                               // -> AGU for misaligned accesses

    output logic         lsu_req_done_o,       // Signals that data request is complete
                                               // (only need to await final data
                                               // response)                        -> to ID/EX

    output logic         lsu_resp_valid_o,     // LSU has response from transaction -> to ID/EX

    // 异常信号
    output logic         load_err_o,
    output logic         store_err_o,

    output logic         busy_o,

    output logic         perf_load_o,
    output logic         perf_store_o
);
```
ibex为32位的riscv处理器，总线宽度32位，访问数据宽度分别为8、16、32位，所以最多需要两次内存访问。系统定义了一个状态位handle\_misaligned\_q标识当前是不是正在进行非对齐访问（访问第二个字）。

在非对齐访问时，需要设置字节使能信号，代码如下：
```
  ///////////////////
  // BE generation //
  ///////////////////

  always_comb begin
    unique case (lsu_type_i) // Data type 00 Word, 01 Half word, 11,10 byte
      2'b00: begin // Writing a word
        if (!handle_misaligned_q) begin // first part of potentially misaligned transaction
          unique case (data_offset)
            2'b00:   data_be = 4'b1111;
            2'b01:   data_be = 4'b1110;
            2'b10:   data_be = 4'b1100;
            2'b11:   data_be = 4'b1000;
            default: data_be = 4'b1111;
          endcase // case (data_offset)
        end else begin // second part of misaligned transaction
          unique case (data_offset)
            2'b00:   data_be = 4'b0000; // this is not used, but included for completeness
            2'b01:   data_be = 4'b0001;
            2'b10:   data_be = 4'b0011;
            2'b11:   data_be = 4'b0111;
            default: data_be = 4'b1111;
          endcase // case (data_offset)
        end
      end

      2'b01: begin // Writing a half word
        if (!handle_misaligned_q) begin // first part of potentially misaligned transaction
          unique case (data_offset)
            2'b00:   data_be = 4'b0011;
            2'b01:   data_be = 4'b0110;
            2'b10:   data_be = 4'b1100;
            2'b11:   data_be = 4'b1000;
            default: data_be = 4'b1111;
          endcase // case (data_offset)
        end else begin // second part of misaligned transaction
          data_be = 4'b0001;
        end
      end

      2'b10,
      2'b11: begin // Writing a byte
        unique case (data_offset)
          2'b00:   data_be = 4'b0001;
          2'b01:   data_be = 4'b0010;
          2'b10:   data_be = 4'b0100;
          2'b11:   data_be = 4'b1000;
          default: data_be = 4'b1111;
        endcase // case (data_offset)
      end

      default:     data_be = 4'b1111;
    endcase // case (lsu_type_i)
  end
```

可以把写入的值，根据字节偏移量循环移位一下，得到真正要写入的值：  
```
  always_comb begin
    unique case (data_offset)
      2'b00:   data_wdata =  lsu_wdata_i[31:0];
      2'b01:   data_wdata = {lsu_wdata_i[23:0], lsu_wdata_i[31:24]};
      2'b10:   data_wdata = {lsu_wdata_i[15:0], lsu_wdata_i[31:16]};
      2'b11:   data_wdata = {lsu_wdata_i[ 7:0], lsu_wdata_i[31: 8]};
      default: data_wdata =  lsu_wdata_i[31:0];
    endcase // case (data_offset)
  end
```

读取时，读取两次，把前一次结果和后一次结果组合到一起就可以获取最终的结果。代码如下：  
```
  // register for unaligned rdata
  always_ff @(posedge clk_i or negedge rst_ni) begin
    if (!rst_ni) begin
      rdata_q <= '0;
    end else if (rdata_update) begin
      rdata_q <= data_rdata_i[31:8];
    end
  end
  ...
  // take care of misaligned words
  always_comb begin
    unique case (rdata_offset_q)
      2'b00:   rdata_w_ext =  data_rdata_i[31:0];
      2'b01:   rdata_w_ext = {data_rdata_i[ 7:0], rdata_q[31:8]};
      2'b10:   rdata_w_ext = {data_rdata_i[15:0], rdata_q[31:16]};
      2'b11:   rdata_w_ext = {data_rdata_i[23:0], rdata_q[31:24]};
      default: rdata_w_ext =  data_rdata_i[31:0];
    endcase
  end
```

读取到的结果还需要进行位扩展操作，代码如下：  
```
  ////////////////////
  // Sign extension //
  ////////////////////

  // sign extension for half words
  always_comb begin
    unique case (rdata_offset_q)
      2'b00: begin
        if (!data_sign_ext_q) begin
          rdata_h_ext = {16'h0000, data_rdata_i[15:0]};
        end else begin
          rdata_h_ext = {{16{data_rdata_i[15]}}, data_rdata_i[15:0]};
        end
      end

      2'b01: begin
        if (!data_sign_ext_q) begin
          rdata_h_ext = {16'h0000, data_rdata_i[23:8]};
        end else begin
          rdata_h_ext = {{16{data_rdata_i[23]}}, data_rdata_i[23:8]};
        end
      end

      2'b10: begin
        if (!data_sign_ext_q) begin
          rdata_h_ext = {16'h0000, data_rdata_i[31:16]};
        end else begin
          rdata_h_ext = {{16{data_rdata_i[31]}}, data_rdata_i[31:16]};
        end
      end

      2'b11: begin
        if (!data_sign_ext_q) begin
          rdata_h_ext = {16'h0000, data_rdata_i[7:0], rdata_q[31:24]};
        end else begin
          rdata_h_ext = {{16{data_rdata_i[7]}}, data_rdata_i[7:0], rdata_q[31:24]};
        end
      end

      default: rdata_h_ext = {16'h0000, data_rdata_i[15:0]};
    endcase // case (rdata_offset_q)
  end

  // sign extension for bytes
  always_comb begin
    unique case (rdata_offset_q)
      2'b00: begin
        if (!data_sign_ext_q) begin
          rdata_b_ext = {24'h00_0000, data_rdata_i[7:0]};
        end else begin
          rdata_b_ext = {{24{data_rdata_i[7]}}, data_rdata_i[7:0]};
        end
      end

      2'b01: begin
        if (!data_sign_ext_q) begin
          rdata_b_ext = {24'h00_0000, data_rdata_i[15:8]};
        end else begin
          rdata_b_ext = {{24{data_rdata_i[15]}}, data_rdata_i[15:8]};
        end
      end

      2'b10: begin
        if (!data_sign_ext_q) begin
          rdata_b_ext = {24'h00_0000, data_rdata_i[23:16]};
        end else begin
          rdata_b_ext = {{24{data_rdata_i[23]}}, data_rdata_i[23:16]};
        end
      end

      2'b11: begin
        if (!data_sign_ext_q) begin
          rdata_b_ext = {24'h00_0000, data_rdata_i[31:24]};
        end else begin
          rdata_b_ext = {{24{data_rdata_i[31]}}, data_rdata_i[31:24]};
        end
      end

      default: rdata_b_ext = {24'h00_0000, data_rdata_i[7:0]};
    endcase // case (rdata_offset_q)
  end
```
## ibex_wb_stage

回写模块，主要负责把计算结果回写到内存，或把从内存加载的值回写到寄存器。接口如下：  
```
module ibex_wb_stage #(
  parameter bit WritebackStage = 1'b0                      // 配置当前模块是否为一级流水线
) (
  // 时钟和复位信号
  input  logic                     clk_i,
  input  logic                     rst_ni,

  input  logic                     en_wb_i,                // 写使能信号
  input  ibex_pkg::wb_instr_type_e instr_type_wb_i,        // 指令类型：加载/存储/其他
  input  logic [31:0]              pc_id_i,                // 来自id模块的pc的值

  output logic                     ready_wb_o,             // 当前模块准备就绪，可以接受新的操作
  output logic                     rf_write_wb_o,          // 写寄存器信号
  output logic                     outstanding_load_wb_o,  // 执行load 操作的输出信号
  output logic                     outstanding_store_wb_o, // 执行store操作的输出信号
  output logic [31:0]              pc_wb_o,                // 输出wb模块的pc
  
  // 来自ID模块的写寄存器控制信号
  input  logic [4:0]               rf_waddr_id_i,          // 寄存器地址
  input  logic [31:0]              rf_wdata_id_i,          // 要写如寄存器的数据
  input  logic                     rf_we_id_i,             // 写使能信号

  // 来自load_store_unit模块的从内存读出的值
  input  logic [31:0]              rf_wdata_lsu_i,         // 从内存读出的值
  input  logic                     rf_we_lsu_i,            // 此信号选择把哪个值写入寄存器
  
  output logic [31:0]              rf_wdata_fwd_wb_o,
  output logic [4:0]               rf_waddr_wb_o,          // 要写入的寄存器地址
  output logic [31:0]              rf_wdata_wb_o,          // 要写入寄存器的值
  output logic                     rf_we_wb_o,             // 写使能信号

  input logic                      lsu_resp_valid_i,

  output logic                     instr_done_wb_o         // 操作完成信号
);
```

当不使用流水线时，信号直接链接：  
```
assign ready_wb_o    = 1'b1;
assign rf_waddr_wb_o         = rf_waddr_id_i;
assign rf_wdata_wb_mux[0]    = rf_wdata_id_i;
assign rf_wdata_wb_mux[1]    = rf_wdata_lsu_i;
assign rf_wdata_wb_mux_we[1] = rf_we_lsu_i;
assign rf_wdata_wb_mux_we[0] = rf_we_id_i;
assign rf_wdata_wb_o = rf_wdata_wb_mux_we[0] ? rf_wdata_wb_mux[0] : rf_wdata_wb_mux[1];
assign rf_we_wb_o    = |rf_wdata_wb_mux_we;
```
当使用流水线时，信号被触发器缓存，在下一个时钟时进行实际的操作：  
```
always_ff @(posedge clk_i) begin
  if(en_wb_i) begin
    rf_we_wb_q      <= rf_we_id_i;
    rf_waddr_wb_q   <= rf_waddr_id_i;
    rf_wdata_wb_q   <= rf_wdata_id_i;
    wb_instr_type_q <= instr_type_wb_i;
    wb_pc_q         <= pc_id_i;
  end
end
assign rf_waddr_wb_o         = rf_waddr_wb_q;
assign rf_wdata_wb_mux[0]    = rf_wdata_wb_q;
assign rf_wdata_wb_mux[1]    = rf_wdata_lsu_i;
assign rf_wdata_wb_mux_we[0] = rf_we_wb_q & wb_valid_q;
assign rf_wdata_wb_mux_we[1] = rf_we_lsu_i;
assign ready_wb_o = ~wb_valid_q | wb_done;
assign rf_wdata_wb_o = rf_wdata_wb_mux_we[0] ? rf_wdata_wb_mux[0] : rf_wdata_wb_mux[1];
assign rf_we_wb_o    = |rf_wdata_wb_mux_we;
```