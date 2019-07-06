---
layout: post

title:  OWASP Android crackmes
date:   2019-05-12 18:20:00 +0800
categories: reverse
tag: frida
---

* content
{:toc}

背景
================
最近在学习frida，就拿OWASP上的crackme进行练习，[下载地址](https://github.com/OWASP/owasp-mstg/blob/master/Crackmes/README.md)，关于该系列crackme分析的文章很多，我只列出相应的frida脚本。

UnCrackable App for Android Level 1
===============
```python
#!/usr/bin/python3

import frida
import sys

def on_message(message, data):
    if message['type'] == 'send':
        print("[*] {0}".format(message['payload']))
    else:
        print(message)

pid = frida.get_usb_device().spawn(["owasp.mstg.uncrackable1"]) # 打开app
process = frida.get_usb_device().attach(pid)
content = ""
with open("script_one.js") as f:
    content = f.read()
if len(content) == 0:
    print("not found script")
    exit(0)
script = process.create_script(content)
script.on("message", on_message)
script.load()
frida.get_usb_device().resume(pid)

sys.stdin.read()
```
```javascript
// hook root检测的三个方法，hook计算password的方法
Java.perform(function() {
    var checkClass = Java.use("sg.vantagepoint.a.c");
    checkClass.a.implementation = function() {
        var result = this.a();
        send("function a: " +  result);
        return false;
    };
    checkClass.b.implementation = function() {
        var result = this.b();
        send("function b: " +  result);
        return false;
    };
    checkClass.c.implementation = function() {
        var result = this.c();
        send("function c: " +  result);
        return false;
    };
    var password = "";
    var passwordClass = Java.use("sg.vantagepoint.a.a");
    passwordClass.a.implementation = function() {
        var result = this.a(arguments[0],arguments[1]);
        for(var i = 0; i < result.length; i++) {
           password += String.fromCharCode(result[i]);
        }
        send("password: " + password);
        return result;
    };
});
```