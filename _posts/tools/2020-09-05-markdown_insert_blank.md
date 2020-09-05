--- 
layout: article

title:  markdown 如何插入多个空格？
date:   2020-09-05 17:10:00 +0800

tags: tools
key:  markdown_blank

---

使用 markdown 发现，即使输入多个空格最终展示上也只有一个空格，那怎么实现展示多个空格？

<!--more-->

## Ideographic Space

之前有了解到 markdown 是将语言转化成 HTML，然后由浏览器引擎渲染成我们看到的效果。从语法中并没有看到可以直接输入多个空格，以为要实现该功能需要使用 HTML 的语法完成。但是偶然发现一篇 markdown 文档，使用了多个空格。使用 hexedit 查看文件的二进制发现空格是使用 **E38080**.

**E38080** 是 UTF-8 编码，对应的 Unicode 编码是**00003000**，是 Ideographic Space，是全角空格，CJK 空格。平时输入的是 ASCII 空格，值是 0x20。

## Linux 上如何输入 Unicode？

现在知道空格的值，但是怎么输入上去呢？ 从[这里](https://fsymbols.com/keyboard/linux/unicode/)了解到，Linux 上输入需要：

* 同时按下：CTRL + SHIFT + U，不释放；
* 然后输入 Unicode 码，如3000；
* 释放 CTRL + SHIFT + U，即输入了对应的字符。

对于 Windows 和 Mac 输入 Unicode，网上教程很多，有需要的可以搜索下。

## 效果

markdown　　　　　　　　　　　　is good!




