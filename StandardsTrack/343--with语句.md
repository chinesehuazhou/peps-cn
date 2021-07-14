# Python 的上下文管理器是怎么设计的？


**PEP原文 ：** [https://www.python.org/dev/peps/pep-0343](https://www.python.org/dev/peps/pep-0343/)

**PEP标题：** PEP 343 -- The "with" Statement

**PEP作者：** Guido van Rossum, Nick Coghlan

**创建日期：** 2005-05-13

**合入版本：** 2.5

**译者** ：豌豆花下猫@Python猫公众号

**PEP翻译计划** ：[https://github.com/chinesehuazhou/peps-cn](https://github.com/chinesehuazhou/peps-cn)

## 摘要

本 PEP 提议在 Python 中新增一种"with"语句，可以取代常规的 try/finally 语句。

在本 PEP 中，上下文管理器提供\_\_enter\_\_() 和 \_\_exit\_\_() 方法，在进入和退出 with 语句体时，这俩方法分别会被调用。

## 作者的批注

本 PEP 最初由 Guido 以第一人称编写，随后由 Nick Coghlan 根据 python-dev 上的讨论，做出了更新补充。所有第一人称的内容都出自于 Guido 的原文。

Python 的 alpha 版本发布周期暴露了本 PEP 以及相关文档和实现[14]中的术语问题。直到 Python 2.5 的第一个 beta 版本发布时，本 PEP 才稳定下来。

是的，本文某些地方的动词时态是混乱的。到现在为止，我们已经创作此 PEP 一年多了，所以，有些原本在未来的事情，现在已经成为过去了:)

## 介绍

经过对 PEP-340 及其替代方案的大量讨论后，我决定撤销 PEP-340，并提出了 PEP-310 的一个小变种。经过更多的讨论后，我又添加了一种机制，可以使用 throw() 方法，在挂起的生成器中抛出异常，或者用一个 close() 方法抛出一个 GeneratorExitexception；这些想法最初是在 python-dev [2] 上提出的，并得到了普遍的认可。我还将关键字改为了“with”。

（Python猫注：PEP-340 也是 Guido 写的，他最初用的关键字是“block”，后来改成了其它 PEP 提议的“with”。）

在本 PEP 被接受后，以下 PEP 由于重叠而被拒绝：

* PEP-310，可靠的获取/释放对。这是 with 语句的原始提案。
* PEP-319，Python 同步/异步代码块。通过提供合适的 with 语句控制器，本 PEP 可以涵盖它的使用场景：对于'synchronize'，我们可以使用示例 1 中的"locking"模板；对于'asynchronize'，我们可以使用类似的"unlock"模板。我认为不必要给代码块加上“匿名的”锁；事实上，应该尽可能地使用明确的互斥锁。

PEP-340 和 PEP-346 也与本 PEP 重叠，但当本 PEP 被提交时，它们就自行撤销了。

关于本 PEP 早期版本的一些讨论，可以在 Python Wiki[3] 上查看。

## 动机与摘要

PEP-340（即匿名的 block 语句）包含了许多强大的创意：使用生成器作为代码块模板、给生成器添加异常处理和终结，等等。除了赞扬之外，它还被很多人所反对，他们不喜欢它是一个（潜在的）循环结构。这意味着块语句中的 break 和 continue 可以中断或继续块语句，即使它原本被当作非循环的资源管理工具。

但是，直到我读了 Raymond Chen 对流量控制宏[1]的抨击时，PEP-340 才走入了末路。Raymond 令人信服地指出，在宏中藏有流程控制会让你的代码变得难以捉摸，我觉得他的论点不仅适用于 C，同样适用于 Python。我意识到，PEP-340 的模板可以隐藏各种控制流；例如，它的示例 4 （auto_retry()）捕获了异常，并将代码块重复三次。

然而，在我看来，PEP-310 的 with 语句并没有隐藏控制流：虽然 finally 代码部分会暂时挂起控制流，但到了最后，控制流会恢复，就好像 finally 子句根本不存在一样。

在 PEP-310 中，它大致提出了以下的语法（"VAR ="部分是可选的）：

```python
with VAR = EXPR:
    BLOCK
```

大致可以理解为：

```python
VAR = EXPR
VAR.__enter__()
try:
    BLOCK
finally:
    VAR.__exit__()
```

现在考虑这个例子：

```python
with f = open("/etc/passwd"):
    BLOCK1
BLOCK2
```

在上例中，第一行就像是一个“if True”，我们知道如果 BLOCK1 在执行时没有抛异常，那么 BLOCK2 将会被执行；如果 BLOCK1 抛出异常，或执行了非局部的 goto （即 break、continue 或 return），那么 BLOCK2 就不会被执行。也就是说，with 语句所加入的魔法并不会影响到这种流程逻辑。

（你可能会问，如果\_\_exit\_\_() 方法因为 bug 导致抛异常怎么办？那么一切都完了——但这并不比其他情况更糟；异常的本质就是，它们可能发生在**任何地方**，你只能接受这一点。即便你写的代码没有 bug，KeyboardInterrupt 异常仍然会导致程序在任意两个虚拟机操作码之间退出。）

这个论点几乎让我采纳了 PEP-310，但是， PEP-340 还有一个亮点让我不忍放弃：使用生成器作为某些抽象化行为的“模板”，例如获取及释放一个锁，或者打开及关闭一个文件，这是一种很强大的想法，通过该 PEP 的例子就能看得出来。

受到 Phillip Eby 对 PEP-340 的反提议（counter-proposal）的启发，我尝试创建一个装饰器，将合适的生成器转换为具有必要的\_\_enter\_\_() 和 \_\_exit\_\_() 方法的对象。我在这里遇到了一个障碍：虽然这对于锁的例子来说并不太难，但是对于打开文件的例子，却不可能做到这一点。我的想法是像这样定义模板：

```python
@contextmanager
def opening(filename):
    f = open(filename)
    try:
        yield f
    finally:
        f.close()
```

并这样使用它：

```python
with f = opening(filename):
    ...read data from f...
```

问题是在 PEP-310 中，EXPR 的调用结果直接分配给 VAR，然后 VAR 的\_\_exit\_\_() 方法会在 BLOCK1 退出时被调用。但是这里，VAR 显然需要接收打开的文件，这意味着\_\_exit\_\_() 必须是文件对象的一个方法。

虽然这可以使用代理类来解决，但会很别扭，同时我还意识到，只需做出一个小小的转变，就能轻轻松松地写出所需的装饰器：让 VAR 接收\_\_enter\_\_() 方法的调用结果，接着保存 EXPR 的值，以便最后调用它的\_\_exit\_\_() 方法。

然后，装饰器可以返回一个包装器的实例，其\_\_enter\_\_() 方法调用生成器的 next() 方法，并返回 next() 所返回的值；包装器实例的\_\_exit\_\_() 方法再次调用 next()，但期望它抛出 StopIteration。（详细信息见下文的生成器装饰器部分。）

因此，最后一个障碍便是 PEP-310 语法：

```python
with VAR = EXPR:
    BLOCK1
```

这是有欺骗性的，因为 VAR 不接收 EXPR 的值。借用 PEP-340 的语法，很容易改成：

```python
with EXPR as VAR:
    BLOCK1
```

在其他的讨论中，人们真的很喜欢能够“看到”生成器中的异常，尽管仅仅是为了记日志；生成器不允许产生（yield）其它的值，因为 with 语句不应该作为循环使用（引发不同的异常是勉强可以接受的）。

为了做到这点，我建议为生成器提供一个新的 throw() 方法，该方法以通常的方式接受 1 到 3 个参数（类型、值、回溯），表示一个异常，并在生成器挂起的地方抛出。

一旦我们有了这个，下一步就是添加另一个生成器方法 close()，它用一个特殊的异常（即 GeneratorExit）调用 throw()，可以令生成器退出。有了这个，在生成器被当作垃圾回收时，可以让程序自动调用 close()。

最后，我们可以允许在 try-finally 语句中使用 yield 语句，因为我们现在可以保证 finally 子句必定被执行。关于终结（finalization）的常见注意事项——进程可能会在没有终结任何对象的情况下突然被终止，而这些对象可能会因程序的周期或内存泄漏而永远存活（在 Python 的实现中，周期或内存泄漏会由 GC 妥善处理）。

请注意，在使用完生成器对象后，我们不保证会立即执行 finally 子句，尽管在 CPython 中是这样实现的。这类似于自动关闭文件：像 CPython 这样的引用计数型解释器，它会在最后一个引用消失时释放一个对象，而使用其他 GC 算法的解释器不保证也是如此。这指的是 Jython、IronPython，可能包括运行在 Parrot 上的 Python。

（关于对生成器所做的更改，可以在 PEP-342 中找到细节，而不是在当前 PEP 中。）

## 用例

请参阅文档末尾的示例部分。

## 规格说明：'with'语句

提出了一种新的语句，语法如下：

```python
with EXPR as VAR:
    BLOCK
```

在这里，“with”和“as”是新的关键字；EXPR 是任意一个表达式（但不是表达式列表），VAR 是一个单一的赋值目标。它不能是以逗号分隔的变量序列，但可以是以圆括号包裹的以逗号分隔的变量序列。（这个限制使得将来的语法扩展可以出现多个逗号分隔的资源，每个资源都有自己的可选 as 子句。）

“as VAR”部分是可选的。

上述语句可以被翻译为:

```python
mgr = (EXPR)
exit = type(mgr).__exit__  # Not calling it yet
value = type(mgr).__enter__(mgr)
exc = True
try:
    try:
        VAR = value  # Only if "as VAR" is present
        BLOCK
    except:
        # The exceptional case is handled here
        exc = False
        if not exit(mgr, *sys.exc_info()):
            raise
        # The exception is swallowed if exit() returns true
finally:
    # The normal and non-local-goto cases are handled here
    if exc:
        exit(mgr, None, None, None)
```

在这里，小写变量（mgr、exit、value、exc）是内部变量，用户不能访问；它们很可能是由特殊的寄存器或堆栈位置来实现。

上述详细的翻译旨在说明确切的语义。解释器会按照顺序查找相关的方法（\_\_exit\_\_、\_\_enter\_\_），如果没有找到，将引发 AttributeError。类似地，如果任何一个调用引发了异常，其效果与上述代码中的效果完全相同。

最后，如果 BLOCK 包含 break、continue 或 return 语句，\_\_exit\_\_() 方法就会被调用，带三个 None 参数，就跟 BLOCK 正常执行完成一样。（也就是说，\_\_exit\_\_() 不会将这些“伪异常”视为异常。）

如果语法中的"as VAR"部分被省略了，则翻译中的"VAR ="部分也要被忽略（但 mgr.\_\_enter\_\_() 仍然会被调用）。

mgr.\_\_exit\_\_() 的调用约定如下。如果 finally 子句是通过 BLOCK 的正常完成或通过非局部 goto（即 BLOCK 中的 break、continue 或 return 语句）到达，则使用三个 None 参数调用mgr.\_\_exit\_\_()。如果 finally 子句是通过 BLOCK 引发的异常到达，则使用异常的类型、值和回溯这三个参数调用 mgr.\_\_exit\_\_()。

重要：如果 mgr.\_\_exit\_\_() 返回“true”，则异常将被“吞灭”。也就是说，如果返回"true"，即便在 with 语句内部发生了异常，也会继续执行 with 语句之后的下一条语句。然而，如果 with 语句通过非局部 goto （break、continue 或 return）跳出，则这个非局部返回将被重置，不管 mgr.\_\_exit\_\_() 的返回值是什么。这个细节的动机是使 mgr.\_\_exit\_\_() 能够吞咽异常，而不使异常产生影响（因为默认的返回值 None为 false，这会导致异常被重新 raise）。吞下异常的主要用途是使编写 @contextmanager 装饰器成为可能，这样被装饰的生成器中的 try/except 代码块的行为就好像生成器的主体在 with-语句里内联展开了一样。

之所以将异常的细节传给\_\_exit\_\_()，而不用 PEP -310 中不带参数的\_\_exit\_\_()，原因是考虑到下面例子 3 的 transactional()。该示例会根据是否发生异常，从而决定提交或回滚事务。我们没有用一个 bool 标志区分是否发生异常，而是传了完整的异常信息，目的是可以记录异常日志。依赖于 sys.exc_info() 获取异常信息的提议被拒绝了；因为 sys.exc_info() 有着非常复杂的语义，它返回的异常信息完全有可能是很久之前就捕获的。有人还提议添加一个布尔值，用于区分是到达 BLOCK 结尾，还是非局部 goto。这因为过于复杂和不必要而被拒绝；对于数据库事务回滚，非局部 goto 应该被认为是正常的。

为了促进 Python 代码中上下文的链接作用，\_\_exit\_\_() 方法不应该继续 raise 传递给它的错误。在这种情况下，\_\_exit\_\_() 方法的调用者应该负责处理 raise。

这样，如果调用者想知道\_\_exit\_\_() 是否调用失败（而不是在传出原始错误之前就完成清理），它就可以自己判断。

如果\_\_exit\_\_() 没有返回错误，那么就可以将\_\_exit\_\_() 方法本身解释为成功（不管原始错误是被传播还是抑制）。

然而，如果\_\_exit\_\_() 向其调用者传播了异常，这就意味着\_\_exit\_\_() 本身已经失败。因此，\_\_exit\_\_() 方法应该避免引发错误，除非它们确实失败了。(允许原始错误继续并不是失败。)

## 过渡计划

在 Python 2.5 中，新语法需要通过 future 引入：

```plain
from __future__ import with_statement
```

它会引入'with'和'as'关键字。如果没有导入，使用'with'或'as'作为标识符时，将导致报错。

在 Python 2.6 中，新语法总是生效的，'with'和'as'已经是关键字。

## 生成器装饰器

随着 PEP-342 被采纳，我们可以编写一个装饰器，令其使用只 yield 一次的生成器来控制 with 语句。这是一个装饰器的粗略示例：

```python
class GeneratorContextManager(object):
   def __init__(self, gen):
       self.gen = gen
   def __enter__(self):
       try:
           return self.gen.next()
       except StopIteration:
           raise RuntimeError("generator didn't yield")
   def __exit__(self, type, value, traceback):
       if type is None:
           try:
               self.gen.next()
           except StopIteration:
               return
           else:
               raise RuntimeError("generator didn't stop")
       else:
           try:
               self.gen.throw(type, value, traceback)
               raise RuntimeError("generator didn't stop after throw()")
           except StopIteration:
               return True
           except:
               # only re-raise if it's *not* the exception that was
               # passed to throw(), because __exit__() must not raise
               # an exception unless __exit__() itself failed.  But
               # throw() has to raise the exception to signal
               # propagation, so this fixes the impedance mismatch
               # between the throw() protocol and the __exit__()
               # protocol.
               #
               if sys.exc_info()[1] is not value:
                   raise
def contextmanager(func):
   def helper(*args, **kwds):
       return GeneratorContextManager(func(*args, **kwds))
   return helper
```

这个装饰器可以这样使用：

```python
@contextmanager
def opening(filename):
   f = open(filename) # IOError is untouched by GeneratorContext
   try:
       yield f
   finally:
       f.close() # Ditto for errors here (however unlikely)
```

这个装饰器的健壮版本将会加入到标准库中。

## 标准库中的上下文管理器

可以将\_\_enter\_\_() 和\_\_exit\_\_() 方法赋予某些对象，如文件、套接字和锁，这样就不用写:

```python
with locking(myLock):
    BLOCK
```

而是简单地写成：

```python
with myLock:
    BLOCK
```

我想我们应该谨慎对待它；它可能会导致以下的错误:

```python
f = open(filename)
with f:
    BLOCK1
with f:
    BLOCK2
```

它可能跟你想的不一样（在进入 block2 之前，f 已经关闭了）。

另一方面，这样的错误很容易诊断；例如，当第二个 with 语句再调用 f.\_\_enter\_\_() 时，上面的生成器装饰器将引发 RuntimeError。如果在一个已关闭的文件对象上调用\_\_enter\_\_，则可能引发类似的错误。

在 Python 2.5中，以下类型被标识为上下文管理器：

```plain
- file
- thread.LockType
- threading.Lock
- threading.RLock
- threading.Condition
- threading.Semaphore
- threading.BoundedSemaphore
```

还将在 decimal 模块添加一个上下文管理器，以支持在 with 语句中使用本地的十进制算术上下文，并在退出 with 语句时，自动恢复原始上下文。

## 标准术语

本 PEP 提议将由\_\_enter\_\_() 和 \_\_exit\_\_() 方法组成的协议称为“上下文管理器协议”，并将实现该协议的对象称为“上下文管理器”。［4]

紧跟着 with 关键字的表达式被称为“上下文表达式”，该表达式提供了上下文管理器在with 代码块中所建立的运行时环境的主要线索。

目前为止， with 语句体中的代码和 as 关键字后面的变量名（一个或多个）还没有特殊的术语。可以使用一般的术语“语句体”和“目标列表”，如果这些术语不清晰，可以使用“with”或“with statement”作为前缀。

考虑到可能存在 decimal 模块的算术上下文这样的对象，因此术语“上下文”是有歧义的。如果想要更加具体的话，可以使用术语“上下文管理器”，表示上下文表达式所创建的具体对象；使用术语“运行时上下文”或者（最好是）"运行时环境"，表示上下文管理器所做出的实际状态的变更。当简单地讨论 with 语句的用法时，歧义性无关紧要，因为上下文表达式完全定义了对运行时环境所做的更改。当讨论 with 语句本身的机制以及如何实际实现上下文管理器时，这些术语的区别才是重要的。

## 缓存上下文管理器

许多上下文管理器（例如文件和基于生成器的上下文）都是一次性的对象。一旦\_\_exit\_\_() 方法被调用，上下文管理器将不再可用（例如：文件已经被关闭，或者底层生成器已经完成执行）。

对于多线程代码，以及嵌套的 with 语句想要使用同一个上下文管理器，最简单的方法是给每个 with 语句一个新的管理器对象。并非巧合的是，标准库中所有支持重用的上下文管理器都来自 threading 模块——它们都被设计用来处理由线程和嵌套使用所产生的问题。

这意味着，为了保存带有特定初始化参数（为了用在多个 with 语句）的上下文管理器，通常需要将它存储在一个无参数的可调用对象，然后在每个语句的上下文表达式中调用，而不是直接把上下文管理器缓存起来。

如果此限制不适用，在受影响的上下文管理器的文档中，应该清楚地指出这一点。

## 解决的问题

以下的问题经由 BDFL 的裁决而解决（并且在 python-dev 上没有重大的反对意见）。

1、当底层的生成器-迭代器行为异常时，GeneratorContextManager 应该引发什么异常？下面引用的内容是 Guido 为本 PEP及 PEP-342 （见[8]）中生成器的 close() 方法选择 RuntimeError 的原因：“我不愿意只是为了它而引入一个新的异常类，因为这不是我想让人们捕获的异常：我想让它变成一个回溯（traceback），被程序员看到并且修复。因此，我认为它们都应该引发 RuntimeError。有一些引发 RuntimeError 的先例：Python 核心代码在检测到无限递归时，遇到未初始化的对象时（以及其它各种各样的情况）。”

2、如果在with语句所涉及的类中没有相关的方法，则最好是抛出AttributeError而不是TypeError。抽象对象C API引发TypeError而不是AttributeError，这只是历史的一个偶然，而不是经过深思熟虑的设计决策[11]。

3、带有\_\_enter\_\_ /\_\_exit\_\_方法的对象被称为“上下文管理器”，将生成器函数转化为上下文管理器工厂的是 contextlib.contextmanager 装饰器。在 2.5版本发布期间，有人提议使用其它的叫法[16]，但没有足够令人信服的理由。

## 拒绝的选项

在长达几个月的时间里，对于是否要抑制异常（从而避免隐藏的流程控制），出现了一场令人痛苦的拉锯战，最终，Guido 决定要抑制异常[13]。

本 PEP 的另一个话题也引起了无休止的争论，即是否要提供一个\_\_context\_\_() 方法，类似于可迭代对象的\_\_iter\_\_() 方法[5][7][9]。源源不断的问题[10][13]在解释它是什么、为什么是那样、以及它是如何工作的，最终导致 Guido 完全抛弃了这个东西[15]（这很让人欢欣鼓舞！）

还有人提议直接使用 PEP-342 的生成器 API 来定义 with 语句[6]，但这很快就不予考虑了，因为它会导致难以编写不基于生成器的上下文管理器。

## 例子

基于生成器的示例依赖于 PEP-342。另外，有些例子是不实用的，因为标准库中有现成的对象可以在 with 语句中直接使用，例如 threading.RLock。

例子中那些函数名所用的时态并不是随意的。过去时态（“-ed”）的函数指的是在\_\_enter\_\_方法中执行，并在\_\_exit\_\_方法中反执行的动作。进行时态（"-ing"）的函数指的是准备在\_\_exit\_\_方法中执行的动作。

1、一个锁的模板，在开始时获取，在离开时释放：

```python
@contextmanager
def locked(lock):
    lock.acquire()
    try:
        yield
    finally:
        lock.release()
```

使用如下：

```python
with locked(myLock):
    # Code here executes with myLock held.  The lock is
    # guaranteed to be released when the block is left (even
    # if via return or by an uncaught exception).
```

2、一个打开文件的模板，确保当代码被执行后，文件会被关闭：

```python
@contextmanager
def opened(filename, mode="r"):
    f = open(filename, mode)
    try:
        yield f
    finally:
        f.close()
```

使用如下:

```python
with opened("/etc/passwd") as f:
    for line in f:
        print line.rstrip()
```

3、一个数据库事务的模板，用于提交或回滚：

```python
@contextmanager
def transaction(db):
    db.begin()
    try:
        yield None
    except:
        db.rollback()
        raise
    else:
        db.commit()
```

4、不使用生成器，重写例子 1：

```python
class locked:
   def __init__(self, lock):
       self.lock = lock
   def __enter__(self):
       self.lock.acquire()
   def __exit__(self, type, value, tb):
       self.lock.release()
```

（这个例子很容易被修改来实现其他相对无状态的例子；这表明，如果不需要保留特殊的状态，就不必要使用生成器。）

5、临时重定向 stdout：

```python
@contextmanager
def stdout_redirected(new_stdout):
    save_stdout = sys.stdout
    sys.stdout = new_stdout
    try:
        yield None
    finally:
        sys.stdout = save_stdout
```

使用如下：

```python
with opened(filename, "w") as f:
    with stdout_redirected(f):
        print "Hello world"
```

当然，这不是线程安全的，但是若不用管理器的话，本身也不是线程安全的。在单线程程序（例如脚本）中，这种做法很受欢迎。

6、opened() 的一个变体，也返回一个错误条件：

```python
@contextmanager
def opened_w_error(filename, mode="r"):
    try:
        f = open(filename, mode)
    except IOError, err:
        yield None, err
    else:
        try:
            yield f, None
        finally:
            f.close()
```

使用如下:

```python
with opened_w_error("/etc/passwd", "a") as (f, err):
    if err:
        print "IOError:", err
    else:
        f.write("guido::0:0::/:/bin/sh\n")
```

7、另一个有用的操作是阻塞信号。它的用法是这样的：

```python
import signal
with signal.blocked():
    # code executed without worrying about signals
```

它的参数是可选的，表示要阻塞的信号列表；在默认情况下，所有信号都被阻塞。具体实现就留给读者作为练习吧。

8、此特性还有一个用途是 Decimal 上下文。下面是 Michael Chermside 发布的一个简单的例子：

```python
import decimal
@contextmanager
def extra_precision(places=2):
    c = decimal.getcontext()
    saved_prec = c.prec
    c.prec += places
    try:
        yield None
    finally:
        c.prec = saved_prec
```

示例用法（摘自 Python 库参考文档）：

```python
def sin(x):
    "Return the sine of x as measured in radians."
    with extra_precision():
        i, lasts, s, fact, num, sign = 1, 0, x, 1, x, 1
        while s != lasts:
            lasts = s
            i += 2
            fact *= i * (i-1)
            num *= x * x
            sign *= -1
            s += num / fact * sign
    # The "+s" rounds back to the original precision,
    # so this must be outside the with-statement:
    return +s
```

9、下面是 decimal 模块的一个简单的上下文管理器：

```python
@contextmanager
def localcontext(ctx=None):
    """Set a new local decimal context for the block"""
    # Default to using the current context
    if ctx is None:
        ctx = getcontext()
    # We set the thread context to a copy of this context
    # to ensure that changes within the block are kept
    # local to the block.
    newctx = ctx.copy()
    oldctx = decimal.getcontext()
    decimal.setcontext(newctx)
    try:
        yield newctx
    finally:
        # Always restore the original context
        decimal.setcontext(oldctx)
```

示例用法:

```python
from decimal import localcontext, ExtendedContext
def sin(x):
    with localcontext() as ctx:
        ctx.prec += 2
        # Rest of sin calculation algorithm
        # uses a precision 2 greater than normal
    return +s # Convert result to normal precision
def sin(x):
    with localcontext(ExtendedContext):
        # Rest of sin calculation algorithm
        # uses the Extended Context from the
        # General Decimal Arithmetic Specification
    return +s # Convert result to normal context
```

10、一个通用的“对象关闭”上下文管理器：

```python
class closing(object):
    def __init__(self, obj):
        self.obj = obj
    def __enter__(self):
        return self.obj
    def __exit__(self, *exc_info):
        try:
            close_it = self.obj.close
        except AttributeError:
            pass
        else:
            close_it()
```

这可以确保关闭任何带有 close 方法的东西，无论是文件、生成器，还是其他东西。它甚至可以在对象并不需要关闭的情况下使用（例如，一个接受了任意可迭代对象的函数）：

```python
# emulate opening():
with closing(open("argument.txt")) as contradiction:
   for line in contradiction:
       print line
# deterministically finalize an iterator:
with closing(iter(data_source)) as data:
   for datum in data:
       process(datum)
```

（Python 2.5 的 contextlib 模块包含了这个上下文管理器的一个版本）

11、PEP-319 给出了一个用例，它也有一个 release() 上下文，能临时释放先前获得的锁；这个用例跟前文的例子 4 很相似，只是交换了 acquire() 和 release() 的调用：

```python
class released:
  def __init__(self, lock):
      self.lock = lock
  def __enter__(self):
      self.lock.release()
  def __exit__(self, type, value, tb):
      self.lock.acquire()
```

示例用法:

```python
with my_lock:
    # Operations with the lock held
    with released(my_lock):
        # Operations without the lock
        # e.g. blocking I/O
    # Lock is held again here
```

12、一个“嵌套型”上下文管理器，自动从左到右嵌套所提供的上下文，可以避免过度缩进：

```python
@contextmanager
def nested(*contexts):
    exits = []
    vars = []
    try:
        try:
            for context in contexts:
                exit = context.__exit__
                enter = context.__enter__
                vars.append(enter())
                exits.append(exit)
            yield vars
        except:
            exc = sys.exc_info()
        else:
            exc = (None, None, None)
    finally:
        while exits:
            exit = exits.pop()
            try:
                exit(*exc)
            except:
                exc = sys.exc_info()
            else:
                exc = (None, None, None)
        if exc != (None, None, None):
            # sys.exc_info() may have been
            # changed by one of the exit methods
            # so provide explicit exception info
            raise exc[0], exc[1], exc[2]
```

示例用法:

```python
with nested(a, b, c) as (x, y, z):
    # Perform operation
```

等价于:

```python
with a as x:
    with b as y:
        with c as z:
            # Perform operation
```

（Python 2.5 的 contextlib 模块包含了这个上下文管理器的一个版本）

## 参考实现

在 2005 年 6 月 27 日的 EuroPython 会议上，Guido 首次采纳了这个 PEP。之后它添加了\_\_context\_\_方法，并被再次采纳。此 PEP 在 Python 2.5 a1 子版本中实现，\_\_context\_\_() 方法在 Python 2.5b1 中被删除。

## 致谢

许多人对这个 PEP 中的想法和概念作出了贡献，包括在 PEP-340 和 PEP-346 的致谢中提到的所有人。

另外，还要感谢（排名不分先后）：Paul Moore, Phillip J. Eby, Greg Ewing, Jason Orendorff, Michael Hudson, Raymond Hettinger, Walter Dörwald, Aahz, Georg Brandl, Terry Reedy, A.M. Kuchling, Brett Cannon，以及所有参与了 python-dev 讨论的人。

## 参考链接

[1]	Raymond Chen's article on hidden flow control[https://devblogs.microsoft.com/oldnewthing/20050106-00/?p=36783](https://devblogs.microsoft.com/oldnewthing/20050106-00/?p=36783)

[2]	Guido suggests some generator changes that ended up in PEP 342[https://mail.python.org/pipermail/python-dev/2005-May/053885.html](https://mail.python.org/pipermail/python-dev/2005-May/053885.html)

[3]	Wiki discussion of PEP 343[http://wiki.python.org/moin/WithStatement](http://wiki.python.org/moin/WithStatement)

[4]	Early draft of some documentation for the with statement[https://mail.python.org/pipermail/python-dev/2005-July/054658.html](https://mail.python.org/pipermail/python-dev/2005-July/054658.html)

[5]	Proposal to add the **with** method[https://mail.python.org/pipermail/python-dev/2005-October/056947.html](https://mail.python.org/pipermail/python-dev/2005-October/056947.html)

[6]	Proposal to use the PEP 342 enhanced generator API directly[https://mail.python.org/pipermail/python-dev/2005-October/056969.html](https://mail.python.org/pipermail/python-dev/2005-October/056969.html)

[7]	Guido lets me (Nick Coghlan) talk him into a bad idea ;)[https://mail.python.org/pipermail/python-dev/2005-October/057018.html](https://mail.python.org/pipermail/python-dev/2005-October/057018.html)

[8]	Guido raises some exception handling questions[https://mail.python.org/pipermail/python-dev/2005-June/054064.html](https://mail.python.org/pipermail/python-dev/2005-June/054064.html)

[9]	Guido answers some questions about the **context** method[https://mail.python.org/pipermail/python-dev/2005-October/057520.html](https://mail.python.org/pipermail/python-dev/2005-October/057520.html)

[10]	Guido answers more questions about the **context** method[https://mail.python.org/pipermail/python-dev/2005-October/057535.html](https://mail.python.org/pipermail/python-dev/2005-October/057535.html)

[11]	Guido says AttributeError is fine for missing special methods[https://mail.python.org/pipermail/python-dev/2005-October/057625.html](https://mail.python.org/pipermail/python-dev/2005-October/057625.html)

[12]	Original PEP 342 implementation patch[http://sourceforge.net/tracker/index.php?func=detail&aid=1223381&group_id=5470&atid=305470](http://sourceforge.net/tracker/index.php?func=detail&aid=1223381&group_id=5470&atid=305470)

[13]	(1, 2) Guido restores the ability to suppress exceptions[https://mail.python.org/pipermail/python-dev/2006-February/061909.html](https://mail.python.org/pipermail/python-dev/2006-February/061909.html)

[14]	A simple question kickstarts a thorough review of PEP 343[https://mail.python.org/pipermail/python-dev/2006-April/063859.html](https://mail.python.org/pipermail/python-dev/2006-April/063859.html)

[15]	Guido kills the **context**() method[https://mail.python.org/pipermail/python-dev/2006-April/064632.html](https://mail.python.org/pipermail/python-dev/2006-April/064632.html)

[16]	Proposal to use 'context guard' instead of 'context manager'[https://mail.python.org/pipermail/python-dev/2006-May/064676.html](https://mail.python.org/pipermail/python-dev/2006-May/064676.html)

## 版权

本文档已进入公共领域。

源文档：[https://github.com/python/peps/blob/master/pep-0343.txt](https://github.com/python/peps/blob/master/pep-0343.txt)


