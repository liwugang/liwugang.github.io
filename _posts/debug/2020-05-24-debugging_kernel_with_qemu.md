---
layout: article

title:  使用QEMU调试Linux内核
date:   2020-05-26 23:00:00 +0800
 
tags: debug
key:  debug_kernel_with_qemu
---

最近准备深入写关于SELinux方面文章，SELinux逻辑主要是在内核，所以需要搭建下调试内核的环境，本文介绍使用QEMU调试，下一篇文章将介绍主机调试virtualbox中centos。

<!--more-->
## 环境搭建

### mkroot工具
mkroot工具作者是landley大佬，landley大佬是著名的toybox工具的owner，toybox是集成绝大多数Linux命令为一个可执行文件中，以简单、小巧和快速等特点被集成到Android中。
mkroot工具的介绍是"Compiles a toybox-based root filesystem and kernel that can boot under qemu."，可以看到直接使用mkroot工具可以帮我们构建一个由QEMU启动的kernel和ramdisk。

在原来工具基础上，我有两处修改，[下载地址](https://github.com/liwugang/mkroot)
1. kernel的下载地址替换为国内清华源；
2. 将目录打包成ramdisk命令提取到单独文件中。

### 运行

在下载工具的根目录下执行：
> ./mkroot.sh  NO_CLEANUP=true kernel

默认情况下编译完后会将源码删掉，为了保留源码，需要带上参数 NO_CLEANUP=true。

![graph]({{"/assets/pictures/debug/mkroot.png" | prepend:site.baseurl}})

从图上看到，根目录下的build目录是源码位置，output/host下有生成的ramdisk、kernel、QEMU启动脚本等文件。

由于编译kernel的选项是采用简单配置，可能不满足需求，此时需要到build目录下的linux源码目录下，重启配置选项编译，然后将生成的bzImage拷贝到output/host或者修改qemu-x86_64.sh脚本，将kernel指向刚生成的bzImage。

## 调试

使用两个终端，终端一在output/host目录下，执行：
> ./qemu-x86_64.sh -s -S

-s是”-gdb tcp::1234“的简写，用于监听tcp::1234端口，-S是让在启动后暂停

终端二在kernel源码目录(build/host-tmp/linux-5.1)下，执行：
> gdb vmlinux

gdb加载kernel
> target remote:1234

连接1234端口

> b start_kernel

在start_kernel函数下断点

> continue

继续执行

此时会断点在start_kernel函数中，当然也可以根据需求设置其他函数断点，然后进行调试。


## 用户态程序执行

要调试kernel的某个函数，默认情况下触发不到怎么办？ 那只有自己写用户态程序进行主动触发。

### 用户态程序要求

由于ramdisk中不包含有共享库，所以用户态程序依赖共享库。生成简单的helloworld可执行文件，用ldd输出可执行文件依赖的共享库

![graph]({{"/assets/pictures/debug/dynamic_so.png" | prepend:site.baseurl}})

可以看到有依赖的共享库，为了能够运行，需要编译时将依赖的都编译进可执行文件中，**gcc或g++编译时增加 -static 参数即可**。

![graph]({{"/assets/pictures/debug/static_exe.png" | prepend:site.baseurl}})

加上参数后，可以看到没有依赖的共享库。

### 打包进ramdisk

![graph]({{"/assets/pictures/debug/ramdisk.png" | prepend:site.baseurl}})

在output/host下，将用户态程序移动到root目录下（可以是root下的任意位置），然后执行cpio.sh将root目录进行打包即可。此时运行qemu-x86_64.sh就会发现可执行文件存在。


