---
layout: article

title:  类似py2exe软件真的能保护python源码吗
date:    2015-11-29 15:52:24 +0800
 
tags: Python
key: python_protection
---

## 背景

最近写了个工具用于对项目中C/C++文件的字符串常量进行自动化加密处理，用python写的，工具效果不错，所以打算在公司内部推广。为了防止代码泄露就考虑不采用直接给源码方式，而python二进制脚本pyc和pyo，虽然提供的不是源码，但可以通过uncompyle2直接得到源码。通过网上资料发现有Windows下的py2exe、Mac下的py2app和跨平台的PyInstaller工具都可以将python脚本打包成可执行文件，第一反应应该满足需要，但有些不放心，故亲自尝试和分析了这些工具。

<!--more-->

## pyexe

该工具用于Windows下将python脚本和python解释器打包成可执行文件。这样可以在没有安装python的机器上运行。打包后会有个library.zip打包的文件，如图:

![graph]({{"/assets/pictures/misc/python_protection/pyexe.png" | prepend:site.baseurl}})

里面包含有所依赖的模块和打包的模块，都是pyc存储，因此可以很容易的通过uncompyle2得到源码。因此基本上是零保护。

## 跨平台工具PyInstaller

使用了这几个工具后发现该工具功能更为强大。打包后的可执行文件不依赖python，可以直接在没装python的机器上运行。要生成不同系统的可执行文件就必须在对应系统上进行打包，而不能一次打包跨平台运行。

该工具可以打包成一个目录或者成一个可执行文件，网上教程很多我也就不介绍了。但打包成一个文件实质执行的时候首先会解压成一个目录，然后才能运行，因此一个文件实质上是对一个目录的压缩存储。该工具提供接口可以使用AES对模块进行加密。

该工具将与系统无关的部分打包进CArchive格式（类似于zip）的文件中，并将该文件放入到生成的可执行文件末尾。PyInstaller提供工具pyi-archive_viewer进行查看，如图:

![graph]({{"/assets/pictures/misc/python_protection/pyinstaller.png" | prepend:site.baseurl}})

下面将对上述图中所列的重点文件进行介绍。

### out00-PYZ.pyz

保存打包所依赖的模块，类似于上述的library.zip，该文件属于ZlibArchive格式，如图：

![graph]({{"/assets/pictures/misc/python_protection/pyz.png" | prepend:site.baseurl}})

从图中可以看出，Table Of Contents部分包含有模块名字、在文件中的偏移和长度。通过这些信息可以得到模块的内容，该内容是压缩的，需要使用zlib进行解压。pyi-archive_viewer也可以查看上述信息。如下图所示：

![graph]({{"/assets/pictures/misc/python_protection/pyi-archive.png" | prepend:site.baseurl}})

### pyimod00_crypto_key

当使用AES加密选项后，才会有该文件，该文件是pyc格式文件，使用uncompyle2得到的内容为：

![graph]({{"/assets/pictures/misc/python_protection/uncompyle2.png" | prepend:site.baseurl}})

从图中可以看出，只有一个key，实际上这就是我使用AES加密的key，因此该key相当于明文存储。

### DealProject

该文件是我处理的python源代码文件，他是直接以源码形式存在的。

可以看出PyInstaller虽然比py2exe复杂点，但同样也是非常容易得到源代码的。

## py2app

同样也对Mac上该工具进行了测试，发现py2app打包后的程序同样存在上述问题，并且不能再没有python的机器上运行，处理的文件也是以源码的形式存在，其他文件是以pyc格式打包在压缩文件中。

## 总结
通过目前主流的打包软件看出，他们只是提供了将python脚本打包成可执行文件，用于在没有python的机器上运行，并没有对python脚本进行保护处理，可以很方便的得到源码。目前了解的解决方案有cython，cython是属于python的超集，他首先会将python代码转化成c语言代码，然后通过c编译器生成可执行文件，这样会克服上述出现的情况，并且可以调用c库的函数，这样提高了程序的性能，并增加了代码的安全性，不失为一种不错的方案。