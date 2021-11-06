---
layout: article

title:  Linux Security Module 框架介绍
date:   2020-10-25 14:17:00 +0800

tags: Linux
key: introduce_lsm
---



LSM 在被合入到 Linux 之前，当时的访问控制机制还不足以提供强大的安全性，存在的一些增强的访问控制机制，由于缺少安全社区的共识而没有被合入到 Linux 中。Linux 是通用操作系统，需要满足不同需求，访问控制机制同样需要满足不同的访问控制机制。在这样的背景下，LSM 因运而生，满足轻量级、通用性和可集成不同的访问控制机制等特点，被合入到 Linux 中。

<!--more-->

## 架构

![graph]({{"/assets/pictures/linux/lsm_architecure.png" | prepend:site.baseurl}})

从上面看到，LSM hook 会插入到访问 kernel 对象前面，DAC 检查之后，然后 LSM 调用系统中启用的访问控制模块，检查是否可以访问。若有多个访问控制模块，会根据初始化的优先顺序执行，都允许访问才能进一步访问 kernel 对象。

## 数据结构

LSM 类似于 VFS，定义了通用的接口，每个访问控制模块可以根据需求，实现这些接口，并添加到 LSM 中去。

### 通用接口

[接口文件 lsm_hook_defs.h](https://github.com/torvalds/linux/blob/master/include/linux/lsm_hook_defs.h)，这里不直接使用函数声明，是为了复用，LSM_HOOK 在不同的结构定义中代表不同。后面会看到。

```c
... ...
LSM_HOOK(int, 0, file_permission, struct file *file, int mask)
LSM_HOOK(int, 0, file_alloc_security, struct file *file)
LSM_HOOK(int, 0, file_ioctl, struct file *file, unsigned int cmd, unsigned long arg)
LSM_HOOK(int, 0, file_mprotect, struct vm_area_struct *vma, unsigned long reqprot, unsigned long prot)
LSM_HOOK(int, 0, file_lock, struct file *file, unsigned int cmd)
LSM_HOOK(int, 0, file_fcntl, struct file *file, unsigned int cmd, unsigned long arg)
LSM_HOOK(int, 0, file_receive, struct file *file)
LSM_HOOK(int, 0, file_open, struct file *file)
... ...
```

### LSM 中接口管理

LSM 中使用 security_hook_heads 将各访问控制模块的接口管理起来。

```c
struct security_hook_heads {
	#define LSM_HOOK(RET, DEFAULT, NAME, ...) struct hlist_head NAME; // 此处定义 LSM_HOOK 为只取接口名字
	#include "lsm_hook_defs.h" // 包含接口文件
	#undef LSM_HOOK
} __randomize_layout;
```

在 security_hook_heads 中 LSM_HOOK 定义为只取接口名字，这样该数据结构是所有接口哈希链表(hlist_head)集合，即实际上为：

```c
struct security_hook_heads {
	... ...
    struct hlist_head file_permission;
    struct hlist_head file_alloc_security;
    struct hlist_head file_ioctl;
    struct hlist_head file_mprotect;
    struct hlist_head file_lock;
    struct hlist_head file_fcntl;
    struct hlist_head file_receive;
    struct hlist_head file_open;
    ... ...
} __randomize_layout;
```

### 访问控制模块的接口

每个访问控制模块需要定义 security_hook_list 数组，security_hook_list 记录单个接口的信息，然后模块初始化函数中会调用 security_add_hooks 将 security_hook_list 数组链接到上面的 security_hook_heads 上。

```c
struct security_hook_list {
	struct hlist_node		list; // 会将该 hlist_node 链接到 security_hook_heads.接口 上
	struct hlist_head		*head; // 指向 security_hook_heads.接口
	union security_list_options	hook; // 接口函数实现
	char				*lsm; // 该模块名字
} __randomize_layout;
```

security_hook_list 的赋值是通过 LSM_HOOK_INIT 宏，调用为 LSM_HOOK_INIT(接口, 接口实现)，会赋值 head 和 hook 成员变量，lsm 实际上是没有使用，list 会在 security_add_hooks 函数中链接起来。

```c
#define LSM_HOOK_INIT(HEAD, HOOK) \
	{ .head = &security_hook_heads.HEAD, .hook = { .HEAD = HOOK } }
```

上面接口函数实现为什么是 union 类型？ 看下 security_list_options 的定义：

```c
union security_list_options {
	#define LSM_HOOK(RET, DEFAULT, NAME, ...) RET (*NAME)(__VA_ARGS__);
	#include "lsm_hook_defs.h"
	#undef LSM_HOOK
};

```
可以看到，是将定义在 lsm_hook_defs.h 的各接口声明组合成 union结构，这样就可以用同一个类型来表示所有的接口函数。


### 访问控制模块接口添加到 LSM

security_add_hooks 函数负责将访问控制模块接口添加到 LSM，hooks 为该模块定义的接口数组，count 为数组大小，lsm 为模块名字。

```c++

void __init security_add_hooks(struct security_hook_list *hooks, int count,
				char *lsm)
{
	int i;

	for (i = 0; i < count; i++) {
		hooks[i].lsm = lsm;
		hlist_add_tail_rcu(&hooks[i].list, hooks[i].head);
	}

... ...
}
```

从上面知道，head 为 security_hook_heads.接口，即只是将每个接口的 security_hook_list.list 添加到 security_hook_heads.接口 的末尾。

### LSM 调用链

LSM 接口只有两种返回类型：void 和 int。即调用时会用下面这种方式。

```c++

#define call_void_hook(FUNC, ...)				\
	do {							\
		struct security_hook_list *P;			\
								\
		hlist_for_each_entry(P, &security_hook_heads.FUNC, list) \
			P->hook.FUNC(__VA_ARGS__);		\
	} while (0)

#define call_int_hook(FUNC, IRC, ...) ({			\
	int RC = IRC;						\
	do {							\
		struct security_hook_list *P;			\
								\
		hlist_for_each_entry(P, &security_hook_heads.FUNC, list) { \
			RC = P->hook.FUNC(__VA_ARGS__);		\
			if (RC != 0)				\
				break;				\
		}						\
	} while (0);						\
	RC;							\
})

```

通过 security_hook_heads 获取到接口的 hlist_head，然后遍历该链表，获取每个模块的 security_hook_list 变量，最后调用实际的接口函数。

### 访问控制模块结构

每个访问控制模块用 lsm_info 结构表示。

```c
struct lsm_info {
	const char *name;	// 访问控制模块名字
	enum lsm_order order;	// 该模块在 LSM 顺序，两种参数，LSM_ORDER_FIRST(目前capabilities使用) 和 LSM_ORDER_MUTABLE(默认)。
	unsigned long flags;    // 支持两种配置，LSM_FLAG_LEGACY_MAJOR 和 LSM_FLAG_EXCLUSIVE(指定该值的多个模块只允许开启一个)。
	int *enabled;		// 根据相应的规则，由 LSM 设置。
	int (*init)(void);	// 各模块的初始化函数
	struct lsm_blob_sizes *blobs; // 模块需要的私有数据大小
};
```

lsm_info 的定义会通过 DEFINE_LSM 和 DEFINE_EARLY_LSM 来定义。这样会将 lsm_info 放置在对应的 section 上。

```c
#define DEFINE_LSM(lsm)							\
	static struct lsm_info __lsm_##lsm				\
		__used __section(.lsm_info.init)			\
		__aligned(sizeof(unsigned long))

#define DEFINE_EARLY_LSM(lsm)						\
	static struct lsm_info __early_lsm_##lsm			\
		__used __section(.early_lsm_info.init)			\
		__aligned(sizeof(unsigned long))
```

## 代码逻辑

上面的数据结构，已经说明了 LSM 的架构，相对逻辑简单，本部分具体看下代码逻辑。


### LSM 初始化

start_kernel 是系统启动执行完架构相关代码后，通用代码的入口，LSM 的初始化也是在这里调用。LSM 会有两种类型，early_lsm 和 lsm。

* early_lsm 用于早期启动的模块，当配置 CONFIG_SECURITY_LOCKDOWN_LSM_EARLY 时lockdown 模块被配置为 early_lsm，调用的函数为 early_security_init。
* lsm 是除 lockdown 外默认配置的方式，调用的函数为 security_init。

#### early_security_init

```c
int __init early_security_init(void)
{
	int i;
	struct hlist_head *list = (struct hlist_head *) &security_hook_heads;
	struct lsm_info *lsm;

	for (i = 0; i < sizeof(security_hook_heads) / sizeof(struct hlist_head); // 1
	     i++)
		INIT_HLIST_HEAD(&list[i]);

	for (lsm = __start_early_lsm_info; lsm < __end_early_lsm_info; lsm++) { // 2
		if (!lsm->enabled)
			lsm->enabled = &lsm_enabled_true;
		prepare_lsm(lsm); // 3
		initialize_lsm(lsm); // 4
	}

	return 0;
}

/* Prepare LSM for initialization. */
static void __init prepare_lsm(struct lsm_info *lsm) // 3
{
	int enabled = lsm_allowed(lsm); // 判断该模块是否允许

	/* Record enablement (to handle any following exclusive LSMs). */
	set_enabled(lsm, enabled);

	/* If enabled, do pre-initialization work. */
	if (enabled) {
		if ((lsm->flags & LSM_FLAG_EXCLUSIVE) && !exclusive) {
			exclusive = lsm;
			init_debug("exclusive chosen: %s\n", lsm->name);
		}

		lsm_set_blob_sizes(lsm->blobs);
	}
}

/* Initialize a given LSM, if it is enabled. */
static void __init initialize_lsm(struct lsm_info *lsm) // 4
{
	if (is_enabled(lsm)) {
		int ret;

		init_debug("initializing %s\n", lsm->name);
		ret = lsm->init(); // 调用初始化的函数 init
		WARN(ret, "%s failed to initialize: %d\n", lsm->name, ret);
	}
}

```

1. 初始化 security_hook_heads；
2. __start_early_lsm_info 和 __end_early_lsm_info 之间是通过 DEFINE_EARLY_LSM 定义的 lsm_info，遍历这些 lsm_info；
3. prepare_lsm 查看是否允许该模块和统计模块所需要的数据大小，并将本模块的数据偏移回填到  lsm_info 中的 blobs 字段，允许的模块需要满足下面条件：
* lsm_info 中 enabled 为 TRUE；
* 若lsm_info 中的 flags 字段包含 LSM_FLAG_EXCLUSIVE 字段，则必须系统还没有相同字段的模块，或者没有该字段。
4. 若模块允许加载，则调用该模块的初始化函数 init。


#### security_init

```c

int __init security_init(void)
{
	struct lsm_info *lsm;

	pr_info("Security Framework initializing\n");

	/*
	 * Append the names of the early LSM modules now that kmalloc() is
	 * available
	 */
	for (lsm = __start_early_lsm_info; lsm < __end_early_lsm_info; lsm++) { // 1
		if (lsm->enabled)
			lsm_append(lsm->name, &lsm_names);
	}

	/* Load LSMs in specified order. */
	ordered_lsm_init();  // 2

	return 0;
}

static void __init ordered_lsm_init(void) // 2
{
	struct lsm_info **lsm;

	ordered_lsms = kcalloc(LSM_COUNT + 1, sizeof(*ordered_lsms),
				GFP_KERNEL);

	if (chosen_lsm_order) {
		if (chosen_major_lsm) {
			pr_info("security= is ignored because it is superseded by lsm=\n");
			chosen_major_lsm = NULL;
		}
		ordered_lsm_parse(chosen_lsm_order, "cmdline"); // 启动时根据用户选择来加载模块
	} else
		ordered_lsm_parse(builtin_lsm_order, "builtin"); // 若用户没有选择，则根据默认的模块顺序选择开启的模块

	for (lsm = ordered_lsms; *lsm; lsm++)
		prepare_lsm(*lsm); // 3

	init_debug("cred blob size     = %d\n", blob_sizes.lbs_cred);
	init_debug("file blob size     = %d\n", blob_sizes.lbs_file);
	init_debug("inode blob size    = %d\n", blob_sizes.lbs_inode);
	init_debug("ipc blob size      = %d\n", blob_sizes.lbs_ipc);
	init_debug("msg_msg blob size  = %d\n", blob_sizes.lbs_msg_msg);
	init_debug("task blob size     = %d\n", blob_sizes.lbs_task);

	/*
	 * Create any kmem_caches needed for blobs
	 */
	if (blob_sizes.lbs_file)
		lsm_file_cache = kmem_cache_create("lsm_file_cache",
						   blob_sizes.lbs_file, 0,
						   SLAB_PANIC, NULL);
	if (blob_sizes.lbs_inode)
		lsm_inode_cache = kmem_cache_create("lsm_inode_cache",
						    blob_sizes.lbs_inode, 0,
						    SLAB_PANIC, NULL);

	lsm_early_cred((struct cred *) current->cred);
	lsm_early_task(current);
	for (lsm = ordered_lsms; *lsm; lsm++)
		initialize_lsm(*lsm); // 4

	kfree(ordered_lsms);
}

```

1. 将之前的 early_lsm 名字添加到全局变量 lsm_names；
2. ordered_lsm_init 函数首先会调用 ordered_lsm_parse 来获取模块，然后调用 prepare_lsm 和 initialize_lsm，这两个函数和 early_security_init 中类似。
* 系统默认的模块加载顺序：lockdown,yama,loadpin,safesetid,integrity,selinux,smack,tomoyo,apparmor,bpf


### LSM 数据大小确定

LSM 数据大小保存在 blob_sizes 中，该值是由加载的模块需要累加获取得到的。每个模块需要的大小初始化在 lsm_info 中的 blobs 字段。计算方法是在 lsm_set_blob_sizes 函数中。

```c

static void __init lsm_set_blob_sizes(struct lsm_blob_sizes *needed)
{
	if (!needed)
		return;

	lsm_set_blob_size(&needed->lbs_cred, &blob_sizes.lbs_cred);
	lsm_set_blob_size(&needed->lbs_file, &blob_sizes.lbs_file);
	/*
	 * The inode blob gets an rcu_head in addition to
	 * what the modules might need.
	 */
	if (needed->lbs_inode && blob_sizes.lbs_inode == 0)
		blob_sizes.lbs_inode = sizeof(struct rcu_head);
	lsm_set_blob_size(&needed->lbs_inode, &blob_sizes.lbs_inode);
	lsm_set_blob_size(&needed->lbs_ipc, &blob_sizes.lbs_ipc);
	lsm_set_blob_size(&needed->lbs_msg_msg, &blob_sizes.lbs_msg_msg);
	lsm_set_blob_size(&needed->lbs_task, &blob_sizes.lbs_task);
}

static void __init lsm_set_blob_size(int *need, int *lbs)
{
	int offset;

	if (*need > 0) {
		offset = *lbs;
		*lbs += *need;
		*need = offset;
	}
}

```

lsm_set_blob_sizes 在 prepare_lsm 中被调用，可以看到 blob_sizes 的每个字段会加上模块的 blobs 对应字段，并且会将 offset 重新赋值给模块的 blobs 对应字段，那执行之后，模块的 blobs 中保存的不是需要的数据大小，而是数据偏移。

## 其他

### 查看系统的访问控制模块

* 若系统配置 CONFIG_SECURITYFS=y，则直接可以查看 /sys/kernel/security/lsm 文件，该文件会打印当前系统加载的模块。
* 若没有配置 CONFIG_SECURITYFS，则需要查看系统的 config，查看对应的模块是否编译进内核。如 Android 系统中，没有配置 CONFIG_SECURITYFS，则可以从 /proc/config.gz 获取系统配置。