---
layout: article

title:  SELinux 编译时 neverallow 如何限制？
date:   2020-03-31 08:53:00 +0800
 
tags: SELinux Android
key:  neverallow_for_build
---

SELinux编译过程中会对配置进行校验，若有违反neverallow的配置会导致编译失败。

## 原理

1. 将所有的的policy文件通过m4合入到一个文件policy.conf中，然后通过checkpolicy编译成二进制policy，编译过程中会进行 neverallow 检查；

2. 将policy.conf文件中去掉只用于编译时使用的规则生成policy_2.conf，然后使用sepolicy-analyze对上面编译的二进制policy用policy_2.conf进行neverallow检查，模拟CTS测试。 

### 为什么有2步？
SELinux有expandattribute关键字，使用 expandattribute attribute_type true|false， 为true时会展开相当于 inline 方法，通过 secilc 编译不会存在在二进制中，但checkpolicy编译还会保留。

第二步实际上去掉expandattribute为true的attribute_type相关的 neverallow规则。

### 限制作用域
由于8.1后是都是project treble，system和vendor的policy会分开，从上面可以看到，policy会打包到一个文件中，neverallow不管放在system还是vendor下，都会限制所有的规则。


## 执行脚本

```bash

include $(CLEAR_VARS)
 
 
LOCAL_MODULE := sepolicy_neverallows
LOCAL_MODULE_CLASS := ETC
LOCAL_MODULE_TAGS := optional
LOCAL_MODULE_PATH := $(TARGET_OUT)/etc/selinux
 
include $(BUILD_SYSTEM)/base_rules.mk
 
# sepolicy_policy.conf - All of the policy for the device.  This is only used to
# check neverallow rules.
sepolicy_policy.conf := $(intermediates)/policy.conf
$(sepolicy_policy.conf): PRIVATE_MLS_SENS := $(MLS_SENS)
$(sepolicy_policy.conf): PRIVATE_MLS_CATS := $(MLS_CATS)
$(sepolicy_policy.conf): PRIVATE_TARGET_BUILD_VARIANT := user
$(sepolicy_policy.conf): PRIVATE_TGT_ARCH := $(my_target_arch)
$(sepolicy_policy.conf): PRIVATE_TGT_WITH_ASAN := $(with_asan)
$(sepolicy_policy.conf): PRIVATE_TGT_WITH_NATIVE_COVERAGE := $(with_native_coverage)
$(sepolicy_policy.conf): PRIVATE_ADDITIONAL_M4DEFS := $(LOCAL_ADDITIONAL_M4DEFS)
$(sepolicy_policy.conf): PRIVATE_SEPOLICY_SPLIT := $(PRODUCT_SEPOLICY_SPLIT)
$(sepolicy_policy.conf): $(call build_policy, $(sepolicy_build_files), \
$(PLAT_PUBLIC_POLICY) $(PLAT_PRIVATE_POLICY) $(PLAT_VENDOR_POLICY) \
$(PRODUCT_PUBLIC_POLICY) $(PRODUCT_PRIVATE_POLICY) \
$(BOARD_VENDOR_SEPOLICY_DIRS) $(BOARD_ODM_SEPOLICY_DIRS))
    $(transform-policy-to-conf)  # 会将所有的policy通过m4合入到policy.conf文件中
    $(hide) sed '/^\s*dontaudit.*;/d' $@ | sed '/^\s*dontaudit/,/;/d' > $@.dontaudit
 
# sepolicy_policy_2.conf - All of the policy for the device.  This is only used to
# check neverallow rules using sepolicy-analyze, similar to CTS.
sepolicy_policy_2.conf := $(intermediates)/policy_2.conf
$(sepolicy_policy_2.conf): PRIVATE_MLS_SENS := $(MLS_SENS)
$(sepolicy_policy_2.conf): PRIVATE_MLS_CATS := $(MLS_CATS)
$(sepolicy_policy_2.conf): PRIVATE_TARGET_BUILD_VARIANT := user
$(sepolicy_policy_2.conf): PRIVATE_EXCLUDE_BUILD_TEST := true  # 用于移除只在编译时使用的规则
$(sepolicy_policy_2.conf): PRIVATE_TGT_ARCH := $(my_target_arch)
$(sepolicy_policy_2.conf): PRIVATE_TGT_WITH_ASAN := $(with_asan)
$(sepolicy_policy_2.conf): PRIVATE_TGT_WITH_NATIVE_COVERAGE := $(with_native_coverage)
$(sepolicy_policy_2.conf): PRIVATE_ADDITIONAL_M4DEFS := $(LOCAL_ADDITIONAL_M4DEFS)
$(sepolicy_policy_2.conf): PRIVATE_SEPOLICY_SPLIT := $(PRODUCT_SEPOLICY_SPLIT)
$(sepolicy_policy_2.conf): $(call build_policy, $(sepolicy_build_files), \
$(PLAT_PUBLIC_POLICY) $(PLAT_PRIVATE_POLICY) $(PLAT_VENDOR_POLICY) \
$(PRODUCT_PUBLIC_POLICY) $(PRODUCT_PRIVATE_POLICY) \
$(BOARD_VENDOR_SEPOLICY_DIRS) $(BOARD_ODM_SEPOLICY_DIRS))
    $(transform-policy-to-conf)
    $(hide) sed '/^\s*dontaudit.*;/d' $@ | sed '/^\s*dontaudit/,/;/d' > $@.dontaudit
 
$(LOCAL_BUILT_MODULE): PRIVATE_SEPOLICY_1 := $(sepolicy_policy.conf)
$(LOCAL_BUILT_MODULE): PRIVATE_SEPOLICY_2 := $(sepolicy_policy_2.conf)
$(LOCAL_BUILT_MODULE): $(sepolicy_policy.conf) $(sepolicy_policy_2.conf) \
  $(HOST_OUT_EXECUTABLES)/checkpolicy $(HOST_OUT_EXECUTABLES)/sepolicy-analyze
ifneq ($(SELINUX_IGNORE_NEVERALLOWS),true)
    $(hide) $(CHECKPOLICY_ASAN_OPTIONS) $(HOST_OUT_EXECUTABLES)/checkpolicy -M -c \
        $(POLICYVERS) -o $@.tmp $(PRIVATE_SEPOLICY_1)  # checkpolicy 使用policy.conf 生成二进制policy
    $(hide) $(HOST_OUT_EXECUTABLES)/sepolicy-analyze $@.tmp neverallow -w -f $(PRIVATE_SEPOLICY_2) || \   # 使用 sepolicy-anlyze对二进制policy用policy_2.conf进行neverallow检查
      ( echo "" 1>&2; \
        echo "sepolicy-analyze failed. This is most likely due to the use" 1>&2; \
        echo "of an expanded attribute in a neverallow assertion. Please fix" 1>&2; \
        echo "the policy." 1>&2; \
        exit 1 )
endif # ($(SELINUX_IGNORE_NEVERALLOWS),true)
    $(hide) touch $@.tmp
    $(hide) mv $@.tmp $@
 
sepolicy_policy.conf :=
sepolicy_policy_2.conf :=
built_sepolicy_neverallows := $(LOCAL_BUILT_MODULE)

```