---
layout: article

title:  Android引入动态库so的方法
date:   2015-08-19 21:02:13 +0800
 
tags: Android
key: android_import_so
sticky: true
---

为了执行效率，会将一些CPU密集性任务如音视频解码、图像处理等放入到so中，还有也会将程序关键核心部分放入到so中，这样增加了逆向的难度。下面将so引入分成本地和第三方两种类型。本地是指包含有源码的。这是不讲如何编译和Android.mk的编写，主要讲在Android项目中如何引入so。

<!--more-->

## 本地so

引入本地的so分成三个步骤:

### ndk项目的编译

这部分网上资料会很多，主要注意事项是函数名格式，如一个类的全名是com.example.firstProgram.ClassName，定义的native函数名为function，则在so中的函数名必须为Java_com_example_firstProgram_ClassName_function，这样通过jni才能找到对应的函数，还有必须要按照C语言的编译和链接规范，，即在C++文件中要使用extern “C”

### 类中引入so

首先要引入对应的so文件，将引入的代码放入到类的static代码块，这样在加载类时会首先将so加载进来。代码：

```java
static{
        System.loadLibrary("XXX");//XXX为so文件名
    }
```

### 将编译后的so引入到项目中lib库下

即使将jni目录放入到项目下，但项目打包时并不会将so文件放入到lib目录下，要通过设置才可以。
找到app目录下的build.gradle，加入下面内容：

```bash
sourceSets {
    main {
        jniLibs.srcDirs = ['../libs']
    }
```
../libs’是表示so文件的目录，相对于build.gradle当前文件位置的。


经过上面三步，就可以将so包含到Android项目中了。

## 三方so

由于第三方so没有源代码，只提供接口，就需要首先自己创建本地so，通过本地C/C++代码来调用第三方so的函数。这里的主要配置是在jni目录下的Android.mk文件中。有两种方式：将三方so放入到系统目录下（如/system/lib/或者/system/lib64）和打包到apk中。

### 放入到系统目录

这种方式类似于引用系统提供的库，需要在Android.mk中提供

> LOCAL_LDLIBS := -lXXX

将so文件放入到/system/lib或者/system/lib64目录下，这种方式适合在root机器或模拟器上进行测试，不用将so打包到apk中。

### 打包到apk中

在jni新建目录如prebuilt，我们这里以libsubstrate.so为例，将libsubstrate.so文件放入到prebuilt目录，并新建Android.mk文件，内容为：

```bash
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := substrate
LOCAL_SRC_FILES := libsubstrate.so
include $(PREBUILT_SHARED_LIBRARY)
```

然后jni目录中的Android.mk中添加下面内容：

> LOCAL_SHARED_LIBRARIES := substrate #上述模块名

并在文件末尾添加：

> include $(LOCAL_PATH)/prebuilt/Android.mk

这样apk打包会将该so文件打包进去，但运行还是有问题，什么问题呢？。。。。。
对了，只是将so文件打包到apk中了，并没有加载进去，和之前一样，在类的static代码块中加入System.loadLibrary(“XXX”），这样在就可以正常运行使用了。

## 总结

由于刚加入到Android阵营中，这些东西都是很基础的，很多内容都是来源于网上，内容很分散，刚开始找的话很费时费力，因此将这些东西整理到起来。方便自己和他人的学习。