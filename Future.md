### Future实例
Future 是 concurrent.futures 模块和 asyncio 包的重要组件。
从 Python3.4 起，标准库中有两个名为 Future 的类，concurrent.futures.Future 和 asyncio.Future 。这两个类的作用相同：两个 Future 类的实例都表示可能已经完成或者尚未完成的廷时计算。

Future 封装完成的操作，可以放入队列，完成的状态可以查询，得到结果(或抛出异常)后可以获取结果(或异常)。

我们要记住一件事：通常情况下自己不应该创建 Future , 而只能由并发框架(concurrent.futures或asyncio)实例化，原因很简单：Future表示终将发生的事情，而确定某件事情会发生的唯一方法是执行的时间已经排定。因此，只有排定将某件事交给concurrent.futures.Executor子类处理时，才会创建concurrent.futures.Future实例。例如，Executor.submit()方法的参数是一个可调用的对象，调用这个方法后会为传入的可调用对象排期，并返回一个Future。

客户端代码不应该改变Future的状态，并发框架在Future表示的延迟计算结束会改变Future的状态，而我们无法控制计算何时结束。

这两种Future都有.done()方法，这个方法不阻塞，返回值是布尔值，指明Future链接的可调用对象是否已经执行，客户端代码通常不会询问Future是否运行结束，而是会等待通知。因此，两个Future类都有.add_done_callback()方法，这个方法只有一个参数，类型是可调用的对象，Future运行结束后会调用指定的可调用对象。
此外，还有.result()方法。在Future运行结束后调用的话，这个方法在两个Future类中的作用相同: 返回可调用对象的结果，或者重新抛出执行可调用对象时抛出的异常。可是，如果Future没有运行结束，result方法在两个Future类中的行为相差很大。对concurrent.futures.Future实例来说，调用f.result()方法会阻塞调用方所在的线程，直到有结果可返回。此时，result方法可以接受可选的timeout参数，如果在指定的时间内Future没有运行完毕，会抛出TimeoutError异常。asyncio.Future.result方法不支持设定超时时间，在那个库中获取Future的结果最好使用yield from结构。不过，对于concurrent.futures.Future实例不能这么做。

这两个库中，有几个函数会返回Future，其他函数则使用Future，以用户易于理解的方式实现自身。使用Executor.map方法属于后者：返回值是一个迭代器，迭代器的__next__方法调用各个Future的result方法，因此我们得到的是各个Future的结果，而非Future本身。

为了从使用的角度理解Future，我们可以使用concurrent.futures.as_completed函数，这个函数的参数是一个Future列表，返回值是一个迭代器，在Future运行结束后产出Future。

为了使用futures.as_completed，需要把较抽象的executor.map调用换成两个for循环：一个创建并排定Future，另一个获取Future的结果。同时，我们会添加几个print调用，显示运行结束前后的Future。

**例子1 futures.map(使用Future)**
``` python
from concurrent import futures
import requests

def wget(*args):
    r=requests.get('http://baidu.com')
    reeturn r.status_code

with futures.ThreadPoolExecutor(20) as executor:
    res=executor.map(wget,range(20))
    print(len(list(res)))
```
**例子2 executor.submit(返回Future)和futures.as_completed(等待全部完成)**
``` python
from concurrent import futures
import requests

def wget(*args):
    r=requests.get('http://baidu.com')
    return r.status_code

with futures.ThreadPoolExecutor(20) as executor:
    to_do=[]
    for i in range(20):
        to_do.append(executor.submit(wget,i))
    for f in futures.as_completed(to_do):
        print(f.result())
```

Executor.map函数易于使用，不过有个特性可能有用，也可能没用，具体情况取决于需求：这个函数返回结果的顺序与调用开始的顺序一致。如果第一个调用生成结果用时10秒，而其他调用只用1秒，代码就会阻塞10秒，获取map方法返回的生成器产出了第一个结果。在此之后，获取后续结果时不会阻塞，因为后续的调用已经结束。如果必须等到获取所有结果后再处理，这种行为没问题；不过，通常更可取的方式是，不管提交的顺序，只要有结果就获取。为此，要把Executor.submit方法和futures.as_completed函数结合起来使用。

executor.submit和futures.as_completed这个组合比exexutor.map更灵活，因为submit方法能处理不同的可调用对象和参数，而executor.map只能处理参数不同的同一个可调用对象。此外，传给futures.as_completed函数的Future集合可以来自多个Executor实例，例如一些由ThreadPoolExecutor实例创建，另一些由ProcessPooolExecutor实例创建。

**例子3 使用tqdm显示进度条**
``` python
from concurrent import futures
import time
import random
import tqdm
from urllib.request import urlopen

def get():
    url='http://coolr.pythonanywhere.com'
    with urlopen(url) as resp:
        read=resp.read()

if __name__ == '__main__':
    with futures.ThreadPoolExecutor(20) as executor:
        to_do = []
        for i in range(1000):
            future = executor.submit(get)
            to_do.append(future)
        diter=futures.as_completed(to_do)
        for i in tqdm.tqdm(diter,total=len(to_do)):pass

```
