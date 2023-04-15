---
title: XCTF-lottery
date: 2021-10-25 19:45:14
updated: 2021-10-25 19:45:14
categories: XCTF
tags: [代码审计]
toc: true
---
XCTF-lottery

<!-- more -->
#####  1.打开网页，注册账号登录，发现猜数字可以获得奖金，足够多的奖金可以购买flag![](https://i.loli.net/2021/10/25/l9LZ4JH6t5kAuQS.png)

![](https://i.loli.net/2021/10/25/VI2fFLs8Cx4X91j.png)

##### 2.目录扫描工具发现robots.txt，访问看到有`git泄露`

![](https://i.loli.net/2021/10/25/lhsP2yL3dHC17Gx.png)

![](https://i.loli.net/2021/10/25/Ywur72VqfvU8415.png)

##### 3.利用githack获取git库相关文件信息

![](https://i.loli.net/2021/10/25/KIAOV9yB3alxH2N.png)

在`api.php`里找到了购买彩票的函数：

```php
function buy($req){
	require_registered();
	require_min_money(2);

	$money = $_SESSION['money'];
	$numbers = $req['numbers'];
	$win_numbers = random_win_nums();
	$same_count = 0;
    //核心代码
	for($i=0; $i<7; $i++){
		if($numbers[$i] == $win_numbers[$i]){
			$same_count++;
		}
	}
	switch ($same_count) {
		case 2:
			$prize = 5;
			break;
		case 3:
			$prize = 20;
			break;
		case 4:
			$prize = 300;
			break;
		case 5:
			$prize = 1800;
			break;
		case 6:
			$prize = 200000;
			break;
		case 7:
			$prize = 5000000;
			break;
		default:
			$prize = 0;
			break;
	}
	$money += $prize - 2;
	$_SESSION['money'] = $money;
	response(['status'=>'ok','numbers'=>$numbers, 'win_numbers'=>$win_numbers, 'money'=>$money, 'prize'=>$prize]);
}
```

在比较彩票的过程中使用了弱比较`==`,并且是一位位的比较，json传递的数据中可以包含布尔类型，因此可以抓包构造全是`true`的数组，即可获得大量奖金

##### 4.抓包，构造payload

![](https://i.loli.net/2021/10/25/ep1bgHzD9tjEqLI.png)

##### 5.多次提交并购买flag

![](https://i.loli.net/2021/10/25/7AVSxWgFHoz5bGB.png)





