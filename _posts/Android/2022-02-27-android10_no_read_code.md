---
layout: article

title:  Android 10 系统库代码段去掉读权限探讨
date:   2022-02-27 13:03:00 +0800
 
tags: Android
key: android10_no_read
---

前段时间在对系统函数进行 hook 检测时，发现了一些异常 crash，并且只发生在 Android 10 系统上，经过对错误日志分析，确认这是 Android 10 新增加的功能：系统库的代码段会映射到可执行内存，没有读权限。hook 检测的逻辑是需要读取指令解析来确认，所以导致了异常。

## 简介

从[官方介绍](https://developer.android.com/about/versions/10/behavior-changes-all#xom-binaries) 可知，增加该功能是为了安全考虑，并且该功能只支持 AArch64 架构，需要硬件和 Kernel 共同支持，硬件提供 PAN(Privileged Access Never) 和 kernel 提供 XOM(eXecute-Only Memory)，详细资料可查看：[Execute-only Memory (XOM) for AArch64 Binaries](https://source.android.com/devices/tech/debug/execute-only-memory)

![graph]({{"/assets/pictures/android/systemlib_execonly.png" | prepend:site.baseurl}})


但是在 Android 11 以及之后的版本上，该功能又被放弃了。不支持的原因主要是 kernel 不在支持 XOM，原因也可以在上面的详细资料中找到。 

## Android 使用原理

从上面和相关资料可知，主要实现在 kernel 和硬件上，Android 只是功能的使用者，下面只讨论 Android 相关的配置。

Android 是一个庞大的系统，不可能有人能了解到该系统的每个细节，我们应该掌握方法，在遇到不懂的问题时，能运用我们的方法快速定位到问题，了解其实现原理。

针对该问题，我们可以从错误日志开始入手，然后一步一步找到系统的改动处。

### 错误日志对应代码

![graph]({{"/assets/pictures/android/execonly_log.png" | prepend:site.baseurl}})

根据错误日志可找到相关代码处：

```c++
// http://aospxref.com/android-10.0.0_r47/xref/system/core/debuggerd/libdebuggerd/tombstone.cpp?r=&mo=3649&fi=108#108
static void dump_probable_cause(log_t* log, const siginfo_t* si, unwindstack::Maps* maps) {
  std::string cause;
  if (si->si_signo == SIGSEGV && si->si_code == SEGV_MAPERR) {
    if (si->si_addr < reinterpret_cast<void*>(4096)) {
      cause = StringPrintf("null pointer dereference");
    } else if (si->si_addr == reinterpret_cast<void*>(0xffff0ffc)) {
      cause = "call to kuser_helper_version";
    } else if (si->si_addr == reinterpret_cast<void*>(0xffff0fe0)) {
      cause = "call to kuser_get_tls";
    } else if (si->si_addr == reinterpret_cast<void*>(0xffff0fc0)) {
      cause = "call to kuser_cmpxchg";
     } else if (si->si_addr == reinterpret_cast<void*>(0xffff0fa0)) {
       cause = "call to kuser_memory_barrier";
     } else if (si->si_addr == reinterpret_cast<void*>(0xffff0f60)) {
       cause = "call to kuser_cmpxchg64";
     }
   } else if (si->si_signo == SIGSEGV && si->si_code == SEGV_ACCERR) {
     unwindstack::MapInfo* map_info = maps->Find(reinterpret_cast<uint64_t>(si->si_addr));
     if (map_info != nullptr && map_info->flags == PROT_EXEC) { // 进程 /proc/[pid]/maps 查询，若找到并且有可执行权限
       cause = "execute-only (no-read) memory access error; likely due to data in .text.";
     }
   } else if (si->si_signo == SIGSYS && si->si_code == SYS_SECCOMP) {
     cause = StringPrintf("seccomp prevented call to disallowed %s system call %d", ABI_STRING,
                          si->si_syscall);
   }
 
   if (!cause.empty()) _LOG(log, logtype::HEADER, "Cause: %s\n", cause.c_str());

```

maps 是进程当前内存信息，出现内存访问异常时，查询异常的地址所在模块，若模块存在并且有可执行权限，则就是触发了代码段读异常。

### 内存代码段配置

查看 Android 10 的进程内存中的系统库，对于代码段来说，只有执行权限，没有读权限，如下图：

![graph]({{"/assets/pictures/android/execonly_maps.png" | prepend:site.baseurl}})

针对内存代码段去掉读权限，首先猜测可能是在 linker 在加载 so 时，做了特殊处理了，去掉了读权限，但是阅读了加载 so 的代码后，未发现有特殊处理。
所以，应该是在打包是进行了特殊配置。

### 文件代码段配置

通过 readelf 查看 libc.so 可知，代码段只有执行权限，没有读权限。

![graph]({{"/assets/pictures/android/execonly_elf.png" | prepend:site.baseurl}})

所以可以确定的是，Android 10 对编译进行了改动，去掉了代码段的读权限。

### Android 打包配置

通过对打包代码查看和验证，下面是相关的配置：

```bash
# http://aospxref.com/android-10.0.0_r47/xref/build/make/core/binary.mk#83
ifneq ($(strip $(ENABLE_XOM)),false) # 总开关
  ifndef LOCAL_IS_HOST_MODULE
    my_xom := true
    # Disable XOM in excluded paths.
    combined_xom_exclude_paths := $(XOM_EXCLUDE_PATHS) \
                                  $(PRODUCT_XOM_EXCLUDE_PATHS)
    ifneq ($(strip $(foreach dir,$(subst $(comma),$(space),$(combined_xom_exclude_paths)),\
           $(filter $(dir)%,$(LOCAL_PATH)))),)
      my_xom := false
    endif

    # Allow LOCAL_XOM to override the above
    ifdef LOCAL_XOM # 单个模块开关
      my_xom := $(LOCAL_XOM)
    endif

    ifeq ($(strip $(my_xom)),true)
      ifeq (arm64,$(TARGET_$(LOCAL_2ND_ARCH_VAR_PREFIX)ARCH))
        ifeq ($(my_use_clang_lld),true)
          my_ldflags += -Wl,-execute-only # 开启时，增加的链接配置
        endif
      endif
    endif
  endif
endif

```

* ENABLE_XROM 是总开关，若设置为 false，则不会开启；
* XOM_EXCLUDE_PATHS 和 PRODUCT_XOM_EXCLUDE_PATHS 是将需要关闭的目录放置在该变量中；
* LOCAL_XOM，在总开关打开情况下，可通过该变量对单个模块进行设置

并且可以看到，最后是通过增加 ldflags += "-Wl,-execute-only" 来完成的。

### ndk 的配置

上面主要是对系统进行配置，可以根据上述原理针对 ndk 配置，在 ndk 的 Android.mk 中增加下面即可：

```bash
LOCAL_LDFLAGS += -fuse-ld=lld -Wl,-execute-only
```

## 解决方案

若我们需要读取代码段，应该怎么处理呢？从 android 官方已经提供了方案：通过 mprotect 增加内存读权限，读取完成后在去掉读权限。

具体的实施有三种方案：

* **访问前增加权限**

在访问前，所有代码段都通过 mprotect 增加读权限，这样逻辑简单，但代码有性能损耗，大多数情况下都不需要

* **检测后增加权限**

通过 Android 版本和内存代码段的权限判断，若是 Android 10 和访问的代码段没有读权限时，在通过 mprotect 增加读权限

* **直接访问**

通过增加信号捕获处理函数，直接读取内存，若有异常，则通过信号处理函数进行忽略，继续后面的操作，这样无需特殊处理

具体采用那种方案，可以根据实际情况处理


## 总结

上面提到的只是冰山一角，相关技术还会涉及到 Kernel、Arm 处理器安全特性、链接器等。

* 作为 Android 从业人员，应该多关注 Android 新版本改动，这样可以及时适配新的改动
* 随着计算机安全防护的发展，有些之前认为理所应当的事情会有改变，要持续学习




