#异步IO

在IO编程中我们知道CPU的速度远远快于磁盘网络等IO。在一个线程中，CPU执行代码的速度极快，然而一旦遇到IO操作，比如读写文件、发送网络数据时，就需要等待IO操作完成，才能进行下一步操作，这种情况称为同步IO。

在IO操作的过程中，当前线程被挂起，而其他需要CPU执行的代码就无法被当前线程执行了。因为一个IO操作就阻塞了当前线程，导致代码无法执行，所以我们必须使用多线程或者多进程并发执行代码，为多个用户服务。每个用户都会分配一个线程，如果遇到IO导致线程被挂起，其他用户的线程不受影响。

多线程和多进程的模型虽然解决了并发问题，但是系统不能无上限地增加线程。由于系统切换线程的开销也很大，所以，线程一旦过多，CPU的时间就花在线程切换上了，导致性能下降。

由于我们要解决的问题是CPU告诉执行能力和IO设备的龟速严重不匹配，多线程和多进程只是解决这一问题的一种方法。另一种方法是异步IO。当代码需要执行一个耗时的IO操作时，它只发出IO指令并不等待IO结果。然后去执行其他代码，一段时间后，当IO结果返回时，再通知CPU进行处理。

可以知道，普通顺序写的代码是无法完成异步IO的，异步IO模型需要一个消息循环，在消息循环中，主线程不断的重复着“读取消息-处理消息”这一过程：
```python
loop = get_event_loop()
while True:
	event = loop.get_event()
	process_event(event)
```

消息模型其实早就应用在桌面程序中了，一个GUI程序的主线程就负责不停地读取消息并处理消息。所有的键盘、鼠标等消息都被发送到GUI程序的消息队列中，然后由GUI程序的主线程处理。由于GUI线程处理键盘鼠标等消息的速度非常快、所以用户感觉不到延迟。某些情况下，GUI线程会在一个消息处理的过程中遇到问题导致一次消息处理时间过长，此时用户会感觉到整个GUI程序停止响应了，敲键盘鼠标都没有反应。这种情况说明在消息模型中，处理一个消息必须非常迅速，否则主线程将无法及时处理消息队列中的其他消息，导致程序看上去停止响应。

当遇到IO操作时，代码只负责发送IO请求，不等待IO结果，然后直接结束本轮消息处理，进入下一轮操作处理过程。当IO操作完成后，将受到一条IO完成的消息，处理该消息时就可以直接获取IO操作的结果。

在发出IO请求到受到IO完成这段时间里面，同步IO模型下，主线程只能挂起，但是异步IO模型下，主线程并没有休息，而是在消息循环中继续处理其他消息。这样在异步IO下，一个线程就可以同时处理多个IO请求，并且没有切换线程的操作。对大多数IO密集型程序，异步IO会大大提高系统的多任务处理能力。

--------------------

##协程

协程，又称微线程、 钎程。英文名Coroutine。

子程序或者成为函数，在所有语言中都是层级调用，比如A调用B，B在执行过程中又调用了C，C执行完毕返回，B执行完毕返回，A执行完毕。所以子程序调用是通过栈实现的，一个线程就是执行一个子程序。子程序调用总是一个入口一个返回，调用顺序是明确的。而协程的调用和子程序不同。

协程看上去也是子程序，但是在执行过程中，在子程序内部可中断，然后转而执行别的子程序，在适当的时候再返回来接着执行。注意在一个子程序中中断，去执行其他子程序，不是调用函数，有点类似CPU中断。比如子程序A、B：
```
def A():
    print('1')
    print('2')
    print('3')

def B():
    print('x')
    print('y')
    print('z')
```

假设由协程执行，在执行A的过程中，可以随时中断，去执行B，B也可能在执行中中断再去执行A，结果可能是：
```
1
2
x
y
3
z
```

但是再A总是没有调用B的，所以协程的调用比起函数调用理解要难。

看起来A、B的执行有点像多线程，但协程的特点在于是一个线程执行。和多线程相比，协程最大的优势就是极高的执行效率。因为子程序切换不是线程切换，而是程序自身控制的 ，因此没有线程切换的开销，和多线程相比，线程数量越多，协程的性能优势就越明显。

第二大优势就是不休要多线程的锁机制，因为只有一个线程，也不存在同时写和变量冲突，在协程中控制共享资源不加锁，只需要判断状态就好了，所以执行效率比多线程高很多。

协程是一个线程执行，为了利用多核CPU，可以用多进程+协程，即充分利用多核，又充分发挥协程的高效率，可获得奇高的性能。

Python对协程的支持是通过generator实现的。

在generator中，我们不但可以通过`for`循环来迭代，还可以不断调用`next()`函数获取由`yield`语句返回下一个值。但是Python的`yield`不但可以返回一个值，还可以接收调用者发出的参数。

例如：传统的生产者-消费者模型就是一个线程写消息，一个线程取消息，通过锁机制控制队列和等待，但一不小心就可能死锁。如果改用协程，生产者生产消息后，直接通过yield跳转到消费者开始执行，等消费者执行完毕后，切换回生产者继续生产，效率极高：
```python
>>> def consumer():
...     r=''
...     while True:
...             n = yield r
...             if not  n:
...                     return
...             print('[CONSUMER] Consuming %s...' % n )
...             r='200 OK'
...
>>> def produce(c):
...     c.send(None)
...     n=0
...     while n < 5:
...             n = n + 1
...             print('[PRODUCER] Producing %s...' %n)
...             r = c.send(n)
...             print('[PRODUCER] Consumer return: %s' % r )
...     c.close()
...
>>> c = consumer()
>>> produce(c)
[PRODUCER] Producing 1...
[CONSUMER] Consuming 1...
[PRODUCER] Consumer return: 200 OK
[PRODUCER] Producing 2...
[CONSUMER] Consuming 2...
[PRODUCER] Consumer return: 200 OK
[PRODUCER] Producing 3...
[CONSUMER] Consuming 3...
[PRODUCER] Consumer return: 200 OK
[PRODUCER] Producing 4...
[CONSUMER] Consuming 4...
[PRODUCER] Consumer return: 200 OK
[PRODUCER] Producing 5...
[CONSUMER] Consuming 5...
[PRODUCER] Consumer return: 200 OK
```

注意到`consumer`函数是一个`generator`，把一个`consumer`传入`produce`后：

1、首先调用`c.send(None)`启动生成器；
2、一旦产生了东西，通过`c.send(n)`切换到`consumer`执行
3、`consumer`通过`yield`拿到消息，处理，又通过`yield`把结果传回。
4、`produce`拿到`consumer`处理的结果，继续生产下一条消息
5、`produce`决定不生产了，通过`c.close()`关闭`consumer`整个过程结束。

整个流程无锁，由一个线程执行，`produce`和`consumer`协作完成任务，所以称为协程。

-----------------------

##asyncio

`asyncio`是python3.4版本引入的标准库，直接内置了对异步IO的支持。`asyncio`的编程模型就是一个消息循环。我们从`asyncio`模块中直接获取一个`Eventloop`的引用，然后把需要执行的协程扔到`Eventloop`中执行，就实现了异步IO。

用`asyncio`实现`Hello world`代码如下：
```python
>>> import asyncio
>>>
>>> @asyncio.coroutine
... def hello():
...     print("Hello world!!")
...     r=yield from asyncio.sleep(1)
...     print("Hello again!")
...
>>> loop = asyncio.get_event_loop()
>>> loop.run_until_complete(hello())
Hello world!!
Hello again!
>>> loop.close()
```

`@asyncio.coroutine`把一个generator标记为coroutine类型，然后，我们就把这个Coroutine扔到Eventloop中执行。

`hello()`首先打印出`Hello world!!`，然后，`yield from`语法可以让我们方便的调用另一个`generator`。由于`asyncio.sleep()`也是一个Coroutine，所以线程不会等待`asyncio.sleep()`，而是直接中断并执行下一个消息循环。当`asyncio.sleep()`返回时，线程就可以从`yield from`拿到返回值，然后接着执行下一行语句。

把`asyncio.sleep(1)`看做一个耗时1秒的IO操作，在此期间，主线程并未等待，而是去执行`Eventloop`中其他可以执行的coroutine了，因此可以实现并发执行。

用Task封装两个`coroutine`：
```python
import threading
import asyncio

@asyncio.coroutine
def hello():
	print('Hello world! (%s)' % threading.currentThread())
	yield from asyncio.sleep(1)
	print('Hello again! (%s)' % threading.currentThread())
		
loop = asyncio.get_event_loop()
tasks = [hello(), hello()]
loop.run_until_complete(asyncio.wait(tasks))
loop.close()
```
结果：
```
D:\笔记\Python\Notepad++>python 1.1.py
Hello world! (<_MainThread(MainThread, started 20036)>)
Hello world! (<_MainThread(MainThread, started 20036)>)
#（停顿1s）
Hello again! (<_MainThread(MainThread, started 20036)>)
Hello again! (<_MainThread(MainThread, started 20036)>)
```

由打印出来的当前线程可以看出，两个`continue`是由同一个线程并发执行的。如果把`asyncio.sleep()`换成真正的IO操作，则多个`coroutine`就可以由一个线程并发执行。我们用`asyncio`的异步网络连接来获取sina、sohu和163的网站首页：
```python
import asyncio

@asyncio.coroutine
def wget(host):
	print('wget %s...' % host)
	connect = asyncio.open_connection(host, 80)
	reader, writer = yield from connect
	header = 'GET / HTTP/1.0\r\nHost: %s\r\n\r\n' %host
	writer.write(header.encode('utf-8'))
	yield from writer.drain()
	while True:
		line = yield from reader.readline()
		if line == b'\r\n':
			break
		print('%s header > %s' %(host, line.decode('utf-8').rstrip()))
	writer.close()

loop = asyncio.get_event_loop()
tasks = [wget(host) for host in['www.sina.com.cn', 'www.sohu.com', 'www.163.com']]
loop.run_until_complete(asyncio.wait(tasks))
loop.close()
```

执行结果：
```python
D:\笔记\Python\Notepad++>python 1.2.py
wget www.sina.com.cn...
wget www.163.com...
wget www.sohu.com...
www.sina.com.cn header > HTTP/1.1 200 OK
www.sina.com.cn header > Server: nginx
www.sina.com.cn header > Date: Mon, 14 Nov 2016 12:17:29 GMT
www.sina.com.cn header > Content-Type: text/html
www.sina.com.cn header > Last-Modified: Mon, 14 Nov 2016 12:17:02 GMT
www.sina.com.cn header > Vary: Accept-Encoding
www.sina.com.cn header > Expires: Mon, 14 Nov 2016 12:18:29 GMT
www.sina.com.cn header > Cache-Control: max-age=60
www.sina.com.cn header > X-Powered-By: shci_v1.03
www.sina.com.cn header > Age: 50
www.sina.com.cn header > Content-Length: 595123
www.sina.com.cn header > X-Cache: HIT from ja108-181.sina.com.cn
www.sina.com.cn header > Connection: close
www.163.com header > HTTP/1.0 302 Moved Temporarily
www.163.com header > Server: Cdn Cache Server V2.0
www.163.com header > Date: Mon, 14 Nov 2016 12:21:18 GMT
www.163.com header > Content-Length: 0
www.163.com header > Location: http://www.163.com/special/0077jt/error_isp.html
www.163.com header > Connection: close
www.sohu.com header > HTTP/1.1 200 OK
www.sohu.com header > Content-Type: text/html
www.sohu.com header > Content-Length: 90665
www.sohu.com header > Connection: close
www.sohu.com header > Date: Mon, 14 Nov 2016 12:18:29 GMT
www.sohu.com header > Server: SWS
www.sohu.com header > Vary: Accept-Encoding
www.sohu.com header > Cache-Control: no-transform, max-age=120
www.sohu.com header > Expires: Mon, 14 Nov 2016 12:20:29 GMT
www.sohu.com header > Last-Modified: Mon, 14 Nov 2016 12:04:28 GMT
www.sohu.com header > Content-Encoding: gzip
www.sohu.com header > X-RS: 10511343.19686393.11189627
www.sohu.com header > FSS-Cache: HIT from 9580427.16723861.11355128
www.sohu.com header > FSS-Proxy: Powered by 3944245.5451583.5718860
```

三个连接是由一个线程通过Coroutine并发完成。

`asyncio`提供了完善的异步IO支持，异步操作需要在`coroutine`中通过`yield from`完成，多个`coroutine`可以封装一组Task然后并发执行。

----------------------

##async/await

用`asyncio`提供的`@asyncio.coroutine`可以把一个generator标记为coroutine类型，然后在coroutine内部用`yield from`调用另一个coroutine实现异步操作。

为了简化并更好地标示异步IO，从python3.5开始引入了新的语法`async`和`await`，可以让Coroutine的代码更加简洁易读。

`async`和`await`是针对coroutine的新语法，要使用新的语法，只需要做两步简单的替换：
>1、把`@asyncio.coroutine`替换为`async`
>2、把`yield from`替换为`await`

对比一下上一节的代码：
```
#old
@asyncio.coroutine
def hello():
	print('Hello world! (%s)' % threading.currentThread())
	yield from asyncio.sleep(1)
	print('Hello again! (%s)' % threading.currentThread())

#new
async def hello():
	print('Hello world! (%s)' % threading.currentThread())
	await asyncio.sleep(1)
	print('Hello again! (%s)' % threading.currentThread())
```

剩下代码保持不变。

----------------------------

##aiohttp

`asyncio`可以实现单线程并发IO操作。如果仅用在客户端，发挥的威力不大。如果把`asyncio`用在服务器端，例如Web服务器，由于HTTP连接就是IO操作，因此可以用单线程+`cortinue`实现多用户的高并发支持。

`asyncio`实现了TCP、UDP、SSL等协议，`aiohttp`则是基于`asyncio`实现了HTTP框架。

安装`aiohttp`然后编写一个HTTP服务器，分别处理以下URL：
>`/`-首页返回`b'<h1>Index</h1>'`
>`/hello/{name}`-根据URL参数返回文本`hello, %s!`

代码如下：
```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-


import asyncio

from aiohttp import web

async def index(request):
    await asyncio.sleep(0.5)
    return web.Response(body=b'<h1>Index</h1>')

async def hello(request):
    await asyncio.sleep(0.5)
    text = '<h1>hello, %s!</h1>' % request.match_info['name']
    return web.Response(body=text.encode('utf-8'))

async def init(loop):
    app = web.Application(loop=loop)
    app.router.add_route('GET', '/', index)
    app.router.add_route('GET', '/hello/{name}', hello)
    srv = await loop.create_server(app.make_handler(), '127.0.0.1', 8000)
    print('Server started at http://127.0.0.1:8000...')
    return srv

loop = asyncio.get_event_loop()
loop.run_until_complete(init(loop))
loop.run_forever()
```

注意`aiohttp`的初始化函数`init()`也是一个`coroutine`，`loop.creat_server()`则利用`asyncio`创建TCP服务。