---
title: BUUCTF-EazyMD5
date: 2021-10-30 17:03:04
updated: 2021-10-30 17:03:04
tags: [sql注入,php相关]
categories: BUUCTF
toc: true
---

php-MD5特性

<!-- more -->

打开题目，尝试注入，毫无回显；bp抓包，看到header部分有hint，是一条sql语句

![](https://i.loli.net/2021/10/30/CTjcBeknNh5MWDv.png)

![](https://i.loli.net/2021/10/30/Ci8brvuk2ZWJ1hY.png)

### 知识点：

#### 1.关于password=md5($pass,true)

![](https://i.loli.net/2021/10/30/vPY6mpqGRTAJl1y.png)

> 1. `content: ffifdyop`
> 2. `hex: 276f722736c95d99e921722cf9ed621c`
> 3. `raw: 'or'6\xc9]\x99\xe9!r,\xf9\xedb\x1c`
> 4. `string: 'or'6]!r,b`

此处添加了 true，所以原始的16字符二进制格式被带到了sql语句当中

echo md5('ffifdyop',true) 的结果是 ---> `'or'6�]��!r,��b`

```
select * from 'admin' where password=''or'6�]��!r,��b'
```

为什么password = ''or'6�]��!r,��b'的返回值会是true呢

因为or后面的单引号里面的字符串（6�]��!r,��b），是数字开头的。当然不能以0开头。

<font color='red'>在mysql里面，在用作布尔型判断时，以1开头的字符串会被当做整型数。要注意的是这种情况是必须要有单引号括起来的</font>

比如``password=‘xxx’ or ‘1xxxxxxxxx’`，那么就相当于`password=‘xxx’ or 1`  ，也相当于`password=‘xxx’ or true`，所以返回值就是true。

<font color='red'>不只是1开头，只要是数字开头都是可以的。</font>

当然如果只有数字的话，就不需要单引号，比如password=‘xxx’ or 1，那么返回值也是true。（xxx指代任意字符）

payload: 

```bash
?password=ffifdyop
```

进入新页面，查看源代码发现：

```php+HTML
<?php
$a = $_GET['a'];
$b = $_GET['b'];

if($a != $b && md5($a) == md5($b)){
    // wow, glzjin wants a girl friend.?>
```

此处是`==`，即是弱比较，绕过有两种方式

#### 2.md5弱碰撞

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

#### 3.md5强比较

md5不能处理数组，所以会返回false，比较结果就是true



payload1:

```bash
/levels91.php?a[]=1&b[]=2
```

![](https://i.loli.net/2021/10/30/qOeSQ5jfnYaGNdC.png)

# 注意：hackbar的url输入框不会因为进入新页面而刷新，要手动loadurl！

![](https://i.loli.net/2021/10/30/UWl4QCkmK3F9YG8.png)

