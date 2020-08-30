---
layout: article

title:  记一次头文件不一致导致的异常
date:   2020-08-30 21:00:00 +0800

tags: debug
key:  header_mismatch

---

最近下载了[SELinux](https://github.com/SELinuxProject/selinux)代码，编译了checkpolicy，但是使用时出现了段错误，本文记录下分析过程。

<!--more-->

## VS code 配置调试环境

由于checkpolicy是自己使用源码编译，可以使用调试来分析问题。gdb是经常使用的命令行调试工具，但为了方便调试，这次是用VS code进行，顺便学习下如何配置。

首先VS code默认是个编辑器，首先要安装C/C++的插件，这样才能进行代码提示、跳转、编译和调试。

![graph]({{"/assets/pictures/debug/install_extension.png" | prepend:site.baseurl}})

安装插件如上图，点击左侧栏最下面插件按钮，默认会显示最常用的，也可以在上面输入框搜索C/C++，我们选择第一个出现的进行安装，由Microsoft提供的。

接下来打开我们要调试的源码。
![graph]({{"/assets/pictures/debug/debug_first.png" | prepend:site.baseurl}})

如上图进行配置调试，首先：
1. 选择菜单Run --> Start Debugging；
2. 出现的调试列表选择C++(GDB/LLDB)。

选择C++(GDB/LLDB)后， 会根据当前打开的文件出现不同菜单，若当前文件为.c或者.cpp等源码文件，会出现如下图：
![graph]({{"/assets/pictures/debug/build_and_debug.png" | prepend:site.baseurl}})

有两种配置方式：
1. Build and debug。该方式是先进行编译，然后在进行调试，并且可以选择不同的编译器；
2. Debug。最后的Default Configuration，该方式只会进行调试，不编译。

若当前文件非.c或者.cpp等源码文件，则会跳过"Build and debug"方式，默认使用上图Default Configuration方式并打开配置：

![graph]({{"/assets/pictures/debug/debug_configuration.png" | prepend:site.baseurl}})

我们只需要填写上“要填写的程序”和“调试参数”就行。

## 调试程序

由于执行会段错误，那先不用设置断点，执行后会断在异常地方。

![graph]({{"/assets/pictures/debug/segment_fault.png" | prepend:site.baseurl}})

如图，执行后断在ebitmap_cpy函数 new->startbit = n->startbit该行，结合左侧局部变量中看到，n = 0x40，问题就出现在这里。

根据左侧的调用堆栈显示，是main调用expand_module，expand_module调用ebitmap_cpy，下面根据栈回溯来分析。

![graph]({{"/assets/pictures/debug/stack_main.png" | prepend:site.baseurl}})

可以看到，在main函数中时policycaps中的node不为0x40。

![graph]({{"/assets/pictures/debug/stack_expand.png" | prepend:site.baseurl}})

而在expand_module中发现已经变为0x40，这个什么原因导致的？

一般出现这种情况都怀疑是多线程中其他线程修改这值导致，但是该程序不是多线程，并且是调用expand_module前后不同。所以问题不是多线程导致。

我们把调用前后的变量添加到watch处，发现地址都相同，并且都属于policydb_t *类型，但为什么不同？当我们添加了sizeof记录这两个变量的长度时，发现竟然不同。

![graph]({{"/assets/pictures/debug/compare.png" | prepend:site.baseurl}})

由于expand_module是属于libsepol，是在引入的静态库libsepol.a中，而main是在checkpolicy中，此时需要查看编译情况。

## 编译检查

![graph]({{"/assets/pictures/debug/makefile.png" | prepend:site.baseurl}})

libsepol.a 是使用SELinux下的libsepol目录生成，而checkpolicy生成是依赖于policydb.h头文件，从Makefile中并没有特殊指定目录，所以应该是使用系统目录下的policydb.h，而libsepol.a是使用SELinux目录下的。

![graph]({{"/assets/pictures/debug/more.png" | prepend:site.baseurl}})

通过对比发现，确实用于生成的libsepol.a的policydb_t类型多了filename_trans_count参数。要修复该问题，只需要将编译checkpolicy的CFLAGS增加 -I../libsepol/include即可，也就是使用和libsepol.a同样的头文件。

## 总结

头文件信息将会打包到程序中，并且也会随时更新，所以在编译代码时要注意头文件的一致，防止不一致导致的问题。特别是在升级系统后，要注意类似问题。

