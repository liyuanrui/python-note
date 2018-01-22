asyncio库: 使用事件循环驱动的协程实现并发

### 线程与协程对比
asyncio包使用的"协程"是较严格的定义。适合asyncio API的协程在定义体中必须使用yield from, 而不能使用yield. 此外，适合asyncio的协程要由调用方驱动，并由调用方通过yield from调用; 或者把协程传给asyncio包中的某个函数，例如asyncio.async(), 从而驱动协程。最后，@asyncio.coroutine装饰器应该用在协程上。

**示例 18-2 通过协程以动画形式显示文本式旋转指针**
``` python3
import asyncio
import itertools
import sys

@asyncio.coroutine
def spin(msg):
    write,flush=sys.stdout.write,sys.stdout.flush
    for char in itertools.cycle('|/-\\'):
        status=char+' '+msg
        write(status)
        flush()
        write('\x08'*len(status))
        try:
            yield from asyncio.sleep(0.1) #3
        except asyncio.CancelledError:    #4
            break
    write(' '*len(status)+'\x08'*len(status))

@asyncio.coroutine
 def slow_function():
      yield from asyncio.sleep(3)
      return 42

@asyncio.coroutine
def supervisor():
    spinner=asyncio.async(spin('thinking!')) #8
    print('spinner object:',spinner)
    result=yield from slow_function()
    spinner.cancel() #11
    return result

def main():
    loop=asyncio.get_event_loop() #12
    result=loop.run_until_complete(supervisor()) #13
    loop.close()
    print('Answer:',result)

if __name__=='__main__':
    main()
```

**3** 使用yield from asyncio.sleep(0.1)代替time.sleep(0.1), 这样的休眠不会阻塞事件循环。
**4** 如果spin函数苏醒后抛出asyncio.CancelledError异常，其原因是发出了取消请求，因此退出循环。
**8** asyncio.async(...)函数排定spin协程的运行时间，使用一个Task对象(Future的子类)包装spin协程，并立即返回。
**11** Task对象可以取消；取消后会在协程当前暂停的yield处抛出asyncio.CancelledError异常，协程可以捕获这个异常，也可以延迟取消，甚至拒绝取消。
**12** 获取时间循环的引用。
**13** 驱动supervisor协程，让它运行完毕，这个协程的返回值是这次调用的返回值。

**#** asyncio.Task对象差不多与threading.Thread对象等效。
**#** Task对象不由自己动手实例化，而是通过把协程传给asyncio.async(...)函数或loop.create_task(...)方法获取。
**#** 获取的Task对象已经排定了运行时间(例如，由asyncio.async函数排定).
**#** 如果想终止任务，可以使用Task.cancel()实例方法，在协程内部抛出CancelledError异常。协程可以在暂停的yield处捕获这个异常，处理终止请求。
**#** supervisor协程必须在main函数中由loop.run_until_complete方法执行。
**#** 协程自身就会同步，因为在任何时刻只有一个协程运行。想要交出控制权时，可以使用yield或yield from把控制权交还给调度程序。这就是能够安全地取消协程地原因:按照定义，协程只能在暂停地yield处取消，因此可以处理CancelledError异常，执行清理操作。

### asyncio.Future与concurrent.futures.Future
如上所述，future只是调度执行某物地结果。在asyncio包中，BaseEventLoop.create_task(...)方法接受一个协程，排定它地运行时间，然后返回一个asyncio.Task实例----也是asyncio.Future类的实例，因为Task是Future的子类，用于包装协程。**这与调用Executor.submit(...)方法创建concurrent.futures.Future实例是一个道理。**

获取asyncio.Future对象的结果通常使用yield from，从中产出结果。而不是.result()方法。
使用yield from处理future，等待future运行完毕这一步无需我们关心，而且不会阻塞事件循环，**因为在asyncio包中，yield from的作用就是把控制权还给事件循环。**


### 从future、任务和协程中产出

在asyncio包中，future和协程关系紧密，因为可以使用yield from从asyncio.Future对象中产出结果。这意味着，如果foo是协程函数(调用后返回协程对象), 抑或是返回Future或Task实例的普通函数，那么可以这样写: res=yield from foo(). 这是asyncio包的API中很多地方可以互换协程与future的原因之一。

为了执行这些操作，必须排定协程的运行时间，然后使用asyncio.Task对象包装协程，对协程来说，获取Task对象有两种主要方式。

**asyncio.async(coro_or_future, *, loop=None)**
这个函数统一了协程和future，第一个参数可以是二者中的任何一个。如果是future或Task对象，那就原封不动地返回。如果是协程，那么async函数会调用loop.create_task(...)方法创建Task对象。loop=关键字参数是可选的，用于传入事件循环，如果没有传入，那么async函数会通过调用asyncio.get_event_loop()函数获取循环对象。

**BaseEventLoop.create_task(coro)**
这个方法排定协程的执行时间，返回一个asyncio.Task对象，如果在自定义的BaseEventLoop子类调用，返回的对象可能是外部库中与Task类兼容的某个类的实例。

asyncio包中有多个函数会自动(内部使用的是asyncio.async函数)把参数指定的协程包装在asyncio.Task对象中，例如BaseEventLoop.run_until_complete(...)方法。

### 使用asyncio和aiohttp包下载

**示例 18-5 使用asyncio和aiohttp包实现的异步下载脚本**
``` python
import asyncio
import aiohttp

BASE_URL='http://flupy.org/data/flags'
CC=['CN','IN','US','ID','BR','PK','NG','BD','RU','JP','MX',    'PH','VN','ET','EG','DE','IR','TR','CD''FR']

@asyncio.coroutine
def get_flag(cc):
    url='{}/{cc}/{cc}.gif'.format(BASE_URL,cc=cc.lower())
    with aiohttp.ClientSession() as session:
        resp=yield from session.get(url)
        image=yield from resp.text()
        return image

@asyncio.coroutine
def download_one(cc):
    image=yield from get_flag(cc)
    print(cc,end=' ')
    with open('{}.gif'.format(cc),'wb') as f:
        f.write(image)
    return cc

def download_many():
    loop=asyncio.get_event_loop()
    to_do=[download_one(cc) for cc in CC]
    wait_coro=asyncio.wait(to_do)  #10
    res,_=loop.run_until_complete(wait_coro) #11
    loop.close()
    return len(res)

if __name__=='__main__':
    download_many()
```

**10** 虽然函数的名称是wait, 但它不是阻塞型函数。wait是一个协程，等传给它的所有协程运行完毕后结束(这是wait的默认行为)
**11** 执行事件循环，直到wait_coro运行结束；事件循环运行的过程中，这个脚本会在这里阻塞。

asyncio.wait(...)协程的参数是一个由future或协程构成的可迭代对象；wait会分别把各个协程包装进一个Task对象。最终的结果是，wait处理的所有对象都通过某种方式变成Future类的实例。wait是协程函数，因此返回的是一个协程或生成器对象；wait_coro变量中存储的正是这种对象。为了驱动协程，我们把协程传给loop.run_until_complete(...)方法。

loop.run_until_complete方法的参数是一个future或协程。如果是协程，run_until_complete方法与wait函数一样，把协程包装进一个Task对象中。**协程、future和任务都能有yield from驱动，**这正是run_until_complete方法对wait函数返回的wait_coro对象做的事。**wait_coro运行结束后返回一个元组，第一个元素是一系列结束的future, 第二个元素是一系列未结束的future**。在示例18-5中，第二个元素始终为空，因此我们把它赋值给-，将其忽略. 但是wait函数有两个关键字参数，如果设定了可能会返回未结束的future, 这两个参数是timeout和return-when.

我们编写的协程链条始终通过把最外层委派生成器传给asyncio包API中的某个函数(如loop.run_until_complete)驱动。也就是说，使用asyncio包时，我们编写的代码不通过调用next(...)或.send(...)方法驱动协程----这一点由asyncio包实现的事件循环去做。

我们编写的协程链条最终通过yield from把职责委派给asyncio包中的某个协程函数或协程方法(例如yield from asyncio.sleep(1))，或者其他库中实现的高层协议的协程(例如resp=yield from aiohttp.request('GET',url)).

也就是说，最内层的子生成器是库中真正执行I/O操作的函数，而不是我们自己编写的函数。

概括起来就是，**使用asyncio包时，我们编写的异步代码中包含由asyncio本身驱动的协程(即委派生成器)，而生成器最终把职责委托给asyncio包或第三方库(aiohttp)中的协程。这种方式相当于架起了管道，让asyncio事件循环(通过我们编写的协程)驱动执行低层异步I/O操作的库函数。**


### 使用asyncio.as_completed函数

在示例18-5中，把一个协程列表传给asyncio.wait函数，经由loop.run_until_complete方法驱动，全部协程运行完毕后，这个函数会返回所有下载结果。可是，为了更新进度条，各个协程运行结束后就要立即获取结果。在线程版示例中为了集成进度条，我们使用的是as_completed生成器函数，幸好，asyncio包提供了这个生成器函数的相应版本。

为了使用asyncio包实现flags示例，我们要重写几个函数，重写后的函数可以供concurrent.futures版重用。之所以要重写，是因为在使用asyncio包的程序中只有一个主线程，而在这个线程中不能有阻塞型调用，因为事件循环也在这个线程中运行。所以，要重写get_flags函数，使用yield from访问网络。现在，由于get_flags是协程，download_one函数必须使用yield from驱动它，因此download_one自己也要变成协程。

之前，在示例18-5中，download_one由download_many驱动：download_one函数由asyncio.wait函数调用，然后传给loop.run_until_complete方法。
现在，为了报告进度并处理错误，我们要更精确地控制，所以把download_many函数中大多数逻辑移到一个新的协程download_core中，只在download_many函数中设置事件循环，以及调度download_coro协程。

**示例 18-7**
``` python
import asyncio
import aiohttp
import tqdm

BASE_URL='http://flupy.org/data/flags'
CC=['CN','IN','US','ID','BR','PK','NG','BD','RU','JP','MX',    'PH','VN','ET','EG','DE','IR','TR','CD''FR']

def save_flag(cc,image):
    with open('{}.gif'.format(cc),'wb') as f:
        f.write(image)

@asyncio.coroutine
def get_flags(cc):
    url='{}/{cc}/{cc}.gif'.format(BASE_URL,cc=cc.lower())
    session=aiohttp.ClientSession()
    try:
        resp=yield from session.get(url)
        image=yield from resp.text()
    except:
        image=None
    finally:
        session.close()
    return image

@asyncio.coroutine
def download_one(cc,semaphore):
    with (yield from semaphore):
        image=yield from get_flags(cc)
        print(cc,end=' ')
        save_flag(cc,image)
        return cc

@asyncio.coroutine
def download_coro():
    semaphore=asyncio.Semaphore(20) #1
    to_do=[download_one(cc,semaphore) for cc in CC]
    item_coro=asyncio.as_completed(to_do) #2
    rets=[]
    for future in tqdm.tqdm(item_coro,total=len(to_do)): #3
        ret=yield from future #4
        rets.append(ret)
    return rets

def download_many():
    loop=asyncio.get_event_loop()
    ret=loop.run_until_complete(download_coro())
    print(len(ret))
    loop.close()

if __name__=='__main__':
    download_many()
```

**1** 创建一个asyncio.Semaphore实例，最多允许激活20个协程
**2** 获取一个迭代器，这个迭代器会在future运行结束后返回future
**3** 把迭代器传给tqdm函数，显示进度
**4** 获取asyncio.Future对象的结果，最简单的方法是使用yield from, 而不是调用future.result()方法


### 使用Executor对象，防止阻塞事件循环

访问本地文件系统会阻塞，在示例18-7中，保存文件会阻塞，保存文件阻塞了客户代码与asyncio事件循环共用的唯一线程，因此保存文件时，整个应用程序都会冻结。
这个问题的解决方法是，使用事件循环对象run_in_executor方法。

asyncio的事件循环在背后维护着一个ThreadPoolExecutor对象，我们可以调用run_in_executor方法，把可调用的对象发给它执行。若想在示例中使用这个功能，download_one协程只有几行代码需要改动。

``` python

@asyncio.coroutine
def download_one(cc,semaphore):
    with (yield from semaphore):
        image=yield from get_flags(cc)
        print(cc,end=' ')
        loop=asyncio.get_event_loop() #1
        loop.run_in_executor(None,save_flag,cc,image) #2
        return cc
```

run_in_executor方法的第一个参数是Executor实例；如果设为None，使用事件循环的默认ThreadPoolExecutor实例。余下的参数是可调用的对象以及参数。


not end...