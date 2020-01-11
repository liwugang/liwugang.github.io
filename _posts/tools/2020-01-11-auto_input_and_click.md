---
layout: article

title:  Python实现自动化点击和输入功能
date:   2020-01-11 19:48:00 +0800
tags: Python
key: auto_input_click 
---



## 背景

由于工作上需要提交大量代码，而进代码平台上需要一个个输入change，效率太低，所以就想写个工具能够自动化进行输入，提高效率。首先想到的是chrome插件，在调研中发现服务端是使用了vue框架，难度比较大，所以就退而求其次，模拟用户输入和点击来实现。使用Python进行实现。

<!--more-->

![graph]({{"/assets/tools/auto_click_input.png" | prepend:site.baseurl}})

## Python 实现

需要先安装依赖的库
> pip3 install PyUserInput 

```python

#!/usr/bin/python3

import time

from pykeyboard  import *
from pymouse import *

m = PyMouse()
k = PyKeyboard()

time.sleep(5)  # 执行后有时间去切换到输入焦点上
print("begin ...")
for i in range(100, 200, 5): # 用100 105 ... 195进行举例140
    k.type_string(str(i)) # 输入 i
    k.tap_key(k.enter_key) # 输入 回车
    time.sleep(1) # 输入等待 1s

x, y = m.position() # 获取当前鼠标位置
m.click(x, y) # 在当前位置点击
```
