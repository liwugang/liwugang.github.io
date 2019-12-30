---
layout: article

title:  A Tool To Automate Multiple Commits Into One
date:   2019-12-30 21:55:00 +0800
tag: git
key: git_merge_commit_en
---


If we have multiple commits or some patches from the software supplier, and want to synthesize one commit and retain the commit message of each one, someone knows that **git rebase -i** can be used to solve it, but this method requires manual for operations. If we have a lot to do, it will took a lot of time. Here I will introduce a tool which to automate multiple commits into one.

excerpt_separator: <!--more-->

## The logic of "git rebase"

When using "git rebase -i", git generates the git-rebase-todo file in the current .git/rebase-merge directory, and then invokes the git editor to let users edit the git-rebase-todo file for processing. So the tool needs to meet:

* Modify the git editor to the tool which we provided;
* The tool processes the git-rebase-todo file.

## Modify the default git editor

Git provides the config command to view the configuration and modify the configuration, and the editor can also be set in this way.

```bash

    git config core.editor  #View the currently used editor
    git config --local --replace-all core.editor NEW_EDITOR # Set the editor to NEW_EDITOR

```

##  Process the git-rebase-todo file

![graph]({{"/assets/tools/git_log.png" | prepend:site.baseurl}})

Now we have three commits and use "git rebase -i", git uses the git editor to open the git-rebase-todo file, as follows:

```bash

pick 310b091 First commit
pick 757b5e5 The second commit
pick c5ca4a9 The third commit

# Rebase a7e56eb..c5ca4a9 onto a7e56eb (3 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup <commit> = like "squash", but discard this commit's log message
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
# .       create a merge commit using the original merge commit's
# .       message (or the oneline, if no original merge commit was
# .       specified). Use -c <commit> to reword the commit message.
#
# These lines can be re-ordered; they are executed from top to bottom.
#
# If you remove a line here THAT COMMIT WILL BE LOST.
#
# However, if you remove everything, the rebase will be aborted.
#
# Note that empty commits are commented out
```

There are three commit in the above. We need to merge the next two commit into the first one. As you can see from the comments, squash merges the commit into the previous commit. Therefore, you need to change the last two pick to squash, which is as follows:

```bash
pick 310b091 First commit
squash 757b5e5 The second commit
squash c5ca4a9 The third commit
```


## The tool

The tool is a python script.

```python
#!/usr/bin/env python3
#encoding: UTF-8

import os
import sys

def change_editor(current_file):
    os.system("git config --local --replace-all  core.editor " + current_file) # Set current_file as git editor
    os.system("git rebase -i") # execute the "git rebase -i" and will invoke the python file later with git-rebase-todo file as argument
    os.system("git config --local --replace-all core.editor vim") # after work reset the git editor to default

def rebase_commits(todo_file):
    with open(todo_file, "r+") as f:
        contents = f.read() # read git-rebase-todo's content
        contents = contents.split("\n")
        first_commit = True
        f.truncate()
        f.seek(0)
        for content in contents:
            if content.startswith("pick"):
                if first_commit:
                    first_commit = False
                else:
                    content = content.replace("pick", "squash") # replace the pick to squash except for the first pick
            f.write(content + "\n")

def main(args):
    if len(args) == 2:
        rebase_commits(args[1]) # process the git-rebase-todo
    else:
        change_editor(os.path.abspath(args[0])) # set git editor to this tool

if __name__ == "__main__":
    main(sys.argv)
```

## Result

After to execute the tool, it merges above three commits into one and retains the messages of them.

```bash
On branch master
Your branch is ahead of 'origin/master' by 1 commit.
  (use "git push" to publish your local commits)

nothing to commit, working tree clean

```
Just one commit.


```bash

commit e1ac798cfe0162f38c908d47720fcadc2d3b0ddd (HEAD -> master)
Author: liwugang <liwugang@163.com>
Date:   Mon Dec 30 22:04:56 2019 +0800

    First commit
    
    The first commit message
    
    Signed-off-by: liwugang <liwugang@163.com>
    
    The second commit
    
    The second commit message
    
    Signed-off-by: liwugang <liwugang@163.com>
    
    The third commit
    
    The third commit message
    
    Signed-off-by: liwugang <liwugang@163.com>

```

The three commit messages are preserved.