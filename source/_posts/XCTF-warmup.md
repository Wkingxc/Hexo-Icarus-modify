---
title: 攻防世界_warmup
date: 2021-09-24 11:58:10
updated: 2021-09-24 11:58:10
tags: [代码审计]
categories: XCTF
toc: true
---
XCTF-warmup签到题

<!-- more -->
 1.打开网页查看源代码，发现source.php

![](https://img-blog.csdnimg.cn/20210822194648656.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

 2.开始代码审计，source代码如下

```php
 <?php
    highlight_file(__FILE__);
    class emmm
    {
        public static function checkFile(&$page)
        {
            $whitelist = ["source"=>"source.php","hint"=>"hint.php"];
            if (! isset($page) || !is_string($page)) {
                echo "you can't see it";
                return false;
            }

            if (in_array($page, $whitelist)) {
                return true;
            }

            $_page = mb_substr(
                $page,
                0,
                mb_strpos($page . '?', '?')
            );
            if (in_array($_page, $whitelist)) {
                return true;
            }

            $_page = urldecode($page);
            $_page = mb_substr(
                $_page,
                0,
                mb_strpos($_page . '?', '?')
            );
            if (in_array($_page, $whitelist)) {
                return true;
            }
            echo "you can't see it";
            return false;
        }
    }

    if (! empty($_REQUEST['file'])
        && is_string($_REQUEST['file'])
        && emmm::checkFile($_REQUEST['file'])
    ) {
        include $_REQUEST['file'];
        exit;
    } else {
        echo "<br><img src=\"https://i.loli.net/2018/11/01/5bdb0d93dc794.jpg\" />";
    }  
?>
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

访问白名单出现的hint.php

 ![](https://img-blog.csdnimg.cn/20210822194859656.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

3.构造payload 

> 既要保证checkfile返回true，又要将file传入的include语句中，我们从第三个if着手
>
> 第三个 if 语句，截取传进参数中首次出现`?`之前的部分，判断该部分是否存在于`$whitelist`数组中，存在则返回 true

payload:?file=hint.php?**/../../../../ffffllllaaaagggg**

**4.解释后面的构造---重点**

给出官方对include语句的说明

![](https://img-blog.csdnimg.cn/20210822195315928.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

 当我们使用../访问文件时，<font color='red'>/将之前的hint.php?给忽略了</font>，只看后面定义的路径。至于用四次../，可以逐渐增加猜测，也可根据ffffllllaaaagggg形式猜测这是四次。

从而访问拿到flag

![](https://img-blog.csdnimg.cn/20210822195439368.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMxNzEwMzMx,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)
