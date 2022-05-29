---
layout: article

title:  监控 magisk su 调用
date:   2022-05-29 12:26:13 +0800
 
tags: Android tools
key: monitor_magisk_su
---

目前市面上 Android 手机 root 采用最多的就是使用 magisk 的方案，方便和通用性比较强。特别是一些黑灰产改机的 rom，是正常 rom + magisk + app 的方案，主要逻辑是在 app 里面，在该 app 中会使用 root 进行执行特权操作，即调用 magisk 的 su 程序，本文介绍一种方案，可以监控 magisk su 调用的命令和返回的结果。

<!--more-->

## Magisk su 的使用

Magisk su 原理比较简单，magisk 在开机后会启动一个高权限 root 用户的进程 magiskd，调用 su 会通过 socket 连接到 magiskd，然后在 magiskd 进程中，会检查调用者权限，权限校验通过后，magiskd 以 root 用户启动 /system/bin/sh 来执行命令，即完成 su 的调用。

su 有两种使用方式：

1. su 后面包含执行的命令，最终调用 sh 时传入该命令，此时 sh 执行完后会立即返回，调用方式为：
* su -c "command"

2. su 后面没有命令参数，调用 sh 像打开正常的交互终端，等待输入，然后输出结果，直到输入退出命令，调用方式为：
* su


对于第一种方式，可以直接在 su 的代码中进行监控，查看输入的命令，而对于第二种方式，调用 sh 后是调用者和 sh 之间的交互，要想监控输入，只有在 sh 中才能监控。


## 监控 su 的原理

为了尽可能少的修改 su 和不修改 sh，可以使用管道的原理，在 su 和 sh 之间增加一层，该层用于监控和传送他们之间的数据。


![graph]({{"/assets/pictures/android/fake_sh.png" | prepend:site.baseurl}})

su 会默认调用 /system/bin/sh，我们将修改为 /data/local/tmp/fake-sh，然后在 fake-sh 中进行数据的监控和转发。


![graph]({{"/assets/pictures/android/fake_sh_pipe.png" | prepend:site.baseurl}})

如上图，除了 su 外，还有4个进程，除了 fake-sh 外，其他进程都是由 fake-sh 创建的子进程，分别为：

* 子进程1： 调用实际的 sh 执行 su 的命令
* 子进程2： 读取 sh 的正常输出，然后打印出该输出，并将输出传给 su
* 子进程3： 读取 sh 的错误输出，然后打印出该错误输出，并将错误输出传给 su

fake-sh 除了创建上面3个子进程外，还有：

* 创建 3个 pipe 管道，用于在进程之间传输数据
* 会读取 su 的输入，即执行的命令，然后打印出该命令，并将该命令传给 sh 进行执行
* 在收到进程1退出信号时，结束进程2和进程3，并退出

## 实现

### fake-sh 的实现

[https://github.com/liwugang/MonitorMagiskSu](https://github.com/liwugang/MonitorMagiskSu)，通过 ndk 进行编译，然后将编译的 fake-sh 导入到 /data/local/tmp/ 目录


### Magisk 改动

Magisk 的改动较少，只需要将 su 使用的 sh 路径从 /system/bin/sh 修改为 /data/local/tmp/fake-sh 即可。

#### 源码修改

修改下面文件：
https://github.com/topjohnwu/Magisk/blob/master/native/jni/su/su.hpp#L9

```bash
--- a/native/jni/su/su.hpp
+++ b/native/jni/su/su.hpp
@@ -6,7 +6,7 @@
 
 #include <db.hpp>
 
-#define DEFAULT_SHELL "/system/bin/sh"
+#define DEFAULT_SHELL "/data/local/tmp/fake-sh"

```

然后重新编译生成 magisk apk 进行安装

#### Patch 现有的 magisk apk

修改的地方只有一处，可以通过 IDA 或者其他工具，对 su 进行 patch 修改，然后在安装，最新的 magisk 版本是 Magisk-v24.3，我在上面 fake-sh 的仓库中提供了修改后的 magisk，可以直接用来使用


## 最后

如果不需要进行监控，可以将 /system/bin/sh 拷贝到 /data/local/tmp/fake-sh，使用默认的 sh 执行

fake-sh 代码：[https://github.com/liwugang/MonitorMagiskSu](https://github.com/liwugang/MonitorMagiskSu)