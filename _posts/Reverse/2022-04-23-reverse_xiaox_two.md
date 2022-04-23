---
layout: article

title:  某多开 app 破解 - 下篇
date:   2022-04-23 16:20:00 +0800

tags: Android reverse
key: reverse_multi_app_two
---

可以从上篇文章 [某多开 app 破解 - 上篇](../17/reverse_xiaox.html) 了解背景和对管理 app 的破解，本篇主要是对分身 app 的防护进行分析和破解

<!--more-->

## 分身 app 防护分析

### Java 侧防护

分身启动后直接发生崩溃，查看日志如下，关键信息用 XXX 代替了：

```bash
04-17 20:31:27.008  3520  3520 W System.err: java.security.NoSuchAlgorithmException: 发生错误
04-17 20:31:27.008  3520  3520 W System.err:    at com.XXX.XXX.plugin.hook.natives.NativeHook.test1(NativeHook.java:1)
04-17 20:31:27.008  3520  3520 W System.err:    at com.XXX.XXX.plugin.hook.natives.NativeHook.makeCrash(Native Method)
04-17 20:31:27.008  3520  3520 W System.err:    at com.XXX.XXX.plugin.hook.natives.NativeHook.goCrash(NativeHook.java:1)
04-17 20:31:27.008  3520  3520 W System.err:    at com.XXX.XXX.g.a(CopyRightUtils.java:16)
04-17 20:31:27.008  3520  3520 W System.err:    at com.XXX.XXXapp.activity.SplashActivity.d(SplashActivity.java:12)
04-17 20:31:27.008  3520  3520 W System.err:    at com.XXX.XXXapp.activity.SplashActivity.onRequestPermissionsResult(SplashActivity.java:2)
04-17 20:31:27.008  3520  3520 W System.err:    at android.app.Activity.dispatchRequestPermissionsResult(Activity.java:8460)
04-17 20:31:27.008  3520  3520 W System.err:    at android.app.Activity.dispatchActivityResult(Activity.java:8308)
04-17 20:31:27.008  3520  3520 W System.err:    at android.app.ActivityThread.deliverResults(ActivityThread.java:5006)
04-17 20:31:27.008  3520  3520 W System.err:    at android.app.ActivityThread.handleSendResult(ActivityThread.java:5054)
04-17 20:31:27.008  3520  3520 W System.err:    at android.app.servertransaction.ActivityResultItem.execute(ActivityResultItem.java:51)
04-17 20:31:27.008  3520  3520 W System.err:    at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:135)
04-17 20:31:27.008  3520  3520 W System.err:    at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:95)
04-17 20:31:27.008  3520  3520 W System.err:    at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2066)
04-17 20:31:27.009  3520  3520 W System.err:    at android.os.Handler.dispatchMessage(Handler.java:106)
04-17 20:31:27.009  3520  3520 W System.err:    at android.os.Looper.loop(Looper.java:223)
04-17 20:31:27.009  3520  3520 W System.err:    at android.app.ActivityThread.main(ActivityThread.java:7664)
04-17 20:31:27.009  3520  3520 W System.err:    at java.lang.reflect.Method.invoke(Native Method)
04-17 20:31:27.009  3520  3520 W System.err:    at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:592)
04-17 20:31:27.009  3520  3520 W System.err:    at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:947)
04-17 20:31:27.164  3589  3589 I Process : Sending signal. PID: 3520 SIG: 9
```

通过堆栈找到发生异常的地方

```java
public class C0125g { // 对应上面 com.XXX.XXX.g 类
    public static void a(Context context) {
        try {
            byte[] digest = MessageDigest.getInstance("SHA1").digest(context.getPackageManager().getPackageInfo(ChaosRuntime.PARENT_PACKAGE, 64).signatures[0].toByteArray()); // 此处是获取管理 app 的签名 SHA1
            StringBuilder sb = new StringBuilder();
            for (byte b2 : digest) {
                String upperCase = Integer.toHexString(b2 & 255).toUpperCase(Locale.US);
                if (upperCase.length() == 1) {
                    sb.append("0");
                }
                sb.append(upperCase);
            }
            if (!"F9074EBEBDDADEE08A78D36EFCCE9393802853C6".equals(sb.toString().toUpperCase())) { // 对签名和预制的值对比，若不同则退出
                Intent intent = new Intent("com.XXX.XXX.KILL_SELF");
                intent.setPackage(context.getPackageName());
                context.sendBroadcast(intent);
                NativeHook.goCrash(); // 主动调用 goCrash
                System.exit(0); // 主动退出
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

从上面代码可以看到，分身 app 会在启动时会对管理 app 进行签名校验，若校验失败，则会主动退出

### Java 防护绕过

绕过比较简单，可以将签名对比的 equals 方法返回值修改为 true，达到绕过该 java 侧防护

具体的操作：
* 将分身 app 的 apk 从手机上拉出来
* 通过 apktool 将其反编译
* 找到对应的 smali 文件，可以通过签名字符串快速定位
* 修改 equals 的返回值始终为 true，修改的逻辑如下
* 然后使用 apk 重新打包，签名后安装测试

```bash
    .line 12
    invoke-virtual {v2}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

    move-result-object v2

    invoke-virtual {v2}, Ljava/lang/String;->toUpperCase()Ljava/lang/String;

    move-result-object v2

    invoke-virtual {v0, v2}, Ljava/lang/String;->equals(Ljava/lang/Object;)Z

    move-result v0

    const/4 v0, 0x1 # 添加的逻辑，将结果始终返回 true

    if-nez v0, :cond_2
```

### Native 侧防护

Java 侧绕过检查之后，发现运行还是会崩溃，但此时是从 native 触发的，日志如下：

```bash

04-17 21:20:44.109  6062  6062 F DEBUG   : signal 6 (SIGABRT), code -1 (SI_QUEUE), fault addr --------                                                                                        
04-17 21:20:44.110  6062  6062 F DEBUG   : Abort message: 'No pending exception expected: java.security.NoSuchAlgorithmException: 发生错误                                                    
04-17 21:20:44.110  6062  6062 F DEBUG   :   at void com.XXX.XXX.plugin.hook.natives.NativeHook.test1() (NativeHook.java:1)                                                                 
04-17 21:20:44.110  6062  6062 F DEBUG   :   at void com.XXX.XXX.plugin.hook.natives.NativeHook.installRedirectHookNative(java.lang.Object, java.lang.String, int, int, boolean, boolean, bo
olean, boolean) (NativeHook.java:-2)                                                                                                                                                          
04-17 21:20:44.110  6062  6062 F DEBUG   :   at void com.XXX.XXX.plugin.hook.natives.NativeHook.installRedirectHook() (NativeHook.java:8)                                                   
04-17 21:20:44.110  6062  6062 F DEBUG   :   at void com.XXX.XXX.plugin.PluginImpl.initRedirectAndHook(android.content.Context, android.content.pm.ApplicationInfo) (PluginImpl.java:52)    
04-17 21:20:44.110  6062  6062 F DEBUG   :   at void com.XXX.XXX.plugin.PluginImpl.bindApplicationNoCheck(java.lang.String, java.lang.String, android.os.ConditionVariable) (PluginImpl.java
:107)                                                                                                                           
04-17 21:20:44.110  6062  6062 F DEBUG   :   at void com.XXX.XXX.plugin.PluginImpl.bindApplication(java.lang.String, java.lang.String) (PluginImpl.java:3)                                  
04-17 21:20:44.110  6062  6062 F DEBUG   :   at boolean com.XXX.XXX.plugin.hook.android.am.g.a(android.os.Message, java.lang.Object) (HCallbackStub.java:87)                                
04-17 21:20:44.110  6062  6062 F DEBUG   :   at boolean com.XXX.XXX.plugin.hook.android.am.g.a(android.os.Message) (HCallbackStub.java:25)                                                  
04-17 21:20:44.110  6062  6062 F DEBUG   :   at boolean com.XXX.XXX.plugin.hook.android.am.g.handleMessage(android.os.Message) (HCallbackStub.java:10)                                      
04-17 21:20:44.110  6062  6062 F DEBUG   :   at void android.os.Handler.dispatchMessage(android.os.Message) (Handler.java:102)                                                                
04-17 21:20:44.110  6062  6062 F DEBUG   :   at void android.os.Looper.loop() (Looper.java:223)                                                                                               
04-17 21:20:44.110  6062  6062 F DEBUG   :   at void android.app.ActivityThread.main(java.lang.String[]) (ActivityThread.java:7664)                                                           
04-17 21:20:44.110  6062  6062 F DEBUG   :   at java.lang.Object java.lang.reflect.Method.invoke(java.lang.Object, java.lang.Object[]) (Method.java:-2)                                       
04-17 21:20:44.110  6062  6062 F DEBUG   :   at void com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run() (RuntimeInit.java:592)                                                     
04-17 21:20:44.110  6062  6062 F DEBUG   :   at void com.android.internal.os.ZygoteInit.main(java.lang.String[]) (ZygoteInit.java:947)                                                        
04-17 21:20:44.110  6062  6062 F DEBUG   : '                                                                                                                                      
04-17 21:20:44.110  6062  6062 F DEBUG   :     x0  0000000000000000  x1  0000000000001792  x2  0000000000000006  x3  0000007fe6aa23a0                                                         
04-17 21:20:44.110  6062  6062 F DEBUG   :     x4  736d74715e62ff1f  x5  736d74715e62ff1f  x6  736d74715e62ff1f  x7  7f7f7f7f7f7f7f7f                                                         04-17 21:20:44.110  6062  6062 F DEBUG   :     x8  00000000000000f0  x9  1ff722a3cb703524  x10 0000000000000000  x11 ffffffc0fffffbdf                                                         
04-17 21:20:44.110  6062  6062 F DEBUG   :     x12 0000000000000001  x13 00000000000007ba  x14 000311f11fb00006  x15 00000000020aa860                                                         
04-17 21:20:44.110  6062  6062 F DEBUG   :     x16 0000007612341c80  x17 0000007612323c10  x18 0000007616c78000  x19 0000000000001792                                                         
04-17 21:20:44.110  6062  6062 F DEBUG   :     x20 0000000000001792  x21 00000000ffffffff  x22 0000000000000000  x23 0000000000000000                                                         
04-17 21:20:44.110  6062  6062 F DEBUG   :     x24 0000007380d6e000  x25 0000000000000043  x26 000000761607786c  x27 0000007380d70000                                                         
04-17 21:20:44.110  6062  6062 F DEBUG   :     x28 0000007380d71000  x29 0000007fe6aa2420                                                                                                     
04-17 21:20:44.110  6062  6062 F DEBUG   :     lr  00000076122d62a0  sp  0000007fe6aa2380  pc  00000076122d62cc  pst 0000000000000000   

... ...
04-17 21:20:44.353  6062  6062 F DEBUG   :       #07 pc 000000000001202c  /data/app/~~NA6OJC104Awax8GaOnClzw==/dkplugin.niu.xje-z4bE-dmP2knenBq5H0oZNA==/lib/arm64/libXXXX.so (BuildId: 98ee9

```

首先定位到发生崩溃的 NativeHook.test1 方法处：

```java
    public static void test1() {
        throw new NoSuchAlgorithmException("发生错误");
    }
```

可以看到，调用该方法是主动触发异常的情形

调用者是 NativeHook.installRedirectHookNative 方法，查看该方法：

```java
private static native void installRedirectHookNative(Object obj, String str, int i, int i2, boolean z, boolean z2, boolean z3, boolean z4);
```
该方法是 native 方法，所以主动调用异常是在 so 里面

通过堆栈找到对应的 so，然后使用 IDA 打开，该 so 没有防护，可以直接使用 F5 转成伪C代码来看，若直接看 installRedirectHookNative 对应的 native，会比较复杂， 可以从 test1 方法入手，回溯方式定位到关键代码如下：

```c++
__int64 __fastcall sub_A294(__int64 a1, __int64 a2) {
 ... ...
  v4 = (*(__int64 (**)(void))(*(_QWORD *)a1 + 248LL))();
  v5 = (*(__int64 (__fastcall **)(__int64, __int64, const char *, const char *))(*(_QWORD *)v2 + 264LL))(
         v2,
         v4,
         "getPackageManager",
         "()Landroid/content/pm/PackageManager;");
  v6 = sub_9F0C(v2, v3, v5);
  v7 = v6;
  v8 = (*(__int64 (__fastcall **)(__int64, __int64))(*(_QWORD *)v2 + 248LL))(v2, v6);
  v9 = (*(__int64 (__fastcall **)(__int64, __int64, const char *, const char *))(*(_QWORD *)v2 + 264LL))(
         v2,
         v8,
         "getPackageInfo",
         "(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;");
  v10 = (*(__int64 (__fastcall **)(__int64, const char *))(*(_QWORD *)v2 + 48LL))(v2, "java/lang/String");
  v11 = (*(__int64 (__fastcall **)(__int64, __int64, const char *, const char *))(*(_QWORD *)v2 + 264LL))(
          v2,
          v10,
          "<init>",
          "([BLjava/lang/String;)V");
  v12 = (*(__int64 (__fastcall **)(__int64, signed __int64))(*(_QWORD *)v2 + 1408LL))(v2, 14LL);
  (*(void (__fastcall **)(__int64, __int64, _QWORD, signed __int64, const char *))(*(_QWORD *)v2 + 1664LL))(
    v2,
    v12,
    0LL,
    14LL,
    "com.XXX.XXXX");
  v13 = (*(__int64 (__fastcall **)(__int64, const char *))(*(_QWORD *)v2 + 1336LL))(v2, "GB2312");
  sub_A0DC(v2, v10, v11, v12, v13);
  (*(void (__fastcall **)(__int64, __int64))(*(_QWORD *)v2 + 184LL))(v2, v10);
  (*(void (__fastcall **)(__int64, __int64))(*(_QWORD *)v2 + 184LL))(v2, v13);
  (*(void (__fastcall **)(__int64, __int64))(*(_QWORD *)v2 + 184LL))(v2, v12);
  v14 = sub_9F0C(v2, v7, v9);
  v15 = v14;
  v16 = (*(__int64 (__fastcall **)(__int64, __int64))(*(_QWORD *)v2 + 248LL))(v2, v14);
  v17 = (*(__int64 (__fastcall **)(__int64, __int64, const char *, const char *))(*(_QWORD *)v2 + 752LL))(
          v2,
          v16,
          "signatures",
          "[Landroid/content/pm/Signature;");
  v18 = (*(__int64 (__fastcall **)(__int64, __int64, __int64))(*(_QWORD *)v2 + 760LL))(v2, v15, v17);
  v19 = v18;
  v20 = (*(__int64 (__fastcall **)(__int64, __int64, _QWORD))(*(_QWORD *)v2 + 1384LL))(v2, v18, 0LL);
  v21 = v20;
  v22 = (*(__int64 (__fastcall **)(__int64, __int64))(*(_QWORD *)v2 + 248LL))(v2, v20);
  v23 = v22;
  v24 = (*(__int64 (__fastcall **)(__int64, __int64, const char *, const char *))(*(_QWORD *)v2 + 264LL))(
          v2,
          v22,
          "toCharsString",
          "()Ljava/lang/String;");
  v25 = sub_9F0C(v2, v21, v24);
  v26 = sub_9DB4(v2, v25);
  (*(void (__fastcall **)(__int64, __int64))(*(_QWORD *)v2 + 184LL))(v2, v25);
  (*(void (__fastcall **)(__int64, __int64))(*(_QWORD *)v2 + 184LL))(v2, v21);
  (*(void (__fastcall **)(__int64, __int64))(*(_QWORD *)v2 + 184LL))(v2, v19);
  (*(void (__fastcall **)(__int64, __int64))(*(_QWORD *)v2 + 184LL))(v2, v15);
  (*(void (__fastcall **)(__int64, __int64))(*(_QWORD *)v2 + 184LL))(v2, v23);
  result = strcmp(
             "308203473082022fa003020102020435ee7721300d06092a864886f70d01010b05003054310b300906035504061302636e310b30090"
             "60355040813026764310b300906035504071302737a310c300a060355040a1303626c79310c300a060355040b1303626c79310f300d"
             "XXX XXX",
             v26);
  if ( (_DWORD)result )
  {
    v28 = (*(__int64 (__fastcall **)(__int64, const char *))(*(_QWORD *)v2 + 48LL))(
            v2,
            "com/XXX/XXXX/plugin/hook/natives/NativeHook");
    v29 = v28;
    v30 = (*(__int64 (__fastcall **)(__int64, __int64, const char *, const char *))(*(_QWORD *)v2 + 904LL))(
            v2,
            v28,
            "test1",
            "()V");
    sub_A1F4(v2, v29, v30);
    result = (*(__int64 (__fastcall **)(__int64, __int64))(*(_QWORD *)v2 + 184LL))(v2, v29);
  }
  return result;
}
```

可以看到，该函数的逻辑是：
* 获取管理 app 的签名，然后和预制的证书进行对比
* 若对比失败，则直接调用 test1 进行崩溃退出

所以，native 的防护仍然是对管理 app 的签名校验

### Native 防护绕过

先找到上面的汇编代码处，如下：

```asm
text:000000000000A578 C0 01 00 90                    ADRP            X0, #a30820347308202@PAGE ; "308203473082022fa003020102020435ee77213"...
.text:000000000000A57C 00 B8 38 91                    ADD             X0, X0, #a30820347308202@PAGEOFF ; "308203473082022fa003020102020435ee77213"...
.text:000000000000A580 E1 03 19 AA                    MOV             X1, X25
.text:000000000000A584 2F FD FF 97                    BL              .strcmp
.text:000000000000A588 E0 03 00 34                    CBZ             W0, loc_A604
.text:000000000000A58C 68 02 40 F9                    LDR             X8, [X19]
.text:000000000000A590 C1 01 00 90                    ADRP            X1, #aComXXXXXXPlu@PAGE ; "com/XXX/XXXX/plugin/hook/natives/Nativ"...
.text:000000000000A594 21 74 34 91                    ADD             X1, X1, #aComXXXXXXPlu@PAGEOFF ; "com/XXX/XXX/plugin/hook/natives/Nativ"...
.text:000000000000A598 E0 03 13 AA                    MOV             X0, X19
.text:000000000000A59C 08 19 40 F9                    LDR             X8, [X8,#0x30]
.text:000000000000A5A0 00 01 3F D6                    BLR             X8
.text:000000000000A5A4 68 02 40 F9                    LDR             X8, [X19]
.text:000000000000A5A8 F4 03 00 AA                    MOV             X20, X0
.text:000000000000A5AC C2 01 00 90                    ADRP            X2, #aTest1@PAGE ; "test1"
.text:000000000000A5B0 C3 01 00 90                    ADRP            X3, #aV@PAGE ; "()V"
.text:000000000000A5B4 08 C5 41 F9                    LDR             X8, [X8,#0x388]
.text:000000000000A5B8 42 28 35 91                    ADD             X2, X2, #aTest1@PAGEOFF ; "test1"
.text:000000000000A5BC 63 40 35 91                    ADD             X3, X3, #aV@PAGEOFF ; "()V"
```

strcmp 是计算的签名和预制证书进行对比， 此时我们需要将 CBZ 修改为相反指令 CBNZ，由于 IDA 不能直接编辑 arm 的指令，所以我们使用 rizin 工具集中的 rz-asm 工具，获取指令的机器码

```bash
rz-asm -a arm -b 64 -d  E0030034 # 获取上面 "CBZ  W0, loc_A604" 具体有偏移的表示，通过机器码
cbz w0, 0x7c
rz-asm -a arm -b 64 "cbnz w0, 0x7c" # 获取 "CBNZ W0, 0x7C" 具体的机器码
e0030035
```

所以应该是将机器码从 E0030034 修改为 E0030035，通过 IDA --> Edit --> Patch program --> Change byte... 进行修改操作，修改完后，可以看到 IDA 的指令也同步进行了修改：

```bash
.text:000000000000A588 E0 03 00 35                    CBNZ            W0, loc_A604
```

修改完后，将改动保存到 so 中，IDA --> Edit --> Patch program --> Apply patches to input file

重新对 apk 打包签名安装后，此时 apk 是可以正常使用了


## 分身 app 修改

我们对分身 app 中进行了修改，验证了修改的可行性。但分身 app 是管理 app 自动生成的，我们需要找到生成的逻辑，然后进行统一的修改。而不是对每个分身 app 进行修改


### 定位关键代码

下面将列出来调用链：

```bash

a(AppEntity appEntity)        // 是点击 “制作分身” 的逻辑
CreateCustomActivity.onCreate // 调用“制作分身”后的界面
CreateCustomActivity.d        // 是点击“开始制作”的调用
CreatingNewActivity.onCreate  // 调用CreatingNewActivity
--> CreatingNewActivity.c           // 是创建的动画效果
--> CreatingNewActivity.d           // 对传入的参数进行校验 和 权限校验
--> --> CreatingNewActivity.b          // 请求服务端，将原 app 信息传入
--> --> C0123v.onResponse              // 对服务端返回值进行处理，有些字段返回会导致中断分身 app 的生成
--> --> CreatingNewActivity.a          // 调用回 CreatingNewActivity 类
--> --> H(this, str, str2)).start()    // 创建单独线程进行分身 app 的生成 
```

```java

public class H implements Runnable {
    public void run() {
        ... ...
        Context applicationContext = this.f673c.getApplicationContext();
        str = this.f673c.f661c;
        if (AppInfoUtils.b(applicationContext, str)) { // 此处是原 app 是判断是32位还是64位
            i5 = this.f673c.k;
            if (i5 == a.n) {
                inputStream = this.f673c.getAssets().open("res"); // 32位打开 res
            } else {
                i7 = this.f673c.k;
                inputStream = s.a(i7, false);
            }
            StringBuilder sb = new StringBuilder();
            sb.append("使用32位的分身 ");
            str15 = this.f673c.f661c;
            sb.append(str15);
            sb.append(",");
            sb.append(this.f671a);
            sb.append(",");
            i6 = this.f673c.k;
            sb.append(i6);
        } else {
            i2 = this.f673c.k;
            if (i2 == a.n) {
                inputStream = this.f673c.getAssets().open("res64"); // 64位打开 res64
            } else {
                i4 = this.f673c.k;
                inputStream = s.a(i4, true);
            }
            StringBuilder sb2 = new StringBuilder();
            sb2.append("使用64位的分身 ");
            str14 = this.f673c.f661c;
            sb2.append(str14);
            sb2.append(",");
            sb2.append(this.f671a);
            sb2.append(",");
            i3 = this.f673c.k;
            sb2.append(i3);
        }
        ... ...
        ea.a(inputStream, inputStream.available(), this.f673c.handler, com.XXX.XXXX.a.a.a().u(), file2.getAbsolutePath());
    }
}

```

{:.warning}
jadx-gui 反编译 H 类时默认会有问题，不能正确显示 java 代码，可以将 File --> Preferences --> Show inconsistent code 选项打开

可以看到会根据原 app 是32位还是64位选择打开 res 或者 res64，该文件保存的肯定是分身 app 所需要的文件，对该文件使用 file 查看，是 zip 压缩文件，并且是有密码保护的

经过对 ea.a 方法深入分析，密码是 com.XXX.XXXX.a.a.a().u() 传入的，而该方法调用的是 AbcUtil.getStr4，即在管理 app 中有提到过，是 native 方法，并且我们也获取到该返回值为固定值，尝试用之前的固定值"Ektech2017!@#"测试解压,发现解压成功了

解压出来是 resource 文件，格式仍然是 zip 压缩文件，继续解压，此时没有密码，解压后发现是分身 app 代码和资源

![graph]({{"/assets/pictures/reverse/multi_open/child_app.png" | prepend:site.baseurl}})

所以，我们将前面对 dex 和 so 的修改同步到这里，然后重新压缩生成 res 和 res64，替换管理 app assets 目录下的这两个文件即可绕过防护

## 分身对多开的校验

此时将上述修改重新打包安装后，分身 app 可以正常启动了，说明我们修改没有问题了。但是又发现一个问题，生成多个分身后，后面的分身是无法启动的，显示“会员已过期，分身无法启动”

可以通过文案的显示，定位到关键代码

### 定位关键代码

可以通过文案的显示，定位到关键代码

```java
                if (!ChaosRuntime.isVipSeted) {
                    requestMemberInfo();  // 这里是从管理 app 请求用户的信息
                }
                if (!ChaosRuntime.isVip && next.d() == 1) { // 如果不是 vip，并且启动的不是第一个分身，就走该逻辑
                    ColorMatrix colorMatrix = new ColorMatrix();
                    colorMatrix.setSaturation(0.0f);
                    imageView.setColorFilter(new ColorMatrixColorFilter(colorMatrix));
                    textView2.setTextColor(Color.parseColor("#AAAAAA"));
                    inflate3.setOnClickListener(new View.OnClickListener() {
                        @Override // android.view.View.OnClickListener
                        public void onClick(View view) {
                            StubBridgePrepareActivity.this.onSelectIsMutiNoVip(next); // 此处为文案显示
                        }
                    });
```

现在管理 app 和分身 app 都没有了防护，可以随便修改了。一种方案是在分身代码处修改此处逻辑，绕过多开的校验，另外的方案就是在管理 app 中修改用户的权限信息，可以通过 requestMemberInfo 深入看下是怎么获取用户信息的

```java
    private void requestMemberInfo() {
        byte[] byteArray;
        try {
            Bundle call = getContentResolver().call(Uri.parse("content://com.XXX.XXXX.PluginProvider"), "getMemberInfo", (String) null, (Bundle) null);
            if (call != null && (byteArray = call.getByteArray("data")) != null) {
                Parcel obtain = Parcel.obtain();
                obtain.unmarshall(byteArray, 0, byteArray.length);
                obtain.setDataPosition(0);
                MemberInfo memberInfo = new MemberInfo(obtain);
                obtain.recycle();
                if (memberInfo != null) {
                    boolean z = true;
                    ChaosRuntime.isVipSeted = true;
                    if (memberInfo.getVipType() != 1) {
                        z = false;
                    }
                    ChaosRuntime.isVip = z;
                }
            }


    public MemberInfo(Parcel parcel) {
        this.imei = parcel.readString(); // imei
        this.vipType = parcel.readInt(); // vip 类型
        this.expiredTime = parcel.readLong(); // vip 过期时间
        this.serverCheckTime = parcel.readLong(); // 服务端检查时间
        this.newVersion = parcel.readInt(); // 新版本号
        this.payMode = parcel.readInt();  // 付款方式
        this.showAd = parcel.readInt();   // 是否显示广告
    }
```

可以看到是通过管理 app 提供的 provider 获取用户信息，并且用户信息包含有：
* imei
* vip 类型
* vip 过期时间
* 服务端检查时间
* 新版本号
* 付款方式
* 是否显示广告


### 修改方式

我是在管理 app 中修改的，通过 PluginProvider 中定位到用户信息的来源，主要是从两个服务端接口获取，我直接修改的是接口返回值，这样在任意验证 vip 地方都可以通过

当然最简单方式就是在上面分身 app 条件判断处进行修改

## 总结

![graph]({{"/assets/pictures/reverse/multi_open/architure.png" | prepend:site.baseurl}})

* 防护手段，主要是对管理 app 的签名校验，自身 so 和分身 app 都会进行校验
* 分身 app 的生成是是通过 res/res64 在结合原 app
* 用户信息的获取，是管理 app 通过 provider 供分身 app 获取

整体破解下来，技术难度不高，但是比较复杂，涉及到的比较多，并且从中也能了解到了多开的大概原理

## 样本下载

[样本 apk](https://github.com/liwugang/liwugang.github.io/tree/master/assets/files/multi_open.apk)



