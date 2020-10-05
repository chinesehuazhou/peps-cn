# PEP 3142 在生成器中添加 while

**PEP 原文：** https://www.python.org/dev/peps/pep-3142/

**PEP 标题：** PEP 3142 -- Add a "while" clause to generator expressions

**PEP 作者：** Gerald Britton <gerald.britton at gmail.com>

**创建日期：** 2009-01-12

**合入版本：** 已被 Rejected

**译者：** [skyleaworlder](https://github.com/skyleaworlder)

## 摘要

这份 `PEP` 主要为通过添加 `while` 语句作为目前 `if` 语句在 **生成器** 中的功能的补充。

## 原理

在 `PEP 289` 中，**生成器** 以 "简洁" 的途径，利用动态生成的对象进行列表生成。目前，生成器使用 `if` 来筛选那些满足条件的对象们。然而，`if` 语句需要对每个对象都进行计算，因为每个对象都有可能因满足条件而被返回。在某些情况下，推导进行到 **某个节点** 时，后续的对象便都不需要考虑了。

比如：

```python
g = (n for n in range(100) if n*n < 50)
```

上述式子等价于使用一个生成器函数（`PEP 255`）：

```python
def __gen(exp):
    for n in exp:
        if n*n < 50:
            yield n
g = __gen(iter(range(10)))
```

这个函数会产生 0，1，2，3，4，5，6，7。但是它同样需要考察 8 到 99 这些数字。如果 `while` 语句可以作用在生成器当中的话，那么它就可以裁去那些冗余的尝试性运算了：

```python
g = (n for n in range(100) while n*n < 50)
```

这个式子会产生 0，1，2，3，4，5，6，7。但是它在 8 的时候就因不满足 `n*n < 50` 的条件而终止了。这等价于这样一个生成器函数：

```python
def __gen(exp):
    for n in exp:
        if n*n < 50:
            yield n
        else:
            break
g = __gen(iter(range(100)))
```

现在，为了得到更好的结果，我们要么编写一个上面那样的生成器函数，要么就使用 `itertools` 里面的 `takewhile` 函数。

```python
from itertools import takewhile
g = takewhile(lambda n: n*n < 50, range(100))
```

这段代码看起来挺常用的（但就是太难看了），但是它做到了上述的语法。除此之外，`takewhile` 需要另外一个匿名函数（上面例子里面写了），这无疑带来了性能上的损失。如下测试：

```python
for n in (n for n in range(100) if 1): pass
```

大概优于下面这段代码 10%：

```python
for n in takewhile(lambda n: 1, range(100)): pass
```

虽然我们获得了相同的结果（第一个例子使用了生成器，而 `takewhile`  是一个 `iterator`）如果仿照 `if` 实现的话，`while` 语句会和现在的 `if` 一样。

读者可能会问了，`if` 和 `while` 语句是不是互斥了，它们两个是不是不能同时作用。通过下面这个例子，我们可以发现，`if` 和 `while` 可以很好地合作：

```python
p = (p for p in primes() if p > 100 while p < 1000)
```

假设 `primes()` 这个函数可以生成素数，那么这个生成器会返回 100 到 1000 之间的素数。

使用 `while` 语句可以保持 “**缩行**” 风格，同时还可以添加一个十分有用的 **短路条件**。

## 致谢

`Raymond Hettinger` 于 2002 年 1 月首次提出了生成器表达式的概念。

## 相关链接

[1] "Generator Expressions" http://www.python.org/dev/peps/pep-0289/

[2] "List Comprehensions" http://www.python.org/dev/peps/pep-0202/

[3] "Simple Generators" http://www.python.org/dev/peps/pep-0255/

## 版权

本文档已进入公共领域。

源文档：https://github.com/python/peps/blob/master/pep-3142.txt