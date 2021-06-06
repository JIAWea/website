---
title: Python进阶用法
date: 2019-10-14 10:58:14
categories: 后端
tags: Python
---

使用Python来编写也有很长一段时间了，也想着如何优化自己的代码，随之也搜了一些问题。
其中印象比较深刻的就是stackoverflow上的一个问题解答了。

<!--more-->

原文：[Hidden features of Python](https://stackoverflow.com/questions/101268/hidden-features-of-python)


**Argument Unpacking**
可以使用 * 和 ** 分别将一个列表和一个字典解包为函数参数
如：
```
def draw_point(x, y):
	# do something


tuple = (6, 8)
dir = {'x': 6, 'y': 8}

draw_point(*tuple)
draw_point(**dir)
```


**Decorators**
装饰器的作用就是在不需要修改原函数代码的前提下增加新的功能，在调用原函数的时候先执行装饰器，如：
```
def print_before_func(func):
	def wrapper(*args, **kwargs):
		print("print before func")
		return func(*args, **kwargs)
	return wrapper


@print_befor_func
def write(text):
	print(text)


write(Hello world)

结果：
print before func
Hello world
```


**Dictionary default .get value**
字典中有一个`get()`方法。如果你使用`dict['key']`的方式而`key`不存在的话就会出现异常。使用`dict.get('key')`的话如果`key`不存在则只会返回`None`。当然`get()`方法提供了第二个参数，如果返回`None`则会返回第二个参数的值。
```
num = dict.get('num', 0)
```

**Enumeration**
使用enumeration包住一个可迭代对象，它会将index和item绑在一起，返回一个enumeration对象，如：
```
a = ['a', 'b', 'c', 'd']
for index, item in enumeration(a):
	print(index, item)


···
0 a
1 b
2 c
3 d
···

```

在同时需要索引和值的时候很有用


**For/else**
语法如下：
```
for i in foo:
	if i == 0:
		break
else:
	print("it was never 0")
```

`else`代码块会在循环正常结束后执行，也就是说没用出现`break`时才会调用`else`代码块，上面代码等价于:
```
found = False
for i in foo:
	if i == 0:
		found = True
		break

if not found:
	print("it was never 0")
```

这个语法挺容易让人混淆的，所有在使用的时候最好是注释一下，以免被其他小伙伴搞错含义了。

上面的代码也等价于：
```
if any(i == 0 for i in foo):
	pass
else:
	print("it was never 0")
```

**Generator expressions**
假如你这样写：
```
x = (n for n in foo if bar(n))
```
你将会得到一个生成器，并可以把它付给一个变量x。现在你可以像下面一样使用生成器：
```
for n in x:
	print(n)
```

这样做的好处就是节省内存了。你不需要中间存储，而如果像下面这样的话，则需要：
```
x = [n for n in foo if bar(n)]
# 列表推导
```

在某些情况下，这会导致极重要的速度提升。
你可以添加许多if语句到生成器的尾端，基本复制for循环嵌套：
```
>>> n = ((a,b) for a in range(0,2) for b in range(4,6))
>>> for i in n:
...   print i 

(0, 4)
(0, 5)
(1, 4)
(1, 5)
```

使用生成器最大的好处就是节省内存了。因为每一个值只有在你需要的时候才会生成，而不像列表推导那样一次性生成所有的结果。


**List stepping**
切片操作符中的步长(step)参数。例如：
```
a = [1,2,3,4,5]
>>> a[::2]  		# iterate over the whole list in 2-increments
[1,3,5]
```

特殊例子`x[::-1]`对‘x反转’来说相当有用。
```
>>> a[::-1]
[5,4,3,2,1]
```

当然，你可以使用`reversed()`函数来实现反转。
区别在于，reversed()返回一个迭代器，所以还需要一个额外的步骤来将结果转换成需要的对象类型。
这个特性在判断例如回文的时候灰常有用，一句话搞定
`True if someseq == someseq[::-1] else False`


**Named string formatting**
%-格式化接收一个字典（也适用于%i%s等）
```
>>> print "The %(foo)s is %(bar)i." % {'foo': 'answer', 'bar':42}
The answer is 42.
```

**try/except/else**
else语句块只有当try语句正常执行（也就是说，except语句未执行）的时候，才会执行。
finally则无论是否异常都会执行。
```
try:
     Normal execution block
except A:
     Exception A handle
except B:
     Exception B handle
except:
     Other exception handle
else:
     if no exception,get here
finally:
     this block will be excuted no matter how it goes above
```
