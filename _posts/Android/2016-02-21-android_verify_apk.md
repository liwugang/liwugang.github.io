---
layout: article

title:  android签名证书文件的解析和签名校验的加强
date:   2016-02-21 14:02:13 +0800
 
tags: Android
key: android_verify_apk
---

这篇文章介绍了apk中META-INF目录下CERT.RSA文件的解析，以及签名校验加强的一些方法。并用C++实现了解析代码，代码地址：[https://github.com/liwugang/pkcs7](https://github.com/liwugang/pkcs7).

<!--more-->

## 文章

{% pdf "/assets/pdf/android签名证书文件的解析和签名校验的加强.pdf" %}

## pkcs7格式解析图

![graph]({{"/assets/pictures/misc/格式解析图.png" | prepend:site.baseurl}})

