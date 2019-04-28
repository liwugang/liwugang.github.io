---
layout: post

title:  Git自动化合并多个Commit
date:   2019-04-28 23:10:00 +0800
categories: 工具
tag: git
---

* content
{:toc}


当我们有多个commit或者从开源处拿到多个commit时，想合成一个commit，并保留每个commit的message时，大家都知道用"git rebase -i"可以解决，但这种方式需要手动进行操作，假如我们要处理的比较多，就想要自动化来处理，下面介绍下怎么自动化处理。

git rebase逻辑
=============
当我们"git rebase -i"后，git在当前.git/rebase-merge目录下生成git-rebase-todo文件，然后调用git editor来让用户编辑git-rebase-todo文件进行处理，如果实现自动化就需要：
1. 修改git editor来使用我们提供的；
2. 脚本来处理进行git-rebase-todo文件的处理。

git editor的修改
===============
git提供config命令来查看配置和修改配置，同样editor也可以这样进行设置。
```bash
    git config core.editor #查看当前使用editor
    git config --local --replace-all  core.editor NEW_EDITOR # 修改当前的仓库的editor为NEW_EDITOR
```

处理git-rebase-todo文件
=====================
```bash
pick 62e0071 first commit
pick 3bd641a second commit
pick 92c03c7 third commit

# 变基 7073047..92c03c7 到 7073047（3 个提交）
#
# 命令:
# p, pick <提交> = 使用提交
# r, reword <提交> = 使用提交，但修改提交说明
# e, edit <提交> = 使用提交，进入 shell 以便进行提交修补
# s, squash <提交> = 使用提交，但融合到前一个提交
# f, fixup <提交> = 类似于 "squash"，但丢弃提交说明日志
# x, exec <命令> = 使用 shell 运行命令（此行剩余部分）
# b, break = 在此处停止（使用 'git rebase --continue' 继续变基）
# d, drop <提交> = 删除提交
# l, label <label> = 为当前 HEAD 打上标记
# t, reset <label> = 重置 HEAD 到该标记
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
# .       创建一个合并提交，并使用原始的合并提交说明（如果没有指定
# .       原始提交，使用注释部分的 oneline 作为提交说明）。使用
# .       -c <提交> 可以编辑提交说明。
#
# 可以对这些行重新排序，将从上至下执行。
#
# 如果您在这里删除一行，对应的提交将会丢失。
#
# 然而，如果您删除全部内容，变基操作将会终止。
#
# 注意空提交已被注释掉
```
上面是有3个commit，需要将后面2个commit合并到第1个，通过后面的注释可以看到，squash是将commit合并到前一个commit上，所以需要将后2个的pick修改为squash，即修改为下面这样：
```bash
pick 62e0071 first commit
squash 3bd641a second commit
squash 92c03c7 third commit
```

Python实现
=========
使用python实现上述逻辑
```python
#!/usr/bin/env python3
#encoding: UTF-8

import os
import sys

def change_editor(current_file):
    os.system("git config --local --replace-all  core.editor " + current_file) # 将当前脚本设置为editor
    os.system("git rebase -i") # 执行rebase，触发调用该脚本进行rebase
    os.system("git config --local --replace-all core.editor vim") # 执行完后将editor设置回vim

def rebase_commits(todo_file):
    with open(todo_file, "r+") as f:
        contents = f.read() # 读取git-rebase-todo文件内容
        contents = contents.split("\n")
        first_commit = True
        f.truncate()
        f.seek(0)
        for content in contents:
            if content.startswith("pick"):
                if first_commit:
                    first_commit = False
                else:
                    content = content.replace("pick", "squash") # 将除了地一个pick修改为squash
            f.write(content + "\n")

def main(args):
    if len(args) == 2:  # 如果将该脚本作为editor，git会调用该脚本，并以git-rebase-todo文件作为第2个参数
        rebase_commits(args[1])
    else:
        change_editor(os.path.abspath(args[0])) # 设置git editor

if __name__ == "__main__":
    main(sys.argv)

```