---
layout: article

title:  如何下载64位的apk？
date:   2021-11-06 15:39:00 +0800
tags: Android
key: download_64_apk
---

64位的apk指的是so库是64位的，在目前的64位架构机型上运行会更有效率，而现在新机型大多数是64位。但目前各大厂商官网提供的都是32位的apk，这样的apk能跑在所有的机型上，以牺牲了部分性能代价。其实也可以在apk里包含有32位和64位的so，但这样apk包会大不小，大厂基本上都不会这样选择。 

<!--more-->

## 64位apk的来源

大厂都会将自家apk上传到各大应用商店里，会分成32位和64位的apk，应用商店在用户下载时，会根据机型选择是使用那种架构的apk。即在64位机型上选择64位的apk。

## 获取64位apk

### 使用64位机型下载

等apk安装完后，使用下面命令获取apk的路径，然后将64位的apk拿出来。

```bash
adb shell pm path [包名] # 获取安装包的路径
```

### 获取下载链接

只是适用于小米机型，其他机型未测试过。

#### 应用商店中关闭“删除安装包”

关闭该选项后，会保留下载记录，这样能拿到下载链接。

![graph]({{"/assets/pictures/android/market_config.png" | prepend:site.baseurl}})

如图所示，在小米应用商店的设置，关闭“删除安装包”选项。

{:.warning}
不关闭选项也可以，只不会需要在下载时进行后面的操作

#### 查看下载记录

应用商店会调用系统的下载管理，使用下面的命令可查看下载记录：

```bash
adb shell dumpsys activity provider com.android.providers.downloads/.provider.DownloadProvider
``` 

这样会dump出所有下载记录，会查看到我们下载的apk记录

![graph]({{"/assets/pictures/android/downloadprovider.png" | prepend:site.baseurl}})

如上图，我们下载了抖音极速版，1处为下载链接，拿到这个链接我们可以在电脑或其他地方下载，2处为本地保存的文件，我们也可以使用该文件。

## 总结

使用小米手机获取64位apk的三种方式：
1. Android通用方式，使用 pm path [包名] 获取apk的地址
2. 查看apk的下载记录，获取下载链接
3. 查看apk的下载记录，获取本地保存的apk