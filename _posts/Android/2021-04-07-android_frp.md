---
layout: article

title:  Android Factory Reset Protection介绍
date:   2021-04-07 23:15:00 +0800
tags: Android
key: android_frp
---

FRP 从字面意思可知，是用于对恢复出厂设置的保护。比如手机丢了，一般情况下小偷都会恢复出厂设置来绕过锁屏密码，这样就能使用了。但若是登录了 google 账号，此时在开机引导处还是要输入密码或者 google 账号才能进入，这就是使用了 FRP。若是从设置里触发恢复出厂设置，不会触发 FRP，可能 google 在设计时认为能进入到设置中操作，此时桌面已经解锁了，那大概率认为是机主所为，所以不会触发。

<!--more-->

## 技术

Android 系统运行中生成的数据和用户数据都是保存在 data 分区，其他分区像 system, boot, vendor等都是只读分区，而恢复出厂设置是会将 data 分区的数据清除，而实现 FRP 肯定是需要保存信息，这样就需要单独分区用来存储 FRP 的信息，即 FRP 分区。该分区不是固定的，是通过属性 ro.frp.pst 获取：

```bash
[ro.frp.pst]: [/dev/block/bootdevice/by-name/frp]

XXX:/ # ls -lZ /dev/block/bootdevice/by-name/frp
lrwxrwxrwx 1 root root u:object_r:block_device:s0  15 1970-11-19 17:43 /dev/block/bootdevice/by-name/frp -> /dev/block/sda9

XXX:/ # ls -lZ /dev/block/sda9
brw------- 1 system system u:object_r:frp_block_device:s0  8,   9 2021-04-02 03:23 /dev/block/sda9

```
从上面可以看到，ro.frp.pst 是 FRP分区的符号文件，实际文件是 /dev/block/sda9，该文件是块设备文件。


### 访问权限

从 DAC 权限上看，只有 system 用户可以访问。SELinux权限上，该块设备的 security label 是 frp_block_device，搜索 system/sepolicy 代码只有 system_server 有可访问的权限。

```bash
allow system_server frp_block_device:blk_file rw_file_perms;
... 

allow gmscore_app persistent_data_block_service:service_manager find;
allow priv_app persistent_data_block_service:service_manager find;
platform_app persistent_data_block_service:service_manager find;
... ...

```

SystemServer 访问该分区是通过 PersistentDataBlockService 类，该类是注册了 persistent_data_block 系统服务，其他模块必须获取系统服务来访问 FRP 分区。从上面配置来看，三方 app 是不可访问，只有特权 APP 才能访问。

## 应用

### Google 原生解锁

![graph]({{"/assets/pictures/android/unlock_flow.png" | prepend:site.baseurl}})

从上述流程图上可以看到：
* 解锁首先需要到开发者选项中打开 OEM 解锁开关，该操作会调用上述 persistent_data_block 系统服务中的 setOemUnlockEnabled 方法，该方法会将 FRP 分区中 unlock 标识设置为1，该标识位于 FRP 分区最后一个字节处；
* 实际解锁使用 "fastboot flashing unlock" 命令，该命令首先会检查 FRP 分区 unlock 标识是否为1，若为1才允许解锁。[相关代码查看](https://git.linaro.org/landing-teams/working/qualcomm/abl.git/tree/QcomModulePkg/Library/FastbootLib/FastbootCmds.c?h=release/LU.UM.1.2.1.r1-23200-QRB5165.0)。


### Google 账号锁

![graph]({{"/assets/pictures/android/google_lock.jpg" | prepend:site.baseurl}})

上图是登录了 google 账号并且恢复出厂设置后，提示输入密码或者输入 google 账号的界面，gmscore 会读取 FRP 分区，根据保存的信息来决定是否需要锁定，锁定时会设置 Settings.Secure.secure_frp_mode=1，相应判断逻辑是在 gmscore 里面，该 app 不开源，所以不能继续分析。