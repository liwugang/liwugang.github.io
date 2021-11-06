---
layout: article

title:  使用frida hook插件化apk
date:   2020-04-12 10:08:00 +0800
 
tags: reverse
key: frida_hook_shadow
---

最近拿到一个XX视频apk样本，里面有视频、直播和小说，没有VIP只能试看30秒，刚好最近学习frida，用来练习下，分析过程中发现是一个插件化的apk，本文记录下分析的过程。

<!--more-->


## 初步分析

首先从AndroidManifest.xml中获取到apk的包名，并且查看下activity情况：

![graph]({{"/assets/pictures/frida_hook_shadow/activity.png" | prepend:site.baseurl}})

发现只有4个Activity，正常情况下一个apk肯定不止这些，所以初步怀疑这只是外壳，真正逻辑是在其他地方，会动态加载进来。

### ps 查看
打开apk
![graph]({{"/assets/pictures/frida_hook_shadow/ps.png" | prepend:site.baseurl}})

可以看到有两个进程，从上图也可以看到，2、3和4处的Activity是运行在plugin进程中，为了确认下视频播放所在的进程，使用dumpsys meminfo查看

### dumpsys meminfo查看
打开任意播放界面

![graph]({{"/assets/pictures/frida_hook_shadow/dumpsys_memory.png" | prepend:site.baseurl}})

确认视频播放是在plugin进程中，此时真正的逻辑已经加载到进程中，查看下plugin进程的maps

### cat /proc/7906/maps

![graph]({{"/assets/pictures/frida_hook_shadow/apk.png" | prepend:site.baseurl}})

从上图可以看到，真正逻辑所在的apk是plugin-shadow-apk-debug.apk，是在该apk的私有文件目录中。

从代码中分析也可知道，此apk是插件化apk，使用的是腾讯开源的插件化框架[Shadow](https://github.com/Tencent/Shadow)，感兴趣的可以去了解下。

## 定位关键代码

既然已经找到真正的apk，那我们就需要定位到关键代码地方。

![graph]({{"/assets/pictures/frida_hook_shadow/screen.png" | prepend:site.baseurl}})

### 字符串定位

![graph]({{"/assets/pictures/frida_hook_shadow/string.png" | prepend:site.baseurl}})

从字符串中定位到有多个类满足，此时一个一个去分析排查太耗时，接下来通过frida来枚举出所有加载的类。

### frida枚举所有加载的类

Java.enumerateLoadedClasses(callbacks) 是用来枚举当前所有加载的类，通过和上述几个关键类对比来找到实际调用的类，callbacks需要提供回调函数，对应onMatch和onComplete。具体如下面：

```javascript

Java.perform(function () {

	// 上述搜索到的多个类
    var key_class = ["com.facebook.plugin.widget.dkplayer.controller.PlayerVideoController",
                     "com.iqiyi.plugin.widget.dkplayer.controller.PlayerVideoController",
                     "com.facebook.plugin.widget.dkplayer.controller.VideoController",
                     "com.iqiyi.plugin.widget.dkplayer.controller.VideoController"]

    Java.enumerateLoadedClasses({
        "onMatch": function(name, handle) {
            for (var i = 0; i < key_class.length; i++) {
                if (key_class[i] == name) {
                    console.log(name);
                }
            }
        },
        "onComplete": function() {
            console.log("success");
        }
    });
});

```

运行结果：

> com.iqiyi.plugin.widget.dkplayer.controller.VideoController

> success

第一行为输出结果，即表示当前使用的类为 com.iqiyi.plugin.widget.dkplayer.controller.VideoController；

第二行为执行完成的日志。

### VideoController 类分析

找到字符串位置

```java
  public int setProgress() {
	  	... ...
        if (this.tryWatchTv != null && position > 0) { // 如果是试看
            pos = (int) (((long) this.stopPlayTime) - position);
            TextView textView = this.tryWatchTv;
            StringBuilder stringBuilder = new StringBuilder();
            stringBuilder.append("剩余试看时间: "); // 此处是我们看到的字符串
            if (pos > 0) {
                j = (long) pos;
            }
            stringBuilder.append(stringForTime(j));
            textView.setText(stringBuilder.toString());
        }
        if (!this.isVip) { // 此处是通过isVip变量执行不同逻辑
            StringBuilder stringBuilder2 = new StringBuilder();
            stringBuilder2.append("position = ");
            stringBuilder2.append(position);
            stringBuilder2.append(" showVipHintTime = ");
            stringBuilder2.append(this.showVipHintTime);
            LogHelper.i(stringBuilder2.toString());
            if (position < ((long) this.showVipHintTime) || this.showVipHintTime <= 0) {
                this.vipHintView.setVisibility(8);
            } else {
                this.vipHintView.setVisibility(0);
            }
            if (position >= ((long) this.stopPlayTime)) {
                this.mMediaPlayer.pause();
            }
        }
		... ...
    }

```
可以看到类中通过isVip变量来执行不同逻辑，继续看下isVip是如何设置的

```java
    public void setVip(boolean isVip) {
        this.isVip = isVip;
        this.tryWatchTv.setVisibility(this.isVip ? 8 : 0);
        if (this.isVip) {
            this.vipHintView.setVisibility(8);
        }
    }
```

可以看到当前类有setVip方法，用于设置该变量，此时可以不用在继续分析调用者，最终都会调用此处，所以我们可以使用frida hook该方法。

## frida hook setVip

```javascript
    var videoController = Java.use("com.iqiyi.plugin.widget.dkplayer.controller.VideoController");
    videoController.setVip.implementation = function() {
        console.log("hook setVip");
        this.setVip(true);
    };
```

运行结果：

```javascript
{'type': 'error', 'description': 'Error: java.lang.ClassNotFoundException: Didn\'t find class "com.iqiyi.plugin.widget.dkplayer.controller.VideoController" on path: DexPathList[[dex file "InMemoryDexFile[cookie=[0, 3983850208]]", zip file "/system/framework/org.apache.http.legacy.boot.jar", zip file "/data/app/com.iqiyi.vider-U7T4aZIQOQyXS5iLBgsNGw==/base.apk"],nativeLibraryDirectories=[/data/app/com.iqiyi.vider-U7T4aZIQOQyXS5iLBgsNGw==/lib/arm, /data/app/com.iqiyi.vider-U7T4aZIQOQyXS5iLBgsNGw==/base.apk!/lib/armeabi-v7a, /system/lib]]', 'stack': 'Error: java.lang.ClassNotFoundException: Didn\'t find class "com.iqiyi.plugin.widget.dkplayer.controller.VideoController" on path: DexPathList[[dex file "InMemoryDexFile[cookie=[0, 3983850208]]", zip file "/system/framework/org.apache.http.legacy.boot.jar", zip file "/data/app/com.iqiyi.vider-U7T4aZIQOQyXS5iLBgsNGw==/base.apk"],nativeLibraryDirectories=[/data/app/com.iqiyi.vider-U7T4aZIQOQyXS5iLBgsNGw==/lib/arm, /data/app/com.iqiyi.vider-U7T4aZIQOQyXS5iLBgsNGw==/base.apk!/lib/armeabi-v7a, /system/lib]]\n    at frida/node_modules/frida-java-bridge/lib/env.js:124\n    at frida/node_modules/frida-java-bridge/lib/class-factory.js:400\n    at frida/node_modules/frida-java-bridge/lib/class-factory.js:781\n    at frida/node_modules/frida-java-bridge/lib/class-factory.js:90\n    at frida/node_modules/frida-java-bridge/lib/class-factory.js:44\n    at /script1.js:23\n    at frida/node_modules/frida-java-bridge/lib/vm.js:11\n    at frida/node_modules/frida-java-bridge/index.js:368\n    at frida/node_modules/frida-java-bridge/index.js:318', 'fileName': 'frida/node_modules/frida-java-bridge/lib/env.js', 'lineNumber': 124, 'columnNumber': 1}
```

从运行结果来看，出现ClassNotFoundException错误，说明没有找到我们要hook的类。


### frida枚举classloader

由于是插件化apk，类加载是在插件化框架自定义的，所以classloader不能使用默认的。我们可以使用Java.enumerateClassLoaders(callbacks)来打印出所有的加载器。


```javascript
Java.perform(function () {
	    Java.enumerateClassLoaders({
        "onMatch": function(loader) {
            console.log(loader);
        },
        "onComplete": function() {
            console.log("success");
        }
    });
});
```
运行结果：

![graph]({{"/assets/pictures/frida_hook_shadow/classloader.png" | prepend:site.baseurl}})

由上面分析可知，真正逻辑代码是在plugin-shadow-apk-debug.apk中，那该apk对应的classloader是com.tencent.shadow.core.loader.classloaders.PluginClassLoader。

### frida指定classloader

来看下Java.ClassFactory中loader的介绍："read-only property providing a wrapper for the class loader currently being used."，loader是当前classloader的wrapper，我们修改classloader可以通过修改该字段。Java.classFactory是默认的class factory，所以我们需要修改的是Java.classFactory.loader。

```javascript
Java.perform(function () {
    Java.enumerateClassLoaders({
        "onMatch": function(loader) {
            if (loader.toString().startsWith("com.tencent.shadow.core.loader.classloaders.PluginClassLoader")) {
                Java.classFactory.loader = loader; // 将当前class factory中的loader指定为我们需要的
            }
        },
        "onComplete": function() {
            console.log("success");
        }
    });
});
```
### 最终脚本

```javascript
Java.perform(function () {
    Java.enumerateClassLoaders({
        "onMatch": function(loader) {
            if (loader.toString().startsWith("com.tencent.shadow.core.loader.classloaders.PluginClassLoader")) {
                Java.classFactory.loader = loader; // 将当前class factory中的loader指定为我们需要的
            }
        },
        "onComplete": function() {
            console.log("success");
        }
    });

	// 此处需要使用Java.classFactory.use
    var videoController = Java.classFactory.use("com.iqiyi.plugin.widget.dkplayer.controller.VideoController");
    videoController.setVip.implementation = function() {
        console.log("hook setVip");
        this.setVip(true);
    };
});
```

运行结果：

![graph]({{"/assets/pictures/frida_hook_shadow/setvip.png" | prepend:site.baseurl}})

可以看到，我们已经成功hook，并且视频上已经没有显示剩余时间。

## frida hook enum

直播和小说的vip判断和视频是不一致的，是通过enum中VIP字段值和1进行对比来判断，具体定位过程和上面类似。

![graph]({{"/assets/pictures/frida_hook_shadow/enum.png" | prepend:site.baseurl}})

判断代码为：
> if (TextUtils.equals("1", PluginEnum.VIP.getValue())) {...}

### enum测试

我们的目的是为了hook VIP，但是对enum的这种用法不是很熟，于是写了个测试程序，来进一步了解

```java
public enum TestEnum {
    A("a"),
    B("b"),
    C("c");

    private String value;
   
    private TestEnum(String value) {
        this.value = value;
    }

    public String getValue() {
        return this.value;
    }

}
```
使用javap打开对应的class文件：

```java
Compiled from "TestEnum.java"
public final class TestEnum extends java.lang.Enum<TestEnum> {
  public static final TestEnum A;
  public static final TestEnum B;
  public static final TestEnum C;
  public static TestEnum[] values();
  public static TestEnum valueOf(java.lang.String);
  public java.lang.String getValue();
  static {};
}
```

从这里可以很明显看到， A、B和C都属于TestEnum中的静态成员变量。来看下调用的smali代码：

```java
    sget-object v3, Lcom/iqiyi/plugin/base/PluginEnum;->VIP:Lcom/iqiyi/plugin/base/PluginEnum;
    invoke-virtual {v3}, Lcom/iqiyi/plugin/base/PluginEnum;->getValue()Ljava/lang/String;
```

从smali上也能看出来类似的逻辑，VIP是com/iqiyi/plugin/base/PluginEnum的静态成员，然后在调用getValue()方法。所以我们hook com/iqiyi/plugin/base/PluginEnum类的getValue方法，然后判断调用者是否为VIP。

### 最终脚本

```javascript
Java.perform(function () {
	var pluginEnum = Java.classFactory.use("com.iqiyi.plugin.base.PluginEnum");
    var String = Java.use("java.lang.String");
    pluginEnum.getValue.implementation = function() {
        var value = this.getValue();
        if (this == "VIP") { // 此时this 或者 this.getString() 返回的是静态成员名
            var vip = String.$new("1");
            this.setValue(vip); // 调用 setValue 修改VIP值
            return vip;
        } else {
            return value;
        }
    }
});
```
## 整体脚本

```javascript
Java.perform(function () {
    Java.enumerateClassLoaders({
        "onMatch": function(loader) {
            if (loader.toString().startsWith("com.tencent.shadow.core.loader.classloaders.PluginClassLoader")) {
                Java.classFactory.loader = loader;
            }
        },
        "onComplete": function() {
            console.log("success");
        }
    });

    var videoController = Java.classFactory.use("com.iqiyi.plugin.widget.dkplayer.controller.VideoController");
    videoController.setVip.implementation = function() {
        console.log("hook setVip");
        this.setVip(true);
    };

	var pluginEnum = Java.classFactory.use("com.iqiyi.plugin.base.PluginEnum");
    var String = Java.use("java.lang.String");
    pluginEnum.getValue.implementation = function() {
        var value = this.getValue();
        if (this == "VIP") {
            var vip = String.$new("1");
            this.setValue(vip);
            return vip;
        } else {
            return value;
        }
    }

});
```

## 总结

通过对该样本的分析，逆向找寻关键代码相对简单，但是在使用frida hook时相对难点，特别是对于frida和插件化不熟的情况下。本文涉及到的有：

1. frida枚举所有加载的类；
2. frida枚举classloader；
3. frida对enum的hook。
