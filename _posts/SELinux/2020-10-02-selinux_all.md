---
layout: article

title:  深入理解 SELinux 
date:   2020-10-03 11:55:00 +0800
 
tags: SELinux
key:  selinux_all
sticky: true
---

本文为 SELinux 专辑，记录相关文章。

<!--more-->

## SELinux

## Android

| 序号 | 文章名 | 概述
| - | - | -
|0|[SELinux 编译时 neverallow 如何限制？](/2020/03/31/neverallow_for_build.html) | 不管定义在system或vendor的neverallow规则，都会限制所有规则

### Domain

### File Contexts

### Property Contexts

| 序号 | 文章名 | 概述
| - | - | - 
|0|[Coredomain 创建新的 property type](/2020/02/29/coredomain_create_new_property.html) |treble架构下，coredomain使用新property type


### Service Contexts


### xTS

| 序号 | 文章名 | 概述
| - | - | - 
|0|[Android CTS中neverallow规则生成过程](/2019/12/29/CTS-neverallow.html) |CTS会进行neverallow测试，修改时不能违反这些规则，介绍Android如何生成neverallow规则