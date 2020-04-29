---
layout: article

title:  Makefile中的ifeq 多条件使用
date:   2015-08-21 20:14:23 +0800
 
tags: Makefile
key: makefile_ifelse
---

网上关于makefile中ifeq的介绍已经很多了，为什么我还要在写这篇文章，因为他们只说了if else两种条件的情况，并没有讲多于两种条件情况的使用。

<!--more-->

多于两种情况的使用很简单，害我尝试很多种方法，如ifeq elifeq等等这些。其实就如同c中的if [else if] [else if]...else的使用一样，举个我使用的例子，Android中的NDK程序android.mk判断当前是哪种CPU架构：

```bash
    ifeq ($(TARGET_ARCH), arm)
        LOCAL_SRC_FILES := ...
    else ifeq ($(TARGET_ARCH), x86)
        LOCAL_SRC_FILES := ...
    else ifeq ($(TARGET_ARCH), mips)
        LOCAL_SRC_FILES := ...
    else 
        LOCAL_SRC_FILES := ...
    endif
```