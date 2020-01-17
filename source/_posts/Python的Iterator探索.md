---
title: Python的Iterator探索
date: 2020-01-17 19:18:19
categories: Python
tags: Python
---
# 前言
用py读取csv，并进行geohash的解码，我对一行代码的原理产生了困惑，本文简要记录当时的所思所想。
<!--more-->

今晚用py读取csv，想要进行`geohash`的解码，但是在一行代码上产生了疑惑，先来看代码：

``` python
import geohash
import csv
# 读取csv至字典
csvFile = open("/home/xcq/文档/MOBIKE_CUP_2017/train.csv", "r")#以只读方式打开
reader = csv.reader(csvFile)
# 建立空字典
geohashed_start_loc = {}
geohashed_end_loc   = {}

for item in reader:
	#忽略第一行     
	if reader.line_num == 1:
		contiune     
	geohashed_start_loc[reader.line_num] = item[5]

csvFile.close()
print(geohashed_start_loc)
```

"忽略第一行"这个操作是在网上看来的，首先它确实行之有效，但我疑惑的是:**为什么是reader.line_num而不是item.line_num？**因为在我的印象里，in后面的东西是不会变的，比如：

``` python
for letter in 'Python':  
	print( '当前字母 :', letter)
```

这一段代码会打印一个一个Python的每个字母，可是"Python"这个字符串始终不应该改变。

因此我觉得在读csv的代码中，`item`应该会依次复制`reader`里的每一个单位，每一个单位都具有一个`line_num`属性来表示当前的行数，所以，使用`item.line_num`来过滤第一行应该更加合理。但是，当我将`reader``改成item`的时候，却报错了：`item`并不具有`line_num`这个属性！我更疑惑了，**这个for_in循环，到底做了什么？**

我首先google这个循环到底是干什么的，得到了如下的合理说明：`for in`语句是一个`语法糖`，具体是这样实现的：

- 调用关键字in后对象的\_\_iter\_\_方法，该方法会返回一个迭代器
- 调用迭代器的\_\_next\_\_方法，返回迭代器中的下一个元素
- 如果没有下一个元素，会抛出异常，for语句会捕获这个异常并终止循环

这些说明让我更糊涂了，**迭代器，next，iter这些都是什么？**我并没有在之前的学习中好好了解这些，所以我决定继续研究。我继续搜寻了好多资料，却始终得不到好的解释，最后还是看的官方文档（官方文档写的真好），在此，我也截取这段文档：

>### Iterator Types
>
>Python supports a concept of iteration over containers. This is implemented using two distinct methods; these are used to allow user-defined classes to support iteration. Sequences, described below in more detail, always support the iteration methods.
>
>One method needs to be defined for container objects to provide iteration support:
>
>- container.**__iter__**()
>
>  Return an iterator object. The object is required to support the iterator protocol described below. If a container supports different types of iteration, additional methods can be provided to specifically request iterators for those iteration types. (An example of an object supporting multiple forms of iteration would be a tree structure which supports both breadth-first and depth-first traversal.) This method corresponds to the tp_iter slot of the type structure for Python objects in the Python/C API.
>
>The iterator objects themselves are required to support the following two methods, which together form the iterator protocol:
>
>- iterator.**__iter__**()
>
>  Return the iterator object itself. This is required to allow both containers and iterators to be used with the for and in statements. This method corresponds to the tp_iter slot of the type structure for Python objects in the Python/C API.
>
>- iterator.**__next__**()
>
>  Return the next item from the container. If there are no further items, raise the StopIteration exception. This method corresponds to the tp_iternext slot of the type structure for Python objects in the Python/C API.
>
>Python defines several iterator objects to support iteration over general and specific sequence types, dictionaries, and other more specialized forms. The specific types are not important beyond their implementation of the iterator protocol.
>
>Once an iterator’s __next__() method raises StopIteration, it must continue to do so on subsequent calls. Implementations that do not obey this property are deemed broken.
>
>### Generator Types
>
>Python’s generators provide a convenient way to implement the iterator protocol. If a container object’s __iter__()method is implemented as a generator, it will automatically return an iterator object (technically, a generator object) supplying the __iter__() and __next__() methods. More information about generators can be found in the documentation for the yield expression.

​     由文档可以很明显得到如下结论：

- `iter()`会返回一个迭代器（如果传入的参数不是迭代器，但是是数组，字符串，元组等容器，则会返回容器的迭代器对象；如果传入的是一个迭代器，则会返回迭代器本身）
- `next()`会返回迭代器的下一个项目（item），下一个，下一个 ，下一个！

既然这样，我似乎明白了什么，我在建立空字典之前`print`了`reader.line_num`，结果是0，我在for循环里也加了`print(reader.line_num)`，发现确实每一轮循环都使`line_num+1`。我还在代码中穿插`print(reader)`，发现打印出来的始终是同一个地址。我心里还是有一点想不明白，就想去阅读csv的原码，然而，花了好多功夫，才发现，`csv.py`有很大一部分是依赖c实现的，也就是说`csv.py`引用了好多`csv.c`的代码，读起来很费力。静下心来认真看了看，确实有发现类似`self.line_num++`的句子。我便大胆作出**假设**：

- `csv.reader()`返回的是一个迭代器，这个迭代器包含有`line_num`这个属性，初值为0（初值为零，方便大众认知，因为现实世界的标号是从1开始，而非0）
- 第一次做`for in`循环的时候，会执行`next(reader)`，将迭代器中的第一个项目传给`item`，并将`line_num加1`，但是，迭代器中的项目只是csv文件某一行的内容，不会有行号信息
- 接下来每轮循环，`reader`的`line_num`都会++，且会按顺序返回迭代器中的项目

​     到此为止，似乎可以很好的解释上面提出的问题了，在解决问题的过程中我也学会了很多没写在这篇笔记中的知识。但是，我又有了新的问题：**python是没有指针的，迭代器执行next()时，是如何判断要返回的是第几个项目呢**？这个问题留给日后再考虑吧。



部分有价值文献：

- <https://docs.python.org/3/library/stdtypes.html#iterator-types>
- <https://www.zhihu.com/question/24868112>
- <https://www.cnblogs.com/huxi/archive/2011/07/01/2095931.html>
- <https://stackoverflow.com/questions/12959968/what-is-csv-in-python>





2019.3.8