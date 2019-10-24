# [译]PEP 3099--Python 3 中不会改变的事情

**导语：** Python 3.8 已经发布了，引进了不少变更点。关于 3.9 预计引入的修改，也披露了一些。我们之前还关注过 [GIL 的移除计划](https://mp.weixin.qq.com/s/8KvQemz0SWq2hw-2aBPv2Q) 和 [Guido 正在开发的新解析器](https://github.com/chinesehuazhou/guido_blog_translation) 等话题，这意味 Python 很有活力，仍在健康地发展着。

Python 3 是比较大胆激进的，抛弃了前一版本的很多陈旧的包袱，但同时，它也是相对克制的（一直如此），社区里提出的很多提议都被否决了。前不久，我分享的[Python 设计和历史的 27 个问题](https://mp.weixin.qq.com/s/zabIvt4dfu_rf7SmGZXqXg) 提到了一些沉淀下来的设计问题，今天这篇译文则聚焦于官方明确否决掉的 24 个设计问题。

大部分问题都是细枝末节（例如大小写、括号、反引号、行宽等等），但细究起来的话，也会挺有意思，欢迎留言讨论。

------

**PEP原文：** https://www.python.org/dev/peps/pep-0399
**PEP标题：** Things that will Not Change in Python 3000
**PEP作者：** Georg Brandl
**创建日期：** 2006-04-04
**译者**：[豌豆花下猫](https://zhuanlan.zhihu.com/pythonCat)（**Python猫** 公众号作者）
**翻译计划**：https://github.com/chinesehuazhou/peps-cn

--------------------

## 目录

- 概要
- 语言核心
- 内置对象
- 标准类型
- 编码风格
- 交互式解释器
- 版权

## 概要

有些想法很糟糕。尽管关于 Python 演化的某些想法具有建设性，但不少想法却与 Python 的基本原则背道而驰，就像是让人围着圈跑步：哪也去不了，即使 Python  3000 允许提出特别的建议，也不管用。

此 PEP 尝试列出 Python 3000 上所有的 BDFL 决断，涉及那些不会发生更改的内容，以及不会引入的新功能，按主题排序，附加简短的说明或者 python-3000 邮件列表的相关线索。

如果你认为应该实现下面所列举的提议，那你最好立即离开计算机，出门去，尽情娱乐。去户外活动，在草地上打盹，都比提出一个“殴打一匹死马”的想法来得有建设性。这算是对你的警告（译注：下面的主意是不会实现的，不要试图改变这些已经是板上钉钉的决断）。

## 语言核心

- Python 3000 不会区分大小写。


- Python 3000 不会从头开始重写。

> 不会使用 C++ 或其它不同于 C 的语言作为实现语言。但是，代码库将逐渐迁移。Joel Spolsky 有一篇出色的文章解释了原因：[http://www.joelonsoftware.com/articles/fog0000000069.html](http://www.joelonsoftware.com/articles/fog0000000069.html)

-  self 不会变成隐式的。 


> 使用显式的 self 是一个好事。消除解析变量时的歧义，可以使得代码更清晰。这还使得函数和方法之间的差异变小。   
>
> 邮件：“提议草案：Python 3.0 使用隐式的 self” [https://mail.python.org/pipermail/python-dev/2006-January/059468.html](https://mail.python.org/pipermail/python-dev/2006-January/059468.html)

-  lambda 不会被改名。 


> 曾经有过提议，在 Python 3000 中移除 lambda。然而，没人能够提出更好的提供匿名函数的方法。所以 lambda 保留了下来。
>
> 但只是说要保持原样。（有人提议）增加它对语句的支持，但这不是一个好想法。因为它需要允许多行 lambda 表达式，这意味着多行表达式可能突然出现，例如，将会允许对函数调用使用多行参数。那真是丑陋。
>
> 邮件：“genexp 语法/lambda”，[https://mail.python.org/pipermail/python-3000/2006-April/001042.html](https://mail.python.org/pipermail/python-3000/2006-April/001042.html) 

-  Python 不会添加可编程的语法。


> 邮件：“是一个声明！是一个函数！两者皆是！”
> [https://mail.python.org/pipermail/python-3000/2006-April/000286.html](https://mail.python.org/pipermail/python-3000/2006-April/000286.html) 

-  不会有类似 zip() 的并行迭代语法。 


> 邮件：“并行迭代语法”，[https://mail.python.org/pipermail/python-3000/2006-March/000210.html](https://mail.python.org/pipermail/python-3000/2006-March/000210.html) 

-  字符串将保持可迭代。


> 邮件：“使字符串不可迭代”，[https://mail.python.org/pipermail/python-3000/2006-April/000759.html ](https://mail.python.org/pipermail/python-3000/2006-April/000759.html) 

-  不会有对生成器表达式或列表推导式的结果进行排序的语法。sorted() 将涵盖所有的使用情况。  


> 邮件：“为生成器表达式添加排序”，[https://mail.python.org/pipermail/python-3000/2006-April/001295.html](https://mail.python.org/pipermail/python-3000/2006-April/001295.html)

-  切片和扩展型切片不会取消（即使\_\_getslice\_\_和\_\_setslice\_\_ API 可能被替换），它们也不会返回标准对象类型的视图。

> 邮件：切片的未来[https://mail.python.org/pipermail/python-3000/2006-May/001563.html](https://mail.python.org/pipermail/python-3000/2006-May/001563.html) 

-  不会禁止在循环结构内重用循环变量。

> 邮件：消除迭代变量的作用域出血(scope bleeding)[https://mail.python.org/pipermail/python-dev/2006-May/064761.html](https://mail.python.org/pipermail/python-dev/2006-May/064761.html)

-  解析器不会比 LL(1) 更复杂。

> 简单胜于复杂。这个想法适用于解析器。将 Python 的语法限制为 LL(1) 解析器是一种好处，而不是诅咒。它使我们带上手铐，不至于发展过度，不至于最终得到些时髦的语法规则，像一些走向无名的动态语言那样，例如 Perl。

-  不会有括号。

> 这太明显了，以至于不需要引用邮件列表。使用`from __future__ import braces` ，你就会得到关于这个问题的明确答案。  

-  不会再有反引号。

> 反引号（`）将不再用作 repr 的简写——但这并不意味着它们可用于其它用途。即使忽略向后兼容性的混乱，这字符本身也会引起太多问题（在某些字体、某些键盘上、在排版书籍时，等等）。
>
> 邮件：“使用反引号作为新运算符”，[https://mail.python.org/pipermail/python-ideas/2007-January/000054.html](https://mail.python.org/pipermail/python-ideas/2007-January/000054.html)

-  引用全局变量名 foo 不会被拼写为 globals.foo。global 语句会保留     

> 邮件：“用全局内置对象替换 globals() 和 global 语句”，[https://mail.python.org/pipermail/python-3000/2006-July/002485.html](https://mail.python.org/pipermail/python-3000/2006-July/002485.html) ，“显式词法作用域（pre-PEP?） ”，[https://mail.python.org/pipermail/python-dev/2006-July/067111.html](https://mail.python.org/pipermail/python-dev/2006-July/067111.html) 

-  不会有替代的绑定操作符，例如 := 。 

> 邮件：“显式词法作用域（pre-PEP?）”，[https://mail.python.org/pipermail/python-dev/2006-July/066995.html](https://mail.python.org/pipermail/python-dev/2006-July/066995.html)

-  我们不会删除容器字面量。也就是说，{expr:expr, ...}，[expr, ...] 和(expr, ...)将保留。

> 邮件：“去除容器字面量”，[https://mail.python.org/pipermail/python-3000/2006-July/002550.html](https://mail.python.org/pipermail/python-3000/2006-July/002550.html)：

-  while 和 for 循环中的 else 子句不会更改语义，也不会被删除。      

> 邮件：“ for/except/else 语法” [https://mail.python.org/pipermail/python-ideas/2009-October/006083.html](https://mail.python.org/pipermail/python-ideas/2009-October/006083.html) 

## 内置对象

-  zip() 不会增加关键字参数或其它机制来防止它在最短序列的末尾停止。 

> 邮件：“对于不同长度的序列，令 zip() 引发异常”，[https://mail.python.org/pipermail/python-3000/2006-August/003338.html](https://mail.python.org/pipermail/python-3000/2006-August/003338.html)

-  hash() 不会成为属性，因为属性应该是易于计算的，但哈希并不一定是这种情况。 

> 邮件：“哈希作为属性/特性”，[https://mail.python.org/pipermail/python-3000/2006-April/000362.html](https://mail.python.org/pipermail/python-3000/2006-April/000362.html) 

## 标准类型

-  遍历字典将继续返回 key。

> 邮件：“遍历字典”，[https://mail.python.org/pipermail/python-3000/2006-April/000283.html](https://mail.python.org/pipermail/python-3000/2006-April/000283.html)
>
> 邮件：让 iter(mapping) 生成 (key, value) 对[https://mail.python.org/pipermail/python-3000/2006-June/002368.html](https://mail.python.org/pipermail/python-3000/2006-June/002368.html)

-  不会有 `frozenlist` 类型。  

> 邮件：“不可变的列表”，[https://mail.python.org/pipermail/python-3000/2006-May/002219.html](https://mail.python.org/pipermail/python-3000/2006-May/002219.html)

-  int 不会支持下标来生成 range。 

> 邮件：“ xrange vs.int .\_\_ getslice\_\_”，[https://mail.python.org/pipermail/python-3000/2006-June/002450.html](https://mail.python.org/pipermail/python-3000/2006-June/002450.html)  

## 编码风格

-  对于 C 和 Python 代码，（推荐的）最大行宽将保持 80 个字符。


> 邮件：“ C 风格指南”，[https://mail.python.org/pipermail/python-3000/2006-March/000131.html](https://mail.python.org/pipermail/python-3000/2006-March/000131.html) 

## 交互式解释器

-  解释器提示（>>>）不会改变。它给 Guido 带来温暖的模糊感觉。


> 邮件：“低垂的果实：更改解释器的提示？”，[https://mail.python.org/pipermail/python-3000/2006-November/004891.html](https://mail.python.org/pipermail/python-3000/2006-November/004891.html)

## 版权

本文档已放置在公共领域。源文档：
[https://github.com/python/peps/blob/master/pep-3099.txt](https://github.com/python/peps/blob/master/pep-3099.txt)

## 译者注

文中出现的“邮件”，原文是“thread”，英文解释应该是：a series of connected messages on a message board on the Internet which have been sent by different people。实际指的是“邮件列表中的议题”，为简洁起见，省译为“邮件”。



