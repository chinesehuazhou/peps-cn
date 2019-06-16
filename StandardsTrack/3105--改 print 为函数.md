# [译] PEP 3105--改 print 为函数

**PEP原文 ：** [https://www.python.org/dev/peps/pep-3105/](https://www.python.org/dev/peps/pep-3105/)

**PEP标题：** Make print a function

**PEP作者：** Georg Brandl

**创建日期：** 2006-11-19

**合入版本：** 3.0

**译者** ：[豌豆花下猫](https://zhuanlan.zhihu.com/pythonCat)（**Python猫** 公众号作者）

**PEP翻译计划** ：https://github.com/chinesehuazhou/peps-cn

## 摘要

标题已说明了一切——本 PEP 提议使用新的内置函数 print() 来替代 print 语句，并建议给此新函数使用特殊的签名（signature ）。

## 原理阐述

`print 语句` 早就被列在了不可靠的语言特性列表中，例如 Guido 的“Python 之悔”（Python Regrets）演讲【1】，并计划在 Python 3000 版本移除。因此，本 PEP 的目的并不新鲜，尽管它可能会在 Python 开发人员中引起较大争议。

以下对 print() 函数的争议是提取自 Guido 本人的 Python-3000 消息【2】：

-  print 是唯一的应用程序级功能，并拥有专属的语句。在 Python 的世界里，当某些任务在不通过编译器的帮助就无法完成的情况下，语法（syntax）通常会被用作最后的手段。在这种异常情况下，print 并不合适。
-  在开发应用程序的时候，人们经常需要用更复杂的东西来代替 print 输出，例如调用 logging，或者调用其它的 I/O 库。至于 print() 函数，这是个直截了当的字符替换，如今它混搭了所有那些括号，还可能会转换 >>stream 样式的语法。
-  为 print 设置特殊的语法只会给进化带来一个更加巨大的屏障，例如这有个猜想，一个新的 printf() 函数不用多久就会出现，跟 print() 函数共存。
-  当需要一个不同的分隔符（不是空格，或者没有分隔符）时，没有简单的方法可以将 print 语句转换成另一个调用。同样地，使用其它一些分隔符而非空格时，根本无法方便地打印对象。
-  如果 print() 是个函数，就可以非常容易地在一个模块内替换它（仅需 def print(*args):...），甚至可以在整个程序内替换（例如放一个不同的方法进 \_\_builtin\_\_.print）。实际上，要做到这点，还可以写一个带 write() 方法的类，然后定向给 `sys.stdout` ，这想法不错，但无疑是一个非常巨大的概念飞跃，而且跟 print 相比，它工作在不同的层级。

## 设计规格

print() 的书写方式取自各种邮件，最近发布在 python-3000 列表里的是【3】：

```
def print(*args, sep=' ', end='\n', file=None)
```

调用像：

```
print(a, b, c, file=sys.stderr)
```

相当于当前的：

```
print >>sys.stderr, a, b, c
```

可选的 sep 与 end 参数相应地指定了每个打印参数之间及之后的内容。

`softspace` 功能（当前在文件上的半秘密属性，用于告诉 print 是否要在第一个条目前插入空格）会被删除。因此，当前版本的以下写法不能被直接转换：

```
print "a",
print
```

它不会在“a”与换行符之间打印一个空格。

（译注：在 3.3 版本，print() 函数又做了改动，增加了默认参数 flush=False）

## 向后兼容性

本 PEP 中提出的改动将致使如今的 print 语句失效。只有那些恰好用括号包围了所有参数的写法才能在 Python 3 版本中生效，至于其它，只有加上了括号的值才能保持原样打印。例如，在 2.x 中：

```
>>> print ("Hello")
Hello
>>> print ("Hello", "world")
('Hello', 'world')
```

而在 3.0 中：

```
>>> print ("Hello")
Hello
>>> print ("Hello", "world")
Hello world
```

幸运的是，因为 print 是 Python 2 中的一个语句，所以它可以被通过自动化工具而检测到，并可靠而精确地替换掉，因此应该没有重大的移植问题（如果有人来写这个工具的话）。

## 实现

更改将在 Python 3000 分支中实现（修订版从 53685 到 53704）。大多数在维库代码（legacy code）已经做转换了，但要抓出发行版本中的每个 print 语句，还需要持续不断地努力。

## 参考资料

[1] http://legacy.python.org/doc/essays/ppt/regrets/PythonRegrets.pdf

[2]Python 3.0 替换 print（Guido van Rossum）

https://mail.python.org/pipermail/python-dev/2005-September/056154.html

[3] py3k 中 print() 的参数（Guido van Rossum）

https://mail.python.org/pipermail/python-3000/2006-November/004485.html

## 版权

本文档已经放置在公共领域。源文档：

[https://github.com/python/peps/blob/master/pep-3105.txt](https://github.com/python/peps/blob/master/pep-3105.txt)
