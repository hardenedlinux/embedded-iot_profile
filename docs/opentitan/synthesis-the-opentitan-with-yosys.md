# Introduction

OpenTitan is the first open source project building a transparent, high-quality reference design and integration guidelines for silicon root of trust (RoT) chips. [Yosys](https://github.com/YosysHQ/yosys) is a framework for RTL synthesis tools. It currently has extensive Verilog-2005 support and provides a basic set of synthesis algorithms for various application domains. OpenTitan is a systemverilog project, but currently yosys only supports a small subset of systemverilog. So we need use [sv2v](https://github.com/zachjs/sv2v)  to convert the source code to verilog.

Open source tools used:  
1. Source code conversion tool [sv2v](https://github.com/zachjs/sv2v)  
2. RTL synthesis tools [yosys](https://github.com/YosysHQ/yosys)  

# Install toolchain

## Install sv2v

sv2v provides pre-built binaries, we can download from [release page] (https://github.com/zachjs/sv2v/releases) according to our own system, then unzip to executable path.

We can also build from source code. Since sv2v is a Haskell project, we need to install Haskell's build system on our computer, please refer to [Stack](https://docs.haskellstack.org/en/stable/README/). Through the following command we can obtain the source code of sv2v and build an executable file.

```
git clone https://github.com/zachjs/sv2v.git
cd sv2v
make
```

When the command completes the executable file is located at `./bin/sv2v`. Copy the executable file to executable path, then the installation is complete.

## Install yosys

If the system you are using is debian, you can install directly via apt-get.

If your system does not provide yosys, you can install from source code. Some prerequisites are need to build yosys, you can install the using the following command:  
```
sudo apt-get install build-essential clang bison flex \
	libreadline-dev gawk tcl-dev libffi-dev git \
	graphviz xdot pkg-config python3 libboost-system-dev \
	libboost-python-dev libboost-filesystem-dev zlib1g-dev
```

Then you need to get the source code of yosys, get it by the following command:  
```
git clone https://github.com/YosysHQ/yosys.git
```

Then build by the following command:  
```
make config-gcc
make
```

Then install by the following command:  
```
make install DESTDIR=path_you_want_to_install_to
``` 

# Transcoding

OpenTitan is a complex project, it has some auxiliary programs to automatically generate some code. By reading the documentation, we can get the source code we need by the following command:  
```
sudo apt-get install autoconf bison build-essential clang-format curl flake8 \
    flex g++ git libelf1 libelf-dev libftdi1-2 libftdi1-dev libssl-dev \
    libusb-1.0-0 make ninja-build pkgconf python3 python3-pip \
    python3-setuptools python3-wheel python3-yaml srecord tree zlib1g-dev

# install some python tools, execute in the path of opentitan
pip3 install --user --requirement python-requirements.txt

# Get gcc toolchain of riscv, execute in the path of opentitan
./util/get-toolchain.py

# install verilator
export VERILATOR_VERSION=4.028
git clone http://git.veripool.org/git/verilator
cd verilator
git checkout v$VERILATOR_VERSION
autoconf
./configure --prefix=/tools/verilator/$VERILATOR_VERSION
make
make install

export PATH=$PATH:/tools/riscv/bin:/tools/verilator/4.028/bin

git clone https://github.com/lowRISC/opentitan.git
cd opentitan
git checkout 75639fe11cd00896a57c34576b8574b6152168df
fusesoc --cores-root . run --target=sim --setup --build lowrisc:systems:top_earlgrey_verilator
```

Then rtl code will be generated in `build/lowrisc_systems_top_earlgrey_verilator_0.1/src`

There are a lot of declarative statements in systemverilog, and sv2v conversion of these statements will not generate any code. And the source code in the opentitan project lacks the include statement, sv2v will not find some declarations and definitions, which will cause some errors. So I need to create a file includes all the source code, the following is the file(opentitan.sv) I used:  
```verilog
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_cipher_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_rstmgr_pkg_0.1/rtl/rstmgr_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_pkg_0.1/rtl/ibex_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_top_earlgrey_pinmux_reg_0.1/rtl/autogen/pinmux_reg_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_hmac_0.1/rtl/hmac_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_hmac_0.1/rtl/hmac_reg_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_nmi_gen_0.1/rtl/nmi_gen_reg_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_reg_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_usb_fs_nb_pe_0.1/rtl/usb_consts_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_alert_handler_component_0.1/rtl/alert_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_alert_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_esc_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_top_earlgrey_rv_plic_0.1/rtl/autogen/rv_plic_reg_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_gpio_0.1/rtl/gpio_reg_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_tlul_headers_0.1/rtl/tlul_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_flash_ctrl_0.1/rtl/flash_ctrl_reg_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_usbdev_0.1/rtl/usbdev_reg_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_constants_top_pkg_0/rtl/top_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_rv_timer_0.1/rtl/rv_timer_reg_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_tracer_0.1/rtl/ibex_tracer_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_spi_device_0.1/rtl/spi_device_reg_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_spi_device_0.1/rtl/spi_device_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_uart_0.1/rtl/uart_reg_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_top_earlgrey_alert_handler_reg_0.1/rtl/autogen/alert_handler_reg_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_xbar_main_0.1/tl_main_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/pulp-platform_riscv-dbg_0.1_0/pulp_riscv_dbg/src/dm_pkg.sv" 
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_abstract_prim_pkg_0.1/prim_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_xbar_peri_0.1/tl_peri_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_rstmgr_0.1/rtl/rstmgr_reg_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_pwrmgr_pkg_0.1/rtl/pwrmgr_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_pwrmgr_0.1/rtl/pwrmgr_reg_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_flash_ctrl_pkg_0.1/rtl/flash_ctrl_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_flash_ctrl_pkg_0.1/rtl/flash_phy_pkg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_pwrmgr_0.1/rtl/pwrmgr_cdc.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_pwrmgr_0.1/rtl/pwrmgr_slow_fsm.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_pwrmgr_0.1/rtl/pwrmgr_wake_info.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_pwrmgr_0.1/rtl/pwrmgr_fsm.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_pwrmgr_0.1/rtl/pwrmgr_reg_top.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_pwrmgr_0.1/rtl/pwrmgr.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_rstmgr_0.1/rtl/rstmgr_ctrl.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_rstmgr_0.1/rtl/rstmgr_info.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_rstmgr_0.1/rtl/rstmgr_por.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_rstmgr_0.1/rtl/rstmgr_reg_top.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_rstmgr_0.1/rtl/rstmgr.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_abstract_flash_0/prim_flash.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_abstract_rom_0/prim_rom.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_abstract_ram_1p_0/prim_ram_1p.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_xbar_peri_0.1/xbar_peri.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_abstract_ram_2p_0/prim_ram_2p.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_abstract_clock_gating_0/prim_clock_gating.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/pulp-platform_riscv-dbg_0.1_0/pulp_riscv_dbg/debug_rom/debug_rom.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_abstract_clock_mux2_0/prim_clock_mux2.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/pulp-platform_riscv-dbg_0.1_0/pulp_riscv_dbg/src/dm_csrs.sv" 
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/pulp-platform_riscv-dbg_0.1_0/pulp_riscv_dbg/src/dmi_cdc.sv" 
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/pulp-platform_riscv-dbg_0.1_0/pulp_riscv_dbg/src/dmi_jtag.sv" 
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/pulp-platform_riscv-dbg_0.1_0/pulp_riscv_dbg/src/dmi_jtag_tap.sv" 
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/pulp-platform_riscv-dbg_0.1_0/pulp_riscv_dbg/src/dm_mem.sv" 
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/pulp-platform_riscv-dbg_0.1_0/pulp_riscv_dbg/src/dm_sba.sv" 
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_xbar_main_0.1/xbar_main.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_rv_plic_component_0.1/rtl/rv_plic_target.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_rv_plic_component_0.1/rtl/rv_plic_gateway.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_alu.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_multdiv_slow.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_controller.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_counters.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_prefetch_buffer.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_compressed_decoder.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_fetch_fifo.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_pmp.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_multdiv_fast.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_cs_registers.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_core.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_register_file_ff.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_ex_block.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_if_stage.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_wb_stage.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_load_store_unit.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_id_stage.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_decoder.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_top_earlgrey_pinmux_reg_0.1/rtl/autogen/pinmux_reg_top.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_xilinx_pad_wrapper_0/rtl/prim_xilinx_pad_wrapper.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_hmac_0.1/rtl/sha2.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_hmac_0.1/rtl/sha2_pad.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_hmac_0.1/rtl/hmac_core.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_hmac_0.1/rtl/hmac_reg_top.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_hmac_0.1/rtl/hmac.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_generic_ram_2p_0/rtl/prim_generic_ram_2p.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_nmi_gen_0.1/rtl/nmi_gen_reg_top.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_nmi_gen_0.1/rtl/nmi_gen.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_generic_flash_0/rtl/prim_generic_flash.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_prng.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_sbox_lut.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_ctr.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_control.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_cipher_control.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_cipher_core.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_key_expand.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_sbox_canright.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_reg_top.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_sub_bytes.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_core.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_mix_single_column.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_shift_rows.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_sbox.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_aes_0.6/rtl/aes_mix_columns.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_usb_fs_nb_pe_0.1/rtl/usb_fs_nb_in_pe.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_usb_fs_nb_pe_0.1/rtl/usb_fs_nb_out_pe.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_usb_fs_nb_pe_0.1/rtl/usb_fs_tx.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_usb_fs_nb_pe_0.1/rtl/usb_fs_rx.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_usb_fs_nb_pe_0.1/rtl/usb_fs_nb_pe.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_usb_fs_nb_pe_0.1/rtl/usb_fs_tx_mux.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_xilinx_rom_0/rtl/prim_xilinx_rom.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_generic_clock_gating_0/rtl/prim_generic_clock_gating.sv"
//`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_systems_top_earlgrey_verilator_0.1/rtl/top_earlgrey_verilator.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_generic_clock_mux2_0/rtl/prim_generic_clock_mux2.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_assert_0.1/rtl/prim_assert.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_diff_decode_0/rtl/prim_diff_decode.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_alert_handler_component_0.1/rtl/alert_handler_reg_wrap.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_alert_handler_component_0.1/rtl/alert_handler.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_alert_handler_component_0.1/rtl/alert_handler_class.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_alert_handler_component_0.1/rtl/alert_handler_esc_timer.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_alert_handler_component_0.1/rtl/alert_handler_ping_timer.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_alert_handler_component_0.1/rtl/alert_handler_accu.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_packer.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_filter_ctr.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_secded_39_32_dec.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_subreg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_arbiter_tree.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_present.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_flop_2sync.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_clock_inverter.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_esc_sender.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_ram_2p_adv.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_esc_receiver.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_arbiter_ppc.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_lfsr.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_filter.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_subreg_ext.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_alert_receiver.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_secded_39_32_enc.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_fifo_async.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_pulse_sync.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_prince.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_sram_arbiter.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_intr_hw.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_ram_2p_async_adv.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_fifo_sync.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_keccak.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_alert_sender.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_systems_top_earlgrey_0.1/rtl/autogen/top_earlgrey.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_systems_top_earlgrey_0.1/rtl/padctl.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_tlul_socket_1n_0.1/rtl/tlul_err_resp.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_tlul_socket_1n_0.1/rtl/tlul_socket_1n.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_rv_dm_0.1/rtl/rv_dm.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_tlul_adapter_host_0.1/rtl/tlul_adapter_host.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_tlul_sram2tlul_0.1/rtl/sram2tlul.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_generic_pad_wrapper_0/rtl/prim_generic_pad_wrapper.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_tlul_adapter_sram_0.1/rtl/tlul_adapter_sram.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_top_earlgrey_rv_plic_0.1/rtl/autogen/rv_plic_reg_top.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_top_earlgrey_rv_plic_0.1/rtl/autogen/rv_plic.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_gpio_0.1/rtl/gpio_reg_top.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_gpio_0.1/rtl/gpio.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_tlul_socket_m1_0.1/rtl/tlul_socket_m1.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_tlul_common_0.1/rtl/tlul_fifo_async.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_tlul_common_0.1/rtl/tlul_fifo_sync.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_tlul_common_0.1/rtl/tlul_assert.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_tlul_common_0.1/rtl/tlul_err.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_tlul_common_0.1/rtl/tlul_assert_multiple.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_flash_ctrl_0.1/rtl/flash_mp.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_flash_ctrl_0.1/rtl/flash_phy.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_flash_ctrl_0.1/rtl/flash_phy_core.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_flash_ctrl_0.1/rtl/flash_rd_ctrl.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_flash_ctrl_0.1/rtl/flash_ctrl.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_flash_ctrl_0.1/rtl/flash_prog_ctrl.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_flash_ctrl_0.1/rtl/flash_erase_ctrl.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_flash_ctrl_0.1/rtl/flash_ctrl_reg_top.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_usbdev_0.1/rtl/usbdev_iomux.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_usbdev_0.1/rtl/usbdev_reg_top.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_usbdev_0.1/rtl/usbdev.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_usbdev_0.1/rtl/usbdev_usbif.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_usbdev_0.1/rtl/usbdev_flop_2syncpulse.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_usbdev_0.1/rtl/usbdev_linkstate.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_xilinx_clock_mux2_0/rtl/prim_xilinx_clock_mux2.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_rv_timer_0.1/rtl/rv_timer_reg_top.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_rv_timer_0.1/rtl/rv_timer.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_rv_timer_0.1/rtl/timer_core.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_pinmux_component_0.1/rtl/pinmux.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_xilinx_ram_2p_0/rtl/prim_xilinx_ram_2p.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_tracer_0.1/rtl/ibex_tracer.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_spi_device_0.1/rtl/spi_fwmode.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_spi_device_0.1/rtl/spi_fwm_txf_ctrl.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_spi_device_0.1/rtl/spi_fwm_rxf_ctrl.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_spi_device_0.1/rtl/spi_device_reg_top.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_spi_device_0.1/rtl/spi_device.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_uart_0.1/rtl/uart_rx.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_uart_0.1/rtl/uart_reg_top.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_uart_0.1/rtl/uart.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_uart_0.1/rtl/uart_tx.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_uart_0.1/rtl/uart_core.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_generic_rom_0/rtl/prim_generic_rom.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_top_earlgrey_alert_handler_reg_0.1/rtl/autogen/alert_handler_reg_top.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_tlul_adapter_reg_0.1/rtl/tlul_adapter_reg.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_xilinx_clock_gating_0/rtl/prim_xilinx_clock_gating.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_generic_ram_1p_0/rtl/prim_generic_ram_1p.sv"
`include "build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_rv_core_ibex_0.1/rtl/rv_core_ibex.sv"
```

Then use the following command to transcode to get verilog.
```
# "SYNTHESIS" macro is used for conditional compilation to remove some non-synthesizable code
sv2v -DSYNTHESIS -I build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_assert_0.1/rtl opentitan.sv > opentitan.v
```

## code fix

The code generated by sv2v cannot be used directly, and there are some problems that need to be fixed. The following are my modifications:  
```
diff -r lowrisc_systems_top_earlgrey_verilator_0.1 ~/workspaces/opentitan/build/lowrisc_systems_top_earlgrey_verilator_0.1
diff -r lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_id_stage.sv /home/merle/workspaces/opentitan/build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_core_0.1/rtl/ibex_id_stage.sv
958c958
< 
---
> /*
1021c1021
< 
---
> */
diff -r lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_tracer_0.1/rtl/ibex_tracer.sv /home/merle/workspaces/opentitan/build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ibex_ibex_tracer_0.1/rtl/ibex_tracer.sv
65c65
< 
---
> /*
643c643
<     */
---
>     * /
913c913
< 
---
> */
diff -r lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_hmac_0.1/rtl/hmac_pkg.sv /home/merle/workspaces/opentitan/build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_ip_hmac_0.1/rtl/hmac_pkg.sv
51,52c51,53
<     sha_word_t conv_data = {<<8{v}};
<     conv_endian = (swap) ? conv_data : v ;
---
>     //sha_word_t conv_data = {<<8{v}};
>     //conv_endian = (swap) ? conv_data : v ;
>     conv_endian = (swap) ? {v[7:0], v[15:8], v[23:16], v[31:24]} : v ;
diff -r lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_alert_receiver.sv /home/merle/workspaces/opentitan/build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_alert_receiver.sv
174c174
< 
---
> /*
213c213
< 
---
> */
diff -r lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_alert_sender.sv /home/merle/workspaces/opentitan/build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_alert_sender.sv
194c194
< 
---
> /*
235c235
< 
---
> */
diff -r lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_cipher_pkg.sv /home/merle/workspaces/opentitan/build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_cipher_pkg.sv
44c44
<   parameter logic [11:0][63:0] PRINCE_ROUND_CONST = {64'hC0AC29B7C97C50DD,
---
>   parameter logic [767:0] PRINCE_ROUND_CONST = {64'hC0AC29B7C97C50DD,
diff -r lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_lfsr.sv /home/merle/workspaces/opentitan/build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_lfsr.sv
302c302
<   if (64'(LfsrType) == 64'("GAL_XOR")) begin : gen_gal_xor
---
>   if (LfsrType == "GAL_XOR") begin : gen_gal_xor
327c327
<   end else if (64'(LfsrType) == "FIB_XNOR") begin : gen_fib_xnor
---
>   end else if (LfsrType == "FIB_XNOR") begin : gen_fib_xnor
392c392
<     if (64'(LfsrType) == 64'("GAL_XOR")) begin
---
>     if (LfsrType == "GAL_XOR") begin
402c402
<     end else if (64'(LfsrType) == "FIB_XNOR") begin
---
>     end else if (LfsrType == "FIB_XNOR") begin
438c438
<   if (ExtSeedSVA) begin : gen_ext_seed_sva
---
> //  if (ExtSeedSVA) begin : gen_ext_seed_sva
442,443c442,443
<     `ASSERT(ExtDefaultSeedInputCheck_A, (seed_en_i && rst_ni) |=> lfsr_q == $past(seed_i))
<   end
---
> //    `ASSERT(ExtDefaultSeedInputCheck_A, (seed_en_i && rst_ni) |=> lfsr_q == $past(seed_i))
> //  end
448c448
<   if (LockupSVA) begin : gen_lockup_mechanism_sva
---
> //  if (LockupSVA) begin : gen_lockup_mechanism_sva
450,452c450,452
<     `ASSERT(LfsrLockupCheck_A, lfsr_en_i && lockup && !seed_en_i |=> !lockup)
<   end
< 
---
> //    `ASSERT(LfsrLockupCheck_A, lfsr_en_i && lockup && !seed_en_i |=> !lockup)
> //  end
> /*
486c486
< 
---
> */
diff -r lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_prince.sv /home/merle/workspaces/opentitan/build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_all_0.1/rtl/prim_prince.sv
81c81
<     data_state[0] ^= prim_cipher_pkg::PRINCE_ROUND_CONST[0][DataWidth-1:0];
---
>     data_state[0] ^= prim_cipher_pkg::PRINCE_ROUND_CONST[0 * 64 + DataWidth-1: 0 * 64 + 0];
106c106
<                             prim_cipher_pkg::PRINCE_ROUND_CONST[k][DataWidth-1:0];
---
>                             prim_cipher_pkg::PRINCE_ROUND_CONST[k * 64 + DataWidth - 1:k * 64 + 0];
142c142
<                              prim_cipher_pkg::PRINCE_ROUND_CONST[10-NumRoundsHalf+k][DataWidth-1:0];
---
>                              prim_cipher_pkg::PRINCE_ROUND_CONST[(10 - NumRoundsHalf + k) * 64 + DataWidth - 1 : (10 - NumRoundsHalf + k) * 64 + 0];
167c167
<               prim_cipher_pkg::PRINCE_ROUND_CONST[11][DataWidth-1:0];
---
>               prim_cipher_pkg::PRINCE_ROUND_CONST[11 * 64 + DataWidth - 1: 11 * 64 + 0];
diff -r lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_diff_decode_0/rtl/prim_diff_decode.sv /home/merle/workspaces/opentitan/build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_diff_decode_0/rtl/prim_diff_decode.sv
210c210
< 
---
> /*
259c259
< 
---
> */
diff -r lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_generic_pad_wrapper_0/rtl/prim_generic_pad_wrapper.sv /home/merle/workspaces/opentitan/build/lowrisc_systems_top_earlgrey_verilator_0.1/src/lowrisc_prim_generic_pad_wrapper_0/rtl/prim_generic_pad_wrapper.sv
34a35
> /*
48c49,50
< 
---
> */
>   assign inout_io = (oe) ? out : 1'bz;
```

### Empty statement

Since the assert statement is not synthesizable, no code will be generated. But there are some assertions in the source code based on the conditions, which will generate empty if statements. Yosys does not accept empty if statements.

The following files have empty sentences that need to be corrected:  
```
	ibex_id_stage.sv
	prim_diff_decode.sv
	prim_lfsr.sv
	prim_alert_receiver.sv
	prim_alert_sender.sv
```

### Stream operator

In systemverilog, stream operators can easily implement endian conversion. sv2v will translate the stream manipulation into a complex function, causing yosys to report an error. We need to modify the corresponding code to achieve endian conversion by means of verilog. This code is located in `hmac_pkg.sv`.

The original implementation of systemverilog is as follows:  
```
  typedef logic [31:0] sha_word_t;

  function automatic sha_word_t conv_endian( input sha_word_t v, input logic swap);
    sha_word_t conv_data = {<<8{v}};
    conv_endian = (swap) ? conv_data : v ;
  endfunction : conv_endian
```

The revised implementation is as follows:  
```
  typedef logic [31:0] sha_word_t;

  function automatic sha_word_t conv_endian( input sha_word_t v, input logic swap);
    conv_endian = (swap) ? {v[7:0], v[15:8], v[23:16], v[31:24]} : v ;
  endfunction : conv_endian
```

### Driving strength
The strength syntax is not synthesizable, so the relevant code needs to be modified. The relevant code is located in the `prim_generic_pad_wrapper.sv` file

The code before modification is as follows:  
```
// driving strength attributes are not supported by verilator
`ifdef VERILATOR
  assign inout_io = (oe) ? out : 1'bz;
`else
  // different driver types
  assign (strong0, strong1) inout_io = (oe && drv == STRONG_DRIVE) ? out : 1'bz;
  assign (pull0, pull1)     inout_io = (oe && drv == WEAK_DRIVE)   ? out : 1'bz;
  // pullup / pulldown termination
  assign (highz0, weak1)    inout_io = pu;
  assign (weak0, highz1)    inout_io = ~pd;
  // fake trireg emulation
  assign (weak0, weak1)     inout_io = (kp) ? inout_io : 1'bz;
`endif
```

The modified code is as follows:
```
assign inout_io = (oe) ? out : 1'bz;
/*
// driving strength attributes are not supported by verilator
`ifdef VERILATOR
  assign inout_io = (oe) ? out : 1'bz;
`else
  // different driver types
  assign (strong0, strong1) inout_io = (oe && drv == STRONG_DRIVE) ? out : 1'bz;
  assign (pull0, pull1)     inout_io = (oe && drv == WEAK_DRIVE)   ? out : 1'bz;
  // pullup / pulldown termination
  assign (highz0, weak1)    inout_io = pu;
  assign (weak0, highz1)    inout_io = ~pd;
  // fake trireg emulation
  assign (weak0, weak1)     inout_io = (kp) ? inout_io : 1'bz;
`endif
*/
```

### ibex_tracer.sv

ibex_tracer is a module for outputting debugging and tracing information. It only has  input signals no output signals. There are a large number of non-synthesizable codes. Errors will occur during synthesis, so comment out the module body.

### prim_lfsr.sv

The prim_lfsr module has a parameter named LfsrType, which is a string. Located in the file prim_lfsr.sv. There are some string comparison codes in the code. These codes use type conversion. Sv2v translates it into a function named sv2v_cast_64. This function will trigger an error. So modify the source code and compare the strings directly.

Source code involved:
```
	prim_cipher_pkg.sv
	prim_prince.sv
```

### Two-dimensional array

There are restrictions on using two-dimensional arrays in verilog. sv2v converts the two-dimensional array declaration statement to a one-dimensional array when performing code conversion, but does not modify the code to access the two-dimensional array, so you need to manually modify the code to access the two-dimensional array.

# Synthesize

First, we need to create a synthetic script(build.ys) for yosys. The content is as follows:  
```
read_verilog opentitan.v
hierarchy -check -top top_earlgrey
synth_ice40
write_blif out.blif
```

Then use the following command to synthesize:  
```
yosys -s build.ys
```









