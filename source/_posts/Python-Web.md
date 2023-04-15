---
title: Python-Web
date: 2021-10-26 13:39:32
updated: 2023-04-11 13:39:32
categories: 编程
tags: [Python]
toc: true
---
python和web相关库使用，爬虫之类随笔

<!-- more -->
## 一、python爬虫基础

常用方法:

```python
import requests
r = requests.get(url)
r.text #字符串数据
r.encoding #编码方式
r.content #获得的字节数据,用.decode()解码成字符串
r.status_code #返回状态
r.headers #响应头
r.request.*** #请求的数据,例如headers
```

#### 1.发送带header的请求

header的形式：字典

```python
headers = {'User-Agent':'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/95.0.4638.54 Safari/537.36 Edg/95.0.1020.30',
}
r = requests.get("http://www.baidu.com",headers=headers)
```

#### 2.发送带参数的请求

参数的形式：字典

```python
payload = {'wd':'CTF'}
r = requests.get("http://www.baidu.com/s?",params=payload,headers=headers)
```

#### 3.发送POST请求

data的形式：字典

```python
r = requests.post(url,data=data,headers=headers)
```

























