###Getting Started

#####Introduction

We got this great rouer to run OpenWrt. It has 128MB NAND Flash, 512MB RAM. We could do a lot things in this machine.

#####OpenWrt Firmware

Official OpenWrt support[1] for the WRT AC Series began under [Chaos Calmer](https://wiki.openwrt.org/toh/linksys/wrt1x00ac_series#stable1) [CC], with [Designated Driver](https://wiki.openwrt.org/toh/linksys/wrt1x00ac_series#development) [DD] being the current Development Branch. 
For daily use, we recommend the [Official OpenWrt Stable branch](https://wiki.openwrt.org/toh/linksys/wrt1x00ac_series#stable1). 
You can simply download `system image` and using wrt1900acs stock rom's "firmware upgrade" function to flash it.   

If you experiencing WiFi stability issues, please following this [instructions](https://wiki.openwrt.org/toh/linksys/wrt1x00ac_series#w8864_mwlwifi) to upgrade the wifi firmware.   

#####Luci interface

Luci is a web ui for management. By default, the url is `http://192.168.1.1`   
After login, we should change the default password. And disable the password authentication and root login with password for SSH access, and add ssh-key. So we can use SSH public-key authentication only   

#####Routine Configuration

As a wireless router, if we want to use the wifi function, we should enable it manually, because it disable by default.

###Advance use   

#####Integrated shadowsocks

In our location, there is something \*\*Special\*\* about network connectivity. So we must take measures to overcome this kind of \*\*Special\*\*. As a temporary measure, here comes the [shadowsocks](https://github.com/shadowsocks/shadowsocks). In OpenWrt we using the [shadowsocks-libev](https://github.com/shadowsocks/shadowsocks-libev) as our choose. We can find the Shadowsocks-libev for OpenWrt in this [link](https://github.com/shadowsocks/openwrt-shadowsocks)

You can build it on your own by following this [instruction](https://github.com/shadowsocks/openwrt-shadowsocks)   

or   

You can using the [prebuild binary](https://bintray.com/aa65535/opkg/shadowsocks-libev/) by Shadowsocks-libev for OpenWrt's maintainer [aa65535](https://github.com/aa65535)    


###Reference:
[1] https://wiki.openwrt.org/toh/linksys/wrt_ac_series
