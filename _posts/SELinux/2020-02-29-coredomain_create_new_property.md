---
layout: article

title:  Coredomain 创建新的 property type
date:   2020-02-29 08:53:00 +0800
 
tags: SELinux Android
key:  coredomain_new_property
---

## 背景

Android 8.1 中引入了 Project Treble 架构，用于将 vendor 下的驱动和 Android system 系统分开，Android system 可以单独升级。SELinux 同样分成了两部分，位于 /system/etc/selinux 下的 platform 部分和位于 /vendor/etc/selinux 下的 vendor 部分。本文分析 Android 大版本升级过程中对 property 增加的一些限制，已经如何绕过这些限制。

<!--more-->

## Coredomain

### 哪些是 coredomain? 

Coredomain 是 attribute，属于 domain (针对进程)或者 type(针对对象，如文件等)的集合。coremain 可以理解为包含 system 下可执行文件和 apps 所运行的 domain 或者说包含所有属于 Android 的 domain。

### 如何查看？

#### 代码中查看

有两种定义方式：

```bash
    1. type XXX ..., coredomain ...;
    2. typeattribute XXX coredomain;
```

上面两种方式都是将 XXX 放入到 coredomain 集合中，即 XXX 是属于 coredomain，对 coredomain 配置的权限也会同步给 XXX，同样对 coredomain 的限制也会限制 XXX。

#### 工具查看

Android system/sepolicy 源码下有 sepolicy-analyze 工具，然后需要编译好的 sepolicy 文件，使用下面命令可以查看 coredomain 都包含那些 domain：

```bash
    sepolicy-analyze 编译好的sepolicy  attribute coredomain
```

{:.warning}
编译好的policy，手机上可以从这两个地方取：1. /vendor(或者odm)/etc/selinux/precompiled_sepolicy; 2. /sys/fs/selinux/policy


## Android P

### 新增限制

升级 Android P 后发现 property 新增了一个 neverallow(neverallow 用于限制配置，明确指定不能干什么)。来看下该 neverallow：

```bash
#https://cs.android.com/android/platform/superproject/+/android-9.0.0_r11:system/sepolicy/public/property.te;l=314
compatible_property_only(`
  # Neverallow coredomain to set vendor properties
  neverallow {
    coredomain
    -init
    -system_writes_vendor_properties_violators
  } {
    property_type
    -apexd_prop
    -audio_prop
    -bluetooth_a2dp_offload_prop
    -bluetooth_audio_hal_prop
    -bluetooth_prop
    -bootloader_boot_reason_prop
    -boottime_prop
    -bpf_progs_loaded_prop
    -config_prop
    -cppreopt_prop
    -ctl_adbd_prop
    -ctl_bootanim_prop
    -ctl_bugreport_prop
    -ctl_console_prop
    -ctl_default_prop
    -ctl_dumpstate_prop
    -ctl_fuse_prop
    -ctl_gsid_prop
    -ctl_interface_restart_prop
    -ctl_interface_start_prop
    -ctl_interface_stop_prop
    -ctl_mdnsd_prop
    -ctl_restart_prop
    -ctl_rildaemon_prop
    -ctl_sigstop_prop
    -ctl_start_prop
    -ctl_stop_prop
    -dalvik_prop
    -debug_prop
    -debuggerd_prop
    -default_prop
    -device_logging_prop
    -dhcp_prop
    -dumpstate_options_prop
    -dumpstate_prop
    -exported2_config_prop
    -exported2_default_prop
    -exported2_radio_prop
    -exported2_system_prop
    -exported2_vold_prop
    -exported3_default_prop
    -exported3_radio_prop
    -exported3_system_prop
    -exported_bluetooth_prop
    -exported_config_prop
    -exported_dalvik_prop
    -exported_default_prop
    -exported_dumpstate_prop
    -exported_ffs_prop
    -exported_fingerprint_prop
    -exported_overlay_prop
    -exported_pm_prop
    -exported_radio_prop
    -exported_secure_prop
    -exported_system_prop
    -exported_system_radio_prop
    -exported_vold_prop
    -exported_wifi_prop
    -extended_core_property_type
    -ffs_prop
    -fingerprint_prop
    -firstboot_prop
    -device_config_activity_manager_native_boot_prop
    -device_config_reset_performed_prop
    -device_config_boot_count_prop
    -device_config_input_native_boot_prop
    -device_config_netd_native_prop
    -device_config_runtime_native_boot_prop
    -device_config_runtime_native_prop
    -device_config_media_native_prop
    -dynamic_system_prop
    -gsid_prop
    -heapprofd_enabled_prop
    -heapprofd_prop
    -hwservicemanager_prop
    -last_boot_reason_prop
    -system_lmk_prop
    -log_prop
    -log_tag_prop
    -logd_prop
    -logpersistd_logging_prop
    -lowpan_prop
    -lpdumpd_prop
    -mmc_prop
    -net_dns_prop
    -net_radio_prop
    -netd_stable_secret_prop
    -nfc_prop
    -overlay_prop
    -pan_result_prop
    -persist_debug_prop
    -persistent_properties_ready_prop
    -pm_prop
    -powerctl_prop
    -radio_prop
    -restorecon_prop
    -safemode_prop
    -serialno_prop
    -shell_prop
    -system_boot_reason_prop
    -system_prop
    -system_radio_prop
    -system_trace_prop
    -test_boot_reason_prop
    -test_harness_prop
    -theme_prop
    -time_prop
    -traced_enabled_prop
    -traced_lazy_prop
    -vendor_default_prop
    -vendor_security_patch_level_prop
    -vold_prop
    -wifi_log_prop
    -wifi_prop
  }:property_service set;
')
```

可以看到限制的是除了 init 和 system_writes_vendor_properties_violators 的 coredomain， 这些 domains 不能设置上述白名单之外的 property type， 这些属于AOSP 定义的 types，所以是限制 system domains 不能设置 vendor properties，符合 Prject Treble 设计思想。

### 如何绕过限制？

由于厂商基本都会：
1. 定义新的 property type，用于细粒度控制访问权限，如 platform_app 或者 system_app 设置新的property，三方 app 可以读；
2. coredomain 需要设置 vendor properties。

但现在增加了 neverallow 限制，这些规则在编译阶段就会报错，所以需要想办法绕过限制。

从上面可以看到限制会排除 init 和 system_writes_vendor_properties_violators，init 是 domain，不能满足要求，我们是需要 platform_app 或者 system_app 有权限。看下 system_writes_vendor_properties_violators 的定义：

```bash
# All system domains which violate the requirement of not writing vendor
# properties.
# TODO(b/78598545): Remove this once there are no violations
attribute system_writes_vendor_properties_violators;

```
首先是 attribute，不是具体的 type， 并且从注释上看到，表示 system domains 违反去写 vendor properties 的集合。这个也是 google 用于在升级中故意新增的，用于临时绕过 neverallow 限制。所以，我们可以需要违反 neverallow 的添加上该 attribute。

**解决方案为：  为需要违反的 domain 添加 system_writes_vendor_properties_violators。**

如要给 platform_app 添加，配置下面规则：

```bash
    typeattribute platform_app system_writes_vendor_properties_violators;
```

## Android Q

### 新增限制

google 在 GTS 中新增了一条测试 case:

```bash
com.google.android.security.gts.SELinuxHostTest#testNoExemptionsForCoreWritingVendorSysprops
```

SELinuxHostTest 是在 GTS GtsSecurityHostTestCases.jar 中，使用 JD-GUI 反编译找到 testNoExemptionsForCoreWritingVendorSysprops 方法：

```java
  public void testNoExemptionsForCoreWritingVendorSysprops() {
    if (this.mFirstApiLevel < 29) { // 使用 firstApi 排除升级的老机型
      return;
    }
    
    // 是获取有 attribute 是 system_writes_vendor_properties_violators 的 domain
    Set<String> types = sepolicyAnalyzeGetTypesAssociatedWithAttribute("system_writes_vendor_properties_violators"); 
    
    if (!types.isEmpty()) {
      List<String> sortedTypes = new ArrayList<String>(types);
      Collections.sort(sortedTypes);
      fail("Policy exempts core domains from ban on writing vendor system properties: " + sortedTypes); // 不为空，则测试失败
    } 
  }
}
```

从上面看到： 1）针对的是 Q 新机型； 2）若指定有 system_writes_vendor_properties_violators 的则失败，即不能为 domain 指定 system_writes_vendor_properties_violators。

所有针对 Q 新机型，不能采用 P 绕过方式。

### 如何绕过限制？


#### 使用现有的 AOSP property type

Neverallow 不能从 domain 进行绕过，只能选择从 property type，即不能定义新的 property，只能使用现有的。这样有两个弊端：

1. 不能对权限细粒度控制，只能在现有权限的基础上增加，而不能减少，这样修改的话势必导致权限扩大；
2. type 名字选择都是有意义，但现在只能从权限和名字上找到平衡，可能权限满足，但名字意义差太远。

那有没有更好的方式？

#### 定义新的 property type

通过对 neverallow 中的每个 type 分析，发现 extended_core_property_type 是 attribute，而不是具体的 type：

```bash

# All properties that are not specific to device but are added from
# outside of AOSP. (e.g. OEM-specific properties)
# These properties are not accessible from device-specific domains
attribute extended_core_property_type;

```

从注释可以看到，这个 attribute 是 google 专门给厂商保留的，太好了。 但是 “These properties are not accessible from device-specific domains” 表示 device-specific domains 不能访问，来看下是怎么限制的：

```bash
# Prevent properties from being read
  neverallow {
    domain
    -coredomain
    -appdomain
    -vendor_init
  } {
    core_property_type
    extended_core_property_type # it
    exported_dalvik_prop
    exported_ffs_prop
    exported_system_radio_prop
    exported2_config_prop
    exported2_system_prop
    exported2_vold_prop
    exported3_default_prop
    exported3_system_prop
    -debug_prop
    -logd_prop
    -nfc_prop
    -powerctl_prop
    -radio_prop
  }:file no_rw_file_perms;
```

可以看到除了 coredomain, appdomain, vendor_init 外都不能访问，那这种方式定义的 property 只能由 coredomain, appdomain 和 vendor_init 来访问。虽然有限制，但是已经能满足大部分需求，如上面提到的规则： platform_app 或者 system_app 设置，三方 apps 去读。

**解决方案：为新定义的 property type 指定 extended_core_property_type。**

{:.warning}
这种方式 vendor domain没有权限去读写。


## 思考

如果有需求是需要 coredomain 和 vendor domain 同时有读写权限，那怎么办？

目前来看，新建 property type不能满足需求，现有 AOSP 存在的 property type 如果没有限制，也尽量不要使用，防止以后有变动，建议：

**1. 业务逻辑上重新设计，尽量避免这种情况出现；**

**2. 使用 init 或者 vendor_init 代替设置，init 和 vendor_init 权限较高，并且限制较少，如对 init script 中配置自动 setprop 规则。**