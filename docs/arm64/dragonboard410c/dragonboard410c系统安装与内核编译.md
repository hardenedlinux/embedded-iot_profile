# 0 概要

此文就dragonboard410c系统安装，内核编译等问题进行描述。此文主要参考官方文档[Dragonboard 410c Installation Guide for Linux and Android](https://github.com/96boards/documentation/wiki/Dragonboard-410c-Installation-Guide-for-Linux-and-Android)编写。



# 1 系统安装

## 1.1 通过SD卡安装系统

### 1.1.1 准备SD卡

1\. 下载SD卡镜像压缩文件[下载](http://builds.96boards.org/releases/dragonboard410c/linaro/debian/latest/dragonboard410c_sdcard_install_debian*.zip)
2\. 解压文件
3\. 写入SD卡，linux系统下通过`sudo dd if=db410c_sd_install_YYY.img of=/dev/XXX bs=4M oflag=sync status=noxfer`命令写入SD卡（XXX为设备名称），win下需要通过[Win32DiskImager tool](http://sourceforge.net/projects/win32diskimager/)把镜像文件烧入SD卡。

### 1.1.2 选择从SD卡启动

在dragonboard410c背后有4个一组的拨码开关，开关开假定为状态1反之为0，默认状态为0000。根据丝印SD BOOT,拨动此开关，即可选择从SD卡启动，这时状态为0100。

### 1.1.3 准备外设

要通过SD卡安装系统，需要键盘鼠标以及HDMI接口的显示器。链接好这些设备。

### 1.1.4 系统安装
开发板连接电源，等待显示器出现安装界面，根据提示操作即可。
在系统安装完后，拨码开关恢复状态0000,重新上电即可。

## 1.2 通过fastboot安装系统

### 1.2.1 安装fastboot

linux通过下命令`sudo apt install android-tools-fastboot`安装
win请自行百度adb工具，一般会附带fastboot

### 1.2.2 准备镜像文件

1\. 下载启动镜像压缩包[下载](http://builds.96boards.org/releases/dragonboard410c/linaro/debian/latest/boot-linaro-jessie-qcom-snapdragon-arm64*.img.gz)
2\. 下载根文件系统镜像压缩包[下载](http://builds.96boards.org/releases/dragonboard410c/linaro/debian/latest/linaro-jessie-developer-qcom-snapdragon-arm64*.img.gz)
3\. 下载后解压获取镜像文件

### 1.2.3 写入内部eMMC存储器

1\.  拨码开关拨到状态0000
2\.  按住音量-（S4）开关，上电
3\.  保持音量-（S4）按下，点击一下电源开关（S2）
4\.  松开音量-（S4）
5\.  插入micro-usb数据线，链接主机
6\.  输入`sudo fastboot devices`测试开发板是否进入fastboot(没有输出即进入fastboot失败)
7\.  输入`sudo fastboot flash boot boot-linaro-jessie-qcom-snapdragon-arm64-BUILD#.img`命令烧入启动镜像文件到boot分区
8\.  输入`sudo fastboot flash rootfs linaro-jessie-developer-qcom-snapdragon-arm64-BUILD#.img`命令烧写根文件系统镜像到rootfs分区
9\.  输入`sudo fastboot reboot`命令重启开发板
10\. 记住，要拔掉micro-usb数据线，不然启动会失败
11\. 用户名和密码都为linaro

# 2 内核编译

## 2.1 获取内核
下载内核源码压缩包[链接](https://git.linaro.org/landing-teams/working/qualcomm/kernel.git)，解压

## 2.2 获取交叉编译工具
下载交叉编译工具[链接](https://releases.linaro.org/components/toolchain/binaries/5.2-2015.11-2/aarch64-linux-gnu/gcc-linaro-5.2-2015.11-2-x86_64_aarch64-linux-gnu.tar.xz)

## 2.3 安装交叉编译工具
解压交叉编译工具到任意目录，设置PATH环境变量即可

## 2.4 为内核编译设置环境变量
```shell
export ARCH=arm64	# 设置内核的目标平台
export CROSS_COMPILE=aarch64-linux-gnu-   #设置交叉编译器前缀
```
## 2.5 内核配置

进入内核源码根目录
```shell
make defconfig distro.config
```
## 2.6 编译

通过`make -j16 Image dtbs modules KERNELRELEASE=4.4.23-linaro-lt-qcom`命令编译内核镜像、device-tree、内核模块

## 2.7 制作启动镜像文件

1\. 下载工具`git clone git://codeaurora.org/quic/kernel/skales，ubuntu下依赖libfdt-dev，需要使用sudo apt install libfdt-dev`命令安装。
2\. 下载初始化内存盘`wget http://builds.96boards.org/releases/dragonboard410c/linaro/debian/16.09/initrd.img-4.4.23-linaro-lt-qcom`
3\. 制作device-tree-blob命令` ./skales/dtbTool -o dt.img -s 2048 arch/arm64/boot/dts/qcom/`
4\. 制作启动镜像，命令
```shell

```

## 2.8 测试内核

`sudo fastboot boot boot-db410c.img`
## 2.9 替换内核

`sudo fastboot falsh boot boot-db410c.img`
