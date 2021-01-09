---
layout: article

title:  博客修改记录
date:   2020-10-02 12:12：12 +0800
 
tags: misc
key: blog_record
---

## 目的

该博客采用[jekyll-TeXt-theme](https://github.com/kitian616/jekyll-TeXt-theme)主题，一个主题不可能让所有人都满意，不满意就会修改，本文会记录下我修改的地方。

## 修改记录

**2020-10-01** 新增置顶功能

**2020-10-02** 新增pdf和ppt显示功能

**2021-01-09** 增加打赏功能

<!--more-->

## 文章置顶功能

### 文章显示原理

主页文章是使用插件jekyll-paginate进行分页，Archive文章不会进行分页，然后都调用_include/article-list.html将文件按照主题规则输出。

### 置顶功能插件

置顶功能现有插件[jekyll-stickyposts](https://github.com/ibrado/jekyll-stickyposts)，单独使用该插件对Archive文章有效果，是由于所有文章都在一页展示。而对于主页会分页文章来使用，只会将置顶的文章置顶在当前页上，而不是第一页上，没有达到理想要求。这就需要在jekyll-paginate分页前进行置顶排序，而jekyll-paginate目前无法实现。该插件原理是将置顶文件排在前面，置顶文章之间和其他文章按照配置的规则进行排序，然后新增字段stickiness，该字段记录所有文章的序号。

了解到插件[jekyll-paginate-v2](https://github.com/sverrirs/jekyll-paginate-v2)满足要求，我们就需要使用该插件。该插件有个配置项sort_field，可以很方面设置根据哪个字段排序，结合jekyll-stickyposts插件，就可以将排序字段设置为有文章序号的stickiness字段。

### 修改显示

现在可以将文章显示在最上面，但还需要增加置顶标识，增加了**<font color=red>[置顶]</font>**标识在文章标题上。

### GitHub Pages插件策略

在本地执行没有问题，但是放在Github Pages上显示不正常，困扰了一天。然后搜索了解到GitHub Pages会限制三方插件执行，可以查看[这里](https://jekyllrb.com/docs/plugins/installation/)。上面使用的两个插件都是三方的，所以都不能使用。但从下面的提示可以看到，可以将本地生成的静态网页上传到Github Pages上。

![graph]({{"/assets/pictures/misc/blog/github_pages_plugins.png" | prepend:site.baseurl}})

解决方法是：
1. 新建一个source分支，该分支为现有代码；
2. 由于GitHub Pages显示master分支，所以将本地生成的静态网页上传到master分支。

### 效果

**主页效果**

![graph]({{"/assets/pictures/misc/blog/top_home.png" | prepend:site.baseurl}})

**Archive效果**

![graph]({{"/assets/pictures/misc/blog/top_archive.png" | prepend:site.baseurl}})

## pdf和ppt显示功能

使用插件[jekyll-pdf-embed](https://github.com/MihajloNesic/jekyll-pdf-embed).

## 打赏功能

使用的代码来自于 [潘柏信的博客](https://github.com/leopardpan/leopardpan.github.io)。

在修改过程中，“打赏”属于链接，默认采用 main.css 中默认红色，直接 css 中配置的白色不起作用，最后采用在 html 代码中增加字体颜色解决。

```html
<p><a href="javascript:void(0)" onclick="dashangToggle()" class="dashang" title="打赏，支持一下"><font color="white">打赏</font></a></p>
```