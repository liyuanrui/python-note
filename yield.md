>lr学习yield的笔记，主要是迭代器，生成器函数，协程。内容来自<<流畅的python>>

笔记中所有的示例都基于Python3.6.2，笔记中所有的人称，都是作者的视角

### 迭代器

#### 前言

迭代是数据处理的基石，扫描内存中放不下的数据集时，我们要找到一种惰性获取数据项的方式，即按需一次获取一个数据项，这就是迭代器模式。

所有生成器都是迭代器，因为生成器完全实现了迭代器接口。

在Python3中，生成器有广泛的用途，即使是range函数也返回一个类似生成器的对象。

在Python中，所有的集合都可以迭代，在Python内部，迭代器用于支持:
```
for 循环 (for是用迭代器实现的)
创建和扩展集合类型
逐行遍历文本文件
列表推导、字典推导和集合推导
元组拆包
调用函数时，使用*拆包实参
```

#### 迭代器示例

```
>>> nl=[1,2,3,4,5]
>>> il=iter(nl)
>>> type(nl)
<class 'list'>          #列表
>>> type(il)
<class 'list_iterator'> #迭代器
>>> 
>>> next(il)
1
```

#### 序列可以迭代的原因：iter函数

内置的iter函数有以下作用:
1. 检查对象是否实现了__iter__方法，如果实现了就调用它，获取一个迭代器。
2. 如果没有实现__iter__方法，但是实现了__getitem__方法，Python会创建一个迭代器，尝试按顺序（从索引0开始）获取元素。
3. 如果尝试失败，Python会抛出TypeError异常，通常会提示"C object is not iterable" (C对象不可迭代)，其中C是目标对象所属的类。

任何Python序列都可迭代的原因是，它们都实现了__getitem__方法。

从Python3.4开始，检查对象 x 能否迭代，最准确的方法是: 调用iter(x)函数，如果不可迭代，再处理TypeError异常。

#### 可迭代的对象和迭代器的对比

**可迭代的对象**

使用iter内置函数可以获取迭代器的对象。如果对象返回迭代器的__iter__方法，那么对象就是可迭代的。

**迭代器**

迭代器是这样的对象，实现了无参数的__next__方法，返回序列中的下一个元素，如果没有元素了，那么抛出StopIteration异常。Python中的迭代器还实现了__iter__方法，因此迭代器也可以迭代。


### 生成器函数

#### 生成器函数的工作原理

只要Python函数的定义体中有yield关键字，该函数就是生成器函数。调用生成器函数时，会返回一个生成器对象。也就是说，生成器函数是生成器工厂。

普通的函数与生成器函数在句法上唯一的区别是，在后者的定义体中有yield关键字。


下面以一个特别简单的函数说明生成器的行为:
```
>>> def gen_123():
...     yield 1
...     yield 2
...     yield 3
...
>>> gen_123
<function gen_123 at 0x...>
>>> gen_123()
<generator object gen_123 at 0x...>
>>> for i in gen_123(): 
    '''生成器是迭代器，会生成传给yield关键字的表达式的值'''
...     print(i)
1
2
3
>>> g = gen_123()
>>> next(g)
1
>>> next(g)
2
>>> next(g)
3
>>> next(g)
'''生成器函数的定义体执行完毕后，生成器对象会抛出StopIteration异常'''
Traceback (most recent call last):
  ...
StopIteration
```

生成器函数会创建一个生成器对象，包装生成器函数的定义体，把生成器传给next(...)函数时，生成器会向前，执行函数定义体的下一个yield语句，返回产出的值，并在函数定义体的当前位置暂停。最终，函数的定义体返回时，外层的生成器对象会抛出StopIteration异常--这一点与迭代器协议一致。

我觉得，使用准确的词语描述重生成器中获取结果的过程，有助于理解生成器。注意，我说的是**产出或生成值**。如果说生成器 "返回" 值，就会让人难以理解。函数返回值；调用生成器函数返回生成值，


在Python3.3之前，如果生成器函数中的return语句有返回值，那么会报错。现在可以这么做，不过retuen语句仍会导致StopIteration异常抛出。调用方可以从异常对象中获取返回值。可是，只有把生成器函数当成协程使用时，这样做才有意义。


使用for循环更清楚地说明了生成器函数定义体的执行过程:
```
>>> def gen_AB():
...     print('start')
...     yield 'A'         #2
...     print('continue') 
...     yield 'B'         #3
...     print('end.')     #4
...
>>> for c in gen_AB():    #5
...     print('-->',c)    
start
--> A    #8
continue
--> B
end.
>>>      #12
```

2. 在for循环中第一次隐式调用next()函数时（序号5），会打印'continue'，然后停在第一个yield语句，生成值'A'。

3. 在for循环中第二次隐式调用next()函数时，会打印'continue'，然后停在第二个yield语句,生成值'B'。

4. 第三次调用next()函数时，会打印'end.'，然后到达函数定义体的末尾，导致生成器对象抛出StopIteration异常。

5. 迭代时，for机制的作用与g = iter(gen_AB())一样，用于获取生成器对象，然后每次迭代时调用next(g)。

8. 生成器函数定义体中的yield 'A'语句会生成值A，提供给for循环使用，而A会赋值给变量c，最终输出--> A。

12. 到达生成器函数定义体的末尾，生成器对象抛出StopIteration异常。for机制会捕获异常，因此循环终止时没有报错。


#### 惰性实现


只要使用的是Python3，思索着做某件事情有没有懒惰的方式，答案通常都是肯定的。

re.finditer函数是re.findall函数的惰性版本，返回的不是列表，而是一个生成器，按需生成re.MatchObject实例。如果有很多匹配，re.finditer能节省大量内存。

```
import re
import reprlib

RE_WORDS = re.compile('\w+')

class Sentence:
    def __init__(self,text):
        self.text=text

    def __repr__(self):
        return 'Setence(%s)' % reprlib.repr(self.text)

    def __iter__(self):
        for math in RE_WORDS.finditer(self.text):  #2
            yield match.group()   #3
```

2. finditer函数构建了一个迭代器，包含self.text中匹配RE_WORDS的单词，产生MatchObject实例。

3. match.group() 方法从MatchObject实例中提取匹配正则表达式的具体文本。


#### 生成器表达式

生成器表达式可以理解为列表推导式的惰性版本，不会迫切地构建列表，而是返回一个生成器，按需惰性生成元素。

```
>>> def gen_AB():
...     print('start')
...     yield 'A'
...     print('continue')
...     yield 'B'
...     print('end.')
...
>>> res1 = [x*3 for x in gen_AB()]
start
continue
end.
>>> for i in res1:print('-->',i)
--> AAA
--> BBB
>>> res2 = (x*3 for x in gen_AB())
>>> res2
<generator object <genexpr> at 0x...>
>>> for i in res2:print('-->',i)
start
--> AAA
continue
--> BBB
end.
```

#### Python 3.3中新出现的句法: yield from

如果生成器函数需要产出**另一个生成器生成的值**，传统的解决方法是使用嵌套的for循环。

```
>>> def chain(*iterables):
...     for it in iterables:
...         for i in it:
...             yield i
...
>>> s = 'ABC'
>>> t = tuple(range(3))
>>> list(chain(s, t))
['A', 'B', 'C', 0, 1, 2]
```

PEP380引入了一个新句法: yield from

```
>>> def chain(*iterables):
...     for i in iterables:
...         yield from i
...
>>> s = 'ABC'
>>> t = tuple(range(3))
>>> list(chain(s, t))
['A', 'B', 'C', 0, 1, 2]
```

可以看出，yield from i 完全代替了内层的for循环。在这个示例中使用yield from是对的，而且代码读起来更顺畅，不过感觉更像语法糖。除了代替循环之外，yield from还会创建通道，把内层生成器直接与外层生成器的客户端联系起来，把生成器当成协程用时，这个通道特别重要，不仅能为客户端代码生成值，还能使用客户端代码提供的值。

#### 把生成器当成协程

Python2.2引入了yield关键字实现的生成器函数，大约五年后，Python2.5实现了PEP342。这个提案为生成器对象添加了额外的方法和功能，其中最值得关注的是.send()方法。

与.__next__()方法一样，.send()方法致使生成器前进到下一个yield语句，不过，.send()方法还允许使用生成器的客户把数据发给自己，即不管传给.send()方法什么参数，那个参数都会成为生成器函数定义体中对应的yield表达式的值。也就是说，.send()方法允许在客户代码和生成器之间双向交换数据。而__next__()方法只允许客户从生成器中获取数据。

这是一项重要的"改进"，甚至改变了生成器的本性，像这样使用的话，生成器就变身 为协程。

在协程中，yield通常出现在赋值语句的右手边，因为yield用于接收客户传给.send()方法的参数，正如David Beazley所说的：
尽管有一些相同之处，但是生成器和协程基本上是两个不同的概念。

### 协程

#### 前言

字典为动词 "to yield" 给出了两个释义: 产出和让步。对于Python生成器中的yield来说，这两个含义都成立。yield item这行代码会产出一个值，提供给next(...)的调用方；此外，还会做出让步，暂停执行生成器，让调用方继续工作，直到需要使用另一个值时再调用next()。调用方会从生成器中拉取值。

从句法上看，协程与生成器类似，都是定义体中包含yield关键字的函数。可是，在协程中，yield通常出现在表达式的右边 (例如，datum = yield)，可以产出值，也可以不产出值--如果yield关键字后面没有表达式，那么生成器产出None。协程可能会从调用方接收数据，不过调用方把数据提供给协程使用的是.send(datum)方法，而不是next(...)函数。通常，调用方会把值推送给协程。

yield关键字甚至还可以不接收或传出数据。不管数据如何流动，yield都是一种流程控制工具，使用它可以实现协作式多任务：协程可以把控制器让步给中心调度程序，从而激活其它的协程。

从根本上把yield视作控制流程的方式，这样就好理解协程了。

前面介绍的生成器函数作用不大，但是进行一系列功能改进之后，得到了Python协程。了解Python协程的进化过程有助于理解各个阶段改进的功能和复杂度。


#### 生成器如何进化成协程

协程的底层架构在PEP342中定义，并在Python2.5实现了。自此之后，yield关键字可以在表达式中使用，而且生成器API增加了.send(value)方法。生成器的调用方法可以使用.send(...)方法发送数据，发送的数据会成为生成器函数的yield表达式的值。
因此，生成器可以作为协程使用，协程是指一个过程，这个过程与调用方协作，产出由调用方提供的值。

除了.send(...)方法，PEP342还添加了.throw(...)和close(...)方法：前者的作用是让调用方抛出异常，在生成器中处理；后者的作用是终止生成器。

协程最近的演进是来自Python3.3实现的PEP380，对生成器函数的句法做了两处改动，以便更好地作为协程使用。

现在，生成器可以返回一个值；以前，如果在生成器中给return语句提供值，会抛出SyntaxError异常。

新引进了yield from句法，使用他可以把复杂的生成器重构成小型的嵌套生成器，省去了之前把生成器的工作委托给子生成器所需的大量样板代码。

#### 用作协程的生成器的基本行为

可能是协程最简单的使用演示
```
>>> def simple_coroutine():  #1
...     print('-> coroutine started')
...     x = yield   #2
...     print('-> coroutine received:', x)
...
>>> my_coro = simple_coroutine()
>>> my_coro   #3
<generator object simple_coroutine at 0x...>
>>> next(my_coro)   #4
-> coroutine started
>>> my_coro.sned(42)   #5
-> coroutine received: 42
Traceback (most recent call last):   #6
  ...
StopIteration

```

1. 协程使用生成器函数定义，定义体中有yield关键字。

2. yield在表达式中使用，如果协程只需从客户那里接收数据，那么产出的值是None--这个值是隐式指定的，因为yield关键字右边没有表达式。

3. 与创建生成器的方式一样，调用函数得到生成器对象。

4. 首先要调用next(...)函数，因为生成器还没启动，没在yield语句处暂停，所以一开始无法发送数据。

5. 调用这个方法后，协程定义体中的yield表达式会计算出42；现在，协程会恢复，一直运行到下一个yield表达式，或者终止。

6. 这里，控制权流动到协程定义体的末尾，导致生成器像往常一样抛出StopIteration异常。

协程可以身处四个状态中的一个，当前状态可以使用inspect.getganeratorstate(...)函数确定，该函数会返回下述字符串中的一个。

```
'GEN_CREATE'    等待开始执行
'GEN_RUNNING'   解释器正在执行
'GEN_SUSPENDED' 在yield表达式处暂停
'GEN_CLOSED'    执行结束
```

只有在多线程中才能看到这个状态，此外，生成器对象在自己身上调用getgenaratorstate函数也行，不过这样做没什么用。

因为send方法的参数会成为暂停的yield表达式的值，所以，仅当协程处于暂停状态时才能调用sned方法，例如my_coro.sned(42)。不过，如果协程还没激活(即，状态是'GEN_CREATED')，情况就不同了。因此，始终要调用next(my_coro)激活协程--也可以调用my_coro.send(None)，效果一样。

如果创建协程对象后立刻把None之外的值发给他，会出现以下错误:
```
>>> my_coro = simple_coroutine()
>>> my_coro.send(42)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: can't send non-None value to a just-startd generator

```

最先调用next(my_coro)函数这一步通常称为"预激"协程(即，让协程向前执行到第一个yield表达式，准备好作为活跃的协程使用)。

产出两个值的协程
```
>>> def simple_coro2(a):
...     print('-> Started: a=', a)
...     b = yield a
...     print('-> Received: b', b)
...     c = yield a + b
...     print('-> Received: c', c)
...
>>> my_coro2 = simple_coro2(14)
>>> next(my_coro2)
-> Started: a = 14
14
>>> my_coro2.send(28)
-> Received: b = 28
42
>>> my_coro2.send(99)
-> Received: c = 99
Treceback (most recent call last):
  ...
StopIteration
```

关键的一点是，协程在yield关键字所在的位置暂停执行。前面说过，在赋值语句中，= 右边的代码在赋值之前执行。因此，对于b = yield a 这行代码来说，等到客户端代码再激活协程时才会设定b的值。这种行为要花点时间才能习惯，不过一定要理解，这样才能弄懂异步编程中yield的作用。



1. 调用next(my_coro2), 打印第一个消息，然后执行yield a, 产出数字14。

2. 调用my_coro2.send(28), 把28赋值给b, 打印第二个消息，然后执行yield a+b，产出数字42.

3. 调用my_coro2.send(99),把99赋值给c，打印第三个消息，协程终止。


#### 示例: 使用协程计算移动平均值

```

```




















