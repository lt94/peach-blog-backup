---
title: 再谈 Python 的写文件操作
tag: [Python]
comments: true
date: 2019-02-28
---







# 前言

关于 Python 对文件的读写操作网上有很多,基本上无外乎都是基于打开文件的 mode 的解释

![](http://ww4.sinaimg.cn/large/d9e82fa4jw1f9ndo67e2sj20fu0dlmzh.jpg)

昨天我媳妇有这样一个需求,读入一个文件如果文件不存在则创建,然后写入,如果文件存在,则读取上次的文件内容,然后添加上这次的内容,是不是很简单,直接用**a或者a+**就可以了. 别着急,重点来了,**文件内容是json数据,追加之后也应该是一个json数据**

说的可能比较抽象,那就举个例子

# 例子
两次写入文件内容,分别为 *{'1':'a'}* 和 *{'2':'b'}*,那么第一次写入,文件不存在,创建并写入,最终结果应该如下:

```json
{
  '1':'a'
}
```

第二次,文件存在,读取数据,合并数据重写写入,结果如下:

```json
{
  '1':'a',
  '2':'b'
}
```

很显然简单的使用 a 或者 a+ 模式是做不到上述要求的,有兴趣的可以自己尝试一下,它们只是简单的追加内容到上次文件末尾

# 实践

针对上述例子的要求,最容易想到的方案就是:
	1.先以 r 的模式,打开文件读取文件数据,得到 dictA
	2.然后以 w 的模式,打开文件,将本次数据 dictB 与 dictA 合并后写入 dictA

首先肯定这种方案没有任何问题,能够解决问题,不过这显然不是最佳方案,因为多次打开文件,增加了 IO 的开销,那有没有只打开一次文件完成上述要求呢?显然是有的,再说解决方案之前,我们先说一下预备知识:**为什么 a 模式下可以追加内容到文件末尾?** 因为在 a 模式下,打开文件后,**文件指针**默认指向文件末尾.ok了,新的方案就这么出来了.

新的方案基于文件指针,我们以 a 或者 a+ 模式打开文件,将文件指针移动到文件开头,读取文件内容,然后合并数据,然后清除文件内容,重新写入合并后的数据.最后看一下代码

```python
import json
import os
new_dict = {'c':'d'}
old_dict = {}
with open('test.json','a+') as f:
    if os.path.getsize('test.json'):
        f.seek(0) # 重点,移动指针
        old_dict = json.load(f)
    merge_dict = {**old_dict,**new_dict}
    f.truncate(0) # 清空之前内容
    json.dump(merge_dict,f)
```

