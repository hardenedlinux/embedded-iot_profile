# Summary

This article is a breif intro about how to run [coreboot](https://www.coreboot.org/) + BBL + Linux kernel on [HiFive Unleashed](https://www.sifive.com/boards/hifive-unleashed).

The current coreboot version( Jan 7 2019) is not able to run linux kernel on HiFive Unleashed yet. We've been using the [workaround version](https://github.com/hardenedlinux/coreboot-HiFiveUnleashed/tree/HiFive-Unleashed-Test-Change) for test provided by HardenedLinux and BBL/linux provided by SiFive. Plz note that we will continue to upstreaming Unleashed code to the coreboot. W/ many thanks to Jonathan Neuschäfer, Philipp Hug and Ron Minnich.

# Get the source code

```bash
git clone -b HiFive-Unleashed-Test-Change git@github.com:hardenedlinux/coreboot-HiFiveUnleashed.git
git clone git@github.com:sifive/freedom-u-sdk.git
```

# Build BBL

Because coreboot will occupy a portion of the memory starting at 0x80000000, BBL cannot run from address 0x80000000. So we need to adjust the starting address of the BBL. Modify freedom-u-sdk/riscv-pk/bbl/bbl.lds as follows:

```diff
diff --git a/bbl/bbl.lds b/bbl/bbl.lds
index 2fd0d7c..181f3ff 100644
--- a/bbl/bbl.lds
+++ b/bbl/bbl.lds
@@ -10,7 +10,7 @@ SECTIONS
   /*--------------------------------------------------------------------*/

   /* Begining of code and text segment */
-  . = 0x80000000;
+  . = 0x82000000;
   _ftext = .;
   PROVIDE( eprol = . );
```

Type make to compile. BBL's elf image is located at freedom-u-sdk/work/riscv-pk/bbl

# Build coreboot

## Build toolchain

```bash
make crossgcc-riscv
```

## Configuration

```bash
make menuconfig
```

- Mainboard->Mainboard vendor, select **SiFive**
- Mainboard->Mainboard model, select **HiFive Unleashed**
- Chipset->Privilege level for payload, select **payload running in m-mode**
- Payload->Add a payload, select **An ELF executable payload**
- Payload->Payload path and filename, Use default value **payload.elf**

## Compile

Copy BBL image from freedom-u-sdk/work/riscv-pk/bbl to coreboot/payload.elf, then type make to compile.

# Burn

There are two ways to burn coreboot, write to spi flash or sdcard.

## Burn to sdcard

```bash
sudo dd build/coreboot.rom /dev/sdx
```

**/dev/sdx** is the device of your sd card reader

## Burn to spi flash

### Get the original firmware

```bash
wget https://static.dev.sifive.com/dev-kits/hifive-unleashed/hifive-unleashed-firmware-1.0.zip
```

### Start the original firmware from the sd card

Unzip the original firmware and burn hifive-unleashed-a00-A.B-YYYY-MM-DD.gpt to the sd card.

```bash
sudo dd if=hifive-unleashed-a00-A.B-YYYY-MM-DD.gpt of=/dev/sdx
```

**/dev/sdx** is the device of your sd card reader

Turn the MSEL DIP switch to 11, connect the USB and network cable, and connect ttyUSB1 via minicom (baud rate 115200 8N1), then press the reset button to restart.

Then log in the Linux by terminal, username: root, password: sifive. In my test the network can not be automatically configured, you need to enter the following command by yourself.

```bash
/etc/init.d/S40network restart
```

### Burn to spi flash

```bash
scp coreboot/build/coreboot.rom root@$target_ip:/tmp/
ssh root@$target_ip "/usr/sbin/flashcp -v /tmp/coreboot.rom /dev/mtd0"
```

**target_ip** is the IP of HiFive Unleashed

# Testing

Turn the MSEL DIP switch to 15/11 (15 for boot from spi flash, 11 for boot from sdcard), connect the USB and network cable, and connect ttyUSB1 via minicom (baud rate 115200 8N1), then press the reset button to restart. Then you will see the log in the terminal.

Linux username: root, password: sifive.
