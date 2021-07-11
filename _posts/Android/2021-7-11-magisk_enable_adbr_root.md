---
layout: article

title:  Magisk 环境下增加 adb root 功能
date:   2021-07-11 22:11:00 +0800
tags: Android magisk
key: magisk_enable_adb_root
---

目前市面上的 Android 手机，要想获取 root 权限，主要有两种方式：
1. 通过官方提供的途径，如小米的开发版，会提供官方 root 能力，但是相对比较麻烦。首先需要解锁，然后需要申请开发者权限，最后在手机上进行 root 升级；
2. 通过解锁手机的 BL(bootloader) 锁，然后使用 magisk 功能获取。

当然除了上面两种方式外， 还可以通过漏洞等其他方式获取 root。

magisk 获取 root 权限是通过放置的 su，adb root 功能并不可用，本文主要是讲在 magisk 环境下增加 adb root 功能。

<!--more-->

## adb root 流程

adb 的架构如下图（摘自于 adb 源码），我们主要关注设备端（即 adbd）的处理逻辑，adb root 即是重启 adbd，让 adbd 以 root 用户运行，这样我们 adb 传入的命令就具有 root 权限。

```
+----------+              +------------------------+
|   ADB    +----------+   |      ADB SERVER        |                   +----------+
|  CLIENT  |          |   |                        |              (USB)|   ADBD   |
+----------+          |   |                     Transport+-------------+ (DEVICE) |
                      |   |                        |                   +----------+
+-----------          |   |                        |
|   ADB    |          v   +                        |                   +----------+
|  CLIENT  +--------->SmartSocket                  |              (USB)|   ADBD   |
+----------+          ^   | (TCP/IP)            Transport+-------------+ (DEVICE) |
                      |   |                        |                   +----------+
+----------+          |   |                        |
|  DDMLIB  |          |   |                     Transport+--+          +----------+
|  CLIENT  +----------+   |                        |        |  (TCP/IP)|   ADBD   |
+----------+              +------------------------+        +----------|(EMULATOR)|
                                                                       +----------+
```

### adb root 处理逻辑

#### adbd 命令处理代码

```c++
unique_fd daemon_service_to_fd(std::string_view name, atransport* transport) {
    ADB_LOG(Service) << "transport " << transport->serial_name() << " opening service " << name;
... ...

    } else if (android::base::ConsumePrefix(&name, "reboot:")) {
        return reboot_device(std::string(name));
    } else if (name.starts_with("root:")) { // 重点
        return create_service_thread("root", restart_root_service);  // 重点
    } else if (name.starts_with("unroot:")) {
        return create_service_thread("unroot", restart_unroot_service);
... ...
```
可以看到 adbd 收到 root 命令后，使用 create_service_thread 启动线程执行 restart_root_service 逻辑。


#### restart_root_service

```c++
void restart_root_service(unique_fd fd) {
    if (getuid() == 0) { // 判断 1
        WriteFdExactly(fd.get(), "adbd is already running as root\n");
        return;
    }
    if (!__android_log_is_debuggable()) { // 判断 2
        WriteFdExactly(fd.get(), "adbd cannot run as root in production builds\n");
        return;
    }

    LOG(INFO) << "adbd restarting as root";
    android::base::SetProperty("service.adb.root", "1"); // 3
    WriteFdExactly(fd.get(), "restarting adbd as root\n");
}
```

1. 首先判断 uid == 0，即已经是 root 用户，返回；
2. 通过  __android_log_is_debuggable 来判断权限，是否运行开启 root；
3. 若权限允许，则设置系统属性 service.adb.root 为 1。


#### __android_log_is_debuggable

```c++
int __android_log_is_debuggable() {
  static int is_debuggable = [] { // 静态变量
    char value[PROP_VALUE_MAX] = {};
    return __system_property_get("ro.debuggable", value) > 0 && !strcmp(value, "1");
  }();

  return is_debuggable;
}

```

可以看到该函数若 ro.debuggable = 1 时返回1， 并且 is_debuggable 是静态变量，即只会初始化赋值一次，若是修改了 ro.debuggable 不会更改该函数返回值，需要重新启动 adbd。

### adb root 是如何生效的？

adb root 的逻辑只是设置了属性 service.adb.root，那是如何重启 adbd 为 root 用户的？

#### adbd 重启

假设： 在init.rc 的脚本注册 service.adb.root 属性的监控逻辑，如配置下面命令：
```bash

on property: service.adb.adb=1
    重启 adbd 为 root 用户
```

但搜索完代码后未发现，所以是其他方式。

搜索代码发现：

```c++

asocket* create_local_service_socket(std::string_view name, atransport* transport) {
#if !ADB_HOST
    if (asocket* s = daemon_service_to_socket(name); s) {
        return s;
    }
#endif
    unique_fd fd = service_to_fd(name, transport);
    if (fd < 0) {
        return nullptr;
    }

    int fd_value = fd.get();
    asocket* s = create_local_socket(std::move(fd));
    LOG(VERBOSE) << "LS(" << s->id << "): bound to '" << name << "' via " << fd_value;

#if !ADB_HOST
    if ((name.starts_with("root:") && getuid() != 0 && __android_log_is_debuggable()) ||
        (name.starts_with("unroot:") && getuid() == 0) || name.starts_with("usb:") ||
        name.starts_with("tcpip:")) {
        D("LS(%d): enabling exit_on_close", s->id);
        s->exit_on_close = 1; // 设置 exit_on_close 值
    }
#endif

    return s;
}

```

此处也调用 __android_log_is_debuggable 进行判断，若满足条件设置 exit_on_close 为1。

```c++
// be sure to hold the socket list lock when calling this
static void local_socket_destroy(asocket* s) {
    int exit_on_close = s->exit_on_close;

    D("LS(%d): destroying fde.fd=%d", s->id, s->fd);

    deferred_close(fdevent_release(s->fde));

    remove_socket(s);
    delete s;

    if (exit_on_close) {
        D("local_socket_destroy: exiting");
        exit(1); // 退出
    }
}
```
在 local_socket_destroy 中判断若 exit_on_close 为 1，则调用 exit 退出，即会有 adbd 重启。

#### adbd 启动

adbd 的启动配置是在 adbd.rc 文件中，如下：

```bash
service adbd /apex/com.android.adbd/bin/adbd --root_seclabel=u:r:su:s0
    class core
    socket adbd seqpacket 660 system system
    disabled
    override
    seclabel u:r:adbd:s0
```

可以看到，没有指定 user 和 group，则 adbd 默认启动为 root 用户。

```c++
int adbd_main(int server_port) {
    ... ...

#if defined(__ANDROID__)
    drop_privileges(server_port);
#endif

    ... ...
```

在启动后会调用 drop_privileges 来执行降权操作。

```c++
static void drop_privileges(int server_port) {
    ScopedMinijail jail(minijail_new());

    gid_t groups[] = {AID_ADB,          AID_LOG,          AID_INPUT,    AID_INET,
                      AID_NET_BT,       AID_NET_BT_ADMIN, AID_SDCARD_R, AID_SDCARD_RW,
                      AID_NET_BW_STATS, AID_READPROC,     AID_UHID,     AID_EXT_DATA_RW,
                      AID_EXT_OBB_RW};
    minijail_set_supplementary_gids(jail.get(), arraysize(groups), groups);

    // Don't listen on a port (default 5037) if running in secure mode.
    // Don't run as root if running in secure mode.
    if (should_drop_privileges()) { // 判断是否需要降权 1
        const bool should_drop_caps = !__android_log_is_debuggable();

        if (should_drop_caps) {
            minijail_use_caps(jail.get(), CAP_TO_MASK(CAP_SETUID) | CAP_TO_MASK(CAP_SETGID));
        }

        minijail_change_gid(jail.get(), AID_SHELL); // 降权 2
        minijail_change_uid(jail.get(), AID_SHELL);
        // minijail_enter() will abort if any priv-dropping step fails.
        minijail_enter(jail.get());

        // Whenever ambient capabilities are being used, minijail cannot
        // simultaneously drop the bounding capability set to just
        // CAP_SETUID|CAP_SETGID while clearing the inheritable, effective,
        // and permitted sets. So we need to do that in two steps.
        using ScopedCaps =
            std::unique_ptr<std::remove_pointer<cap_t>::type, std::function<void(cap_t)>>;
        ScopedCaps caps(cap_get_proc(), &cap_free);
        if (cap_clear_flag(caps.get(), CAP_INHERITABLE) == -1) {  // 降权 2
            PLOG(FATAL) << "cap_clear_flag(INHERITABLE) failed";
        }
        if (cap_clear_flag(caps.get(), CAP_EFFECTIVE) == -1) {
            PLOG(FATAL) << "cap_clear_flag(PEMITTED) failed";
        }
        if (cap_clear_flag(caps.get(), CAP_PERMITTED) == -1) {
            PLOG(FATAL) << "cap_clear_flag(PEMITTED) failed";
        }
        if (cap_set_proc(caps.get()) != 0) {
            PLOG(FATAL) << "cap_set_proc() failed";
        }

        D("Local port disabled");
    } else {
        // minijail_enter() will abort if any priv-dropping step fails.
        minijail_enter(jail.get());

        if (root_seclabel != nullptr) {
            if (selinux_android_setcon(root_seclabel) < 0) { // 不降权 3
                LOG(FATAL) << "Could not set SELinux context";
            }
        }
    }
}
```

1. 调用 should_drop_privileges 判断是否需要降权；
2. 若降权，则 修改用户和用户组为 shell，并对 capabilities 权限清空；
3. 若不降权，则设置 security contexts 为 u:r:su:s0，root_seclabel 在 adbd.rc 中指定。

```c++
static bool should_drop_privileges() {
    // The properties that affect `adb root` and `adb unroot` are ro.secure and
    // ro.debuggable. In this context the names don't make the expected behavior
    // particularly obvious.
    //
    // ro.debuggable:
    //   Allowed to become root, but not necessarily the default. Set to 1 on
    //   eng and userdebug builds.
    //
    // ro.secure:
    //   Drop privileges by default. Set to 1 on userdebug and user builds.
    bool ro_secure = android::base::GetBoolProperty("ro.secure", true);
    bool ro_debuggable = __android_log_is_debuggable();

    // Drop privileges if ro.secure is set...
    bool drop = ro_secure; // 初始值

    // ... except "adb root" lets you keep privileges in a debuggable build.
    std::string prop = android::base::GetProperty("service.adb.root", "");
    bool adb_root = (prop == "1");
    bool adb_unroot = (prop == "0");
    if (ro_debuggable && adb_root) {
        drop = false; // 修改逻辑
    }
    // ... and "adb unroot" lets you explicitly drop privileges.
    if (adb_unroot) {
        drop = true;
    }

    return drop;
}
```

从上面修改逻辑可以看到，若 ro.debuggable == 1 && service.adb.root == 1 时 drop 为 false，即不降权。

#### 总结

adb root 生效需要的操作为：

* 修改 ro.debuggable 为 1
* 杀掉 adbd，让 __android_log_is_debuggable 返回为 1
* 调用 adb root 用于配置 service.adb.root = 1

## Magisk 工具使用

### 属性设置

使用 magisk 提供的修改属性工具 resetprop，将 ro.debuggable 设置为 1。

### 权限处理

设置了属性执行后失败，查看是缺少 SELinux 权限：

```bash
[ 5252.630174] type=1400 audit(1626010790.323:6145): avc: denied { setcurrent } for comm="adbd" scontext=u:r:adbd:s0 tcontext=u:r:adbd:s0 tclass=process permissive=1
[ 5252.630353] type=1400 audit(1626010790.323:6146): avc: denied { dyntransition } for comm="adbd" scontext=u:r:adbd:s0 tcontext=u:r:su:s0 tclass=process permissive=1
```

缺少下面权限：
```bash
allow adbd self:process setcurrent;
allow adbd su:process dyntransition;
```

查看 AOSP 代码发现如下：

```bash
userdebug_or_eng(`
  allow adbd self:process setcurrent;
  allow adbd su:process dyntransition;
')
```
可以看到，只有在 userdebug 和 eng 版本中才会有，所以正常 user 是不能使用 adb root 的。还有为了 adbd 转成 su domain后不受限制，我们将 su 配置为 permissive 状态。

使用 magisk 中的 magiskpolicy 工具将 su 配置为 permissive，即：

```bash
magiskpolicy --live 'permissive { su }'
```

## 实现

```bash
#!/system/bin/sh
su -c "resetprop ro.debuggable 1"
su -c "resetprop service.adb.root 1" # 减少调用 adb root
su -c "magiskpolicy --live 'allow adbd adbd process setcurrent'" # 配置缺少的权限
su -c "magiskpolicy --live 'allow adbd su process dyntransition'" # 配置缺少的权限
su -c "magiskpolicy --live 'permissive { su }'" # 将 su 配置为 permissive，防止后续命令执行缺少权限
su -c "kill -9 `ps -A | grep adbd | awk '{print $2}' `" # 杀掉 adbd

```

执行上面脚本，然后再执行 adb shell 即可，此时已具备 root 权限。


## 总结

1. adbd 正常是 root 用户启动，然后通过降权到 shell 用户，我们是通过修改属性来达到不降权；
2. 还有 adb root 操作正常情况下是用在 userdebug 和 eng 版本中，而这些版本我们外部是拿不到的，只有 user 版本，而 user 版本在 SELinux 权限上限制会更多，所以需要处理权限问题。