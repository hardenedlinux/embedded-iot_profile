# Summary

This article is used to show how to run coreboot on HiFive Unleashed and run
Linux by BBL

The current community version of coreboot can not run linux, here we use the
test version provided by hardenedlinux, and BBL and linux provided by sifive.

# Get the source code

```
git clone -b HiFive-Unleashed-Test-Change git@github.com:hardenedlinux/coreboot-HiFiveUnleashed.git
git clone git@github.com:sifive/freedom-u-sdk.git
```

# Build BBL

Because coreboot will occupy a portion of the memory starting at 0x80000000, BBL
cannot run from address 0x80000000. So we need to adjust the starting address of
the BBL. Modify freedom-u-sdk/riscv-pk/bbl/bbl.lds as follows:

```
diff --git a/bbl/bbl.lds b/bbl/bbl.lds
index 2fd0d7c..181f3ff 100644
--- a/bbl/bbl.lds
+++ b/bbl/bbl.lds
@@ -10,7 +10,7 @@ SECTIONS
   /*--------------------------------------------------------------------*/
 
   /* Begining of code and text segment */
-  . = 0x80000000;
+  . = 0x80200000;
   _ftext = .;
   PROVIDE( eprol = . );
```

Type make to compile. BBL's elf image is located at freedom-u-sdk/work/riscv-pk/bbl

# Build coreboot

## Build toolchain

```
make crossgcc-riscv
```

## Configuration

```
make menuconfig
```

- Mainboard->Mainboard vendor, select **SiFive**
- Mainboard->Mainboard model, select **HiFive Unleashed**
- Chipset->Privilege level for payload, select **payload running in m-mode**
- Payload->Add a payload, select **An ELF executable payload**
- Payload->Payload path and filename, Use default value **payload.elf**

## Compile

Copy BBL image from freedom-u-sdk/work/riscv-pk/bbl to coreboot/payload.elf, then
type make to compile.

# Burn

## Get the original firmware

```
wget https://static.dev.sifive.com/dev-kits/hifive-unleashed/hifive-unleashed-firmware-1.0.zip
```

## Start the original firmware from the sd card

Unzip the original firmware and burn hifive-unleashed-a00-A.B-YYYY-MM-DD.gpt to
the sd card.

```
sudo dd if=hifive-unleashed-a00-A.B-YYYY-MM-DD.gpt of=/dev/sdx
```

/dev/sdx is the device of your sd card reader

Turn the MSEL DIP switch to 11, connect the USB and network cable, and connect
ttyUSB1 via minicom (baud rate 115200 8N1), then press the reset button to
restart.

Then log in the Linux by terminal, username: root, password: sifive. In my test
the network can not be automatically configured, you need to enter the following
command by yourself.

```
/etc/init.d/S40network restart
```

## Burn to spi flash

```
scp coreboot/build/coreboot.rom root@$target_ip:/tmp/
ssh root@$target_ip "/usr/sbin/flashcp -v /tmp/coreboot.rom /dev/mtd0"
```

target_ip is the IP of HiFive Unleashed

# Testing

Turn the MSEL DIP switch to 15, connect the USB and network cable, and connect
ttyUSB1 via minicom (baud rate 115200 8N1), then press the reset button to
restart. Then you will see the log in the terminal.

Linux username: root, password: sifive.

