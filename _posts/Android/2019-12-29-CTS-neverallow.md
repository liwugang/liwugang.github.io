---
layout: article

title:  Android CTS中neverallow规则生成过程
date:   2019-12-29 08:51:00 +0800
tags: Android SELinux
key: cts_neverallow
---

CTS里面SELinux相关测试中neverallow测试项占绝大多数，Android系统开发者都应该知道，在修改sepolicy时，需要确保不能违反这些neverallow规则，不然会过不了CTS。CTS中nerverallow测试都是在SELinuxNeverallowRulesTest.java文件中，并且从AOSP代码中发现该文件不是人工提交的，而是通过python脚本生成的，为了以后更好的修改sepolicy，就需要了解下SELinuxNeverallowRulesTest.java是如何生成的。

<!--more-->

## Makefile

首先看下SELinuxNeverallowRulesTest.java的生成的[Makefile](http://androidxref.com/9.0.0_r3/xref/cts/hostsidetests/security/Android.mk#69).

```makefile
selinux_general_policy := $(call intermediates-dir-for,ETC,general_sepolicy.conf)/general_sepolicy.conf

selinux_neverallow_gen := cts/tools/selinux/SELinuxNeverallowTestGen.py

selinux_neverallow_gen_data := cts/tools/selinux/SELinuxNeverallowTestFrame.py

LOCAL_ADDITIONAL_DEPENDENCIES := $(COMPATIBILITY_TESTCASES_OUT_cts)/sepolicy-analyze

LOCAL_GENERATED_SOURCES := $(call local-generated-sources-dir)/android/cts/security/SELinuxNeverallowRulesTest.java # 目标文件

$(LOCAL_GENERATED_SOURCES) : PRIVATE_SELINUX_GENERAL_POLICY := $(selinux_general_policy)
$(LOCAL_GENERATED_SOURCES) : $(selinux_neverallow_gen) $(selinux_general_policy) $(selinux_neverallow_gen_data)
	mkdir -p $(dir $@)
	$< $(PRIVATE_SELINUX_GENERAL_POLICY) $@
# $< 为：右边依赖的第一个元素， 即 $(selinux_neverallow_gen) = cts/tools/selinux/SELinuxNeverallowTestGen.py
# $@ 为：左边目标，即要生成的目标文件SELinuxNeverallowRulesTest.java
# 这条命令相当于 cts/tools/selinux/SELinuxNeverallowTestGen.py $(call intermediates-dirfor,ETC,general_sepolicy.conf)/general_sepolicy.conf SELinuxNeverallowRulesTest.java

include $(BUILD_CTS_HOST_JAVA_LIBRARY)
```

从上面可以看到，执行SELinuxNeverallowTestGen.py general_sepolicy.conf SELinuxNeverallowRulesTest.java会生成SELinuxNeverallowRulesTest.java文件。

## general_sepolicy.conf 生成

该文件的生成[Makfile](http://androidxref.com/9.0.0_r3/xref/system/sepolicy/Android.mk#773)

```makefile
# SELinux policy embedded into CTS.
# CTS checks neverallow rules of this policy against the policy of the device under test.
##################################
include $(CLEAR_VARS)

LOCAL_MODULE := general_sepolicy.conf # 目标文件
LOCAL_MODULE_CLASS := ETC
LOCAL_MODULE_TAGS := tests

include $(BUILD_SYSTEM)/base_rules.mk

$(LOCAL_BUILT_MODULE): PRIVATE_MLS_SENS := $(MLS_SENS)
$(LOCAL_BUILT_MODULE): PRIVATE_MLS_CATS := $(MLS_CATS)
$(LOCAL_BUILT_MODULE): PRIVATE_TARGET_BUILD_VARIANT := user
$(LOCAL_BUILT_MODULE): PRIVATE_TGT_ARCH := $(my_target_arch)
$(LOCAL_BUILT_MODULE): PRIVATE_WITH_ASAN := false
$(LOCAL_BUILT_MODULE): PRIVATE_SEPOLICY_SPLIT := cts
$(LOCAL_BUILT_MODULE): PRIVATE_COMPATIBLE_PROPERTY := cts
$(LOCAL_BUILT_MODULE): $(call build_policy, $(sepolicy_build_files), \
$(PLAT_PUBLIC_POLICY) $(PLAT_PRIVATE_POLICY)) # PLAT_PUBLIC_POLICY = syetem/sepolicy/public PLAT_PRIVATE_POLICY = system/sepolicy/private
	$(transform-policy-to-conf) # 这里是使用m4将te规则文件都处理合成为目标文件$@，即general_sepolicy.conf
	$(hide) sed '/dontaudit/d' $@ > $@.dontaudit

##################################
```

可以看到，general_sepolicy.conf 文件是将system/sepolicy/public和system/sepolicy/private规则文件整合在一起，而这些目录包含的是AOSP sepolicy大多数配置信息。

## SELinuxNeverallowTestGen.py 脚本逻辑

生成的逻辑都是在该脚本中，下面脚本我调整了顺序，方便说明执行的逻辑，[脚本代码](http://androidxref.com/9.0.0_r3/xref/cts/tools/selinux/SELinuxNeverallowTestGen.py)

```python
#!/usr/bin/env python

import re
import sys
import SELinuxNeverallowTestFrame

usage = "Usage: ./SELinuxNeverallowTestGen.py <input policy file> <output cts java source>"

if __name__ == "__main__":
    # check usage
    if len(sys.argv) != 3:
        print usage
        exit(1)
    input_file = sys.argv[1]
    output_file = sys.argv[2]

    # 这三个变量是同目录下SELinuxNeverallowTestFrame.py文件中的内容，是生成java文件的模版
    src_header = SELinuxNeverallowTestFrame.src_header
    src_body = SELinuxNeverallowTestFrame.src_body
    src_footer = SELinuxNeverallowTestFrame.src_footer

    # grab the neverallow rules from the policy file and transform into tests
    neverallow_rules = extract_neverallow_rules(input_file) # 提取neverallow规则从general_sepolicy.conf中
    i = 0
    for rule in neverallow_rules:
        src_body += neverallow_rule_to_test(rule, i)
        i += 1
    # 然后将neverallow规则写入到SELinuxNeverallowRulesTest.java文件中
    with open(output_file, 'w') as out_file:
        out_file.write(src_header)
        out_file.write(src_body)
        out_file.write(src_footer)

# extract_neverallow_rules - takes an intermediate policy file and pulls out the
# neverallow rules by taking all of the non-commented text between the 'neverallow'
# keyword and a terminating ';'
# returns: a list of rules
def extract_neverallow_rules(policy_file):
    with open(policy_file, 'r') as in_file:
        policy_str = in_file.read()

        # full-Treble only tests are inside sections delimited by BEGIN_TREBLE_ONLY
        # and END_TREBLE_ONLY comments.

        # uncomment TREBLE_ONLY section delimiter lines
        remaining = re.sub(
            r'^\s*#\s*(BEGIN_TREBLE_ONLY|END_TREBLE_ONLY|BEGIN_COMPATIBLE_PROPERTY_ONLY|END_COMPATIBLE_PROPERTY_ONLY)',
            r'\1', # group 引用
            policy_str,
            flags = re.M) # 该方法是将 #开头的注释行任意空格后跟着BEGIN_TREBLE_ONLY、END_TREBLE_ONLY、BEGIN_COMPATIBLE_PROPERTY_ONLY和END_COMPATIBLE_PROPERTY_ONLY时，替换为这些关键字，即去掉注释
        # remove comments 
        remaining = re.sub(r'#.+?$', r'', remaining, flags = re.M)  # 将文件中的 # 开头注释行去掉
        # match neverallow rules
        lines = re.findall(
            r'^\s*(neverallow\s.+?;|BEGIN_TREBLE_ONLY|END_TREBLE_ONLY|BEGIN_COMPATIBLE_PROPERTY_ONLY|END_COMPATIBLE_PROPERTY_ONLY)',
            remaining,
            flags = re.M |re.S) # 将neverallow和以这几个关键字开头的行取出来

        # extract neverallow rules from the remaining lines
        # 这些关键字会修饰里面的neverallowrules，若treble_only_depth > 1 说明是适用于treble系统， 若compatible_property_only_depth > 1，说明适用于 compatible_property 系统
        rules = list()
        treble_only_depth = 0
        compatible_property_only_depth = 0
        for line in lines:
            if line.startswith("BEGIN_TREBLE_ONLY"):
                treble_only_depth += 1
                continue
            elif line.startswith("END_TREBLE_ONLY"):
                if treble_only_depth < 1:
                    exit("ERROR: END_TREBLE_ONLY outside of TREBLE_ONLY section")
                treble_only_depth -= 1
                continue
            elif line.startswith("BEGIN_COMPATIBLE_PROPERTY_ONLY"):
                compatible_property_only_depth += 1
                continue
            elif line.startswith("END_COMPATIBLE_PROPERTY_ONLY"):
                if compatible_property_only_depth < 1:
                    exit("ERROR: END_COMPATIBLE_PROPERTY_ONLY outside of COMPATIBLE_PROPERTY_ONLY section")
                compatible_property_only_depth -= 1
                continue
            rule = NeverallowRule(line)
            rule.treble_only = (treble_only_depth > 0)
            rule.compatible_property_only = (compatible_property_only_depth > 0)
            rules.append(rule)

        if treble_only_depth != 0:
            exit("ERROR: end of input while inside TREBLE_ONLY section")
        if compatible_property_only_depth != 0:
            exit("ERROR: end of input while inside COMPATIBLE_PROPERTY_ONLY section")

        return rules

# neverallow_rule_to_test - takes a neverallow statement and transforms it into
# the output necessary to form a cts unit test in a java source file.
# returns: a string representing a generic test method based on this rule.

# 将neverallowrules 替换到java模版中
def neverallow_rule_to_test(rule, test_num):
    squashed_neverallow = rule.statement.replace("\n", " ")
    method  = SELinuxNeverallowTestFrame.src_method
    method = method.replace("testNeverallowRules()",
        "testNeverallowRules" + str(test_num) + "()")
    method = method.replace("$NEVERALLOW_RULE_HERE$", squashed_neverallow)
    method = method.replace(
        "$FULL_TREBLE_ONLY_BOOL_HERE$",
        "true" if rule.treble_only else "false")
    method = method.replace(
        "$COMPATIBLE_PROPERTY_ONLY_BOOL_HERE$",
        "true" if rule.compatible_property_only else "false")
    return method

```
#### 总结下脚本功能

1. 将BEGIN_TREBLE_ONLY|END_TREBLE_ONLY|BEGIN_COMPATIBLE_PROPERTY_ONLY|
END_COMPATIBLE_PROPERTY_ONLY这几个关键字前面的注释去掉，以便后面解析时使用；
2. 删除冗余的注释行；
3. 取neverallow和上面四个关键字的部分进行解析，并根据下面情况对treble_only和compatible_property_only进行设置； 

    * neverallow 包含在BEGIN_TREBLE_ONLY和END_TREBLE_ONLY之间，treble_only被设置为true；
    * neverallow 包含在BEGIN_COMPATIBLE_PROPERTY_ONLY和END_COMPATIBLE_PROPERTY_ONLY之间，compatible_property_only被设置为true；
    * neverallow 不在任何BEGIN_TREBLE_ONLY/END_TREBLE_ONLY和BEGIN_COMPATIBLE_PROPERTY_ONLY/END_COMPATIBLE_PROPERTY_ONLY之间，则treble_only和compatible_property_only都被设置为false。
4. 然后用neverallow部分、treble_only和compatible_property_only值对下面方法模板中的$NEVERALLOW_RULE_HERE$、$FULL_TREBLE_ONLY_BOOL_HERE$和$COMPATIBLE_PROPERTY_ONLY_BOOL_HERE$分别替换。
```java
src_method = """
    @RestrictedBuildTest
    public void testNeverallowRules() throws Exception {
        String neverallowRule = "$NEVERALLOW_RULE_HERE$";
        boolean fullTrebleOnly = $FULL_TREBLE_ONLY_BOOL_HERE$;
        boolean compatiblePropertyOnly = $COMPATIBLE_PROPERTY_ONLY_BOOL_HERE$;

        if ((fullTrebleOnly) && (!isFullTrebleDevice())) {
            // This test applies only to Treble devices but this device isn't one
            return;
        }
        if ((compatiblePropertyOnly) && (!isCompatiblePropertyEnforcedDevice())) {
            // This test applies only to devices on which compatible property is enforced but this
            // device isn't one
            return;
        }

        // If sepolicy is split and vendor sepolicy version is behind platform's,
        // only test against platform policy.
        File policyFile =
                (isSepolicySplit() && mVendorSepolicyVersion < P_SEPOLICY_VERSION) ?
                deviceSystemPolicyFile :
                devicePolicyFile;

        /* run sepolicy-analyze neverallow check on policy file using given neverallow rules */
        ProcessBuilder pb = new ProcessBuilder(sepolicyAnalyze.getAbsolutePath(),
                policyFile.getAbsolutePath(), "neverallow", "-w", "-n",
                neverallowRule);
        pb.redirectOutput(ProcessBuilder.Redirect.PIPE);
        pb.redirectErrorStream(true);
        Process p = pb.start();
        p.waitFor();
        BufferedReader result = new BufferedReader(new InputStreamReader(p.getInputStream()));
        String line;
        StringBuilder errorString = new StringBuilder();
        while ((line = result.readLine()) != null) {
            errorString.append(line);
            errorString.append("\\n");
        }
        assertTrue("The following errors were encountered when validating the SELinux"
                   + "neverallow rule:\\n" + neverallowRule + "\\n" + errorString,
                   errorString.length() == 0);
    }
```

## 本地生成 SELinuxNeverallowRulesTest.java 文件

在修改SELinux后，想确定下是否满足neverallow规则，虽然编译过程中会进行neverallow检查，但由于打包时间比较耗时，如果在本地生成的话，那速度会更快。

### 本地生成 SELinuxNeverallowRulesTest.java 命令

默认是在源码的根目录

> make general_sepolicy.conf

> cts/tools/selinux/SELinuxNeverallowTestGen.py out/target/product/cepheus/obj/ETC/general_sepolicy.conf_intermediates/general_sepolicy.conf  SELinuxNeverallowRulesTest.java


由于某些规则是使用attribute，可能不是很明显，还需要结合其他方法来确定。

## 总结

从生成代码中可以看到，neverallow规则都属于AOSP system/sepolicy/private和system/sepolicy/public中的neverallow，所以在添加规则时不能修改neverallow，也不能违背。

## 附件

[cts_neverallow.zip](https://github.com/liwugang/liwugang.github.io/tree/master/assets/files/cts_neverallow.zip)，中包含有：

> SELinuxNeverallowTestGen.py 脚本

> general_sepolicy.conf

> SELinuxNeverallowTestFrame.py Java测试代码模板

> first 为SELinuxNeverallowTestGen.py第一步执行的结果

> second 为SELinuxNeverallowTestGen.py第二步执行的结果

> SELinuxNeverallowRulesTest.java 为生成的文件

后面三个文件是前三个文件所生成，执行命令为：

> SELinuxNeverallowTestGen.py general_sepolicy.conf SELinuxNeverallowRulesTest.java
