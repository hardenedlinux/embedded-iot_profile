# Summary

This article is a breif intro about how to run [coreboot](https://www.coreboot.org/) + [opensbi](https://github.com/riscv/opensbi) + linux kernel on [HiFive Unleashed](https://www.sifive.com/boards/hifive-unleashed). 

Separate hardware-initiated code for a simpler upgrade. To do this, a loader called riscv-gptl was added to load opensbi and linux from the sdcard and run as a coreboot payload.

# Get Sources Code

```
git clone git@github.com:hardenedlinux/coreboot-HiFiveUnleashed.git -b gptl coreboot
git clone git@github.com:wxjstz/opensbi.git -b gptl
git clone git@github.com:wxjstz/riscv-gptl.git
git clone git@github.com:sifive/freedom-u-sdk.git
```

# intro riscv-gptl

riscv-gptl is a loader that runs in M mode and loads the riscv-gptl image from the GPT partition of the **4750544c-0000-0000-0000-524953432d56** partition type.

riscv-gptl image is a binary, which contains opensbi, kernel, optional fdt and optional initrd. This image has a header `struct gptl_header` to describe the layout of this files (opensbi, kernel, fdt and initrd).
```c
struct gptl_header {
	uint32_t magic;		/* "GPTL" -> 0x4750544c */

	uint32_t version;
	uint32_t header_size;
	uint32_t total_size;

	uint32_t opensbi_offset;
	uint32_t opensbi_size;

	uint32_t kernel_offset;
	uint32_t kernel_size;

	uint32_t initrd_offset;
	uint32_t initrd_size;

	uint32_t fdt_offset;
	uint32_t fdt_size;

	char commandline[0];
};
```

If you want to adapt to a specific motherboard, you need to implement `struct platform_interface`
```c
struct platform_interface {
	int hart_num;
	char *platform_name;
	void (*init) (void);
	void (*putchar) (char c);
	size_t logic_block_size;
	void *(*read) (size_t offset, size_t size, void *buff);
};
```

Currently using opensbi's FW\_JUMP firmware to provide sbi services. FW\_JUMP needs to load openbsi/kernel to a fixed address and reserve memory with fixed address for fdt.

Build dependencies with the following environment variables

- **CROSS\_COMPILE**: Must be defined. The name prefix of the RISC-V compiler toolchain executables, e.g. **riscv64-elf-** if the gcc executable used is **riscv64-elf-gcc**.
- **PLATFORM**: Must be defined. A subdirectory under the platform that specifies the target platform
- **DEBUG**: The build option of riscv-gptl to generate DEBUG macro, when $(DEBUG) = y.
- **CONFIG\_FW\_START**: The starting address of riscv-gptl, the default is 0x80000000
- **CONFIG\_HEAP\_SIZE**: The heap size of riscv-gptl, the default is 65536.
- **CONFIG\_STACK\_SIZE**: The stack size of riscv-gptl, the default is 65536.
- **CONFIG\_RISCV\_ARCH**: The build option of riscv-gptl, -march=$(CONFIG\_RISCV\_ARCH)
- **CONFIG\_RISCV\_ABI**: The build option of riscv-gptl, -mabi=$(CONFIG\_RISCV\_ABI)
- **CONFIG\_RISCV\_CODEMODEL**: The build option of riscv-gptl, -mcmodel=$(CONFIG\_RISCV\_CODEMODEL)
- **CONFIG\_OPENSBI\_ADDR**: Must be defined. The running address of opensbi, related to the opensbi build parameter
- **CONFIG\_KERNEL\_ADDR**: Must be defined. The running address of kernel, related to the opensbi build parameter
- **CONFIG\_FDT\_ADDR**: Must be defined. The memory address reserved for opensbi, used by opensbi to modify fdt, related to the opensbi build parameter
- **CONFIG\_RESERVED\_FOR\_FDT**: Must be defined. The fdt memory size reserved for opensbi, used to calculate the memory start address of the initrd

# Build

## riscv-gptl

```
cd riscv-gptl
. ./platform/hifive-unleashed/build.env
make CROSS_COMPILE=riscv64-elf- CONFIG_FW_START=0x82000000 DEBUG=y

# build riscv-tool which used to create riscv-gptl image
make -C util/gptl-tool
```

## coreboot

```
cd coreboot
make menuconfig
```

- Mainboard->Mainboard vendor, select **SiFive**
- Mainboard->Mainboard model, select **HiFive Unleashed**
- Payload->Add a payload, select **An ELF executable payload**
- Payload->Payload path and filename, Use default value **payload.elf**

```
cp ../riscv-gptl/build/program.elf ./payload.elf
make
```

## opensbi

```
cd opensbi
make CROSS_COMPILE=riscv64-elf- PLATFORM=sifive/fu540
```

## linux

```
cd freedom-u-sdk
make
riscv64-unknown-elf-copy -O binary work/linux/vmlinux-stripped vmlinux.bin
```

## create riscv-gptl image

```
cd riscv-gptl
build/util/gptl-tools/gptl-tools -O gptl.img -o ../opensbi/build/platform/sifive/fu540/firmware/fw_jump.bin -k ../freedom-u-sdk/vmlinux.bin
```


# Burn

Insert the sd card into your computer's card reader

```
sudo dd if=coreboot/build/coreboot.rom of=/dev/sdX; sync
sudo dd if=riscv-gptl/gptl.img of=/dev/sdX2; sync
```

# Testing

Turn the MSEL DIP switch to 11, and connect ttyUSB1 via minicom (baud rate 115200 8N1), then press the reset button to restart. Then you will see the log in the terminal.

Linux username: root, password: sifive.

