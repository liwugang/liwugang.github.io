---
layout: article

title:  批量获取亲宝宝应用中的照片和视频
date:   2021-04-10 18:34:00 +0800

tags: reverse
key: reverse_qinbaobao
---


亲宝宝该款软件用起来不错，可以将手机上的孩子的图片和视频归纳起来，并且按照时间线排序；还可以添加亲戚进入，有社交属性，这样上传的孩子照片，亲戚都能看到，很方便。若想下载图片或者视频，只能单个操作，不能批量进行，之前有PC版本可以批量导出，但官方现在不支持。所以我就逆向了亲宝宝的Android版本，来看怎么批量获取。

<!--more-->

## 分析

正常情况下，应该分析客户端如何从服务端获取到照片的信息，然后根据这些信息去下载，这是正向思维。但是客户端与服务端通信协议，这是每个公司都会花费更多精力去保护，防止竞争对手或者恶意者用于非法用途。由于文件和视频都比较大，肯定是需要缓存下来，我们就从本地保存的数据分析，并结合抓包和逆向。

### 下载地址

抓包后发现，图片和视频的下载地址分别为下面格式，并且在请求时不需要额外参数，那么接下来问题是如何获取文件名。

```bash
http://stphoto-bj.qbb6.com/photo/XXX.jpg
http://stvideo-bj.qbb6.com/video/XXX.mp4

```
### 信息保存数据库btime.db

结合逆向，发现图片和视频的的数据都放在了btime.db数据库TbActivity表中，如下图：

![graph]({{"/assets/pictures/reverse/qinbaobao_btime.png" | prepend:site.baseurl}})


上图右边是文件类型，图片、视频还是音频。data中是具体的数据：

```
{
    "actiTime": 1617255239000,
    "commentNum": 0,
    "createTime": 1617258829823,
    "des": "这是你第一次游泳，还挺勇敢的。",
    ... ...
    "itemList": [
        {
            "actid": 5224163125,
            "data": "{\"fid\":XXX,\"secret\":\"XXXX\",\"owner\":XXXX,\"farm\":10038,\"fileType\":2001,\"size\":91974876,\"uploadTime\":1617270373000,\"duration\":180800,\"width\":1280,\"height\":720,\"status\":1}",
            "itemid": 7,
            "type": 1
        },
        {
            "actid": 5224163125,
            "data": "{\"ftid\":-1,\"des\":\"第一次游泳\"}",
            "itemid": 8,
            "type": 7
        }
    ],
    "itemNum": 2,
    "likeItems": "",
    "likeList": [],
    ... ...
    "ownerName": "妈妈",
    ... ...
    "updateTime": 1617270496378
}
```

itemList中包含有照片或者视频的信息，经测试发现，fid_secret 组合起来是文件名，那么在结合前面的链接就可以获取到照片或者视频了。

### 缓存数据

由于btime.db是在该应用的私有目录中，非root版本无法获取，所以对于非root版本还需要继续分析。接下来看应用的缓存数据。

#### 位置确定

一般情况下，应用都会将大数据放在外置存储中，来看外置空间中的该应用目录： /sdcard/Android/data/com.dw.btime

![graph]({{"/assets/pictures/reverse/qinbaobao_size.png" | prepend:site.baseurl}})

{:.warning}
借助命令： "du -h -d 1" 来查看不同目录下大小

从上图看到，图片缓存在cache/ImageCache或者cache/thumbnail中，视频肯定是在files/exoplayer目录下。

#### 照片

经过逆向代码分析，cache/ImageCache中的文件名是经过hash之后的，所以不能使用。而cache/thumbnail中的文件是最终文件名，但是压缩过的图片，所以我们只取文件名，然后结合上面链接下载最终图片。

#### 视频

![graph]({{"/assets/pictures/reverse/qinbaobao_exo.png" | prepend:site.baseurl}})

图上是files/exoplayer目录文件，经搜索可知，exo是exoplayer的视频格式，所以可以将这些文件导出，然后想办法通过exoplayer将这些文件转化成mp4格式。但有个非exo的文件，该文件很显眼，打开后发现该文件有视频完整链接，哈哈，看来可以直接从该文件中获取链接了。

![graph]({{"/assets/pictures/reverse/qinbaobao_exo_file.png" | prepend:site.baseurl}})

### 总结

* root版本，可以直接从/data/data/com.dw.btime/databases/btime.db获取到文件信息；
* 非root版本，从缓存中获取照片文件名和视频链接，然后在下载。（该种方式一定要在手机上点击过图片或者视频，这样应用才会生成缓存文件）


## 工具

本人使用python实现了上述两种方式的获取，[代码地址.](https://github.com/liwugang/misc/blob/main/tools/qinbaobao.py)
