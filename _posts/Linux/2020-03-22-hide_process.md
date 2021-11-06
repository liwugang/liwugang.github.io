---
layout: article

title:  Linux 下如何隐藏自己不被发现？
date:   2020-03-22 19:12:00 +0800
 
tags: Linux
key: hide_process
---

可能在某些情况下，自己运行的程序不想或者不方便被其他人看到，就需要隐藏运行的进程。或者某些攻击者采用了本文介绍的隐藏技术，也可以让大家看到如何进行对抗。
隐藏有两种方法：
1. kernel 层面，不对用户层暴露该进程的信息，进程不被看见；
2. 用户层可以看到该进程信息，但不是以真实的身份出现，而是伪造成别的程序出现，达到隐藏自己的目的。

方法一是需要修改内核，本文主要是讲第二种方法。


<!--more-->

## 常用获取进程信息工具

在 Linux 环境下，主要是通过 top、ps 等工具来查看当前运行的进程，他们主要是通过读取 /proc/ 文件系统来获取进程信息，读取哪些内容，我们可以通过 strace (该工具用于监控系统调用和信号) 工具查看，我们以 296277 进程为例。

### top 获取进程信息

#### 296277 进程信息

```bash
   PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                                                                              
 296277 wangshu+  20   0    2272   1380   1300 S   0.0   0.0   0:00.00 bash 
```
top 查看到 296277 进程的信息，我们一般主要关心的是最后一列，进程名。

#### strace 跟踪结果
```c
stat("/proc/296277", {st_mode=S_IFDIR|0555, st_size=0, ...}) = 0
openat(AT_FDCWD, "/proc/296277/stat", O_RDONLY) = 9
read(9, "296277 (bash) S 296271 296271 23"..., 1024) = 322
close(9)                                = 0
openat(AT_FDCWD, "/proc/296277/statm", O_RDONLY) = 9
read(9, "568 355 335 2 0 78 0\n", 1024) = 21
close(9)
```

从上面可以看到，主要是读取了 /proc/296277/stat 和 /proc/296277/statm 文件。

* /proc/296277/stat 内容

```bash
296277 (bash) S 296271 296271 230491 34840 296271 4194304 94 155 0 0 0 0 0 0 20 0 1 0 94413592 2326528 345 18446744073709551615 94055941185536 94055941191257 140723269084624 0 0 0 0 4096 0 0 0 0 17 11 0 0 0 0 0 94055941201384 94055941202080 94055956852736 140723269087783 140723269087820 140723269087820 140723269091283 0
```
* /proc/296277/statm 内容

```bash
568 345 325 2 0 78 0
```
可以看到 top 中的进程名是使用 /proc/296277/stat 中的第二项 (bash)。

### ps -aux 获取进程信息

#### 296277 进程信息

```bash
wangshu+  296277  0.0  0.0   2272  1380 pts/24   S+   18:59   0:00 /bin/bash
```
ps -aux 查看到 296277 进程的信息，最后一列是 bash 的完整路径名。

#### strace 跟踪结果

```c
stat("/proc/296277", {st_mode=S_IFDIR|0555, st_size=0, ...}) = 0
openat(AT_FDCWD, "/proc/296277/stat", O_RDONLY) = 6
read(6, "296277 (bash) S 296271 296271 23"..., 2048) = 322
close(6)                                = 0
openat(AT_FDCWD, "/proc/296277/status", O_RDONLY) = 6
read(6, "Name:\tbash\nUmask:\t0022\nState:\tS "..., 2048) = 1095
close(6)                                = 0
openat(AT_FDCWD, "/proc/296277/cmdline", O_RDONLY) = 6
read(6, "/bin/bash\0", 131072)          = 10
read(6, "", 131035)                     = 0
close(6)                                = 0
stat("/dev/pts24", 0x7ffe4ce31630)      = -1 ENOENT (No such file or directory)
stat("/dev/pts/24", {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0x18), ...}) = 0
stat("/etc/localtime", {st_mode=S_IFREG|0644, st_size=545, ...}) = 0
stat("/etc/localtime", {st_mode=S_IFREG|0644, st_size=545, ...}) = 0
write(1, "wangshu+  296277  0.0  0.0   227"..., 104) = 104
```
从上面可以看到，主要是读取了 /proc/296277/stat、/proc/296277/status 和 /proc/296277/cmdline。

* /proc/296277/cmdline 内容

```bash
/bin/bash
```

ps -aux 中进程名可以看到是从 /proc/296277/cmdline 读取到的。

{:.warning}
上面列出来proc信息，可以从[proc文件内容介绍](http://man7.org/linux/man-pages/man5/proc.5.html)获取更详细的介绍。

### 小结

从上面实验中可以看到，进程名主要是通过 /proc/进程号/stat 和 /proc/进程号/cmdline 文件中获取。

## 隐藏技术实现

/proc/进程号/stat 和 /proc/进程号/cmdline 这些接口都是内核实现的，通过阅读内核代码得到：
* /proc/进程号/stat 中的进程名对应的可执行文件的文件名；
* /proc/进程号/cmdline 内容对应的是启动该进程的使用的路径，对应的是入口函数 main(int argc, char *argv[]) 中的 argv[0]。

为了实现隐藏，我们将伪造为 bash 进程， bash 进程比较常见，并且一般不会被过多关注。所以需要进行的操作有：
1. 文件名修改为 bash；
2. 运行时将 argv[0] 修改成 /bin/bash；
3. 为了防止从 /proc/进程号/maps 中发现没有包含 /bin/bash，我们将 /bin/bash 文件 mmap 进我们的进程中。

{:.warning}
2中 Android 环境中不能直接拿到argv[0] 怎么办？ 在 Unix 环境下，有定义全局变量 __progname_full， 该变量指向了 argv[0]，可以直接使用 __progname_full 进行修改，当然在 Linux 下同样适用。

### 代码示例
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <sys/types.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <sys/mman.h>

extern const char *__progname_full; // 引入该变量

int main(int argc, char **argv)
{
	int pid = getpid();
	printf("current pid: %d\n", pid);
	char buffer[100] = {0};
	char cmdline[100] = {0};
	sprintf(cmdline, "cat /proc/%d/cmdline", pid);
	system(cmdline);
	printf("\n");
	scanf("%s", &buffer);
	printf("%d change the name: %s\n", getpid(), buffer);

	//int size = strlen(__progname_full);      // 使用 __progname_full 进行修改 cmdline
	//printf("name len: %d\n", size);
	//memset((char *)__progname_full, 0, size);
	//strcpy((char *)__progname_full, buffer);

	int size = strlen(argv[0]);   // 使用 argv[0] 进行修改 cmdline
	printf("name len: %d\n", size);
	memset((char *)argv[0], 0, size);
	strcpy((char *)argv[0], buffer);

	int fd = open(buffer, O_RDONLY);     // 打开伪造后的可执行文件， mmap 到我们进程中
	if (fd < 0) {
		printf("not found the file:%s\n", buffer);
		return -1;
	}
	struct stat st;
	fstat(fd, &st);
	void *addr = mmap(NULL, st.st_size, PROT_READ, MAP_PRIVATE, fd, 0); // 可读 map
	if (addr == MAP_FAILED) {
		printf("mmap failed\n");
	}
	void *addr1 = mmap(NULL, st.st_size, PROT_READ | PROT_WRITE, MAP_PRIVATE, fd, 0); // 读写 map
	if (addr1 == MAP_FAILED) {
		printf("mmap1 failed\n");
	}
	void *addr2 = mmap(NULL, st.st_size, PROT_READ | PROT_EXEC, MAP_PRIVATE, fd, 0); // 读可执行 map
	if (addr2 == MAP_FAILED) {
		printf("mmap2 failed\n");
	}
	system(cmdline);
	printf("\n");
	scanf("%s", &buffer);
	munmap(addr, st.st_size);
	munmap(addr1, st.st_size);
	munmap(addr2, st.st_size);
	close(fd);

	return 0;
}
```
#### 执行结果

```bash
current pid: 301929  # 当前进程
/home/f/doing/linux/change_name/bash  # 伪造前 cmdline
/bin/bash # 要伪造的可执行文件
301929 change the name: /bin/bash
name len: 36
/bin/bash # 伪造后的 cmdline

```
### 工具查看结果

#### ps -aux 查看

![graph]({{"/assets/pictures/hide_process/ps.png" | prepend:site.baseurl}})

#### top 查看

![graph]({{"/assets/pictures/hide_process/top.png" | prepend:site.baseurl}})

可以看到都已经是 bash。


## 技术对抗

上述隐藏方式可以骗过 ps 和 top 工具，但是只是修改了名字，但可执行文件等信息没有改变，所以从其他信息可以推断出是进程名是伪造的，下面阐述几种方式。

### /proc/进程名/exe

在 /proc/进程名/exe 中保存的真正执行的可执行文件，所以可以将该文件和伪造后的 /bin/bash 文件对比，若发现不同可以证明进程名是伪造的，但此时还不能确认真正的可执行文件，只能和文件系统中的其他可执行文件对比进行确认。

### /proc/进程名/maps

Linux 中内存的布局是有一定的规律的，如下图，地址从低到高，分别是ELF可执行文件、Data和Bss区域、HEAP、memory map区域、stack等。我们可以根据这些信息来确认真正的可执行文件。

![graph]({{"/assets/pictures/hide_process/linux_memory_map.png" | prepend:site.baseurl}})

来看下伪造进程的内存布局：

![graph]({{"/assets/pictures/hide_process/proc_maps.png" | prepend:site.baseurl}})

从上面可以看出来，红色框是ELF可执行文件、Data和Bss区域，其实就是进程的真正可执行文件，接着是 HEAP，后面是我们 mmap 伪造的 bash 文件，所以通过 /proc进程名/maps  可以确认出真正的可执行文件为 /home/f/doing/linux/change_name/bash。


### 小结

上面中 /proc/进程名/exe 能简单判断出进程名是否伪造，但不能确认真正的可执行文件，而通过 /proc/进程名/maps 可以非常直观的看到可执行文件的信息。除了这两种方式外，可能还有其他方法，比如 /proc/进程名/status 中 VmExe 字段，该字段是记录 text 段大小，和显示的可执行文件进行对比，确认是否有伪造。

## 总结

虽然原理上比较简单，但是提醒我们有这么一种方式，可以简单的骗过我们通过ps和top看到的信息。若发现系统中有异常，也可以通过技术对抗中提到的方式，确认是否有恶意程序存在。