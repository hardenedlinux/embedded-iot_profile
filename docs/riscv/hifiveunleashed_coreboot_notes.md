# 概要

这篇文章主要用于介绍如何在HiFive Unleashed上运行coreboot，并通过BBL运行linux。

当前coreboot的社区版本还不能支持linux的运行，这里使用hardenedlinux的测试版本。linux
和BBL使用sifive提供的版本。

# 源码获取

```
git clone -b HiFive-Unleashed-Test-Change git@github.com:hardenedlinux/coreboot-HiFiveUnleashed.git
git clone git@github.com:sifive/freedom-u-sdk.git
```

# 编译bbl镜像

因为coreboot会占用一部分内存，所以bbl不能从0x80000000处开始运行，需要往后移动一点。
这里需要对freedom-u-sdk/riscv-pk/bbl/bbl.lds进行修正，修正如下：

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

然后执行make编译。bbl的elf镜像位于freedom-u-sdk/work/riscv-pk/bbl

# 编译coreboot

## 第一步需要编译出riscv工具链

```
make crossgcc-riscv
```

## 配置编译选项

```
make menuconfig
```

Mainboard->Mainboard vendor，选中SiFive

Mainboard->Mainboard model，选中HiFive Unleashed

Chipset->Privilege level for payload，选中payload running in m-mode

Payload->Add a payload，选中An ELF executable payload

Payload->Payload path and filename，文件名不要修改使用默认的payload.elf

## 编译

拷贝bbl镜像freedom-u-sdk/work/riscv-pk/bbl到coreboot/payload.elf,然后在coreboot下
执行make

# 烧录

## 获取原厂固件

https://static.dev.sifive.com/dev-kits/hifive-unleashed/hifive-unleashed-firmware-1.0.zip

## 从sd卡启动原厂固件

解压下载到的原厂固件，把hifive-unleashed-a00-A.B-YYYY-MM-DD.gpt烧写到sd卡

```
sudo dd if=hifive-unleashed-a00-A.B-YYYY-MM-DD.gpt of=/dev/sdx
```

把MSEL拨到11，连接USB和网线，电脑通过minicom连接ttyUSB1（波特率115200 8N1），按复位
键重启HiFive Unleashed

然后在终端登陆linux（用户名：root，密码：sifive），在我测试中网络不能上电启动需要自己
输入以下命令启动网络
```
/etc/init.d/S40network restart
```

## 烧写coreboot到spi flash

```
scp coreboot/build/coreboot.rom root@$target_ip:/tmp/
ssh root@$target_ip "/usr/sbin/flashcp -v /tmp/coreboot.rom /dev/mtd0"
```
其中，target_ip为HiFive Unleashed获取到的ip地址

# 测试

把MSEL拨到11，连接USB和网线，电脑通过minicom连接ttyUSB1（波特率115200 8N1），按复位
键重启HiFive Unleashed，这时在终端将看到启动的log

linux用户名：root，密码：sifive

