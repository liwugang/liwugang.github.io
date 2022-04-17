---
layout: article

title:  某多开 app 破解 - 上篇
date:   2022-04-17 10:46:00 +0800

tags: Android reverse
key: reverse_multi_app
---

之前对多开 app 进行了调研，发现了某多开 app 比较优秀，使用起来广告少，崩溃也相对较少，大厂 app 都能稳定运行。但是免费版只支持多开一个，要是多于一个需要会员，故就对该 app 进行了逆向分析，本文只是记录下破解过程，用于学习研究。

本文核心不是分析多开的原理，而是分析非会员不能多开的逻辑，破解掉该防护，以达到非会员也能多开的效果。

为了后面方便对涉及到的 app 称呼，本文把要分析的多开 app 又称为管理 app，要多开的 app 称为原 app，多开出来的 app 称为分身 app。

<!--more-->


## 工具

* jadx-gui
* charles
* IDA
* frida
* elf-dump-fix，用于从内存中 dump dex，([下载地址](https://github.com/maiyao1988/elf-dump-fix/))
* rizin，用于获取 arm 指令的字节码 (和 radare2 类似，从 radare2 独立出来)，下篇用到

## 获取核心的 dex

正常情况下，逆向是首先需要定位到关键代码处，然后从关键代码处开始分析。特别是对于有界面的某个操作，先找到界面对应的 activity，然后分析界面上的操作和显示等。

对于我们的管理 app，就是先进入到不能多开的界面，然后定位其 activity，界面如下图：

![graph]({{"/assets/pictures/reverse/multi_open/main_activity.png" | prepend:site.baseurl}})


### 获取界面 activity

获取 Android 当前界面可以通过各种工具，如 ddms，最简单的方式就是通过 adb shell 命令获取，获取方式如下：

```bash
adb shell dumpsys activity |grep mResumedActivity  # Linux 和 MacOs 可用，Windows 没有测试，请自行使用 findstr 替换 grep 测试
```

结果为：

```bash
mResumedActivity: ActivityRecord{67a7b7a u0 com.XXX.XXXX/.widget.create.SelectCreateAppActivity t294}
```
所以，管理 app 的关键的 activity 类为 SelectCreateAppActivity

通过 jadx-gui 对 apk 打开，然后没有发现 SelectCreateAppActivity 类，并且从类名可以看出来，该 apk 是被加固的，所以核心的 dex 是被隐藏的
 
![graph]({{"/assets/pictures/reverse/multi_open/not_found_activity.png" | prepend:site.baseurl}})

### 寻找核心的 dex

对于加固的 dex，最终都是要加载到内存中执行，有的方式我们可以从内存 maps 中找到蛛丝马迹，有的是使用匿名内存方式加载，这种需要使用 frida dex dump 这类脚本在内存中搜索 dex

对于我们分析的管理 app，比较幸运的是直接能从内存中找到一个 dex，如下图：

![graph]({{"/assets/pictures/reverse/multi_open/search_maps.png" | prepend:site.baseurl}})

将 dex 从手机上取出来，重新使用 jadx-gui 对其打开，发现存在上面的类 SelectCreateAppActivity，所以这个 dex 就是我们找的核心 dex，但是发现没有指令，所有指令是被 nop 填充的，真正的指令被抽取了，如下图：

![graph]({{"/assets/pictures/reverse/multi_open/actual_dex.png" | prepend:site.baseurl}})

### 获取完整的 dex

指令被抽取后，在运行前肯定要回填，所以我们可以把内存的该 dex dump出来，我们使用 elf-dump-fix 工具，该工具主要是 dump so 内存，然后进行 elf 修复。此时我们只需要 dump dex 内存即可

![graph]({{"/assets/pictures/reverse/multi_open/dump_dex.png" | prepend:site.baseurl}})

此时使用 jadx-gui 打开时提示错误，dex 的 checksum 校验失败，打开显示下面错误，这是由于 dex 在内存中被修改导致

![graph]({{"/assets/pictures/reverse/multi_open/checksum_error.png" | prepend:site.baseurl}})

为了验证下 dump 出来的 dex 包含有指令，我们给 jadx-gui 提供绕过 checksum 的检测的参数：-Pdex-input.verify-checksum=no，然后重新使用 jadx-gui 打开：

```bash
jadx-gui -Pdex-input.verify-checksum=no XXX.dex
```

重新打开后，确认有代码了，说明我们 dump 出来的 dex 是完整的，也就是我们要找到的核心 dex

![graph]({{"/assets/pictures/reverse/multi_open/activity_content.png" | prepend:site.baseurl}})


### dex checksum 修复

在系统加载 dex 时会对 dex 的完整性通过 checksum 进行校验，指令重新写回时会修改 dex，导致 dex 的校验失败，此时我们可以 jadx-gui 中的提示信息将实际的 checksum 写回到 dex 中进行修复。

从 dex header 的结构可知，checksum 是在 dex 文件的 8字节偏移，此时我们把正确的 checksum 写入到位置即可，存放是采用小端方式，由上面的错误信息可知：实际的 checksum 是 0x14950f15，写入小端字节序是：15 0f 95 14

```c
 struct Header {
    uint8_t magic_[8] = {};
    uint32_t checksum_ = 0;  // See also location_checksum_
    ... ...
  };
```

修复完成后，可通过 jadx-gui 正常打开 dex，此时得到的就是我们将要分析的核心 dex

## 代码逻辑分析

### 定位关键代码

寻找关键代码的两种思路：

* 直接从代码侧出发，dialog 显示是通过点击应用右侧 "制作分身" button 触发的，通过找到显示 app 信息的逻辑，然后找到对应的 button 的点击事件

* 另外一种是通过提示文案寻找，通过文案中 “普通用户只能双开应用” 找到是在 layout 下的 dialog_muti_need_vip.xml，然后通过 dialog_muti_need_vip 定位到显示 dialog 的代码处，为 D 类中的 d 方法，然后找到调用处


App 信息的类

```java
SelectCreateAppActivity.java
public class d extends me.yokeyword.indexablerv.d<AppEntity> {
    ... ...
    public void a(RecyclerView.ViewHolder viewHolder, AppEntity appEntity) {
        e eVar = (e) viewHolder;
        eVar.f710b.setText(appEntity.getName());
        com.bumptech.glide.c.b(SelectCreateAppActivity.this.getApplicationContext()).d(AppInfoUtils.a(SelectCreateAppActivity.this, appEntity.getPackageName())).a(s.f1879d).a(eVar.f709a);
        eVar.f710b.setText(appEntity.getName());
        if (appEntity.getIsSupport() == 1) {
            eVar.f711c.setOnClickListener(new V(this, appEntity)); // 设置点击 “制作分身”  的事件接收器
            return;
        }
        ... ...
    }
}
```

走到 button 的点击事件接收器类

```java

public class V implements View.OnClickListener {
    ... ...
    @Override // android.view.View.OnClickListener
    public void onClick(View view) {
        SelectCreateAppActivity.this.onClickApp(this.f718a); // 接收到点击事件后，调用 SelectCreateAppActivity 的 onClickApp 方法
    }
    ...
}
```
最终调用的方法

```java
    public void onClickApp(AppEntity appEntity) {
        if (1 != appEntity.getIsSupport()) {
            T.a(this, "正在适配中，敬请期待...");
        } else if (!PackageNameGenarator.d(this, appEntity.getPackageName())) {
            T.a(this, "您的手机上还没有" + appEntity.getName() + ",请先安装官方最新版。");
        } else if (!com.XXX.XXX.a.a.a().A()) {
            T.a(this, "初始化失败，请尝试退出后重新打开XXXX");
        } else if (StringUtils.isNotBlank(com.XXX.XXX.a.a.a().u())) {
            a(appEntity); // 通过文案可知走到这里
        } else {
            T.a(this, "初始化失败，请尝试退出后重新打开XXXX");
        }
    }
```

通过从文案的排查，可知走的是 a 方法

```java

    private void a(AppEntity appEntity) {
        if (com.XXX.XXX.a.a.a().w() != 0 || com.XXX.XXX.a.a.a().E() || !h.a(this, appEntity.getPackageName())) {
            Intent intent = new Intent(this, CreateCustomActivity.class);
            intent.putExtra("entity", appEntity);
            intent.putExtra("ct", this.j);
            startActivity(intent);
            overridePendingTransition(R$anim.in_from_right, R$anim.out_to_left);
            return;
        }
        D.d(this); // 这是显示文案的对话框，所以上面是判断多开和会员的逻辑
    }
```

该方法是核心方法，D.d 是显示不能多开的对话框，通过第一行的 if 后面的条件判断是否可以进行多开

条件一和条件二是通过对 com.XXX.XXX.a.a.a() 中的 w() 和 E() 方法判断，这个类中的信息都是从服务端传递过来的

条件三是通过遍历设备上的 applist 寻找，是否有 metaData 中的 PLUGIN_PACKAGE 为原 app 的包名，若有则表示已经多开过了

```java
    public static boolean a(Context context, String str) {
        try {
            List<PackageInfo> installedPackages = context.getPackageManager().getInstalledPackages(128);
            for (int i = 0; i < installedPackages.size(); i++) {
                PackageInfo packageInfo = installedPackages.get(i);
                if ((packageInfo.applicationInfo.flags & 1) == 0 && (packageInfo.packageName.startsWith("dkmodel") || packageInfo.packageName.startsWith("dkplugin") || .....)) {
                    try {
                        new PluginInfo();
                        String string = packageInfo.applicationInfo.metaData.getString("PLUGIN_PACKAGE", "");
                        if (StringUtils.isNotBlank(string) && string.equals(str)) {
                            return true;
                        }
                    } catch (Exception e2) {
                        e2.printStackTrace();
                    }
                }
            }
        } catch (Exception e3) {
            e3.printStackTrace();
        }
        return false;
    }
```

### 绕过多开限制

#### 尝试 - 本地修改包名绕过

经过上面分析，条件一和条件二都是需要对服务端返回值修改，而条件三是本地包名的判断，是不是可以将分身 app 中的 metadata 里的 PLUGIN_PACKAGE 改成任意包名绕过限制？

经尝试，修改成不存在的包名后，分身 app 无法启动，并且查看 maps 可以发现，其实他会包含有原 app 包的 apk

如下面对夸克进行多开，分身 app 包名是 dkplugin.azg.hjo，在 maps 中能发现有夸克的包名

![graph]({{"/assets/pictures/reverse/multi_open/shadow.png" | prepend:site.baseurl}})

从该现象中也告诉我们，即使多开 app 会在系统中单独安装分身 app，但他任然依赖于原 app，我们不能将原 app 卸载

#### 总结

只能通过修改条件一或者条件二绕过多开限制，而这两个信息都是从服务端传过来的，所以需要修改 dex 代码


## 脱壳

该壳的核心 so 代码是用 OLLVM 加固，分析起来比较麻烦，为了能够快速得到目的，就没有继续死磕。通过网上资料可知，该壳相对简单，脱壳可参考[这里](https://www.imwen.cn/2020/03/29/271.html)

* 把 Androidmanifest.xml 中的
name 改为加固 classes.dex 中以下路径 com.wrapper.proxyapplication.WrapperProxyApplication 中调用的实际 application 类

* 删除下面文件

```bash
删除 apk 中的下列文件

assets/0OO00l111l1l
assets/0OO00oo01l1l
assets/0OO00oo11l1l
assets/o0oooOO0ooOo.dat
assets/tosversion
lib/libshell-super.2019.so
lib/libshella-4.2.0.9.so
tencent_stub
```

* 然后将核心 dex 放入，重新打包签名即可

脱壳后并且将上面条件一设置大于0时，运行后出错，出错信息为：

![graph]({{"/assets/pictures/reverse/multi_open/so_error.png" | prepend:site.baseurl}})


## so 分析

通过字符串搜索，只有两处会有该提示：

```java
    public void onClickApp(AppEntity appEntity) {
        if (1 != appEntity.getIsSupport()) {
            T.a(this, "正在适配中，敬请期待...");
        } else if (!PackageNameGenarator.d(this, appEntity.getPackageName())) {
            T.a(this, "您的手机上还没有" + appEntity.getName() + ",请先安装官方最新版。");
        } else if (!com.XXX.XXX.a.a.a().A()) { // 第一处：服务端获取配置标识
            T.a(this, "初始化失败，请尝试退出后重新打开XXXX");
        } else if (StringUtils.isNotBlank(com.XXX.XXX.a.a.a().u())) { // 第二处：u() 返回空
            a(appEntity);
        } else {
            T.a(this, "初始化失败，请尝试退出后重新打开XXXX");
        }
    }
```

第一处是从服务端获取配置标识，若失败了才会打印日志退出。而在使用 charles 抓包可知，从服务端获取到的配置是成功的。所以失败原因是发生第二处，即是 com.XXX.XXX.a.a.a().u() 返回空导致

```java

    public String u() {
        if (!StringUtils.isNotBlank(this.r)) { // 服务端信息，不为空
            return "";
        }
        try {
            AbcUtil abcUtil = this.H;
            return AbcUtil.getStr4(Application.getInstance().getApplicationContext(), this.r);
        } catch (Exception e2) {
            e2.printStackTrace();
            return "";
        }
    }

```

u 调用空可能存在三个地方：
* this.r 为空，这是服务端返回的信息，排除该情况
* 调用 AbcUtil.getStr4 后返回为空
* 或者调用 AbcUtil.getStr4 抛出异常

```java

public class AbcUtil {
    static {
        System.loadLibrary("jiagu1");
    }

    public static native String getStr4(Object obj, String str);

    public static native boolean saveD(Context context, String str, DeviceEntity deviceEntity);
}

```

可以看到， abcUtil 中的两个方法实际上调用的是 libjiagu1.so 中的函数

使用 IDA 对该 so 打开，然后发现并没有找到 getStr4 函数，而是有一大段数据，在结合有使用 svc 调用的 mmap 等函数，猜测可能是在运行时进行动态解密


![graph]({{"/assets/pictures/reverse/multi_open/svc.png" | prepend:site.baseurl}})


### 获取动态解密后的 so

此时可以等管理 app 运行后，从内存中 dump 出来 so 文件，dump 使用的工具任是 elf-dump-fix


![graph]({{"/assets/pictures/reverse/multi_open/dump_so.png" | prepend:site.baseurl}})

和上面 dump dex 不同的是，最后一个参数设为 1， 要对 elf 进行修复

此时在通过 IDA 打开是可以找到 getStr4 函数

### getStr4 函数分析

![graph]({{"/assets/pictures/reverse/multi_open/so_code.png" | prepend:site.baseurl}})

getStr4 的函数逻辑为：

* 调用 sub_28F4 函数，若为 true，则直接返回 0
* 然后调用 java 侧的 StringUtils 类中的 str2 方法，然后将结果返回


查看 sub_28F4 函数如下：

![graph]({{"/assets/pictures/reverse/multi_open/so_sig.png" | prepend:site.baseurl}})

从使用的 API 可以明显看到，这是在获取 apk 签名，由于我们重新对 apk 签名过，所以在此时签名校验会失败，才导致脱壳后不可用


### 绕过签名校验

直接修改 so 简单，但是 so 会有动态解密过程，需要将修改转化成解密前的修改，这样修改比较麻烦，为了更快完成目标，并且对 so 的两个方法进行了分析，都是判断签名后直接调用 java 侧方法，所以我们可以将 so 去掉，直接在 java 侧修改调用对应的 java 方法

通过 frida 对 getStr4 函数的 hook 监控，发现 getStr4 返回固定的值，所以为了更简单的修改，直接将 java 侧 getStr4 返回该固定值，并且去掉 so 的加载：

```bash

# 修复前

.method static constructor <clinit>()V
    .locals 1

    const-string v0, "jiagu1"

    .line 1
    invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V

    return-void
.end method

.method public static native getStr4(Ljava/lang/Object;Ljava/lang/String;)Ljava/lang/String;
.end method

# 修改后

.method static constructor <clinit>()V
    .locals 1

    #const-string v0, "jiagu1"

    .line 1
    #invoke-static {v0}, Ljava/lang/System;->loadLibrary(Ljava/lang/String;)V

    return-void
.end method


.method public static getStr4(Ljava/lang/Object;Ljava/lang/String;)Ljava/lang/String;
    .registers 3

    const-string v0, "Ektech2017!@#" # 返回的固定值

    return-object v0

.end method

```
将 so 的加载注释掉，并且将 getStr4 直接返回固定值

修改完后重新打包签名安装，此时发现是可用的，如下图可以多开任意个，没有限制：

![graph]({{"/assets/pictures/reverse/multi_open/multi_open_success.png" | prepend:site.baseurl}})


## 总结

本篇主要是对管理 app 的逆向分析破解，主要涉及到：

* 核心 dex 的获取
* 定位关键代码
* 脱壳并修改关键代码
* 解密 so 的获取
* so 核心代码分析
* 绕过 so 的签名校验

本文中涉及到的一些技术点并没有仔细列举，大家可以根据自行去分析，如：
* 条件一和条件二是服务端的配置
* charles 使用，并且通过 charles 测试修改服务端配置
* frida 的使用

最后本篇只是分析了管理 app，此时点击分身 app 是会闪退，这是由于分身 app 还会有安全防护，该防护等下篇再分析

## 样本下载


[样本 apk](https://github.com/liwugang/liwugang.github.io/tree/master/assets/files/multi_open.apk)



